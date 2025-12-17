# Chapter 30: 필터 관리 애플리케이션

## 난이도: ★★★★☆ (중급-고급)

## 학습 목표
- WPF 기반 관리 콘솔 UI 구현
- MVVM 패턴으로 드라이버 통신 통합
- 실시간 이벤트 모니터링 구현
- 정책 관리 인터페이스 구축

## 30.1 관리 콘솔 아키텍처

### 전체 시스템 구조

```
+------------------------------------------------------------------+
|                    DRM Filter 관리 콘솔                           |
+------------------------------------------------------------------+
|                                                                   |
|  +------------------------+  +--------------------------------+  |
|  |                        |  |                                |  |
|  |   실시간 모니터링       |  |        정책 관리               |  |
|  |   - 파일 이벤트 로그    |  |   - 보호 경로 목록             |  |
|  |   - 차단 알림           |  |   - 확장자 필터                |  |
|  |   - 통계 대시보드       |  |   - 프로세스 예외              |  |
|  |                        |  |                                |  |
|  +------------------------+  +--------------------------------+  |
|                                                                   |
|  +------------------------+  +--------------------------------+  |
|  |                        |  |                                |  |
|  |   드라이버 상태         |  |        감사 로그               |  |
|  |   - 연결 상태           |  |   - 이벤트 검색                |  |
|  |   - 성능 메트릭         |  |   - 보고서 생성                |  |
|  |   - 설정 관리           |  |   - 내보내기                   |  |
|  |                        |  |                                |  |
|  +------------------------+  +--------------------------------+  |
|                                                                   |
+------------------------------------------------------------------+
            |                                   |
            v                                   v
+------------------------+     +--------------------------------+
|   FilterCommunication  |     |      PolicyDatabase            |
|   (Chapter 29)         |     |      (SQLite/JSON)             |
+------------------------+     +--------------------------------+
            |
            v
+------------------------------------------------------------------+
|                    Minifilter Driver                              |
+------------------------------------------------------------------+
```

### 프로젝트 구조

```
DrmFilterManager/
├── App.xaml                    # 애플리케이션 진입점
├── App.xaml.cs
├── GlobalUsings.cs            # 전역 using
├── Models/                    # 데이터 모델
│   ├── FileEvent.cs
│   ├── BlockEvent.cs
│   ├── ProtectionPolicy.cs
│   └── DriverStatus.cs
├── ViewModels/                # MVVM ViewModels
│   ├── MainViewModel.cs
│   ├── MonitorViewModel.cs
│   ├── PolicyViewModel.cs
│   └── SettingsViewModel.cs
├── Views/                     # XAML Views
│   ├── MainWindow.xaml
│   ├── MonitorView.xaml
│   ├── PolicyView.xaml
│   └── SettingsView.xaml
├── Services/                  # 서비스 계층
│   ├── IDriverService.cs
│   ├── DriverService.cs
│   ├── IPolicyRepository.cs
│   └── PolicyRepository.cs
├── Converters/               # 값 변환기
│   └── StatusToColorConverter.cs
└── Themes/                   # 스타일/리소스
    └── Generic.xaml
```

## 30.2 프로젝트 설정

### 프로젝트 파일

```xml
<!-- DrmFilterManager.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <ApplicationManifest>app.manifest</ApplicationManifest>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.0" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
    <PackageReference Include="System.Data.SQLite.Core" Version="1.0.119" />
    <PackageReference Include="LiveChartsCore.SkiaSharpView.WPF" Version="2.0.0-rc3.3" />
  </ItemGroup>

</Project>
```

### GlobalUsings.cs

```csharp
// GlobalUsings.cs
// 전역 using 선언
// Global using declarations

global using System.Collections.ObjectModel;
global using System.ComponentModel;
global using System.Windows;
global using System.Windows.Controls;
global using System.Windows.Data;
global using System.Windows.Input;
global using System.Windows.Media;
global using System.Windows.Threading;

global using CommunityToolkit.Mvvm.ComponentModel;
global using CommunityToolkit.Mvvm.Input;
global using CommunityToolkit.Mvvm.Messaging;

global using Microsoft.Extensions.DependencyInjection;
```

### 애플리케이션 매니페스트 (관리자 권한)

```xml
<!-- app.manifest -->
<?xml version="1.0" encoding="utf-8"?>
<assembly manifestVersion="1.0" xmlns="urn:schemas-microsoft-com:asm.v1">
  <assemblyIdentity version="1.0.0.0" name="DrmFilterManager.app"/>
  <trustInfo xmlns="urn:schemas-microsoft-com:asm.v2">
    <security>
      <requestedPrivileges xmlns="urn:schemas-microsoft-com:asm.v3">
        <!-- 관리자 권한 요청 - 드라이버 통신에 필요 -->
        <!-- Request administrator privileges - required for driver communication -->
        <requestedExecutionLevel level="requireAdministrator" uiAccess="false" />
      </requestedPrivileges>
    </security>
  </trustInfo>
</assembly>
```

## 30.3 데이터 모델

### 이벤트 모델

```csharp
// Models/FileEvent.cs
// 파일 이벤트 모델
// File event model

namespace DrmFilterManager.Models;

/// <summary>
/// 파일 시스템 이벤트
/// File system event
/// </summary>
public sealed class FileEvent
{
    public DateTime Timestamp { get; init; }
    public string FilePath { get; init; } = string.Empty;
    public string FileName => Path.GetFileName(FilePath);
    public string Directory => Path.GetDirectoryName(FilePath) ?? string.Empty;
    public FileOperation Operation { get; init; }
    public uint ProcessId { get; init; }
    public string ProcessName { get; init; } = string.Empty;
    public EventResult Result { get; init; }
    public string? Details { get; init; }

    /// <summary>
    /// 표시용 문자열
    /// Display string
    /// </summary>
    public string DisplayText =>
        $"[{Timestamp:HH:mm:ss}] {Operation}: {FileName}";
}

/// <summary>
/// 파일 작업 유형
/// File operation type
/// </summary>
public enum FileOperation
{
    Create,
    Read,
    Write,
    Delete,
    Rename,
    SetInfo,
    Unknown
}

/// <summary>
/// 이벤트 결과
/// Event result
/// </summary>
public enum EventResult
{
    Allowed,
    Blocked,
    Monitored
}
```

```csharp
// Models/BlockEvent.cs
// 차단 이벤트 모델
// Block event model

namespace DrmFilterManager.Models;

/// <summary>
/// 차단 이벤트
/// Block event
/// </summary>
public sealed class BlockEvent
{
    public DateTime Timestamp { get; init; }
    public string FilePath { get; init; } = string.Empty;
    public string FileName => Path.GetFileName(FilePath);
    public FileOperation Operation { get; init; }
    public uint ProcessId { get; init; }
    public string ProcessName { get; init; } = string.Empty;
    public string BlockReason { get; init; } = string.Empty;
    public string PolicyName { get; init; } = string.Empty;

    /// <summary>
    /// 심각도 레벨
    /// Severity level
    /// </summary>
    public SeverityLevel Severity =>
        Operation switch
        {
            FileOperation.Delete => SeverityLevel.High,
            FileOperation.Write => SeverityLevel.Medium,
            FileOperation.Rename => SeverityLevel.Medium,
            _ => SeverityLevel.Low
        };
}

public enum SeverityLevel
{
    Low,
    Medium,
    High,
    Critical
}
```

