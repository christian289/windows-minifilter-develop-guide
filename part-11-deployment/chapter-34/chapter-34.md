# Chapter 34: 업데이트와 유지보수

## 난이도: ★★★★☆ (중급-고급)

## 학습 목표
- 드라이버 버전 관리 전략 수립
- 안전한 업데이트 메커니즘 구현
- 롤백 및 복구 시스템 설계
- 원격 배포 및 관리 방법 이해

## 34.1 버전 관리 전략

### 시맨틱 버전 관리

```
+------------------------------------------------------------------+
|                    드라이버 버전 체계                              |
+------------------------------------------------------------------+
|                                                                   |
|   Major.Minor.Patch.Build                                         |
|   예: 1.2.3.456                                                   |
|                                                                   |
|   +----------+--------------------------------------------------+  |
|   | Major    | 호환성 없는 API 변경                              |  |
|   |          | - 통신 프로토콜 변경                              |  |
|   |          | - 컨텍스트 구조체 변경                            |  |
|   +----------+--------------------------------------------------+  |
|   | Minor    | 하위 호환 기능 추가                               |  |
|   |          | - 새로운 IOCTL 추가                               |  |
|   |          | - 새로운 정책 유형 지원                           |  |
|   +----------+--------------------------------------------------+  |
|   | Patch    | 버그 수정                                         |  |
|   |          | - 보안 패치                                       |  |
|   |          | - 성능 개선                                       |  |
|   +----------+--------------------------------------------------+  |
|   | Build    | 빌드 번호                                         |  |
|   |          | - CI/CD 자동 증가                                 |  |
|   |          | - 빌드 추적용                                     |  |
|   +----------+--------------------------------------------------+  |
|                                                                   |
+------------------------------------------------------------------+
```

### 버전 정보 관리

```c
// Version.h
// 드라이버 버전 정의
// Driver version definitions

#pragma once

// 버전 번호
// Version numbers
#define DRM_FILTER_VERSION_MAJOR    1
#define DRM_FILTER_VERSION_MINOR    2
#define DRM_FILTER_VERSION_PATCH    3
#define DRM_FILTER_VERSION_BUILD    456

// 버전 문자열
// Version string
#define DRM_FILTER_VERSION_STRING   "1.2.3.456"

// 통신 프로토콜 버전 (클라이언트 호환성 확인용)
// Communication protocol version (for client compatibility)
#define DRM_PROTOCOL_VERSION        0x00010002  // v1.2

// 최소 지원 클라이언트 버전
// Minimum supported client version
#define DRM_MIN_CLIENT_VERSION      0x00010000  // v1.0

// 버전 비교 매크로
// Version comparison macro
#define MAKE_VERSION(major, minor) \
    (((major) << 16) | (minor))

#define VERSION_MAJOR(v)    (((v) >> 16) & 0xFFFF)
#define VERSION_MINOR(v)    ((v) & 0xFFFF)

// 빌드 정보
// Build information
#define DRM_BUILD_DATE      __DATE__
#define DRM_BUILD_TIME      __TIME__

#ifdef _DEBUG
#define DRM_BUILD_TYPE      "Debug"
#else
#define DRM_BUILD_TYPE      "Release"
#endif
```