```csharp
// Models/ProtectionPolicy.cs
// 보호 정책 모델
// Protection policy model

namespace DrmFilterManager.Models;

/// <summary>
/// 보호 정책
/// Protection policy
/// </summary>
public sealed partial class ProtectionPolicy : ObservableObject
{
    [ObservableProperty]
    private int _id;

    [ObservableProperty]
    private string _name = string.Empty;

    [ObservableProperty]
    private string _path = string.Empty;

    [ObservableProperty]
    private PolicyType _type;

    [ObservableProperty]
    private ProtectionFlags _flags;

    [ObservableProperty]
    private bool _isEnabled = true;

    [ObservableProperty]
    private DateTime _createdAt;

    [ObservableProperty]
    private DateTime? _lastModified;

    /// <summary>
    /// 요약 설명
    /// Summary description
    /// </summary>
    public string Summary
    {
        get
        {
            var actions = new List<string>();
            if (Flags.HasFlag(ProtectionFlags.Write)) actions.Add("쓰기");
            if (Flags.HasFlag(ProtectionFlags.Delete)) actions.Add("삭제");
            if (Flags.HasFlag(ProtectionFlags.Rename)) actions.Add("이름변경");
            if (Flags.HasFlag(ProtectionFlags.Move)) actions.Add("이동");

            return actions.Count > 0
                ? string.Join(", ", actions) + " 차단"
                : "없음";
        }
    }
}

public enum PolicyType
{
    Path,       // 경로 기반
    Extension,  // 확장자 기반
    Process,    // 프로세스 기반
    Content     // 콘텐츠 기반
}

[Flags]
public enum ProtectionFlags
{
    None = 0x0000,
    Read = 0x0001,
    Write = 0x0002,
    Execute = 0x0004,
    Rename = 0x0008,
    Delete = 0x0010,
    Move = 0x0020,
    All = Read | Write | Execute | Rename | Delete | Move
}
```

```csharp
// Models/DriverStatus.cs
// 드라이버 상태 모델
// Driver status model

namespace DrmFilterManager.Models;

/// <summary>
/// 드라이버 상태 정보
/// Driver status information
/// </summary>
public sealed class DriverStatus
{
    public bool IsConnected { get; init; }
    public string Version { get; init; } = "Unknown";
    public DateTime? LastHeartbeat { get; init; }
    public int ActivePolicies { get; init; }
    public DriverStatistics Statistics { get; init; } = new();
}

/// <summary>
/// 드라이버 통계
/// Driver statistics
/// </summary>
public sealed class DriverStatistics
{
    public long TotalEventsProcessed { get; init; }
    public long FilesScanned { get; init; }
    public long OperationsBlocked { get; init; }
    public long OperationsAllowed { get; init; }
    public double AverageLatencyMs { get; init; }

    /// <summary>
    /// 차단율 백분율
    /// Block rate percentage
    /// </summary>
    public double BlockRatePercent =>
        TotalEventsProcessed > 0
            ? (double)OperationsBlocked / TotalEventsProcessed * 100
            : 0;
}
```

## 30.4 서비스 계층

### 드라이버 서비스 인터페이스

```csharp
// Services/IDriverService.cs
// 드라이버 서비스 인터페이스
// Driver service interface

namespace DrmFilterManager.Services;

/// <summary>
/// 드라이버 통신 서비스 인터페이스
/// Driver communication service interface
/// </summary>
public interface IDriverService : IDisposable
{
    /// <summary>
    /// 연결 상태
    /// Connection status
    /// </summary>
    bool IsConnected { get; }

    /// <summary>
    /// 연결 상태 변경 이벤트
    /// Connection status changed event
    /// </summary>
    event EventHandler<bool>? ConnectionChanged;

    /// <summary>
    /// 파일 이벤트 수신
    /// File event received
    /// </summary>
    event EventHandler<FileEvent>? FileEventReceived;

    /// <summary>
    /// 차단 이벤트 수신
    /// Block event received
    /// </summary>
    event EventHandler<BlockEvent>? BlockEventReceived;

    /// <summary>
    /// 드라이버에 연결
    /// Connect to driver
    /// </summary>
    Task<bool> ConnectAsync();

    /// <summary>
    /// 연결 해제
    /// Disconnect
    /// </summary>
    Task DisconnectAsync();

    /// <summary>
    /// 드라이버 상태 조회
    /// Get driver status
    /// </summary>
    Task<DriverStatus> GetStatusAsync();

    /// <summary>
    /// 정책 업데이트 전송
    /// Send policy update
    /// </summary>
    Task SendPolicyUpdateAsync(ProtectionPolicy policy);

    /// <summary>
    /// 정책 삭제 전송
    /// Send policy removal
    /// </summary>
    Task SendPolicyRemovalAsync(string path);

    /// <summary>
    /// 설정 변경 전송
    /// Send configuration change
    /// </summary>
    Task SendConfigChangeAsync(bool enableMonitoring, bool enableBlocking);
}
```

### 드라이버 서비스 구현

```csharp
// Services/DriverService.cs
// 드라이버 서비스 구현
// Driver service implementation

using System.Diagnostics;
using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;

namespace DrmFilterManager.Services;

/// <summary>
/// 드라이버 통신 서비스 구현
/// Driver communication service implementation
/// </summary>
public sealed class DriverService : IDriverService
{
    private const string PortName = "\\DrmFilterPort";

    private SafeFileHandle? _portHandle;
    private CancellationTokenSource? _cts;
    private Task? _listenerTask;
    private bool _isConnected;

    public bool IsConnected => _isConnected;

    public event EventHandler<bool>? ConnectionChanged;
    public event EventHandler<FileEvent>? FileEventReceived;
    public event EventHandler<BlockEvent>? BlockEventReceived;

    // P/Invoke 선언
    // P/Invoke declarations

    [DllImport("fltlib.dll", CharSet = CharSet.Unicode)]
    private static extern uint FilterConnectCommunicationPort(
        string portName,
        uint options,
        IntPtr context,
        uint sizeOfContext,
        IntPtr securityAttributes,
        out SafeFileHandle port);

    [DllImport("fltlib.dll")]
    private static extern uint FilterSendMessage(
        SafeFileHandle port,
        IntPtr inBuffer,
        uint inBufferSize,
        IntPtr outBuffer,
        uint outBufferSize,
        out uint bytesReturned);

    [DllImport("fltlib.dll")]
    private static extern uint FilterGetMessage(
        SafeFileHandle port,
        IntPtr messageBuffer,
        uint messageBufferSize,
        IntPtr overlapped);

    public async Task<bool> ConnectAsync()
    {
        if (_isConnected)
        {
            return true;
        }

        return await Task.Run(() =>
        {
            try
            {
                uint result = FilterConnectCommunicationPort(
                    PortName,
                    0,
                    IntPtr.Zero,
                    0,
                    IntPtr.Zero,
                    out _portHandle);

                if (result != 0)
                {
                    Debug.WriteLine($"드라이버 연결 실패: 0x{result:X8}");
                    // Driver connection failed
                    return false;
                }

                _isConnected = true;
                _cts = new CancellationTokenSource();
                _listenerTask = Task.Run(MessageListenerLoop);

                ConnectionChanged?.Invoke(this, true);
                return true;
            }
            catch (Exception ex)
            {
                Debug.WriteLine($"연결 예외: {ex.Message}");
                // Connection exception
                return false;
            }
        });
    }

    public async Task DisconnectAsync()
    {
        if (!_isConnected)
        {
            return;
        }

        _cts?.Cancel();

        if (_listenerTask != null)
        {
            await _listenerTask.WaitAsync(TimeSpan.FromSeconds(5))
                .ConfigureAwait(false);
        }

        _portHandle?.Dispose();
        _portHandle = null;
        _isConnected = false;

        ConnectionChanged?.Invoke(this, false);
    }

    private async Task MessageListenerLoop()
    {
        const int bufferSize = 4096;
        IntPtr buffer = Marshal.AllocHGlobal(bufferSize);

        try
        {
            while (_cts is { IsCancellationRequested: false })
            {
                uint result = FilterGetMessage(
                    _portHandle!,
                    buffer,
                    (uint)bufferSize,
                    IntPtr.Zero);

                if (result != 0)
                {
                    if (result == 0x800704D3) // ERROR_OPERATION_ABORTED
                    {
                        break;
                    }

                    await Task.Delay(100);
                    continue;
                }

                ProcessMessage(buffer);
            }
        }
        catch (OperationCanceledException)
        {
            // 정상 종료
            // Normal shutdown
        }
        finally
        {
            Marshal.FreeHGlobal(buffer);
        }
    }

    private void ProcessMessage(IntPtr buffer)
    {
        // 헤더 건너뛰기 (FilterMessageHeader)
        // Skip header
        IntPtr messagePtr = buffer + 16;  // sizeof(FILTER_MESSAGE_HEADER)

        // 메시지 타입 읽기
        // Read message type
        int messageType = Marshal.ReadInt32(messagePtr);

        switch (messageType)
        {
            case 2:  // MSG_TYPE_FILE_EVENT
                ProcessFileEvent(messagePtr);
                break;

            case 5:  // MSG_TYPE_BLOCK_NOTIFICATION
                ProcessBlockEvent(messagePtr);
                break;
        }
    }

    private void ProcessFileEvent(IntPtr messagePtr)
    {
        // 메시지 파싱
        // Parse message
        uint processId = (uint)Marshal.ReadInt32(messagePtr + 4);
        long timestamp = Marshal.ReadInt64(messagePtr + 12);
        string filePath = Marshal.PtrToStringUni(messagePtr + 20, 260)?.TrimEnd('\0') ?? "";
        uint operation = (uint)Marshal.ReadInt32(messagePtr + 540);

        var fileEvent = new FileEvent
        {
            Timestamp = DateTime.FromFileTimeUtc(timestamp),
            FilePath = filePath,
            ProcessId = processId,
            ProcessName = GetProcessName(processId),
            Operation = (FileOperation)operation,
            Result = EventResult.Monitored
        };

        FileEventReceived?.Invoke(this, fileEvent);
    }

    private void ProcessBlockEvent(IntPtr messagePtr)
    {
        uint processId = (uint)Marshal.ReadInt32(messagePtr + 4);
        long timestamp = Marshal.ReadInt64(messagePtr + 12);
        string filePath = Marshal.PtrToStringUni(messagePtr + 20, 260)?.TrimEnd('\0') ?? "";
        uint operation = (uint)Marshal.ReadInt32(messagePtr + 540);

        var blockEvent = new BlockEvent
        {
            Timestamp = DateTime.FromFileTimeUtc(timestamp),
            FilePath = filePath,
            ProcessId = processId,
            ProcessName = GetProcessName(processId),
            Operation = (FileOperation)operation,
            BlockReason = "정책에 의해 차단됨"  // Blocked by policy
        };

        BlockEventReceived?.Invoke(this, blockEvent);
    }

    private static string GetProcessName(uint processId)
    {
        try
        {
            var process = Process.GetProcessById((int)processId);
            return process.ProcessName;
        }
        catch
        {
            return $"PID:{processId}";
        }
    }

    public async Task<DriverStatus> GetStatusAsync()
    {
        if (!_isConnected)
        {
            return new DriverStatus { IsConnected = false };
        }

        // 실제로는 드라이버에서 상태 조회
        // In reality, query status from driver
        await Task.Delay(10);

        return new DriverStatus
        {
            IsConnected = true,
            Version = "1.0.0",
            LastHeartbeat = DateTime.Now,
            ActivePolicies = 5,
            Statistics = new DriverStatistics
            {
                TotalEventsProcessed = 15000,
                FilesScanned = 12500,
                OperationsBlocked = 42,
                OperationsAllowed = 14958,
                AverageLatencyMs = 0.5
            }
        };
    }

    public async Task SendPolicyUpdateAsync(ProtectionPolicy policy)
    {
        if (!_isConnected || _portHandle == null)
        {
            throw new InvalidOperationException("드라이버에 연결되지 않음");
            // Not connected to driver
        }

        // 메시지 구성 및 전송
        // Build and send message
        await Task.Run(() =>
        {
            // 구현 - Chapter 29의 SendMessage 참조
            // Implementation - see SendMessage from Chapter 29
            Debug.WriteLine($"정책 업데이트: {policy.Path}");
            // Policy update
        });
    }

    public async Task SendPolicyRemovalAsync(string path)
    {
        if (!_isConnected)
        {
            return;
        }

        await Task.Run(() =>
        {
            Debug.WriteLine($"정책 삭제: {path}");
            // Policy removal
        });
    }

    public async Task SendConfigChangeAsync(bool enableMonitoring, bool enableBlocking)
    {
        if (!_isConnected)
        {
            return;
        }

        await Task.Run(() =>
        {
            Debug.WriteLine($"설정 변경: Monitor={enableMonitoring}, Block={enableBlocking}");
            // Config change
        });
    }

    public void Dispose()
    {
        DisconnectAsync().Wait(TimeSpan.FromSeconds(5));
        _cts?.Dispose();
    }
}
```

### 정책 저장소

```csharp
// Services/IPolicyRepository.cs
// 정책 저장소 인터페이스
// Policy repository interface

namespace DrmFilterManager.Services;

/// <summary>
/// 정책 저장소 인터페이스
/// Policy repository interface
/// </summary>
public interface IPolicyRepository
{
    Task<IReadOnlyList<ProtectionPolicy>> GetAllAsync();
    Task<ProtectionPolicy?> GetByIdAsync(int id);
    Task<int> AddAsync(ProtectionPolicy policy);
    Task UpdateAsync(ProtectionPolicy policy);
    Task DeleteAsync(int id);
    Task<IReadOnlyList<ProtectionPolicy>> GetEnabledAsync();
}
```

```csharp
// Services/PolicyRepository.cs
// SQLite 기반 정책 저장소
// SQLite-based policy repository

using System.Data.SQLite;

namespace DrmFilterManager.Services;

/// <summary>
/// SQLite 정책 저장소
/// SQLite policy repository
/// </summary>
public sealed class PolicyRepository : IPolicyRepository, IDisposable
{
    private readonly SQLiteConnection _connection;

    public PolicyRepository(string dbPath)
    {
        var connectionString = $"Data Source={dbPath};Version=3;";
        _connection = new SQLiteConnection(connectionString);
        _connection.Open();

        InitializeDatabase();
    }

    private void InitializeDatabase()
    {
        const string createTableSql = """
            CREATE TABLE IF NOT EXISTS Policies (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL,
                Path TEXT NOT NULL,
                Type INTEGER NOT NULL,
                Flags INTEGER NOT NULL,
                IsEnabled INTEGER NOT NULL DEFAULT 1,
                CreatedAt TEXT NOT NULL,
                LastModified TEXT
            );
            """;

        using var cmd = new SQLiteCommand(createTableSql, _connection);
        cmd.ExecuteNonQuery();
    }

    public async Task<IReadOnlyList<ProtectionPolicy>> GetAllAsync()
    {
        const string sql = "SELECT * FROM Policies ORDER BY CreatedAt DESC";

        var policies = new List<ProtectionPolicy>();

        using var cmd = new SQLiteCommand(sql, _connection);
        using var reader = await cmd.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            policies.Add(MapFromReader(reader));
        }

        return policies;
    }

    public async Task<ProtectionPolicy?> GetByIdAsync(int id)
    {
        const string sql = "SELECT * FROM Policies WHERE Id = @Id";

        using var cmd = new SQLiteCommand(sql, _connection);
        cmd.Parameters.AddWithValue("@Id", id);

        using var reader = await cmd.ExecuteReaderAsync();
        if (await reader.ReadAsync())
        {
            return MapFromReader(reader);
        }

        return null;
    }

    public async Task<int> AddAsync(ProtectionPolicy policy)
    {
        const string sql = """
            INSERT INTO Policies (Name, Path, Type, Flags, IsEnabled, CreatedAt)
            VALUES (@Name, @Path, @Type, @Flags, @IsEnabled, @CreatedAt);
            SELECT last_insert_rowid();
            """;

        using var cmd = new SQLiteCommand(sql, _connection);
        cmd.Parameters.AddWithValue("@Name", policy.Name);
        cmd.Parameters.AddWithValue("@Path", policy.Path);
        cmd.Parameters.AddWithValue("@Type", (int)policy.Type);
        cmd.Parameters.AddWithValue("@Flags", (int)policy.Flags);
        cmd.Parameters.AddWithValue("@IsEnabled", policy.IsEnabled ? 1 : 0);
        cmd.Parameters.AddWithValue("@CreatedAt", DateTime.Now.ToString("O"));

        var result = await cmd.ExecuteScalarAsync();
        return Convert.ToInt32(result);
    }

    public async Task UpdateAsync(ProtectionPolicy policy)
    {
        const string sql = """
            UPDATE Policies
            SET Name = @Name, Path = @Path, Type = @Type,
                Flags = @Flags, IsEnabled = @IsEnabled, LastModified = @LastModified
            WHERE Id = @Id
            """;

        using var cmd = new SQLiteCommand(sql, _connection);
        cmd.Parameters.AddWithValue("@Id", policy.Id);
        cmd.Parameters.AddWithValue("@Name", policy.Name);
        cmd.Parameters.AddWithValue("@Path", policy.Path);
        cmd.Parameters.AddWithValue("@Type", (int)policy.Type);
        cmd.Parameters.AddWithValue("@Flags", (int)policy.Flags);
        cmd.Parameters.AddWithValue("@IsEnabled", policy.IsEnabled ? 1 : 0);
        cmd.Parameters.AddWithValue("@LastModified", DateTime.Now.ToString("O"));

        await cmd.ExecuteNonQueryAsync();
    }

    public async Task DeleteAsync(int id)
    {
        const string sql = "DELETE FROM Policies WHERE Id = @Id";

        using var cmd = new SQLiteCommand(sql, _connection);
        cmd.Parameters.AddWithValue("@Id", id);

        await cmd.ExecuteNonQueryAsync();
    }

    public async Task<IReadOnlyList<ProtectionPolicy>> GetEnabledAsync()
    {
        const string sql = "SELECT * FROM Policies WHERE IsEnabled = 1";

        var policies = new List<ProtectionPolicy>();

        using var cmd = new SQLiteCommand(sql, _connection);
        using var reader = await cmd.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            policies.Add(MapFromReader(reader));
        }

        return policies;
    }

    private static ProtectionPolicy MapFromReader(System.Data.Common.DbDataReader reader)
    {
        return new ProtectionPolicy
        {
            Id = reader.GetInt32(0),
            Name = reader.GetString(1),
            Path = reader.GetString(2),
            Type = (PolicyType)reader.GetInt32(3),
            Flags = (ProtectionFlags)reader.GetInt32(4),
            IsEnabled = reader.GetInt32(5) == 1,
            CreatedAt = DateTime.Parse(reader.GetString(6)),
            LastModified = reader.IsDBNull(7) ? null : DateTime.Parse(reader.GetString(7))
        };
    }

    public void Dispose()
    {
        _connection.Close();
        _connection.Dispose();
    }
}
```