```c
// VersionInfo.c
// 버전 정보 조회 및 호환성 검사
// Version info query and compatibility check

#include "Version.h"

// 버전 정보 구조체
// Version info structure
typedef struct _DRIVER_VERSION_INFO {
    ULONG StructSize;
    ULONG VersionMajor;
    ULONG VersionMinor;
    ULONG VersionPatch;
    ULONG VersionBuild;
    ULONG ProtocolVersion;
    CHAR BuildDate[16];
    CHAR BuildTime[16];
    CHAR BuildType[16];
} DRIVER_VERSION_INFO, *PDRIVER_VERSION_INFO;

// 버전 정보 가져오기
// Get version info
NTSTATUS
GetDriverVersionInfo(
    _Out_ PDRIVER_VERSION_INFO VersionInfo
    )
{
    if (VersionInfo == NULL) {
        return STATUS_INVALID_PARAMETER;
    }

    RtlZeroMemory(VersionInfo, sizeof(DRIVER_VERSION_INFO));

    VersionInfo->StructSize = sizeof(DRIVER_VERSION_INFO);
    VersionInfo->VersionMajor = DRM_FILTER_VERSION_MAJOR;
    VersionInfo->VersionMinor = DRM_FILTER_VERSION_MINOR;
    VersionInfo->VersionPatch = DRM_FILTER_VERSION_PATCH;
    VersionInfo->VersionBuild = DRM_FILTER_VERSION_BUILD;
    VersionInfo->ProtocolVersion = DRM_PROTOCOL_VERSION;

    RtlStringCbCopyA(VersionInfo->BuildDate, sizeof(VersionInfo->BuildDate), DRM_BUILD_DATE);
    RtlStringCbCopyA(VersionInfo->BuildTime, sizeof(VersionInfo->BuildTime), DRM_BUILD_TIME);
    RtlStringCbCopyA(VersionInfo->BuildType, sizeof(VersionInfo->BuildType), DRM_BUILD_TYPE);

    return STATUS_SUCCESS;
}

// 클라이언트 호환성 검사
// Check client compatibility
NTSTATUS
CheckClientCompatibility(
    _In_ ULONG ClientProtocolVersion
    )
{
    // 클라이언트 버전이 너무 오래됨
    // Client version too old
    if (ClientProtocolVersion < DRM_MIN_CLIENT_VERSION) {
        DbgPrint("[DrmFilter] 클라이언트 버전 너무 낮음: 0x%08X (최소: 0x%08X)\n",
                 ClientProtocolVersion, DRM_MIN_CLIENT_VERSION);
        // [DrmFilter] Client version too low

        return STATUS_REVISION_MISMATCH;
    }

    // Major 버전 불일치 - 호환 불가
    // Major version mismatch - incompatible
    if (VERSION_MAJOR(ClientProtocolVersion) != VERSION_MAJOR(DRM_PROTOCOL_VERSION)) {
        DbgPrint("[DrmFilter] 프로토콜 Major 버전 불일치\n");
        // [DrmFilter] Protocol major version mismatch

        return STATUS_REVISION_MISMATCH;
    }

    // 호환됨
    // Compatible
    return STATUS_SUCCESS;
}

// 연결 시 버전 교환
// Version exchange on connect
NTSTATUS
CommConnectNotify(
    _In_ PFLT_PORT ClientPort,
    _In_opt_ PVOID ServerPortCookie,
    _In_reads_bytes_opt_(SizeOfContext) PVOID ConnectionContext,
    _In_ ULONG SizeOfContext,
    _Outptr_result_maybenull_ PVOID *ConnectionPortCookie
    )
{
    NTSTATUS status;

    // 연결 컨텍스트에서 클라이언트 버전 확인
    // Check client version from connection context
    if (ConnectionContext != NULL && SizeOfContext >= sizeof(ULONG)) {
        ULONG clientVersion = *(PULONG)ConnectionContext;

        status = CheckClientCompatibility(clientVersion);
        if (!NT_SUCCESS(status)) {
            DbgPrint("[DrmFilter] 클라이언트 호환성 검사 실패\n");
            // [DrmFilter] Client compatibility check failed
            return status;
        }
    }

    // ... 나머지 연결 처리

    return STATUS_SUCCESS;
}
```

## 34.2 업데이트 메커니즘

### 핫 업데이트 전략

```c
// HotUpdate.c
// 서비스 중단 최소화 업데이트
// Update with minimal service interruption

#include <fltKernel.h>

// 업데이트 상태
// Update state
typedef enum _UPDATE_STATE {
    UPDATE_IDLE,
    UPDATE_PREPARING,
    UPDATE_IN_PROGRESS,
    UPDATE_COMPLETING,
    UPDATE_FAILED
} UPDATE_STATE;

typedef struct _UPDATE_CONTEXT {
    UPDATE_STATE State;
    ULONG NewVersion;
    UNICODE_STRING NewDriverPath;
    KEVENT CompletionEvent;
    NTSTATUS Result;
} UPDATE_CONTEXT, *PUPDATE_CONTEXT;

UPDATE_CONTEXT g_UpdateContext = { 0 };

// 업데이트 준비
// Prepare for update
NTSTATUS
PrepareForUpdate(
    _In_ PUNICODE_STRING NewDriverPath,
    _In_ ULONG NewVersion
    )
{
    NTSTATUS status = STATUS_SUCCESS;

    // 상태 확인
    // Check state
    if (g_UpdateContext.State != UPDATE_IDLE) {
        return STATUS_DEVICE_BUSY;
    }

    DbgPrint("[DrmFilter] 업데이트 준비 중: v%d.%d\n",
             VERSION_MAJOR(NewVersion),
             VERSION_MINOR(NewVersion));
    // [DrmFilter] Preparing for update

    g_UpdateContext.State = UPDATE_PREPARING;
    g_UpdateContext.NewVersion = NewVersion;

    // 새 드라이버 경로 복사
    // Copy new driver path
    g_UpdateContext.NewDriverPath.Buffer = ExAllocatePool2(
        POOL_FLAG_PAGED,
        NewDriverPath->MaximumLength,
        'dpUd'
    );

    if (g_UpdateContext.NewDriverPath.Buffer == NULL) {
        g_UpdateContext.State = UPDATE_IDLE;
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    RtlCopyUnicodeString(&g_UpdateContext.NewDriverPath, NewDriverPath);

    // 현재 상태 저장
    // Save current state
    status = SaveCurrentState();
    if (!NT_SUCCESS(status)) {
        ExFreePoolWithTag(g_UpdateContext.NewDriverPath.Buffer, 'dpUd');
        g_UpdateContext.State = UPDATE_IDLE;
        return status;
    }

    // 새 연결 차단 (기존 연결은 유지)
    // Block new connections (keep existing)
    BlockNewConnections();

    KeInitializeEvent(&g_UpdateContext.CompletionEvent, NotificationEvent, FALSE);

    g_UpdateContext.State = UPDATE_IN_PROGRESS;

    DbgPrint("[DrmFilter] 업데이트 준비 완료\n");
    // [DrmFilter] Update preparation complete

    return STATUS_SUCCESS;
}

// 현재 상태 저장
// Save current state
NTSTATUS
SaveCurrentState(
    VOID
    )
{
    // 활성 정책을 레지스트리에 저장
    // Save active policies to registry
    SavePoliciesToRegistry();

    // 통계 정보 저장
    // Save statistics
    SaveStatisticsToRegistry();

    // 열린 파일 컨텍스트 정보 저장
    // Save open file context info
    SaveOpenFileContexts();

    return STATUS_SUCCESS;
}

// 활성 작업 완료 대기
// Wait for active operations to complete
NTSTATUS
WaitForActiveOperations(
    _In_ PLARGE_INTEGER Timeout
    )
{
    LARGE_INTEGER waitTime;
    LONG activeOps;

    if (Timeout != NULL) {
        waitTime = *Timeout;
    } else {
        waitTime.QuadPart = -100000000;  // 10초
    }

    DbgPrint("[DrmFilter] 활성 작업 완료 대기 중...\n");
    // [DrmFilter] Waiting for active operations...

    // 활성 작업 카운터 폴링
    // Poll active operation counter
    LARGE_INTEGER startTime, currentTime;
    KeQuerySystemTime(&startTime);

    do {
        activeOps = g_ActiveOperationCount;

        if (activeOps == 0) {
            DbgPrint("[DrmFilter] 모든 작업 완료\n");
            // [DrmFilter] All operations complete
            return STATUS_SUCCESS;
        }

        // 짧은 대기
        // Brief wait
        LARGE_INTEGER delay = { .QuadPart = -100000 };  // 10ms
        KeDelayExecutionThread(KernelMode, FALSE, &delay);

        KeQuerySystemTime(&currentTime);

    } while ((currentTime.QuadPart - startTime.QuadPart) < (-waitTime.QuadPart));

    DbgPrint("[DrmFilter] 타임아웃: %ld 작업 남음\n", activeOps);
    // [DrmFilter] Timeout: operations remaining

    return STATUS_TIMEOUT;
}

// 업데이트 실행 (사용자 모드에서 호출)
// Execute update (called from user mode)
NTSTATUS
ExecuteUpdate(
    VOID
    )
{
    NTSTATUS status;

    if (g_UpdateContext.State != UPDATE_IN_PROGRESS) {
        return STATUS_INVALID_DEVICE_STATE;
    }

    DbgPrint("[DrmFilter] 업데이트 실행 중...\n");
    // [DrmFilter] Executing update...

    // 1. 활성 작업 완료 대기
    // 1. Wait for active operations
    status = WaitForActiveOperations(NULL);
    if (!NT_SUCCESS(status) && status != STATUS_TIMEOUT) {
        g_UpdateContext.State = UPDATE_FAILED;
        return status;
    }

    // 2. 드라이버 언로드 준비
    // 2. Prepare for driver unload
    FltUnregisterFilter(g_FilterHandle);

    // 3. 완료 이벤트 설정
    // 3. Set completion event
    g_UpdateContext.Result = STATUS_SUCCESS;
    KeSetEvent(&g_UpdateContext.CompletionEvent, IO_NO_INCREMENT, FALSE);

    // 사용자 모드 서비스가 새 드라이버 로드
    // User mode service loads new driver

    return STATUS_SUCCESS;
}
```