## 30.5 ViewModels

### 메인 ViewModel

```csharp
// ViewModels/MainViewModel.cs
// 메인 윈도우 ViewModel
// Main window ViewModel

namespace DrmFilterManager.ViewModels;

/// <summary>
/// 메인 ViewModel
/// Main ViewModel
/// </summary>
public sealed partial class MainViewModel : ObservableObject
{
    private readonly IDriverService _driverService;

    [ObservableProperty]
    private string _title = "DRM Filter Manager";

    [ObservableProperty]
    private bool _isConnected;

    [ObservableProperty]
    private string _connectionStatus = "연결 끊김";
    // Disconnected

    [ObservableProperty]
    private string _driverVersion = "-";

    [ObservableProperty]
    private object? _currentView;

    // 하위 ViewModels
    // Sub ViewModels
    public MonitorViewModel MonitorVM { get; }
    public PolicyViewModel PolicyVM { get; }
    public SettingsViewModel SettingsVM { get; }

    public MainViewModel(
        IDriverService driverService,
        MonitorViewModel monitorVM,
        PolicyViewModel policyVM,
        SettingsViewModel settingsVM)
    {
        _driverService = driverService;
        MonitorVM = monitorVM;
        PolicyVM = policyVM;
        SettingsVM = settingsVM;

        // 연결 상태 변경 이벤트 구독
        // Subscribe to connection status change
        _driverService.ConnectionChanged += OnConnectionChanged;

        // 기본 뷰 설정
        // Set default view
        CurrentView = MonitorVM;
    }

    private void OnConnectionChanged(object? sender, bool connected)
    {
        Application.Current.Dispatcher.Invoke(() =>
        {
            IsConnected = connected;
            ConnectionStatus = connected ? "연결됨" : "연결 끊김";
            // Connected / Disconnected
        });
    }

    [RelayCommand]
    private async Task ConnectAsync()
    {
        ConnectionStatus = "연결 중...";
        // Connecting...

        bool success = await _driverService.ConnectAsync();

        if (success)
        {
            var status = await _driverService.GetStatusAsync();
            DriverVersion = status.Version;
        }
        else
        {
            ConnectionStatus = "연결 실패";
            // Connection failed

            MessageBox.Show(
                "드라이버에 연결할 수 없습니다.\n드라이버가 로드되어 있는지 확인하세요.",
                // Cannot connect to driver. Check if driver is loaded.
                "연결 오류",
                // Connection Error
                MessageBoxButton.OK,
                MessageBoxImage.Warning);
        }
    }

    [RelayCommand]
    private async Task DisconnectAsync()
    {
        await _driverService.DisconnectAsync();
    }

    [RelayCommand]
    private void ShowMonitor() => CurrentView = MonitorVM;

    [RelayCommand]
    private void ShowPolicy() => CurrentView = PolicyVM;

    [RelayCommand]
    private void ShowSettings() => CurrentView = SettingsVM;
}
```

### 모니터링 ViewModel

```csharp
// ViewModels/MonitorViewModel.cs
// 실시간 모니터링 ViewModel
// Real-time monitoring ViewModel

namespace DrmFilterManager.ViewModels;

/// <summary>
/// 모니터링 ViewModel
/// Monitoring ViewModel
/// </summary>
public sealed partial class MonitorViewModel : ObservableObject
{
    private readonly IDriverService _driverService;
    private readonly int _maxEvents = 500;

    [ObservableProperty]
    private bool _isMonitoring;

    [ObservableProperty]
    private long _totalEvents;

    [ObservableProperty]
    private long _blockedCount;

    [ObservableProperty]
    private FileEvent? _selectedEvent;

    /// <summary>
    /// 파일 이벤트 목록
    /// File event list
    /// </summary>
    public ObservableCollection<FileEvent> Events { get; } = [];

    /// <summary>
    /// 차단 이벤트 목록
    /// Block event list
    /// </summary>
    public ObservableCollection<BlockEvent> BlockedEvents { get; } = [];

    /// <summary>
    /// 필터링된 이벤트 뷰
    /// Filtered event view
    /// </summary>
    public ICollectionView EventsView { get; }

    [ObservableProperty]
    private string _filterText = string.Empty;

    [ObservableProperty]
    private FileOperation? _filterOperation;

    public MonitorViewModel(IDriverService driverService)
    {
        _driverService = driverService;

        // 이벤트 구독
        // Subscribe to events
        _driverService.FileEventReceived += OnFileEventReceived;
        _driverService.BlockEventReceived += OnBlockEventReceived;

        // CollectionView 설정
        // Setup CollectionView
        EventsView = CollectionViewSource.GetDefaultView(Events);
        EventsView.Filter = FilterEvents;
    }

    private void OnFileEventReceived(object? sender, FileEvent e)
    {
        Application.Current.Dispatcher.Invoke(() =>
        {
            // 최대 개수 유지
            // Maintain max count
            while (Events.Count >= _maxEvents)
            {
                Events.RemoveAt(Events.Count - 1);
            }

            Events.Insert(0, e);
            TotalEvents++;
        });
    }

    private void OnBlockEventReceived(object? sender, BlockEvent e)
    {
        Application.Current.Dispatcher.Invoke(() =>
        {
            while (BlockedEvents.Count >= _maxEvents)
            {
                BlockedEvents.RemoveAt(BlockedEvents.Count - 1);
            }

            BlockedEvents.Insert(0, e);
            BlockedCount++;

            // 차단 알림 표시 (선택적)
            // Show block notification (optional)
            // ShowBlockNotification(e);
        });
    }

    private bool FilterEvents(object obj)
    {
        if (obj is not FileEvent evt)
        {
            return false;
        }

        // 텍스트 필터
        // Text filter
        if (!string.IsNullOrWhiteSpace(FilterText))
        {
            bool matchPath = evt.FilePath.Contains(FilterText, StringComparison.OrdinalIgnoreCase);
            bool matchProcess = evt.ProcessName.Contains(FilterText, StringComparison.OrdinalIgnoreCase);

            if (!matchPath && !matchProcess)
            {
                return false;
            }
        }

        // 작업 필터
        // Operation filter
        if (FilterOperation.HasValue && evt.Operation != FilterOperation.Value)
        {
            return false;
        }

        return true;
    }

    partial void OnFilterTextChanged(string value)
    {
        EventsView.Refresh();
    }

    partial void OnFilterOperationChanged(FileOperation? value)
    {
        EventsView.Refresh();
    }

    [RelayCommand]
    private void ClearEvents()
    {
        Events.Clear();
        BlockedEvents.Clear();
        TotalEvents = 0;
        BlockedCount = 0;
    }

    [RelayCommand]
    private void ToggleMonitoring()
    {
        IsMonitoring = !IsMonitoring;

        // 드라이버에 모니터링 상태 전송
        // Send monitoring state to driver
        _driverService.SendConfigChangeAsync(IsMonitoring, true);
    }

    [RelayCommand]
    private async Task ExportEventsAsync()
    {
        var dialog = new Microsoft.Win32.SaveFileDialog
        {
            Filter = "CSV 파일 (*.csv)|*.csv|모든 파일 (*.*)|*.*",
            // CSV file / All files
            DefaultExt = ".csv",
            FileName = $"events_{DateTime.Now:yyyyMMdd_HHmmss}.csv"
        };

        if (dialog.ShowDialog() == true)
        {
            await ExportToCsvAsync(dialog.FileName);
        }
    }

    private async Task ExportToCsvAsync(string filePath)
    {
        var lines = new List<string>
        {
            "Timestamp,FilePath,Operation,ProcessId,ProcessName,Result"
        };

        foreach (var evt in Events)
        {
            lines.Add($"\"{evt.Timestamp:O}\",\"{evt.FilePath}\",\"{evt.Operation}\",{evt.ProcessId},\"{evt.ProcessName}\",\"{evt.Result}\"");
        }

        await File.WriteAllLinesAsync(filePath, lines);

        MessageBox.Show(
            $"{Events.Count}개 이벤트를 내보냈습니다.",
            // Exported events
            "내보내기 완료",
            // Export Complete
            MessageBoxButton.OK,
            MessageBoxImage.Information);
    }
}
```