### 사용자 모드 업데이트 서비스

```csharp
// UpdateService.cs
// 드라이버 업데이트 관리 서비스
// Driver update management service

using System.Diagnostics;
using System.ServiceProcess;

namespace DrmFilterService;

/// <summary>
/// 드라이버 업데이트 관리자
/// Driver update manager
/// </summary>
public sealed class DriverUpdateManager
{
    private readonly ILogger<DriverUpdateManager> _logger;
    private readonly IDriverService _driverService;
    private readonly string _driversPath;

    public DriverUpdateManager(
        ILogger<DriverUpdateManager> logger,
        IDriverService driverService,
        string driversPath)
    {
        _logger = logger;
        _driverService = driverService;
        _driversPath = driversPath;
    }

    /// <summary>
    /// 업데이트 사용 가능 여부 확인
    /// Check if update is available
    /// </summary>
    public async Task<UpdateInfo?> CheckForUpdateAsync()
    {
        try
        {
            // 현재 버전 조회
            // Get current version
            var currentVersion = await _driverService.GetVersionAsync();

            // 서버에서 최신 버전 확인
            // Check latest version from server
            using var httpClient = new HttpClient();
            var response = await httpClient.GetStringAsync(
                "https://update.example.com/drmfilter/version");

            var latestVersion = Version.Parse(response.Trim());

            if (latestVersion > currentVersion)
            {
                return new UpdateInfo
                {
                    CurrentVersion = currentVersion,
                    NewVersion = latestVersion,
                    DownloadUrl = $"https://update.example.com/drmfilter/{latestVersion}/package.cab"
                };
            }

            return null;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "업데이트 확인 실패");
            // Update check failed
            return null;
        }
    }

    /// <summary>
    /// 업데이트 다운로드
    /// Download update
    /// </summary>
    public async Task<string> DownloadUpdateAsync(UpdateInfo updateInfo, IProgress<int>? progress = null)
    {
        var downloadPath = Path.Combine(_driversPath, "updates", $"v{updateInfo.NewVersion}");
        Directory.CreateDirectory(downloadPath);

        var packagePath = Path.Combine(downloadPath, "package.cab");

        _logger.LogInformation("업데이트 다운로드 중: {Version}", updateInfo.NewVersion);
        // Downloading update

        using var httpClient = new HttpClient();
        using var response = await httpClient.GetAsync(
            updateInfo.DownloadUrl,
            HttpCompletionOption.ResponseHeadersRead);

        response.EnsureSuccessStatusCode();

        var totalBytes = response.Content.Headers.ContentLength ?? -1;
        var downloadedBytes = 0L;

        await using var contentStream = await response.Content.ReadAsStreamAsync();
        await using var fileStream = File.Create(packagePath);

        var buffer = new byte[81920];
        int bytesRead;

        while ((bytesRead = await contentStream.ReadAsync(buffer)) > 0)
        {
            await fileStream.WriteAsync(buffer.AsMemory(0, bytesRead));
            downloadedBytes += bytesRead;

            if (totalBytes > 0)
            {
                progress?.Report((int)(downloadedBytes * 100 / totalBytes));
            }
        }

        // 패키지 검증
        // Verify package
        if (!await VerifyPackageAsync(packagePath))
        {
            File.Delete(packagePath);
            throw new InvalidOperationException("패키지 검증 실패");
            // Package verification failed
        }

        // 패키지 압축 해제
        // Extract package
        await ExtractPackageAsync(packagePath, downloadPath);

        return downloadPath;
    }

    /// <summary>
    /// 패키지 서명 검증
    /// Verify package signature
    /// </summary>
    private async Task<bool> VerifyPackageAsync(string packagePath)
    {
        return await Task.Run(() =>
        {
            var startInfo = new ProcessStartInfo
            {
                FileName = "signtool.exe",
                Arguments = $"verify /pa \"{packagePath}\"",
                UseShellExecute = false,
                RedirectStandardOutput = true,
                CreateNoWindow = true
            };

            using var process = Process.Start(startInfo);
            process?.WaitForExit();

            return process?.ExitCode == 0;
        });
    }

    /// <summary>
    /// 업데이트 설치
    /// Install update
    /// </summary>
    public async Task InstallUpdateAsync(string updatePath)
    {
        _logger.LogInformation("업데이트 설치 중...");
        // Installing update...

        try
        {
            // 1. 드라이버에 업데이트 준비 알림
            // 1. Notify driver to prepare for update
            await _driverService.PrepareForUpdateAsync();

            // 2. 드라이버 언로드
            // 2. Unload driver
            _logger.LogInformation("드라이버 언로드 중...");
            // Unloading driver...

            await RunFltmcAsync("unload", "DrmFilter");

            // 3. 파일 교체
            // 3. Replace files
            _logger.LogInformation("파일 교체 중...");
            // Replacing files...

            var systemDriverPath = Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.System),
                "drivers",
                "DrmFilter.sys");

            // 백업 생성
            // Create backup
            var backupPath = systemDriverPath + ".bak";
            if (File.Exists(systemDriverPath))
            {
                File.Copy(systemDriverPath, backupPath, true);
            }

            // 새 드라이버 복사
            // Copy new driver
            var newDriverPath = Path.Combine(updatePath, "DrmFilter.sys");
            File.Copy(newDriverPath, systemDriverPath, true);

            // 4. 드라이버 로드
            // 4. Load driver
            _logger.LogInformation("드라이버 로드 중...");
            // Loading driver...

            await RunFltmcAsync("load", "DrmFilter");

            // 5. 연결 재설정
            // 5. Reconnect
            await _driverService.ConnectAsync();

            _logger.LogInformation("업데이트 완료!");
            // Update complete!
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "업데이트 실패, 롤백 중...");
            // Update failed, rolling back...

            await RollbackAsync();
            throw;
        }
    }

    /// <summary>
    /// 롤백 수행
    /// Perform rollback
    /// </summary>
    public async Task RollbackAsync()
    {
        _logger.LogWarning("롤백 수행 중...");
        // Performing rollback...

        try
        {
            // 드라이버 언로드
            // Unload driver
            await RunFltmcAsync("unload", "DrmFilter");

            // 백업 복원
            // Restore backup
            var systemDriverPath = Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.System),
                "drivers",
                "DrmFilter.sys");

            var backupPath = systemDriverPath + ".bak";

            if (File.Exists(backupPath))
            {
                File.Copy(backupPath, systemDriverPath, true);
            }

            // 드라이버 로드
            // Load driver
            await RunFltmcAsync("load", "DrmFilter");

            _logger.LogInformation("롤백 완료");
            // Rollback complete
        }
        catch (Exception ex)
        {
            _logger.LogCritical(ex, "롤백 실패!");
            // Rollback failed!
            throw;
        }
    }

    private static async Task RunFltmcAsync(string command, string driverName)
    {
        var startInfo = new ProcessStartInfo
        {
            FileName = "fltmc.exe",
            Arguments = $"{command} {driverName}",
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            CreateNoWindow = true
        };

        using var process = Process.Start(startInfo);
        if (process != null)
        {
            await process.WaitForExitAsync();

            if (process.ExitCode != 0)
            {
                var error = await process.StandardError.ReadToEndAsync();
                throw new InvalidOperationException($"fltmc {command} 실패: {error}");
            }
        }
    }

    private static async Task ExtractPackageAsync(string cabPath, string extractPath)
    {
        var startInfo = new ProcessStartInfo
        {
            FileName = "expand.exe",
            Arguments = $"\"{cabPath}\" -F:* \"{extractPath}\"",
            UseShellExecute = false,
            CreateNoWindow = true
        };

        using var process = Process.Start(startInfo);
        if (process != null)
        {
            await process.WaitForExitAsync();
        }
    }
}

/// <summary>
/// 업데이트 정보
/// Update information
/// </summary>
public sealed class UpdateInfo
{
    public required Version CurrentVersion { get; init; }
    public required Version NewVersion { get; init; }
    public required string DownloadUrl { get; init; }
}
```

## 34.3 롤백 메커니즘

### 자동 롤백 시스템

```c
// RollbackSystem.c
// 자동 롤백 시스템
// Automatic rollback system

#include <fltKernel.h>

// 롤백 정보 저장 위치
// Rollback info storage location
#define ROLLBACK_REGISTRY_PATH L"\\Registry\\Machine\\System\\CurrentControlSet\\Services\\DrmFilter\\Rollback"

typedef struct _ROLLBACK_INFO {
    ULONG PreviousVersion;
    WCHAR PreviousDriverPath[260];
    LARGE_INTEGER UpdateTime;
    ULONG FailureCount;
    BOOLEAN RollbackRequired;
} ROLLBACK_INFO, *PROLLBACK_INFO;

// 롤백 정보 저장
// Save rollback info
NTSTATUS
SaveRollbackInfo(
    _In_ ULONG CurrentVersion,
    _In_ PCUNICODE_STRING CurrentDriverPath
    )
{
    NTSTATUS status;
    HANDLE keyHandle;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING keyPath;

    RtlInitUnicodeString(&keyPath, ROLLBACK_REGISTRY_PATH);

    InitializeObjectAttributes(
        &oa,
        &keyPath,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        NULL
    );

    status = ZwCreateKey(
        &keyHandle,
        KEY_WRITE,
        &oa,
        0,
        NULL,
        REG_OPTION_NON_VOLATILE,
        NULL
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 버전 저장
    // Save version
    UNICODE_STRING valueName;
    RtlInitUnicodeString(&valueName, L"PreviousVersion");

    status = ZwSetValueKey(
        keyHandle,
        &valueName,
        0,
        REG_DWORD,
        &CurrentVersion,
        sizeof(CurrentVersion)
    );

    // 경로 저장
    // Save path
    if (NT_SUCCESS(status)) {
        RtlInitUnicodeString(&valueName, L"PreviousPath");

        status = ZwSetValueKey(
            keyHandle,
            &valueName,
            0,
            REG_SZ,
            CurrentDriverPath->Buffer,
            CurrentDriverPath->Length + sizeof(WCHAR)
        );
    }

    // 업데이트 시간 저장
    // Save update time
    if (NT_SUCCESS(status)) {
        LARGE_INTEGER currentTime;
        KeQuerySystemTime(&currentTime);

        RtlInitUnicodeString(&valueName, L"UpdateTime");

        status = ZwSetValueKey(
            keyHandle,
            &valueName,
            0,
            REG_QWORD,
            &currentTime,
            sizeof(currentTime)
        );
    }

    ZwClose(keyHandle);

    return status;
}

// 롤백 필요 여부 확인
// Check if rollback is needed
BOOLEAN
CheckRollbackRequired(
    VOID
    )
{
    NTSTATUS status;
    HANDLE keyHandle;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING keyPath;
    ULONG failureCount = 0;

    RtlInitUnicodeString(&keyPath, ROLLBACK_REGISTRY_PATH);

    InitializeObjectAttributes(
        &oa,
        &keyPath,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        NULL
    );

    status = ZwOpenKey(&keyHandle, KEY_READ, &oa);

    if (!NT_SUCCESS(status)) {
        return FALSE;
    }

    // 실패 카운트 읽기
    // Read failure count
    UNICODE_STRING valueName;
    RtlInitUnicodeString(&valueName, L"FailureCount");

    KEY_VALUE_PARTIAL_INFORMATION valueInfo;
    ULONG resultLength;

    status = ZwQueryValueKey(
        keyHandle,
        &valueName,
        KeyValuePartialInformation,
        &valueInfo,
        sizeof(valueInfo) + sizeof(ULONG),
        &resultLength
    );

    if (NT_SUCCESS(status) && valueInfo.Type == REG_DWORD) {
        failureCount = *(PULONG)valueInfo.Data;
    }

    ZwClose(keyHandle);

    // 3회 이상 실패 시 롤백 필요
    // Rollback needed after 3+ failures
    return (failureCount >= 3);
}

// 실패 카운트 증가
// Increment failure count
VOID
IncrementFailureCount(
    VOID
    )
{
    NTSTATUS status;
    HANDLE keyHandle;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING keyPath;

    RtlInitUnicodeString(&keyPath, ROLLBACK_REGISTRY_PATH);

    InitializeObjectAttributes(
        &oa,
        &keyPath,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        NULL
    );

    status = ZwOpenKey(&keyHandle, KEY_READ | KEY_WRITE, &oa);

    if (!NT_SUCCESS(status)) {
        return;
    }

    // 현재 카운트 읽기
    // Read current count
    UNICODE_STRING valueName;
    RtlInitUnicodeString(&valueName, L"FailureCount");

    UCHAR buffer[sizeof(KEY_VALUE_PARTIAL_INFORMATION) + sizeof(ULONG)];
    PKEY_VALUE_PARTIAL_INFORMATION valueInfo = (PKEY_VALUE_PARTIAL_INFORMATION)buffer;
    ULONG resultLength;
    ULONG failureCount = 0;

    status = ZwQueryValueKey(
        keyHandle,
        &valueName,
        KeyValuePartialInformation,
        valueInfo,
        sizeof(buffer),
        &resultLength
    );

    if (NT_SUCCESS(status) && valueInfo->Type == REG_DWORD) {
        failureCount = *(PULONG)valueInfo->Data;
    }

    // 증가
    // Increment
    failureCount++;

    // 저장
    // Save
    ZwSetValueKey(
        keyHandle,
        &valueName,
        0,
        REG_DWORD,
        &failureCount,
        sizeof(failureCount)
    );

    DbgPrint("[DrmFilter] 실패 카운트 증가: %lu\n", failureCount);
    // [DrmFilter] Failure count incremented

    ZwClose(keyHandle);
}

// 실패 카운트 초기화 (성공 시)
// Reset failure count (on success)
VOID
ResetFailureCount(
    VOID
    )
{
    NTSTATUS status;
    HANDLE keyHandle;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING keyPath;

    RtlInitUnicodeString(&keyPath, ROLLBACK_REGISTRY_PATH);

    InitializeObjectAttributes(
        &oa,
        &keyPath,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        NULL
    );

    status = ZwOpenKey(&keyHandle, KEY_WRITE, &oa);

    if (!NT_SUCCESS(status)) {
        return;
    }

    UNICODE_STRING valueName;
    RtlInitUnicodeString(&valueName, L"FailureCount");

    ULONG zero = 0;

    ZwSetValueKey(
        keyHandle,
        &valueName,
        0,
        REG_DWORD,
        &zero,
        sizeof(zero)
    );

    ZwClose(keyHandle);

    DbgPrint("[DrmFilter] 실패 카운트 초기화\n");
    // [DrmFilter] Failure count reset
}

// 드라이버 시작 시 롤백 확인
// Check rollback on driver start
NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath
    )
{
    NTSTATUS status;

    // 롤백 필요 여부 확인
    // Check if rollback needed
    if (CheckRollbackRequired()) {
        DbgPrint("[DrmFilter] 롤백이 필요합니다!\n");
        // [DrmFilter] Rollback required!

        // 롤백 플래그 설정 - 사용자 모드 서비스가 처리
        // Set rollback flag - user mode service handles it
        SetRollbackRequired(TRUE);
    }

    // 정상 초기화
    // Normal initialization
    status = InitializeDriver(DriverObject, RegistryPath);

    if (NT_SUCCESS(status)) {
        // 성공 시 실패 카운트 초기화
        // Reset failure count on success
        ResetFailureCount();
    } else {
        // 실패 시 카운트 증가
        // Increment count on failure
        IncrementFailureCount();
    }

    return status;
}
```