### 정책 ViewModel

```csharp
// ViewModels/PolicyViewModel.cs
// 정책 관리 ViewModel
// Policy management ViewModel

namespace DrmFilterManager.ViewModels;

/// <summary>
/// 정책 관리 ViewModel
/// Policy management ViewModel
/// </summary>
public sealed partial class PolicyViewModel : ObservableObject
{
    private readonly IDriverService _driverService;
    private readonly IPolicyRepository _policyRepository;

    [ObservableProperty]
    private ProtectionPolicy? _selectedPolicy;

    [ObservableProperty]
    private bool _isEditing;

    // 편집 중인 정책 속성
    // Editing policy properties
    [ObservableProperty]
    private string _editName = string.Empty;

    [ObservableProperty]
    private string _editPath = string.Empty;

    [ObservableProperty]
    private PolicyType _editType;

    [ObservableProperty]
    private bool _editProtectWrite;

    [ObservableProperty]
    private bool _editProtectDelete;

    [ObservableProperty]
    private bool _editProtectRename;

    [ObservableProperty]
    private bool _editProtectMove;

    /// <summary>
    /// 정책 목록
    /// Policy list
    /// </summary>
    public ObservableCollection<ProtectionPolicy> Policies { get; } = [];

    public PolicyViewModel(IDriverService driverService, IPolicyRepository policyRepository)
    {
        _driverService = driverService;
        _policyRepository = policyRepository;
    }

    [RelayCommand]
    private async Task LoadPoliciesAsync()
    {
        var policies = await _policyRepository.GetAllAsync();

        Policies.Clear();
        foreach (var policy in policies)
        {
            Policies.Add(policy);
        }
    }

    [RelayCommand]
    private void NewPolicy()
    {
        SelectedPolicy = null;
        IsEditing = true;

        // 기본값 설정
        // Set defaults
        EditName = "새 정책";
        // New Policy
        EditPath = @"C:\Protected\*";
        EditType = PolicyType.Path;
        EditProtectWrite = true;
        EditProtectDelete = true;
        EditProtectRename = true;
        EditProtectMove = false;
    }

    [RelayCommand]
    private void EditPolicy()
    {
        if (SelectedPolicy == null)
        {
            return;
        }

        IsEditing = true;

        // 편집 필드에 로드
        // Load into edit fields
        EditName = SelectedPolicy.Name;
        EditPath = SelectedPolicy.Path;
        EditType = SelectedPolicy.Type;
        EditProtectWrite = SelectedPolicy.Flags.HasFlag(ProtectionFlags.Write);
        EditProtectDelete = SelectedPolicy.Flags.HasFlag(ProtectionFlags.Delete);
        EditProtectRename = SelectedPolicy.Flags.HasFlag(ProtectionFlags.Rename);
        EditProtectMove = SelectedPolicy.Flags.HasFlag(ProtectionFlags.Move);
    }

    [RelayCommand]
    private async Task SavePolicyAsync()
    {
        // 유효성 검사
        // Validation
        if (string.IsNullOrWhiteSpace(EditName))
        {
            MessageBox.Show("정책 이름을 입력하세요.", "입력 오류");
            // Enter policy name / Input Error
            return;
        }

        if (string.IsNullOrWhiteSpace(EditPath))
        {
            MessageBox.Show("보호 경로를 입력하세요.", "입력 오류");
            // Enter protection path / Input Error
            return;
        }

        // 플래그 구성
        // Build flags
        var flags = ProtectionFlags.None;
        if (EditProtectWrite) flags |= ProtectionFlags.Write;
        if (EditProtectDelete) flags |= ProtectionFlags.Delete;
        if (EditProtectRename) flags |= ProtectionFlags.Rename;
        if (EditProtectMove) flags |= ProtectionFlags.Move;

        if (SelectedPolicy == null)
        {
            // 새 정책 추가
            // Add new policy
            var newPolicy = new ProtectionPolicy
            {
                Name = EditName,
                Path = EditPath,
                Type = EditType,
                Flags = flags,
                IsEnabled = true,
                CreatedAt = DateTime.Now
            };

            int id = await _policyRepository.AddAsync(newPolicy);
            newPolicy.Id = id;

            Policies.Add(newPolicy);

            // 드라이버에 전송
            // Send to driver
            await _driverService.SendPolicyUpdateAsync(newPolicy);
        }
        else
        {
            // 기존 정책 수정
            // Update existing policy
            SelectedPolicy.Name = EditName;
            SelectedPolicy.Path = EditPath;
            SelectedPolicy.Type = EditType;
            SelectedPolicy.Flags = flags;
            SelectedPolicy.LastModified = DateTime.Now;

            await _policyRepository.UpdateAsync(SelectedPolicy);

            // 드라이버에 전송
            // Send to driver
            await _driverService.SendPolicyUpdateAsync(SelectedPolicy);
        }

        IsEditing = false;
    }

    [RelayCommand]
    private void CancelEdit()
    {
        IsEditing = false;
    }

    [RelayCommand]
    private async Task DeletePolicyAsync()
    {
        if (SelectedPolicy == null)
        {
            return;
        }

        var result = MessageBox.Show(
            $"'{SelectedPolicy.Name}' 정책을 삭제하시겠습니까?",
            // Delete policy?
            "정책 삭제",
            // Delete Policy
            MessageBoxButton.YesNo,
            MessageBoxImage.Question);

        if (result == MessageBoxResult.Yes)
        {
            await _policyRepository.DeleteAsync(SelectedPolicy.Id);

            // 드라이버에서 제거
            // Remove from driver
            await _driverService.SendPolicyRemovalAsync(SelectedPolicy.Path);

            Policies.Remove(SelectedPolicy);
            SelectedPolicy = null;
        }
    }

    [RelayCommand]
    private async Task TogglePolicyAsync()
    {
        if (SelectedPolicy == null)
        {
            return;
        }

        SelectedPolicy.IsEnabled = !SelectedPolicy.IsEnabled;
        await _policyRepository.UpdateAsync(SelectedPolicy);

        if (SelectedPolicy.IsEnabled)
        {
            await _driverService.SendPolicyUpdateAsync(SelectedPolicy);
        }
        else
        {
            await _driverService.SendPolicyRemovalAsync(SelectedPolicy.Path);
        }
    }

    [RelayCommand]
    private void BrowsePath()
    {
        var dialog = new System.Windows.Forms.FolderBrowserDialog
        {
            Description = "보호할 폴더를 선택하세요"
            // Select folder to protect
        };

        if (dialog.ShowDialog() == System.Windows.Forms.DialogResult.OK)
        {
            EditPath = dialog.SelectedPath + @"\*";
        }
    }
}
```

## 30.6 XAML Views

### 메인 윈도우

```xml
<!-- Views/MainWindow.xaml -->
<Window x:Class="DrmFilterManager.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:vm="clr-namespace:DrmFilterManager.ViewModels"
        xmlns:views="clr-namespace:DrmFilterManager.Views"
        mc:Ignorable="d"
        Title="{Binding Title}"
        Height="700" Width="1000"
        WindowStartupLocation="CenterScreen"
        d:DataContext="{d:DesignInstance vm:MainViewModel}">

    <Window.Resources>
        <!-- 연결 상태 색상 변환기 -->
        <!-- Connection status color converter -->
        <SolidColorBrush x:Key="ConnectedBrush" Color="#4CAF50"/>
        <SolidColorBrush x:Key="DisconnectedBrush" Color="#F44336"/>
    </Window.Resources>

    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- 상단 메뉴/툴바 -->
        <!-- Top menu/toolbar -->
        <Border Grid.Row="0" Background="#1976D2" Padding="10">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>

                <!-- 제목 -->
                <!-- Title -->
                <TextBlock Grid.Column="0"
                           Text="DRM Filter Manager"
                           FontSize="20"
                           FontWeight="Bold"
                           Foreground="White"
                           VerticalAlignment="Center"/>

                <!-- 네비게이션 버튼 -->
                <!-- Navigation buttons -->
                <StackPanel Grid.Column="1"
                            Orientation="Horizontal"
                            HorizontalAlignment="Center">
                    <Button Content="모니터링"
                            Command="{Binding ShowMonitorCommand}"
                            Style="{StaticResource NavButtonStyle}"
                            Margin="5,0"/>
                    <Button Content="정책 관리"
                            Command="{Binding ShowPolicyCommand}"
                            Style="{StaticResource NavButtonStyle}"
                            Margin="5,0"/>
                    <Button Content="설정"
                            Command="{Binding ShowSettingsCommand}"
                            Style="{StaticResource NavButtonStyle}"
                            Margin="5,0"/>
                </StackPanel>

                <!-- 연결 상태 -->
                <!-- Connection status -->
                <StackPanel Grid.Column="2"
                            Orientation="Horizontal"
                            VerticalAlignment="Center">
                    <Ellipse Width="12" Height="12"
                             Margin="0,0,8,0">
                        <Ellipse.Style>
                            <Style TargetType="Ellipse">
                                <Setter Property="Fill" Value="{StaticResource DisconnectedBrush}"/>
                                <Style.Triggers>
                                    <DataTrigger Binding="{Binding IsConnected}" Value="True">
                                        <Setter Property="Fill" Value="{StaticResource ConnectedBrush}"/>
                                    </DataTrigger>
                                </Style.Triggers>
                            </Style>
                        </Ellipse.Style>
                    </Ellipse>
                    <TextBlock Text="{Binding ConnectionStatus}"
                               Foreground="White"
                               VerticalAlignment="Center"
                               Margin="0,0,15,0"/>
                    <Button Content="연결"
                            Command="{Binding ConnectCommand}"
                            Visibility="{Binding IsConnected, Converter={StaticResource BoolToVisibilityInverseConverter}}"
                            Style="{StaticResource ConnectButtonStyle}"/>
                    <Button Content="연결 해제"
                            Command="{Binding DisconnectCommand}"
                            Visibility="{Binding IsConnected, Converter={StaticResource BoolToVisibilityConverter}}"
                            Style="{StaticResource DisconnectButtonStyle}"/>
                </StackPanel>
            </Grid>
        </Border>

        <!-- 메인 콘텐츠 영역 -->
        <!-- Main content area -->
        <ContentControl Grid.Row="1" Content="{Binding CurrentView}">
            <ContentControl.Resources>
                <DataTemplate DataType="{x:Type vm:MonitorViewModel}">
                    <views:MonitorView/>
                </DataTemplate>
                <DataTemplate DataType="{x:Type vm:PolicyViewModel}">
                    <views:PolicyView/>
                </DataTemplate>
                <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
                    <views:SettingsView/>
                </DataTemplate>
            </ContentControl.Resources>
        </ContentControl>

        <!-- 상태 바 -->
        <!-- Status bar -->
        <Border Grid.Row="2" Background="#E0E0E0" Padding="10,5">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>

                <TextBlock Grid.Column="0"
                           Text="{Binding DriverVersion, StringFormat='드라이버 버전: {0}'}"/>
                <TextBlock Grid.Column="1"
                           Text="{Binding MonitorVM.TotalEvents, StringFormat='이벤트: {0:N0}'}"
                           Margin="20,0"/>
                <TextBlock Grid.Column="2"
                           Text="{Binding MonitorVM.BlockedCount, StringFormat='차단: {0:N0}'}"
                           Foreground="#F44336"/>
            </Grid>
        </Border>
    </Grid>
</Window>
```

### 모니터링 뷰