## 34.4 원격 관리

### 중앙 관리 서버 통신

```csharp
// RemoteManagement.cs
// 원격 관리 클라이언트
// Remote management client

using System.Net.Http.Json;
using System.Security.Cryptography;
using System.Text;

namespace DrmFilterService;

/// <summary>
/// 중앙 관리 서버 클라이언트
/// Central management server client
/// </summary>
public sealed class ManagementClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ManagementClient> _logger;
    private readonly string _serverUrl;
    private readonly string _deviceId;
    private readonly string _apiKey;

    public ManagementClient(
        ILogger<ManagementClient> logger,
        string serverUrl,
        string deviceId,
        string apiKey)
    {
        _logger = logger;
        _serverUrl = serverUrl;
        _deviceId = deviceId;
        _apiKey = apiKey;

        _httpClient = new HttpClient
        {
            BaseAddress = new Uri(serverUrl)
        };

        _httpClient.DefaultRequestHeaders.Add("X-API-Key", apiKey);
        _httpClient.DefaultRequestHeaders.Add("X-Device-ID", deviceId);
    }

    /// <summary>
    /// 서버에 상태 보고
    /// Report status to server
    /// </summary>
    public async Task ReportStatusAsync(DriverStatus status)
    {
        try
        {
            var payload = new
            {
                deviceId = _deviceId,
                timestamp = DateTime.UtcNow,
                driverVersion = status.Version,
                isEnabled = status.IsEnabled,
                activePolicies = status.ActivePolicies,
                statistics = new
                {
                    filesScanned = status.Statistics.FilesScanned,
                    blockedOperations = status.Statistics.OperationsBlocked,
                    averageLatencyMs = status.Statistics.AverageLatencyMs
                }
            };

            var response = await _httpClient.PostAsJsonAsync("/api/devices/status", payload);
            response.EnsureSuccessStatusCode();
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "상태 보고 실패");
            // Status report failed
        }
    }

    /// <summary>
    /// 서버에서 정책 가져오기
    /// Fetch policies from server
    /// </summary>
    public async Task<IEnumerable<PolicyDto>> FetchPoliciesAsync()
    {
        try
        {
            var response = await _httpClient.GetAsync($"/api/devices/{_deviceId}/policies");
            response.EnsureSuccessStatusCode();

            var policies = await response.Content.ReadFromJsonAsync<PolicyDto[]>();
            return policies ?? [];
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "정책 가져오기 실패");
            // Policy fetch failed
            throw;
        }
    }

    /// <summary>
    /// 명령 확인 및 실행
    /// Check and execute commands
    /// </summary>
    public async Task<CommandDto?> CheckForCommandAsync()
    {
        try
        {
            var response = await _httpClient.GetAsync($"/api/devices/{_deviceId}/commands/pending");

            if (response.StatusCode == System.Net.HttpStatusCode.NoContent)
            {
                return null;
            }

            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<CommandDto>();
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "명령 확인 실패");
            // Command check failed
            return null;
        }
    }

    /// <summary>
    /// 명령 결과 보고
    /// Report command result
    /// </summary>
    public async Task ReportCommandResultAsync(string commandId, bool success, string? message = null)
    {
        try
        {
            var payload = new
            {
                commandId,
                success,
                message,
                completedAt = DateTime.UtcNow
            };

            var response = await _httpClient.PostAsJsonAsync(
                $"/api/devices/{_deviceId}/commands/{commandId}/result",
                payload);

            response.EnsureSuccessStatusCode();
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "명령 결과 보고 실패");
            // Command result report failed
        }
    }

    /// <summary>
    /// 이벤트 전송
    /// Send event
    /// </summary>
    public async Task SendEventAsync(SecurityEvent securityEvent)
    {
        try
        {
            var payload = new
            {
                deviceId = _deviceId,
                timestamp = securityEvent.Timestamp,
                eventType = securityEvent.Type.ToString(),
                filePath = securityEvent.FilePath,
                processName = securityEvent.ProcessName,
                processId = securityEvent.ProcessId,
                action = securityEvent.Action.ToString(),
                details = securityEvent.Details
            };

            var response = await _httpClient.PostAsJsonAsync("/api/events", payload);
            response.EnsureSuccessStatusCode();
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "이벤트 전송 실패");
            // Event send failed
        }
    }
}

// DTOs
public record PolicyDto(
    int Id,
    string Name,
    string Path,
    string Type,
    uint Flags,
    bool IsEnabled);

public record CommandDto(
    string Id,
    string Type,
    Dictionary<string, object>? Parameters,
    DateTime CreatedAt);

public record SecurityEvent(
    DateTime Timestamp,
    SecurityEventType Type,
    string FilePath,
    string ProcessName,
    uint ProcessId,
    SecurityAction Action,
    string? Details);

public enum SecurityEventType
{
    FileAccess,
    PolicyViolation,
    SuspiciousActivity,
    ConfigChange
}

public enum SecurityAction
{
    Allowed,
    Blocked,
    Logged
}
```