```xml
<!-- Views/MonitorView.xaml -->
<UserControl x:Class="DrmFilterManager.Views.MonitorView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:vm="clr-namespace:DrmFilterManager.ViewModels"
             mc:Ignorable="d"
             d:DataContext="{d:DesignInstance vm:MonitorViewModel}">

    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- 필터 영역 -->
        <!-- Filter area -->
        <Border Grid.Row="0"
                Background="#F5F5F5"
                CornerRadius="5"
                Padding="10"
                Margin="0,0,0,10">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="200"/>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="120"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>

                <TextBlock Grid.Column="0"
                           Text="검색:"
                           VerticalAlignment="Center"
                           Margin="0,0,10,0"/>
                <TextBox Grid.Column="1"
                         Text="{Binding FilterText, UpdateSourceTrigger=PropertyChanged}"
                         VerticalContentAlignment="Center"/>

                <TextBlock Grid.Column="2"
                           Text="작업:"
                           VerticalAlignment="Center"
                           Margin="20,0,10,0"/>
                <ComboBox Grid.Column="3"
                          SelectedValue="{Binding FilterOperation}"
                          SelectedValuePath="Tag">
                    <ComboBoxItem Content="전체" Tag="{x:Null}"/>
                    <ComboBoxItem Content="생성" Tag="{x:Static vm:FileOperation.Create}"/>
                    <ComboBoxItem Content="읽기" Tag="{x:Static vm:FileOperation.Read}"/>
                    <ComboBoxItem Content="쓰기" Tag="{x:Static vm:FileOperation.Write}"/>
                    <ComboBoxItem Content="삭제" Tag="{x:Static vm:FileOperation.Delete}"/>
                    <ComboBoxItem Content="이름변경" Tag="{x:Static vm:FileOperation.Rename}"/>
                </ComboBox>

                <StackPanel Grid.Column="5"
                            Orientation="Horizontal">
                    <ToggleButton Content="{Binding IsMonitoring, Converter={StaticResource MonitoringTextConverter}}"
                                  IsChecked="{Binding IsMonitoring}"
                                  Command="{Binding ToggleMonitoringCommand}"
                                  Padding="15,5"
                                  Margin="0,0,10,0"/>
                    <Button Content="지우기"
                            Command="{Binding ClearEventsCommand}"
                            Padding="15,5"
                            Margin="0,0,10,0"/>
                    <Button Content="내보내기"
                            Command="{Binding ExportEventsCommand}"
                            Padding="15,5"/>
                </StackPanel>
            </Grid>
        </Border>

        <!-- 이벤트 탭 -->
        <!-- Event tabs -->
        <TabControl Grid.Row="1">
            <!-- 모든 이벤트 탭 -->
            <!-- All Events tab -->
            <TabItem Header="모든 이벤트">
                <DataGrid ItemsSource="{Binding EventsView}"
                          SelectedItem="{Binding SelectedEvent}"
                          AutoGenerateColumns="False"
                          IsReadOnly="True"
                          CanUserAddRows="False"
                          GridLinesVisibility="Horizontal"
                          HeadersVisibility="Column"
                          AlternatingRowBackground="#FAFAFA">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="시간"
                                            Binding="{Binding Timestamp, StringFormat='{}{0:HH:mm:ss.fff}'}"
                                            Width="100"/>
                        <DataGridTextColumn Header="작업"
                                            Binding="{Binding Operation}"
                                            Width="80"/>
                        <DataGridTextColumn Header="파일"
                                            Binding="{Binding FileName}"
                                            Width="200"/>
                        <DataGridTextColumn Header="경로"
                                            Binding="{Binding Directory}"
                                            Width="*"/>
                        <DataGridTextColumn Header="프로세스"
                                            Binding="{Binding ProcessName}"
                                            Width="120"/>
                        <DataGridTextColumn Header="PID"
                                            Binding="{Binding ProcessId}"
                                            Width="60"/>
                        <DataGridTextColumn Header="결과"
                                            Binding="{Binding Result}"
                                            Width="80"/>
                    </DataGrid.Columns>
                </DataGrid>
            </TabItem>

            <!-- 차단 이벤트 탭 -->
            <!-- Blocked Events tab -->
            <TabItem Header="차단된 작업">
                <DataGrid ItemsSource="{Binding BlockedEvents}"
                          AutoGenerateColumns="False"
                          IsReadOnly="True"
                          CanUserAddRows="False"
                          GridLinesVisibility="Horizontal"
                          AlternatingRowBackground="#FFEBEE">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="시간"
                                            Binding="{Binding Timestamp, StringFormat='{}{0:HH:mm:ss}'}"
                                            Width="80"/>
                        <DataGridTextColumn Header="작업"
                                            Binding="{Binding Operation}"
                                            Width="80"/>
                        <DataGridTextColumn Header="파일"
                                            Binding="{Binding FilePath}"
                                            Width="*"/>
                        <DataGridTextColumn Header="프로세스"
                                            Binding="{Binding ProcessName}"
                                            Width="120"/>
                        <DataGridTextColumn Header="차단 이유"
                                            Binding="{Binding BlockReason}"
                                            Width="200"/>
                    </DataGrid.Columns>
                    <DataGrid.RowStyle>
                        <Style TargetType="DataGridRow">
                            <Style.Triggers>
                                <DataTrigger Binding="{Binding Severity}" Value="High">
                                    <Setter Property="Background" Value="#FFCDD2"/>
                                </DataTrigger>
                            </Style.Triggers>
                        </Style>
                    </DataGrid.RowStyle>
                </DataGrid>
            </TabItem>
        </TabControl>

        <!-- 선택된 이벤트 상세 정보 -->
        <!-- Selected event details -->
        <Border Grid.Row="2"
                Background="#F5F5F5"
                CornerRadius="5"
                Padding="10"
                Margin="0,10,0,0"
                Visibility="{Binding SelectedEvent, Converter={StaticResource NullToVisibilityConverter}}">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                </Grid.ColumnDefinitions>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>

                <TextBlock Grid.Row="0" Grid.Column="0"
                           Text="전체 경로:"
                           FontWeight="Bold"
                           Margin="0,0,10,5"/>
                <TextBlock Grid.Row="0" Grid.Column="1"
                           Text="{Binding SelectedEvent.FilePath}"
                           TextWrapping="Wrap"/>

                <TextBlock Grid.Row="1" Grid.Column="0"
                           Text="상세:"
                           FontWeight="Bold"
                           Margin="0,0,10,0"/>
                <TextBlock Grid.Row="1" Grid.Column="1"
                           Text="{Binding SelectedEvent.Details}"
                           TextWrapping="Wrap"/>
            </Grid>
        </Border>
    </Grid>
</UserControl>
```

### 정책 관리 뷰