### 명령 처리 핸들러

```csharp
// CommandHandler.cs
// 원격 명령 처리
// Remote command handling

namespace DrmFilterService;

/// <summary>
/// 원격 명령 핸들러
/// Remote command handler
/// </summary>
public sealed class CommandHandler
{
    private readonly ILogger<CommandHandler> _logger;
    private readonly IDriverService _driverService;
    private readonly IPolicyRepository _policyRepository;
    private readonly DriverUpdateManager _updateManager;
    private readonly ManagementClient _managementClient;

    public CommandHandler(
        ILogger<CommandHandler> logger,
        IDriverService driverService,
        IPolicyRepository policyRepository,
        DriverUpdateManager updateManager,
        ManagementClient managementClient)
    {
        _logger = logger;
        _driverService = driverService;
        _policyRepository = policyRepository;
        _updateManager = updateManager;
        _managementClient = managementClient;
    }

    /// <summary>
    /// 명령 실행
    /// Execute command
    /// </summary>
    public async Task ExecuteCommandAsync(CommandDto command)
    {
        _logger.LogInformation("명령 실행: {Type} ({Id})", command.Type, command.Id);
        // Executing command

        try
        {
            bool success = command.Type switch
            {
                "UPDATE_POLICIES" => await HandleUpdatePoliciesAsync(command.Parameters),
                "RELOAD_DRIVER" => await HandleReloadDriverAsync(),
                "UPDATE_DRIVER" => await HandleUpdateDriverAsync(command.Parameters),
                "COLLECT_LOGS" => await HandleCollectLogsAsync(command.Parameters),
                "ENABLE_MONITORING" => await HandleEnableMonitoringAsync(true),
                "DISABLE_MONITORING" => await HandleEnableMonitoringAsync(false),
                "RESTART_SERVICE" => await HandleRestartServiceAsync(),
                _ => throw new NotSupportedException($"알 수 없는 명령: {command.Type}")
            };

            await _managementClient.ReportCommandResultAsync(
                command.Id,
                success,
                success ? "명령 실행 성공" : "명령 실행 실패");
            // Command execution success/failure
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "명령 실행 실패: {Type}", command.Type);
            // Command execution failed

            await _managementClient.ReportCommandResultAsync(
                command.Id,
                false,
                ex.Message);
        }
    }

    private async Task<bool> HandleUpdatePoliciesAsync(Dictionary<string, object>? parameters)
    {
        _logger.LogInformation("정책 업데이트 중...");
        // Updating policies...

        // 서버에서 정책 가져오기
        // Fetch policies from server
        var policies = await _managementClient.FetchPoliciesAsync();

        // 로컬 정책 업데이트
        // Update local policies
        foreach (var policyDto in policies)
        {
            var policy = new ProtectionPolicy
            {
                Id = policyDto.Id,
                Name = policyDto.Name,
                Path = policyDto.Path,
                Type = Enum.Parse<PolicyType>(policyDto.Type),
                Flags = (ProtectionFlags)policyDto.Flags,
                IsEnabled = policyDto.IsEnabled,
                CreatedAt = DateTime.Now
            };

            var existing = await _policyRepository.GetByIdAsync(policyDto.Id);
            if (existing == null)
            {
                await _policyRepository.AddAsync(policy);
            }
            else
            {
                await _policyRepository.UpdateAsync(policy);
            }

            // 드라이버에 적용
            // Apply to driver
            if (policy.IsEnabled)
            {
                await _driverService.SendPolicyUpdateAsync(policy);
            }
            else
            {
                await _driverService.SendPolicyRemovalAsync(policy.Path);
            }
        }

        _logger.LogInformation("{Count}개 정책 업데이트됨", policies.Count());
        // policies updated

        return true;
    }

    private async Task<bool> HandleReloadDriverAsync()
    {
        _logger.LogInformation("드라이버 다시 로드 중...");
        // Reloading driver...

        await _driverService.DisconnectAsync();

        // fltmc를 통한 언로드/로드
        // Unload/load via fltmc
        await RunProcessAsync("fltmc", "unload DrmFilter");
        await Task.Delay(1000);
        await RunProcessAsync("fltmc", "load DrmFilter");

        await _driverService.ConnectAsync();

        return _driverService.IsConnected;
    }

    private async Task<bool> HandleUpdateDriverAsync(Dictionary<string, object>? parameters)
    {
        if (parameters == null || !parameters.TryGetValue("version", out var versionObj))
        {
            throw new ArgumentException("버전 정보 필요");
            // Version info required
        }

        var version = Version.Parse(versionObj.ToString()!);

        var updateInfo = new UpdateInfo
        {
            CurrentVersion = await _driverService.GetVersionAsync(),
            NewVersion = version,
            DownloadUrl = parameters.GetValueOrDefault("downloadUrl")?.ToString()
                ?? throw new ArgumentException("다운로드 URL 필요")
        };

        var downloadPath = await _updateManager.DownloadUpdateAsync(updateInfo);
        await _updateManager.InstallUpdateAsync(downloadPath);

        return true;
    }

    private async Task<bool> HandleCollectLogsAsync(Dictionary<string, object>? parameters)
    {
        _logger.LogInformation("로그 수집 중...");
        // Collecting logs...

        // 로그 파일 수집 및 압축
        // Collect and compress log files
        var logsPath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.CommonApplicationData),
            "DrmFilter",
            "Logs");

        var archivePath = Path.Combine(Path.GetTempPath(), $"DrmFilterLogs_{DateTime.Now:yyyyMMdd_HHmmss}.zip");

        if (Directory.Exists(logsPath))
        {
            System.IO.Compression.ZipFile.CreateFromDirectory(logsPath, archivePath);

            // 서버에 업로드 (구현 필요)
            // Upload to server (implementation needed)
            // await UploadLogsAsync(archivePath);

            File.Delete(archivePath);
        }

        return true;
    }

    private async Task<bool> HandleEnableMonitoringAsync(bool enable)
    {
        await _driverService.SendConfigChangeAsync(enable, true);
        return true;
    }

    private async Task<bool> HandleRestartServiceAsync()
    {
        _logger.LogInformation("서비스 재시작 중...");
        // Restarting service...

        // 자기 자신 재시작은 별도 프로세스 필요
        // Restarting self requires separate process
        var restartScript = Path.Combine(Path.GetTempPath(), "restart_service.ps1");
        await File.WriteAllTextAsync(restartScript, @"
            Start-Sleep -Seconds 2
            Restart-Service DrmFilterService -Force
        ");

        Process.Start(new ProcessStartInfo
        {
            FileName = "powershell.exe",
            Arguments = $"-ExecutionPolicy Bypass -File \"{restartScript}\"",
            UseShellExecute = true,
            CreateNoWindow = true
        });

        return true;
    }

    private static async Task RunProcessAsync(string fileName, string arguments)
    {
        using var process = Process.Start(new ProcessStartInfo
        {
            FileName = fileName,
            Arguments = arguments,
            UseShellExecute = false,
            CreateNoWindow = true
        });

        if (process != null)
        {
            await process.WaitForExitAsync();
        }
    }
}
```

## 34.5 정리

이 챕터에서 학습한 내용:

1. **버전 관리**
   - 시맨틱 버전 체계
   - 프로토콜 호환성 검사
   - 클라이언트 버전 확인

2. **업데이트 메커니즘**
   - 핫 업데이트 전략
   - 서비스 중단 최소화
   - 파일 교체 프로세스

3. **롤백 시스템**
   - 자동 롤백 조건
   - 실패 카운트 관리
   - 백업 복원

4. **원격 관리**
   - 중앙 서버 통신
   - 명령 처리 시스템
   - 상태 보고

Part 11이 완료되었습니다. 다음 Part 12에서는 **고급 주제**를 다룹니다.

---

**다음 챕터 예고**: Chapter 35에서는 가상화 환경 지원, 컨테이너 호환성, 클라우드 환경에서의 드라이버 운영을 학습합니다.