```xml
<!-- Views/PolicyView.xaml -->
<UserControl x:Class="DrmFilterManager.Views.PolicyView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:vm="clr-namespace:DrmFilterManager.ViewModels"
             mc:Ignorable="d"
             Loaded="UserControl_Loaded"
             d:DataContext="{d:DesignInstance vm:PolicyViewModel}">

    <Grid Margin="10">
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="*"/>
            <ColumnDefinition Width="350"/>
        </Grid.ColumnDefinitions>

        <!-- 정책 목록 -->
        <!-- Policy list -->
        <Border Grid.Column="0"
                BorderBrush="#E0E0E0"
                BorderThickness="1"
                CornerRadius="5"
                Margin="0,0,10,0">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>

                <!-- 목록 헤더 -->
                <!-- List header -->
                <Border Grid.Row="0"
                        Background="#F5F5F5"
                        Padding="10">
                    <Grid>
                        <TextBlock Text="보호 정책 목록"
                                   FontWeight="Bold"
                                   FontSize="14"/>
                        <StackPanel Orientation="Horizontal"
                                    HorizontalAlignment="Right">
                            <Button Content="새 정책"
                                    Command="{Binding NewPolicyCommand}"
                                    Padding="10,5"
                                    Margin="0,0,5,0"/>
                            <Button Content="삭제"
                                    Command="{Binding DeletePolicyCommand}"
                                    Padding="10,5"
                                    IsEnabled="{Binding SelectedPolicy, Converter={StaticResource NullToBoolConverter}}"/>
                        </StackPanel>
                    </Grid>
                </Border>

                <!-- 정책 리스트 -->
                <!-- Policy ListView -->
                <ListView Grid.Row="1"
                          ItemsSource="{Binding Policies}"
                          SelectedItem="{Binding SelectedPolicy}"
                          BorderThickness="0">
                    <ListView.ItemTemplate>
                        <DataTemplate>
                            <Border Padding="10"
                                    BorderBrush="#E0E0E0"
                                    BorderThickness="0,0,0,1">
                                <Grid>
                                    <Grid.ColumnDefinitions>
                                        <ColumnDefinition Width="*"/>
                                        <ColumnDefinition Width="Auto"/>
                                    </Grid.ColumnDefinitions>
                                    <Grid.RowDefinitions>
                                        <RowDefinition Height="Auto"/>
                                        <RowDefinition Height="Auto"/>
                                        <RowDefinition Height="Auto"/>
                                    </Grid.RowDefinitions>

                                    <TextBlock Grid.Row="0" Grid.Column="0"
                                               Text="{Binding Name}"
                                               FontWeight="SemiBold"
                                               FontSize="14"/>

                                    <ToggleButton Grid.Row="0" Grid.Column="1"
                                                  IsChecked="{Binding IsEnabled}"
                                                  Content="{Binding IsEnabled, Converter={StaticResource EnabledTextConverter}}"
                                                  Padding="10,3"/>

                                    <TextBlock Grid.Row="1" Grid.ColumnSpan="2"
                                               Text="{Binding Path}"
                                               Foreground="Gray"
                                               TextTrimming="CharacterEllipsis"
                                               Margin="0,3,0,0"/>

                                    <TextBlock Grid.Row="2" Grid.ColumnSpan="2"
                                               Text="{Binding Summary}"
                                               Foreground="#1976D2"
                                               FontSize="11"
                                               Margin="0,3,0,0"/>
                                </Grid>
                            </Border>
                        </DataTemplate>
                    </ListView.ItemTemplate>
                </ListView>
            </Grid>
        </Border>

        <!-- 정책 편집 패널 -->
        <!-- Policy edit panel -->
        <Border Grid.Column="1"
                BorderBrush="#E0E0E0"
                BorderThickness="1"
                CornerRadius="5"
                Visibility="{Binding IsEditing, Converter={StaticResource BoolToVisibilityConverter}}">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>

                <!-- 편집 헤더 -->
                <!-- Edit header -->
                <Border Grid.Row="0"
                        Background="#1976D2"
                        Padding="10">
                    <TextBlock Text="정책 편집"
                               Foreground="White"
                               FontWeight="Bold"/>
                </Border>

                <!-- 편집 폼 -->
                <!-- Edit form -->
                <ScrollViewer Grid.Row="1"
                              VerticalScrollBarVisibility="Auto"
                              Padding="15">
                    <StackPanel>
                        <!-- 이름 -->
                        <!-- Name -->
                        <TextBlock Text="정책 이름"
                                   FontWeight="SemiBold"
                                   Margin="0,0,0,5"/>
                        <TextBox Text="{Binding EditName, UpdateSourceTrigger=PropertyChanged}"
                                 Padding="8"
                                 Margin="0,0,0,15"/>

                        <!-- 유형 -->
                        <!-- Type -->
                        <TextBlock Text="정책 유형"
                                   FontWeight="SemiBold"
                                   Margin="0,0,0,5"/>
                        <ComboBox SelectedValue="{Binding EditType}"
                                  SelectedValuePath="Tag"
                                  Padding="8"
                                  Margin="0,0,0,15">
                            <ComboBoxItem Content="경로 기반" Tag="{x:Static vm:PolicyType.Path}"/>
                            <ComboBoxItem Content="확장자 기반" Tag="{x:Static vm:PolicyType.Extension}"/>
                            <ComboBoxItem Content="프로세스 기반" Tag="{x:Static vm:PolicyType.Process}"/>
                        </ComboBox>

                        <!-- 경로 -->
                        <!-- Path -->
                        <TextBlock Text="보호 경로"
                                   FontWeight="SemiBold"
                                   Margin="0,0,0,5"/>
                        <Grid Margin="0,0,0,15">
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="Auto"/>
                            </Grid.ColumnDefinitions>
                            <TextBox Grid.Column="0"
                                     Text="{Binding EditPath, UpdateSourceTrigger=PropertyChanged}"
                                     Padding="8"/>
                            <Button Grid.Column="1"
                                    Content="..."
                                    Command="{Binding BrowsePathCommand}"
                                    Padding="10,0"
                                    Margin="5,0,0,0"/>
                        </Grid>

                        <!-- 보호 옵션 -->
                        <!-- Protection options -->
                        <TextBlock Text="보호 옵션"
                                   FontWeight="SemiBold"
                                   Margin="0,0,0,10"/>
                        <Border Background="#F5F5F5"
                                CornerRadius="5"
                                Padding="10">
                            <StackPanel>
                                <CheckBox Content="쓰기 차단"
                                          IsChecked="{Binding EditProtectWrite}"
                                          Margin="0,0,0,8"/>
                                <CheckBox Content="삭제 차단"
                                          IsChecked="{Binding EditProtectDelete}"
                                          Margin="0,0,0,8"/>
                                <CheckBox Content="이름 변경 차단"
                                          IsChecked="{Binding EditProtectRename}"
                                          Margin="0,0,0,8"/>
                                <CheckBox Content="이동 차단"
                                          IsChecked="{Binding EditProtectMove}"/>
                            </StackPanel>
                        </Border>
                    </StackPanel>
                </ScrollViewer>

                <!-- 버튼 영역 -->
                <!-- Button area -->
                <Border Grid.Row="2"
                        Background="#F5F5F5"
                        Padding="10">
                    <StackPanel Orientation="Horizontal"
                                HorizontalAlignment="Right">
                        <Button Content="취소"
                                Command="{Binding CancelEditCommand}"
                                Padding="20,8"
                                Margin="0,0,10,0"/>
                        <Button Content="저장"
                                Command="{Binding SavePolicyCommand}"
                                Background="#1976D2"
                                Foreground="White"
                                Padding="20,8"/>
                    </StackPanel>
                </Border>
            </Grid>
        </Border>

        <!-- 정책 선택 안내 (편집 모드 아닐 때) -->
        <!-- Policy selection guide (when not editing) -->
        <Border Grid.Column="1"
                BorderBrush="#E0E0E0"
                BorderThickness="1"
                CornerRadius="5"
                Visibility="{Binding IsEditing, Converter={StaticResource BoolToVisibilityInverseConverter}}">
            <StackPanel VerticalAlignment="Center"
                        HorizontalAlignment="Center">
                <TextBlock Text="정책을 선택하거나"
                           TextAlignment="Center"
                           Foreground="Gray"/>
                <TextBlock Text="새 정책을 추가하세요"
                           TextAlignment="Center"
                           Foreground="Gray"/>
                <Button Content="새 정책 추가"
                        Command="{Binding NewPolicyCommand}"
                        Margin="0,20,0,0"
                        Padding="20,10"/>
            </StackPanel>
        </Border>
    </Grid>
</UserControl>
```

## 30.7 애플리케이션 시작 설정

### App.xaml.cs

```csharp
// App.xaml.cs
// 애플리케이션 진입점 및 DI 설정
// Application entry point and DI configuration

using DrmFilterManager.Services;
using DrmFilterManager.ViewModels;
using DrmFilterManager.Views;

namespace DrmFilterManager;

public partial class App : Application
{
    private ServiceProvider? _serviceProvider;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // DI 컨테이너 구성
        // Configure DI container
        var services = new ServiceCollection();
        ConfigureServices(services);
        _serviceProvider = services.BuildServiceProvider();

        // 메인 윈도우 생성 및 표시
        // Create and show main window
        var mainWindow = _serviceProvider.GetRequiredService<MainWindow>();
        mainWindow.Show();
    }

    private void ConfigureServices(IServiceCollection services)
    {
        // 서비스 등록
        // Register services
        services.AddSingleton<IDriverService, DriverService>();
        services.AddSingleton<IPolicyRepository>(sp =>
        {
            var dbPath = Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                "DrmFilter",
                "policies.db");

            Directory.CreateDirectory(Path.GetDirectoryName(dbPath)!);
            return new PolicyRepository(dbPath);
        });

        // ViewModel 등록
        // Register ViewModels
        services.AddSingleton<MainViewModel>();
        services.AddSingleton<MonitorViewModel>();
        services.AddSingleton<PolicyViewModel>();
        services.AddSingleton<SettingsViewModel>();

        // View 등록
        // Register Views
        services.AddSingleton<MainWindow>();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        // 리소스 정리
        // Cleanup resources
        if (_serviceProvider is IDisposable disposable)
        {
            disposable.Dispose();
        }

        base.OnExit(e);
    }
}
```

## 30.8 정리

이 챕터에서 학습한 내용:

1. **WPF 관리 콘솔 아키텍처**
   - MVVM 패턴 적용
   - 서비스 계층 분리
   - 의존성 주입 구성

2. **실시간 모니터링**
   - 이벤트 스트림 표시
   - 필터링 및 검색
   - 내보내기 기능

3. **정책 관리 UI**
   - CRUD 인터페이스
   - SQLite 데이터 저장
   - 드라이버 동기화

4. **드라이버 통신 통합**
   - FilterCommunication 래퍼
   - 이벤트 기반 업데이트
   - 상태 모니터링

5. **사용자 경험**
   - 연결 상태 표시
   - 알림 시스템
   - 직관적 네비게이션

Part 9가 완료되었습니다. 다음 Part 10에서는 **테스트와 최적화**를 다룹니다.

---

**다음 챕터 예고**: Chapter 31에서는 Minifilter 드라이버의 단위 테스트, 통합 테스트, 스트레스 테스트 방법론을 학습합니다.
