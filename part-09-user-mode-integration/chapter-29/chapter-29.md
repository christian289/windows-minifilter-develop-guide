# Chapter 29: 사용자 모드 서비스 설계

## 난이도: ★★★★☆ (중급-고급)

## 학습 목표
- 커널-사용자 모드 통신 아키텍처 이해
- FilterManager 통신 포트 구현
- Windows 서비스로 드라이버 관리
- 비동기 메시지 처리 패턴 구현

## 29.1 통신 아키텍처 개요

### 왜 사용자 모드 컴포넌트가 필요한가?

Minifilter 드라이버는 커널 모드에서 실행되어 강력한 파일 시스템 제어가 가능하지만, 몇 가지 제한이 있습니다:

```
+------------------------------------------------------------------+
|                     시스템 아키텍처                               |
+------------------------------------------------------------------+
|                                                                   |
|  +------------------+     +------------------+                    |
|  | UI 애플리케이션   |     | 관리 콘솔        |                    |
|  | (WPF/WinForms)   |     | (설정/모니터링)   |                    |
|  +--------+---------+     +--------+---------+                    |
|           |                        |                              |
|           v                        v                              |
|  +--------------------------------------------------+            |
|  |           사용자 모드 서비스                       |            |
|  |  - 드라이버 통신 관리                             |            |
|  |  - 정책 데이터베이스 관리                         |            |
|  |  - 로그 수집 및 분석                              |            |
|  |  - 네트워크 통신 (중앙 서버)                      |            |
|  +------------------------+--------------------------+            |
|                           |                                       |
|  =========================|====================================== |
|                           | FilterManager Communication Port      |
|  =========================|====================================== |
|                           v                                       |
|  +--------------------------------------------------+            |
|  |              Minifilter 드라이버                  |            |
|  |  - 파일 I/O 모니터링                              |            |
|  |  - 정책 적용                                      |            |
|  |  - 이벤트 생성                                    |            |
|  +--------------------------------------------------+            |
|                                                                   |
+------------------------------------------------------------------+
```

### 커널 모드의 한계

```c
// 커널 모드에서 할 수 없거나 어려운 작업들
// Things that are difficult or impossible in kernel mode

// 1. 네트워크 통신 - 중앙 서버와 정책 동기화 불가
// Network communication - cannot sync with central server
// NTSTATUS status = HttpSendRequest(...);  // 불가능!

// 2. 데이터베이스 접근 - SQL Server, SQLite 등 사용 불가
// Database access - cannot use SQL Server, SQLite, etc.
// SqlConnection conn = ...;  // 불가능!

// 3. 복잡한 UI - 사용자에게 결정 요청
// Complex UI - requesting user decisions
// MessageBox("파일을 삭제하시겠습니까?");  // 불가능!

// 4. 고급 암호화 - CNG/CAPI 일부만 사용 가능
// Advanced encryption - only partial CNG/CAPI available

// 5. 파일 파싱 라이브러리 - XML, JSON, ZIP 등
// File parsing libraries - XML, JSON, ZIP, etc.
```

### .NET 개발자를 위한 비교

```csharp
// C# - Named Pipes 통신 (비슷한 개념)
// C# - Named Pipes communication (similar concept)

// 서버 측
using var server = new NamedPipeServerStream("MyPipe");
await server.WaitForConnectionAsync();
using var reader = new StreamReader(server);
string message = await reader.ReadLineAsync();

// 클라이언트 측
using var client = new NamedPipeClientStream(".", "MyPipe");
await client.ConnectAsync();
using var writer = new StreamWriter(client);
await writer.WriteLineAsync("Hello from client");

// FilterManager Communication Port는 이와 유사하지만
// 커널 모드 드라이버와 통신하는 점이 다릅니다
```

## 29.2 FilterManager 통신 포트

### 드라이버 측 포트 설정

```c
// CommunicationPort.h
// 통신 포트 헤더 파일
// Communication port header file

#pragma once
#include <fltKernel.h>

// 포트 이름 정의
// Port name definition
#define DRM_PORT_NAME   L"\\DrmFilterPort"

// 최대 연결 수
// Maximum connections
#define MAX_CONNECTIONS 10

// 메시지 타입 정의
// Message type definitions
typedef enum _MESSAGE_TYPE {
    MSG_TYPE_POLICY_UPDATE = 1,    // 정책 업데이트
    MSG_TYPE_FILE_EVENT = 2,       // 파일 이벤트 알림
    MSG_TYPE_STATUS_REQUEST = 3,   // 상태 요청
    MSG_TYPE_STATUS_RESPONSE = 4,  // 상태 응답
    MSG_TYPE_BLOCK_NOTIFICATION = 5, // 차단 알림
    MSG_TYPE_CONFIG_CHANGE = 6     // 설정 변경
} MESSAGE_TYPE;

// 커널에서 사용자 모드로 보내는 메시지
// Message from kernel to user mode
typedef struct _DRIVER_MESSAGE {
    MESSAGE_TYPE Type;
    ULONG ProcessId;
    ULONG ThreadId;
    LONGLONG Timestamp;
    WCHAR FilePath[260];
    ULONG Operation;           // IRP_MJ_xxx
    NTSTATUS ResultStatus;
    ULONG AdditionalDataSize;
    UCHAR AdditionalData[1];   // 가변 길이 데이터
} DRIVER_MESSAGE, *PDRIVER_MESSAGE;

// 사용자 모드에서 커널로 보내는 메시지
// Message from user mode to kernel
typedef struct _USER_MESSAGE {
    MESSAGE_TYPE Type;
    union {
        struct {
            ULONG PolicyVersion;
            ULONG PolicyFlags;
            WCHAR ProtectedPath[260];
        } PolicyUpdate;

        struct {
            BOOLEAN EnableMonitoring;
            BOOLEAN EnableBlocking;
            ULONG LogLevel;
        } ConfigChange;

        struct {
            ULONG Reserved;
        } StatusRequest;
    } Data;
} USER_MESSAGE, *PUSER_MESSAGE;

// 응답 메시지
// Reply message
typedef struct _REPLY_MESSAGE {
    NTSTATUS Status;
    ULONG DataSize;
    UCHAR Data[1];
} REPLY_MESSAGE, *PREPLY_MESSAGE;

// 통신 포트 컨텍스트
// Communication port context
typedef struct _PORT_CONTEXT {
    PFLT_PORT ClientPort;
    HANDLE ProcessId;
    BOOLEAN IsAdmin;
    LONG MessageCount;
} PORT_CONTEXT, *PPORT_CONTEXT;

// 전역 통신 데이터
// Global communication data
typedef struct _COMMUNICATION_DATA {
    PFLT_PORT ServerPort;
    PFLT_PORT ClientPort;
    LONG ConnectedClients;
    LIST_ENTRY MessageQueue;
    KSPIN_LOCK QueueLock;
    KEVENT NewMessageEvent;
} COMMUNICATION_DATA, *PCOMMUNICATION_DATA;

// 함수 선언
// Function declarations
NTSTATUS
CommInitialize(
    _In_ PFLT_FILTER Filter
    );

VOID
CommUninitialize(
    VOID
    );

NTSTATUS
CommSendMessage(
    _In_ PDRIVER_MESSAGE Message,
    _In_ ULONG MessageSize
    );
```

### 드라이버 측 포트 구현

```c
// CommunicationPort.c
// 통신 포트 구현
// Communication port implementation

#include "CommunicationPort.h"

// 전역 통신 데이터
// Global communication data
COMMUNICATION_DATA g_CommData = { 0 };

// 연결 콜백 - 클라이언트가 연결할 때 호출
// Connect callback - called when client connects
NTSTATUS
CommConnectNotify(
    _In_ PFLT_PORT ClientPort,
    _In_opt_ PVOID ServerPortCookie,
    _In_reads_bytes_opt_(SizeOfContext) PVOID ConnectionContext,
    _In_ ULONG SizeOfContext,
    _Outptr_result_maybenull_ PVOID *ConnectionPortCookie
    )
{
    PPORT_CONTEXT portContext = NULL;
    NTSTATUS status = STATUS_SUCCESS;

    UNREFERENCED_PARAMETER(ServerPortCookie);

    // 연결 수 확인
    // Check connection count
    if (InterlockedIncrement(&g_CommData.ConnectedClients) > MAX_CONNECTIONS) {
        InterlockedDecrement(&g_CommData.ConnectedClients);

        DbgPrint("[DrmFilter] 최대 연결 수 초과\n");
        // [DrmFilter] Maximum connections exceeded

        return STATUS_CONNECTION_REFUSED;
    }

    // 포트 컨텍스트 할당
    // Allocate port context
    portContext = ExAllocatePool2(
        POOL_FLAG_NON_PAGED,
        sizeof(PORT_CONTEXT),
        'tCmC'
        );

    if (portContext == NULL) {
        InterlockedDecrement(&g_CommData.ConnectedClients);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // 컨텍스트 초기화
    // Initialize context
    portContext->ClientPort = ClientPort;
    portContext->ProcessId = PsGetCurrentProcessId();
    portContext->IsAdmin = SeSinglePrivilegeCheck(
        SeExports->SeSecurityPrivilege,
        UserMode
        );
    portContext->MessageCount = 0;

    // 클라이언트 포트 저장 (단일 클라이언트 가정)
    // Save client port (assuming single client)
    g_CommData.ClientPort = ClientPort;

    *ConnectionPortCookie = portContext;

    DbgPrint("[DrmFilter] 클라이언트 연결됨: PID=%lu, Admin=%d\n",
             HandleToUlong(portContext->ProcessId),
             portContext->IsAdmin);
    // [DrmFilter] Client connected: PID=%lu, Admin=%d

    return STATUS_SUCCESS;
}

// 연결 해제 콜백
// Disconnect callback
VOID
CommDisconnectNotify(
    _In_opt_ PVOID ConnectionCookie
    )
{
    PPORT_CONTEXT portContext = (PPORT_CONTEXT)ConnectionCookie;

    if (portContext != NULL) {
        DbgPrint("[DrmFilter] 클라이언트 연결 해제: PID=%lu, Messages=%ld\n",
                 HandleToUlong(portContext->ProcessId),
                 portContext->MessageCount);
        // [DrmFilter] Client disconnected: PID=%lu, Messages=%ld

        // 클라이언트 포트 초기화
        // Clear client port
        if (g_CommData.ClientPort == portContext->ClientPort) {
            g_CommData.ClientPort = NULL;
        }

        ExFreePoolWithTag(portContext, 'tCmC');
    }

    InterlockedDecrement(&g_CommData.ConnectedClients);
}

// 메시지 수신 콜백 - 사용자 모드에서 메시지를 보낼 때 호출
// Message receive callback - called when user mode sends message
NTSTATUS
CommMessageNotify(
    _In_opt_ PVOID PortCookie,
    _In_reads_bytes_opt_(InputBufferLength) PVOID InputBuffer,
    _In_ ULONG InputBufferLength,
    _Out_writes_bytes_to_opt_(OutputBufferLength, *ReturnOutputBufferLength) PVOID OutputBuffer,
    _In_ ULONG OutputBufferLength,
    _Out_ PULONG ReturnOutputBufferLength
    )
{
    PPORT_CONTEXT portContext = (PPORT_CONTEXT)PortCookie;
    PUSER_MESSAGE userMsg = (PUSER_MESSAGE)InputBuffer;
    PREPLY_MESSAGE reply = (PREPLY_MESSAGE)OutputBuffer;
    NTSTATUS status = STATUS_SUCCESS;

    *ReturnOutputBufferLength = 0;

    // 입력 검증
    // Input validation
    if (InputBuffer == NULL || InputBufferLength < sizeof(USER_MESSAGE)) {
        return STATUS_INVALID_PARAMETER;
    }

    // 메시지 카운트 증가
    // Increment message count
    if (portContext != NULL) {
        InterlockedIncrement(&portContext->MessageCount);
    }

    DbgPrint("[DrmFilter] 메시지 수신: Type=%d\n", userMsg->Type);
    // [DrmFilter] Message received: Type=%d

    // 메시지 타입에 따른 처리
    // Process based on message type
    switch (userMsg->Type) {

        case MSG_TYPE_POLICY_UPDATE:
            // 정책 업데이트 처리
            // Handle policy update
            status = HandlePolicyUpdate(&userMsg->Data.PolicyUpdate);
            break;

        case MSG_TYPE_CONFIG_CHANGE:
            // 설정 변경 처리
            // Handle configuration change
            status = HandleConfigChange(&userMsg->Data.ConfigChange);
            break;

        case MSG_TYPE_STATUS_REQUEST:
            // 상태 요청 처리
            // Handle status request
            if (OutputBuffer != NULL &&
                OutputBufferLength >= sizeof(REPLY_MESSAGE)) {
                status = HandleStatusRequest(reply, ReturnOutputBufferLength);
            }
            break;

        default:
            status = STATUS_INVALID_PARAMETER;
            break;
    }

    return status;
}

// 정책 업데이트 처리
// Handle policy update
NTSTATUS
HandlePolicyUpdate(
    _In_ PVOID PolicyData
    )
{
    // 정책 업데이트 데이터 구조
    // Policy update data structure
    struct {
        ULONG PolicyVersion;
        ULONG PolicyFlags;
        WCHAR ProtectedPath[260];
    } *policy = PolicyData;

    DbgPrint("[DrmFilter] 정책 업데이트: Version=%lu, Flags=0x%08X, Path=%ws\n",
             policy->PolicyVersion,
             policy->PolicyFlags,
             policy->ProtectedPath);
    // [DrmFilter] Policy update: Version=%lu, Flags=0x%08X, Path=%ws

    // 여기에서 정책 매니저 업데이트
    // Update policy manager here
    // PolicyManagerUpdate(policy->ProtectedPath, policy->PolicyFlags);

    return STATUS_SUCCESS;
}

// 설정 변경 처리
// Handle configuration change
NTSTATUS
HandleConfigChange(
    _In_ PVOID ConfigData
    )
{
    struct {
        BOOLEAN EnableMonitoring;
        BOOLEAN EnableBlocking;
        ULONG LogLevel;
    } *config = ConfigData;

    DbgPrint("[DrmFilter] 설정 변경: Monitor=%d, Block=%d, LogLevel=%lu\n",
             config->EnableMonitoring,
             config->EnableBlocking,
             config->LogLevel);
    // [DrmFilter] Config change: Monitor=%d, Block=%d, LogLevel=%lu

    // 전역 설정 업데이트
    // Update global settings
    // g_Settings.EnableMonitoring = config->EnableMonitoring;
    // g_Settings.EnableBlocking = config->EnableBlocking;
    // g_Settings.LogLevel = config->LogLevel;

    return STATUS_SUCCESS;
}

// 상태 요청 처리
// Handle status request
NTSTATUS
HandleStatusRequest(
    _Out_ PREPLY_MESSAGE Reply,
    _Out_ PULONG ReplyLength
    )
{
    // 상태 정보 구조
    // Status information structure
    typedef struct _DRIVER_STATUS {
        ULONG Version;
        ULONG ActivePolicies;
        LONGLONG FilesScanned;
        LONGLONG BlockedOperations;
        BOOLEAN IsEnabled;
    } DRIVER_STATUS;

    DRIVER_STATUS status = { 0 };

    // 상태 정보 수집
    // Collect status information
    status.Version = 0x00010000;  // v1.0
    status.ActivePolicies = 5;    // PolicyManagerGetCount();
    status.FilesScanned = 12345;  // g_Statistics.FilesScanned;
    status.BlockedOperations = 42; // g_Statistics.BlockedOps;
    status.IsEnabled = TRUE;       // g_Settings.IsEnabled;

    // 응답 구성
    // Build reply
    Reply->Status = STATUS_SUCCESS;
    Reply->DataSize = sizeof(DRIVER_STATUS);
    RtlCopyMemory(Reply->Data, &status, sizeof(DRIVER_STATUS));

    *ReplyLength = FIELD_OFFSET(REPLY_MESSAGE, Data) + sizeof(DRIVER_STATUS);

    return STATUS_SUCCESS;
}

// 통신 초기화
// Initialize communication
NTSTATUS
CommInitialize(
    _In_ PFLT_FILTER Filter
    )
{
    NTSTATUS status;
    PSECURITY_DESCRIPTOR sd = NULL;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING portName;

    // 보안 디스크립터 생성 - 모든 사용자 접근 허용 (프로덕션에서는 제한 필요)
    // Create security descriptor - allow all users (restrict in production)
    status = FltBuildDefaultSecurityDescriptor(&sd, FLT_PORT_ALL_ACCESS);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DrmFilter] 보안 디스크립터 생성 실패: 0x%08X\n", status);
        // [DrmFilter] Failed to create security descriptor: 0x%08X
        return status;
    }

    // 포트 이름 초기화
    // Initialize port name
    RtlInitUnicodeString(&portName, DRM_PORT_NAME);

    InitializeObjectAttributes(
        &oa,
        &portName,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        sd
        );

    // 통신 포트 생성
    // Create communication port
    status = FltCreateCommunicationPort(
        Filter,
        &g_CommData.ServerPort,
        &oa,
        NULL,                      // ServerPortCookie
        CommConnectNotify,         // ConnectNotify
        CommDisconnectNotify,      // DisconnectNotify
        CommMessageNotify,         // MessageNotify
        MAX_CONNECTIONS            // MaxConnections
        );

    // 보안 디스크립터 해제
    // Free security descriptor
    FltFreeSecurityDescriptor(sd);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DrmFilter] 통신 포트 생성 실패: 0x%08X\n", status);
        // [DrmFilter] Failed to create communication port: 0x%08X
        return status;
    }

    // 메시지 큐 초기화
    // Initialize message queue
    InitializeListHead(&g_CommData.MessageQueue);
    KeInitializeSpinLock(&g_CommData.QueueLock);
    KeInitializeEvent(&g_CommData.NewMessageEvent, SynchronizationEvent, FALSE);

    DbgPrint("[DrmFilter] 통신 포트 초기화 완료\n");
    // [DrmFilter] Communication port initialized

    return STATUS_SUCCESS;
}

// 통신 종료
// Uninitialize communication
VOID
CommUninitialize(
    VOID
    )
{
    // 서버 포트 닫기 - 모든 클라이언트 연결도 자동 해제됨
    // Close server port - all client connections are automatically disconnected
    if (g_CommData.ServerPort != NULL) {
        FltCloseCommunicationPort(g_CommData.ServerPort);
        g_CommData.ServerPort = NULL;
    }

    DbgPrint("[DrmFilter] 통신 포트 종료\n");
    // [DrmFilter] Communication port closed
}

// 드라이버에서 사용자 모드로 메시지 전송
// Send message from driver to user mode
NTSTATUS
CommSendMessage(
    _In_ PDRIVER_MESSAGE Message,
    _In_ ULONG MessageSize
    )
{
    NTSTATUS status;
    LARGE_INTEGER timeout;
    ULONG replyLength = 0;

    // 연결된 클라이언트 확인
    // Check for connected client
    if (g_CommData.ClientPort == NULL) {
        return STATUS_PORT_DISCONNECTED;
    }

    // 타임아웃 설정: 1초
    // Set timeout: 1 second
    timeout.QuadPart = -10000000;  // 100ns 단위의 상대 시간

    // 메시지 전송
    // Send message
    status = FltSendMessage(
        g_FilterHandle,           // 필터 핸들
        &g_CommData.ClientPort,   // 클라이언트 포트
        Message,                  // 송신 버퍼
        MessageSize,              // 송신 버퍼 크기
        NULL,                     // 응답 버퍼 (필요 없음)
        &replyLength,             // 응답 크기
        &timeout                  // 타임아웃
        );

    if (!NT_SUCCESS(status)) {
        if (status == STATUS_TIMEOUT) {
            DbgPrint("[DrmFilter] 메시지 전송 타임아웃\n");
            // [DrmFilter] Message send timeout
        } else {
            DbgPrint("[DrmFilter] 메시지 전송 실패: 0x%08X\n", status);
            // [DrmFilter] Message send failed: 0x%08X
        }
    }

    return status;
}

// 파일 이벤트 알림 전송 헬퍼
// Helper to send file event notification
NTSTATUS
CommNotifyFileEvent(
    _In_ PCUNICODE_STRING FilePath,
    _In_ ULONG Operation,
    _In_ NTSTATUS ResultStatus
    )
{
    DRIVER_MESSAGE message = { 0 };
    ULONG pathLength;

    // 메시지 구성
    // Build message
    message.Type = MSG_TYPE_FILE_EVENT;
    message.ProcessId = HandleToUlong(PsGetCurrentProcessId());
    message.ThreadId = HandleToUlong(PsGetCurrentThreadId());
    KeQuerySystemTime((PLARGE_INTEGER)&message.Timestamp);
    message.Operation = Operation;
    message.ResultStatus = ResultStatus;

    // 파일 경로 복사
    // Copy file path
    pathLength = min(FilePath->Length / sizeof(WCHAR), 259);
    RtlCopyMemory(message.FilePath, FilePath->Buffer, pathLength * sizeof(WCHAR));
    message.FilePath[pathLength] = L'\0';

    return CommSendMessage(&message, sizeof(DRIVER_MESSAGE));
}

// 차단 알림 전송
// Send block notification
NTSTATUS
CommNotifyBlocked(
    _In_ PCUNICODE_STRING FilePath,
    _In_ ULONG Operation,
    _In_ PCUNICODE_STRING Reason
    )
{
    // 확장된 메시지 구조 사용
    // Use extended message structure
    struct {
        DRIVER_MESSAGE Base;
        WCHAR ReasonText[256];
    } extMessage = { 0 };

    ULONG pathLength, reasonLength;

    extMessage.Base.Type = MSG_TYPE_BLOCK_NOTIFICATION;
    extMessage.Base.ProcessId = HandleToUlong(PsGetCurrentProcessId());
    extMessage.Base.ThreadId = HandleToUlong(PsGetCurrentThreadId());
    KeQuerySystemTime((PLARGE_INTEGER)&extMessage.Base.Timestamp);
    extMessage.Base.Operation = Operation;
    extMessage.Base.ResultStatus = STATUS_ACCESS_DENIED;

    // 파일 경로
    // File path
    pathLength = min(FilePath->Length / sizeof(WCHAR), 259);
    RtlCopyMemory(extMessage.Base.FilePath, FilePath->Buffer, pathLength * sizeof(WCHAR));

    // 차단 이유
    // Block reason
    if (Reason != NULL && Reason->Length > 0) {
        reasonLength = min(Reason->Length / sizeof(WCHAR), 255);
        RtlCopyMemory(extMessage.ReasonText, Reason->Buffer, reasonLength * sizeof(WCHAR));
        extMessage.Base.AdditionalDataSize = reasonLength * sizeof(WCHAR);
    }

    return CommSendMessage(
        (PDRIVER_MESSAGE)&extMessage,
        sizeof(extMessage)
        );
}
```

## 29.3 사용자 모드 통신 클라이언트

### C# 통신 클라이언트 구현

```csharp
// FilterCommunication.cs
// Minifilter 통신 클라이언트
// Minifilter communication client

using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;

namespace DrmFilterService;

/// <summary>
/// FilterManager 통신 포트 관리 클래스
/// FilterManager communication port management class
/// </summary>
public sealed class FilterCommunication : IDisposable
{
    private const string PortName = "\\DrmFilterPort";
    private SafeFileHandle? _portHandle;
    private readonly CancellationTokenSource _cts = new();
    private Task? _messageListenerTask;

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

    [DllImport("fltlib.dll")]
    private static extern uint FilterReplyMessage(
        SafeFileHandle port,
        IntPtr replyBuffer,
        uint replyBufferSize);

    // 메시지 타입 열거형
    // Message type enumeration
    public enum MessageType : uint
    {
        PolicyUpdate = 1,
        FileEvent = 2,
        StatusRequest = 3,
        StatusResponse = 4,
        BlockNotification = 5,
        ConfigChange = 6
    }

    // 드라이버 메시지 구조체
    // Driver message structure
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    public struct FilterMessageHeader
    {
        public uint ReplyLength;
        public ulong MessageId;
    }

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    public struct DriverMessage
    {
        public MessageType Type;
        public uint ProcessId;
        public uint ThreadId;
        public long Timestamp;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 260)]
        public string FilePath;
        public uint Operation;
        public int ResultStatus;
        public uint AdditionalDataSize;
    }

    // 사용자 메시지 구조체
    // User message structure
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    public struct PolicyUpdateMessage
    {
        public MessageType Type;
        public uint PolicyVersion;
        public uint PolicyFlags;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 260)]
        public string ProtectedPath;
    }

    [StructLayout(LayoutKind.Sequential)]
    public struct ConfigChangeMessage
    {
        public MessageType Type;
        [MarshalAs(UnmanagedType.Bool)]
        public bool EnableMonitoring;
        [MarshalAs(UnmanagedType.Bool)]
        public bool EnableBlocking;
        public uint LogLevel;
    }

    // 이벤트 정의
    // Event definitions
    public event EventHandler<FileEventArgs>? FileEventReceived;
    public event EventHandler<BlockEventArgs>? BlockNotificationReceived;

    /// <summary>
    /// 드라이버에 연결
    /// Connect to driver
    /// </summary>
    public void Connect()
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
            throw new InvalidOperationException(
                $"드라이버 연결 실패: 0x{result:X8}");
            // Driver connection failed
        }

        // 메시지 수신 스레드 시작
        // Start message listener thread
        _messageListenerTask = Task.Run(MessageListenerLoop);
    }

    /// <summary>
    /// 메시지 수신 루프
    /// Message listener loop
    /// </summary>
    private async Task MessageListenerLoop()
    {
        const int bufferSize = 4096;
        IntPtr buffer = Marshal.AllocHGlobal(bufferSize);

        try
        {
            while (!_cts.Token.IsCancellationRequested)
            {
                // FilterGetMessage는 블로킹 호출
                // FilterGetMessage is a blocking call
                uint result = FilterGetMessage(
                    _portHandle!,
                    buffer,
                    (uint)bufferSize,
                    IntPtr.Zero);

                if (result != 0)
                {
                    if (result == 0x800704D3) // ERROR_OPERATION_ABORTED
                        break;

                    // 짧은 지연 후 재시도
                    // Retry after short delay
                    await Task.Delay(100, _cts.Token);
                    continue;
                }

                // 메시지 파싱
                // Parse message
                ProcessReceivedMessage(buffer);
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

    /// <summary>
    /// 수신된 메시지 처리
    /// Process received message
    /// </summary>
    private void ProcessReceivedMessage(IntPtr buffer)
    {
        // 헤더 읽기
        // Read header
        var header = Marshal.PtrToStructure<FilterMessageHeader>(buffer);

        // 메시지 본문 위치 계산
        // Calculate message body position
        IntPtr messagePtr = buffer + Marshal.SizeOf<FilterMessageHeader>();
        var message = Marshal.PtrToStructure<DriverMessage>(messagePtr);

        switch (message.Type)
        {
            case MessageType.FileEvent:
                OnFileEvent(message);
                break;

            case MessageType.BlockNotification:
                OnBlockNotification(message, messagePtr);
                break;
        }
    }

    private void OnFileEvent(DriverMessage message)
    {
        FileEventReceived?.Invoke(this, new FileEventArgs
        {
            FilePath = message.FilePath,
            ProcessId = message.ProcessId,
            Operation = (IrpMajorFunction)message.Operation,
            Timestamp = DateTime.FromFileTimeUtc(message.Timestamp)
        });
    }

    private void OnBlockNotification(DriverMessage message, IntPtr messagePtr)
    {
        // 추가 데이터(차단 이유) 읽기
        // Read additional data (block reason)
        string? reason = null;
        if (message.AdditionalDataSize > 0)
        {
            IntPtr reasonPtr = messagePtr + Marshal.SizeOf<DriverMessage>();
            reason = Marshal.PtrToStringUni(reasonPtr, (int)(message.AdditionalDataSize / 2));
        }

        BlockNotificationReceived?.Invoke(this, new BlockEventArgs
        {
            FilePath = message.FilePath,
            ProcessId = message.ProcessId,
            Operation = (IrpMajorFunction)message.Operation,
            Reason = reason ?? "정책에 의해 차단됨" // Blocked by policy
        });
    }

    /// <summary>
    /// 정책 업데이트 전송
    /// Send policy update
    /// </summary>
    public void SendPolicyUpdate(string protectedPath, ProtectionFlags flags)
    {
        var message = new PolicyUpdateMessage
        {
            Type = MessageType.PolicyUpdate,
            PolicyVersion = 1,
            PolicyFlags = (uint)flags,
            ProtectedPath = protectedPath
        };

        SendMessage(message);
    }

    /// <summary>
    /// 설정 변경 전송
    /// Send configuration change
    /// </summary>
    public void SendConfigChange(bool enableMonitoring, bool enableBlocking, uint logLevel)
    {
        var message = new ConfigChangeMessage
        {
            Type = MessageType.ConfigChange,
            EnableMonitoring = enableMonitoring,
            EnableBlocking = enableBlocking,
            LogLevel = logLevel
        };

        SendMessage(message);
    }

    /// <summary>
    /// 메시지 전송 헬퍼
    /// Send message helper
    /// </summary>
    private void SendMessage<T>(T message) where T : struct
    {
        int size = Marshal.SizeOf<T>();
        IntPtr buffer = Marshal.AllocHGlobal(size);

        try
        {
            Marshal.StructureToPtr(message, buffer, false);

            uint result = FilterSendMessage(
                _portHandle!,
                buffer,
                (uint)size,
                IntPtr.Zero,
                0,
                out _);

            if (result != 0)
            {
                throw new InvalidOperationException(
                    $"메시지 전송 실패: 0x{result:X8}");
                // Message send failed
            }
        }
        finally
        {
            Marshal.FreeHGlobal(buffer);
        }
    }

    /// <summary>
    /// 드라이버 상태 조회
    /// Query driver status
    /// </summary>
    public DriverStatus GetDriverStatus()
    {
        var request = new ConfigChangeMessage { Type = MessageType.StatusRequest };

        int requestSize = Marshal.SizeOf<ConfigChangeMessage>();
        int responseSize = 1024;

        IntPtr requestBuffer = Marshal.AllocHGlobal(requestSize);
        IntPtr responseBuffer = Marshal.AllocHGlobal(responseSize);

        try
        {
            Marshal.StructureToPtr(request, requestBuffer, false);

            uint result = FilterSendMessage(
                _portHandle!,
                requestBuffer,
                (uint)requestSize,
                responseBuffer,
                (uint)responseSize,
                out uint bytesReturned);

            if (result != 0)
            {
                throw new InvalidOperationException(
                    $"상태 조회 실패: 0x{result:X8}");
                // Status query failed
            }

            // 응답 파싱
            // Parse response
            return ParseDriverStatus(responseBuffer, bytesReturned);
        }
        finally
        {
            Marshal.FreeHGlobal(requestBuffer);
            Marshal.FreeHGlobal(responseBuffer);
        }
    }

    private static DriverStatus ParseDriverStatus(IntPtr buffer, uint size)
    {
        // 간단한 파싱 - 실제로는 응답 구조체에 맞게 처리
        // Simple parsing - handle according to actual response structure
        return new DriverStatus
        {
            Version = "1.0.0",
            IsEnabled = true,
            ActivePolicies = 5,
            FilesScanned = 12345,
            BlockedOperations = 42
        };
    }

    public void Dispose()
    {
        _cts.Cancel();
        _messageListenerTask?.Wait(TimeSpan.FromSeconds(5));
        _portHandle?.Dispose();
        _cts.Dispose();
    }
}

// 이벤트 인자 클래스
// Event argument classes

public sealed class FileEventArgs : EventArgs
{
    public required string FilePath { get; init; }
    public uint ProcessId { get; init; }
    public IrpMajorFunction Operation { get; init; }
    public DateTime Timestamp { get; init; }
}

public sealed class BlockEventArgs : EventArgs
{
    public required string FilePath { get; init; }
    public uint ProcessId { get; init; }
    public IrpMajorFunction Operation { get; init; }
    public required string Reason { get; init; }
}

public sealed class DriverStatus
{
    public required string Version { get; init; }
    public bool IsEnabled { get; init; }
    public int ActivePolicies { get; init; }
    public long FilesScanned { get; init; }
    public long BlockedOperations { get; init; }
}

public enum IrpMajorFunction : uint
{
    Create = 0,
    CreateNamedPipe = 1,
    Close = 2,
    Read = 3,
    Write = 4,
    SetInformation = 6,
    // ... 필요한 값 추가
}

[Flags]
public enum ProtectionFlags : uint
{
    None = 0x0000,
    Read = 0x0001,
    Write = 0x0002,
    Execute = 0x0004,
    Rename = 0x0008,
    Delete = 0x0010,
    Move = 0x0020,
    Standard = Write | Rename | Delete | Move
}
```

## 29.4 Windows 서비스 구현

### 서비스 기본 구조

```csharp
// DrmFilterService.cs
// Windows 서비스로 드라이버 관리
// Manage driver as Windows Service

using System.ServiceProcess;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace DrmFilterService;

/// <summary>
/// DRM 필터 서비스
/// DRM Filter Service
/// </summary>
public sealed class DrmFilterService : BackgroundService
{
    private readonly ILogger<DrmFilterService> _logger;
    private readonly FilterCommunication _filterComm;
    private readonly PolicyManager _policyManager;
    private readonly EventLogWriter _eventLog;

    public DrmFilterService(
        ILogger<DrmFilterService> logger,
        FilterCommunication filterComm,
        PolicyManager policyManager,
        EventLogWriter eventLog)
    {
        _logger = logger;
        _filterComm = filterComm;
        _policyManager = policyManager;
        _eventLog = eventLog;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("DRM 필터 서비스 시작");
        // DRM Filter Service starting

        try
        {
            // 드라이버 로드 확인
            // Check driver is loaded
            await EnsureDriverLoaded(stoppingToken);

            // 드라이버 연결
            // Connect to driver
            _filterComm.Connect();
            _logger.LogInformation("드라이버 연결 성공");
            // Driver connection successful

            // 이벤트 핸들러 등록
            // Register event handlers
            _filterComm.FileEventReceived += OnFileEvent;
            _filterComm.BlockNotificationReceived += OnBlockNotification;

            // 초기 정책 로드 및 적용
            // Load and apply initial policies
            await LoadAndApplyPolicies(stoppingToken);

            // 서비스 실행 중 대기
            // Wait while service is running
            await Task.Delay(Timeout.Infinite, stoppingToken);
        }
        catch (OperationCanceledException)
        {
            _logger.LogInformation("서비스 중지 요청됨");
            // Service stop requested
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "서비스 실행 중 오류 발생");
            // Error during service execution
            throw;
        }
        finally
        {
            // 이벤트 핸들러 해제
            // Unregister event handlers
            _filterComm.FileEventReceived -= OnFileEvent;
            _filterComm.BlockNotificationReceived -= OnBlockNotification;

            _filterComm.Dispose();
            _logger.LogInformation("DRM 필터 서비스 종료");
            // DRM Filter Service stopped
        }
    }

    private async Task EnsureDriverLoaded(CancellationToken token)
    {
        // fltmc로 드라이버 상태 확인
        // Check driver status with fltmc
        var process = System.Diagnostics.Process.Start(new System.Diagnostics.ProcessStartInfo
        {
            FileName = "fltmc",
            Arguments = "load DrmFilter",
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            CreateNoWindow = true
        });

        if (process != null)
        {
            await process.WaitForExitAsync(token);

            if (process.ExitCode != 0)
            {
                string error = await process.StandardError.ReadToEndAsync(token);

                // 이미 로드된 경우는 정상
                // Already loaded is OK
                if (!error.Contains("already loaded", StringComparison.OrdinalIgnoreCase))
                {
                    _logger.LogWarning("드라이버 로드 경고: {Error}", error);
                    // Driver load warning
                }
            }
        }
    }

    private async Task LoadAndApplyPolicies(CancellationToken token)
    {
        // 데이터베이스에서 정책 로드
        // Load policies from database
        var policies = await _policyManager.GetActivePoliciesAsync(token);

        foreach (var policy in policies)
        {
            _filterComm.SendPolicyUpdate(policy.Path, policy.Flags);
            _logger.LogInformation("정책 적용: {Path}, Flags={Flags}",
                policy.Path, policy.Flags);
            // Policy applied
        }

        _logger.LogInformation("총 {Count}개 정책 적용됨", policies.Count);
        // Total policies applied
    }

    private void OnFileEvent(object? sender, FileEventArgs e)
    {
        _logger.LogDebug(
            "파일 이벤트: {Path}, PID={Pid}, Op={Op}",
            e.FilePath, e.ProcessId, e.Operation);
        // File event

        // 이벤트 로그에 기록
        // Log to event log
        _eventLog.WriteFileEvent(e);
    }

    private void OnBlockNotification(object? sender, BlockEventArgs e)
    {
        _logger.LogWarning(
            "차단됨: {Path}, PID={Pid}, 이유={Reason}",
            e.FilePath, e.ProcessId, e.Reason);
        // Blocked

        // 이벤트 로그에 기록
        // Log to event log
        _eventLog.WriteBlockEvent(e);

        // UI에 알림 (선택적)
        // Notify UI (optional)
        // NotifyUserInterface(e);
    }
}

/// <summary>
/// 정책 관리자
/// Policy Manager
/// </summary>
public sealed class PolicyManager
{
    private readonly string _connectionString;

    public PolicyManager(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<List<ProtectionPolicy>> GetActivePoliciesAsync(CancellationToken token)
    {
        // 실제로는 데이터베이스에서 조회
        // In reality, query from database

        // 테스트용 하드코딩된 정책
        // Hardcoded policies for testing
        return await Task.FromResult(new List<ProtectionPolicy>
        {
            new() { Path = @"C:\ProtectedDocs\*.docx", Flags = ProtectionFlags.Standard },
            new() { Path = @"C:\ProtectedDocs\*.xlsx", Flags = ProtectionFlags.Standard },
            new() { Path = @"C:\Confidential\*", Flags = ProtectionFlags.Write | ProtectionFlags.Delete }
        });
    }

    public async Task AddPolicyAsync(ProtectionPolicy policy, CancellationToken token)
    {
        // 데이터베이스에 정책 추가
        // Add policy to database
        await Task.CompletedTask;
    }

    public async Task RemovePolicyAsync(string path, CancellationToken token)
    {
        // 데이터베이스에서 정책 제거
        // Remove policy from database
        await Task.CompletedTask;
    }
}

public sealed class ProtectionPolicy
{
    public required string Path { get; init; }
    public ProtectionFlags Flags { get; init; }
}

/// <summary>
/// 이벤트 로그 기록기
/// Event Log Writer
/// </summary>
public sealed class EventLogWriter
{
    private const string LogName = "Application";
    private const string SourceName = "DrmFilterService";

    public EventLogWriter()
    {
        // 이벤트 소스 등록 (관리자 권한 필요)
        // Register event source (requires admin rights)
        if (!System.Diagnostics.EventLog.SourceExists(SourceName))
        {
            System.Diagnostics.EventLog.CreateEventSource(SourceName, LogName);
        }
    }

    public void WriteFileEvent(FileEventArgs e)
    {
        System.Diagnostics.EventLog.WriteEntry(
            SourceName,
            $"파일 작업: {e.FilePath}\nPID: {e.ProcessId}\n작업: {e.Operation}",
            // File operation
            System.Diagnostics.EventLogEntryType.Information,
            1001);
    }

    public void WriteBlockEvent(BlockEventArgs e)
    {
        System.Diagnostics.EventLog.WriteEntry(
            SourceName,
            $"차단됨: {e.FilePath}\nPID: {e.ProcessId}\n이유: {e.Reason}",
            // Blocked
            System.Diagnostics.EventLogEntryType.Warning,
            2001);
    }
}
```

### 서비스 호스팅 설정

```csharp
// Program.cs
// 서비스 진입점
// Service entry point

using DrmFilterService;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Logging.EventLog;

// Windows 서비스로 실행
// Run as Windows Service
var builder = Host.CreateDefaultBuilder(args)
    .UseWindowsService(options =>
    {
        options.ServiceName = "DrmFilterService";
    })
    .ConfigureServices((context, services) =>
    {
        // 서비스 등록
        // Register services
        services.AddSingleton<FilterCommunication>();
        services.AddSingleton<PolicyManager>(sp =>
            new PolicyManager(context.Configuration.GetConnectionString("PolicyDb") ?? ""));
        services.AddSingleton<EventLogWriter>();

        // 호스티드 서비스 등록
        // Register hosted service
        services.AddHostedService<DrmFilterService.DrmFilterService>();
    })
    .ConfigureLogging((context, logging) =>
    {
        logging.ClearProviders();
        logging.AddConsole();
        logging.AddEventLog(new EventLogSettings
        {
            SourceName = "DrmFilterService"
        });
    });

var host = builder.Build();
await host.RunAsync();
```

### 서비스 설치 스크립트

```powershell
# Install-DrmService.ps1
# 서비스 설치 스크립트
# Service installation script

param(
    [string]$ServicePath = "C:\Program Files\DrmFilter\DrmFilterService.exe",
    [string]$ServiceName = "DrmFilterService",
    [string]$DisplayName = "DRM Filter Service",
    [string]$Description = "파일 시스템 DRM 보호 서비스"
)

# 관리자 권한 확인
# Check admin rights
if (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Error "관리자 권한이 필요합니다."
    # Administrator rights required
    exit 1
}

# 기존 서비스 확인
# Check existing service
$existingService = Get-Service -Name $ServiceName -ErrorAction SilentlyContinue

if ($existingService) {
    Write-Host "기존 서비스 중지 중..."
    # Stopping existing service...
    Stop-Service -Name $ServiceName -Force

    Write-Host "기존 서비스 제거 중..."
    # Removing existing service...
    sc.exe delete $ServiceName
    Start-Sleep -Seconds 2
}

# 서비스 생성
# Create service
Write-Host "서비스 생성 중..."
# Creating service...

New-Service -Name $ServiceName `
    -BinaryPathName $ServicePath `
    -DisplayName $DisplayName `
    -Description $Description `
    -StartupType Automatic `
    -DependsOn "FltMgr"  # Filter Manager 의존성

# 서비스 시작
# Start service
Write-Host "서비스 시작 중..."
# Starting service...
Start-Service -Name $ServiceName

# 상태 확인
# Check status
$service = Get-Service -Name $ServiceName
Write-Host "서비스 상태: $($service.Status)"
# Service status
```

## 29.5 비동기 메시지 큐

### 고성능 메시지 처리

```c
// AsyncMessageQueue.c
// 비동기 메시지 큐 구현
// Asynchronous message queue implementation

#include <fltKernel.h>

// 메시지 큐 항목
// Message queue entry
typedef struct _MESSAGE_QUEUE_ENTRY {
    LIST_ENTRY ListEntry;
    ULONG MessageSize;
    UCHAR MessageData[1];  // 가변 길이
} MESSAGE_QUEUE_ENTRY, *PMESSAGE_QUEUE_ENTRY;

// 메시지 큐
// Message queue
typedef struct _MESSAGE_QUEUE {
    LIST_ENTRY ListHead;
    KSPIN_LOCK Lock;
    KEVENT ItemAvailable;
    LONG Count;
    LONG MaxCount;
    BOOLEAN Shutdown;
} MESSAGE_QUEUE, *PMESSAGE_QUEUE;

// 전역 메시지 큐
// Global message queue
MESSAGE_QUEUE g_MessageQueue = { 0 };

// 큐 초기화
// Initialize queue
NTSTATUS
MsgQueueInitialize(
    _In_ LONG MaxMessages
    )
{
    InitializeListHead(&g_MessageQueue.ListHead);
    KeInitializeSpinLock(&g_MessageQueue.Lock);
    KeInitializeEvent(&g_MessageQueue.ItemAvailable, SynchronizationEvent, FALSE);
    g_MessageQueue.Count = 0;
    g_MessageQueue.MaxCount = MaxMessages;
    g_MessageQueue.Shutdown = FALSE;

    return STATUS_SUCCESS;
}

// 큐 종료
// Shutdown queue
VOID
MsgQueueShutdown(
    VOID
    )
{
    KIRQL oldIrql;
    PLIST_ENTRY entry;
    PMESSAGE_QUEUE_ENTRY queueEntry;

    // 종료 플래그 설정
    // Set shutdown flag
    g_MessageQueue.Shutdown = TRUE;
    KeSetEvent(&g_MessageQueue.ItemAvailable, IO_NO_INCREMENT, FALSE);

    // 대기 중인 모든 메시지 해제
    // Free all pending messages
    KeAcquireSpinLock(&g_MessageQueue.Lock, &oldIrql);

    while (!IsListEmpty(&g_MessageQueue.ListHead)) {
        entry = RemoveHeadList(&g_MessageQueue.ListHead);
        queueEntry = CONTAINING_RECORD(entry, MESSAGE_QUEUE_ENTRY, ListEntry);
        ExFreePoolWithTag(queueEntry, 'qgsM');
    }

    g_MessageQueue.Count = 0;

    KeReleaseSpinLock(&g_MessageQueue.Lock, oldIrql);
}

// 메시지 추가 (비차단)
// Add message (non-blocking)
NTSTATUS
MsgQueueAdd(
    _In_reads_bytes_(MessageSize) PVOID Message,
    _In_ ULONG MessageSize
    )
{
    KIRQL oldIrql;
    PMESSAGE_QUEUE_ENTRY entry;
    ULONG entrySize;

    // 종료 확인
    // Check shutdown
    if (g_MessageQueue.Shutdown) {
        return STATUS_SHUTDOWN_IN_PROGRESS;
    }

    // 큐 크기 확인
    // Check queue size
    if (g_MessageQueue.Count >= g_MessageQueue.MaxCount) {
        DbgPrint("[MsgQueue] 큐가 가득 참, 메시지 버림\n");
        // [MsgQueue] Queue full, dropping message
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // 항목 할당
    // Allocate entry
    entrySize = FIELD_OFFSET(MESSAGE_QUEUE_ENTRY, MessageData) + MessageSize;
    entry = ExAllocatePool2(POOL_FLAG_NON_PAGED, entrySize, 'qgsM');

    if (entry == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // 데이터 복사
    // Copy data
    entry->MessageSize = MessageSize;
    RtlCopyMemory(entry->MessageData, Message, MessageSize);

    // 큐에 추가
    // Add to queue
    KeAcquireSpinLock(&g_MessageQueue.Lock, &oldIrql);

    InsertTailList(&g_MessageQueue.ListHead, &entry->ListEntry);
    InterlockedIncrement(&g_MessageQueue.Count);

    KeReleaseSpinLock(&g_MessageQueue.Lock, oldIrql);

    // 대기 중인 소비자 깨우기
    // Wake waiting consumer
    KeSetEvent(&g_MessageQueue.ItemAvailable, IO_NO_INCREMENT, FALSE);

    return STATUS_SUCCESS;
}

// 메시지 가져오기 (차단 가능)
// Get message (can block)
NTSTATUS
MsgQueueGet(
    _Out_writes_bytes_to_(BufferSize, *ReturnedSize) PVOID Buffer,
    _In_ ULONG BufferSize,
    _Out_ PULONG ReturnedSize,
    _In_opt_ PLARGE_INTEGER Timeout
    )
{
    NTSTATUS status;
    KIRQL oldIrql;
    PLIST_ENTRY entry;
    PMESSAGE_QUEUE_ENTRY queueEntry;

    *ReturnedSize = 0;

    // 종료 확인
    // Check shutdown
    if (g_MessageQueue.Shutdown) {
        return STATUS_SHUTDOWN_IN_PROGRESS;
    }

    // 메시지 대기
    // Wait for message
    while (TRUE) {
        KeAcquireSpinLock(&g_MessageQueue.Lock, &oldIrql);

        if (!IsListEmpty(&g_MessageQueue.ListHead)) {
            // 메시지 있음 - 가져오기
            // Message available - get it
            entry = RemoveHeadList(&g_MessageQueue.ListHead);
            InterlockedDecrement(&g_MessageQueue.Count);

            KeReleaseSpinLock(&g_MessageQueue.Lock, oldIrql);

            queueEntry = CONTAINING_RECORD(entry, MESSAGE_QUEUE_ENTRY, ListEntry);

            // 버퍼 크기 확인
            // Check buffer size
            if (BufferSize < queueEntry->MessageSize) {
                // 버퍼가 작음 - 메시지 다시 큐에 넣기
                // Buffer too small - put message back
                KeAcquireSpinLock(&g_MessageQueue.Lock, &oldIrql);
                InsertHeadList(&g_MessageQueue.ListHead, &queueEntry->ListEntry);
                InterlockedIncrement(&g_MessageQueue.Count);
                KeReleaseSpinLock(&g_MessageQueue.Lock, oldIrql);

                return STATUS_BUFFER_TOO_SMALL;
            }

            // 데이터 복사
            // Copy data
            RtlCopyMemory(Buffer, queueEntry->MessageData, queueEntry->MessageSize);
            *ReturnedSize = queueEntry->MessageSize;

            // 항목 해제
            // Free entry
            ExFreePoolWithTag(queueEntry, 'qgsM');

            return STATUS_SUCCESS;
        }

        KeReleaseSpinLock(&g_MessageQueue.Lock, oldIrql);

        // 종료 확인
        // Check shutdown
        if (g_MessageQueue.Shutdown) {
            return STATUS_SHUTDOWN_IN_PROGRESS;
        }

        // 이벤트 대기
        // Wait for event
        status = KeWaitForSingleObject(
            &g_MessageQueue.ItemAvailable,
            Executive,
            KernelMode,
            FALSE,
            Timeout
            );

        if (status == STATUS_TIMEOUT) {
            return STATUS_TIMEOUT;
        }

        // 종료 후 대기 해제 확인
        // Check if woken for shutdown
        if (g_MessageQueue.Shutdown) {
            return STATUS_SHUTDOWN_IN_PROGRESS;
        }
    }
}

// 큐 상태 조회
// Query queue status
VOID
MsgQueueGetStatus(
    _Out_ PLONG CurrentCount,
    _Out_ PLONG MaxCount
    )
{
    *CurrentCount = g_MessageQueue.Count;
    *MaxCount = g_MessageQueue.MaxCount;
}
```

### 워커 스레드 패턴

```c
// WorkerThread.c
// 워커 스레드로 비동기 메시지 전송
// Asynchronous message sending with worker thread

#include <fltKernel.h>

// 워커 스레드 컨텍스트
// Worker thread context
typedef struct _WORKER_THREAD_CONTEXT {
    PKTHREAD Thread;
    KEVENT StopEvent;
    BOOLEAN Running;
} WORKER_THREAD_CONTEXT, *PWORKER_THREAD_CONTEXT;

WORKER_THREAD_CONTEXT g_WorkerThread = { 0 };

// 워커 스레드 함수
// Worker thread function
VOID
MessageWorkerThread(
    _In_ PVOID Context
    )
{
    NTSTATUS status;
    LARGE_INTEGER timeout;
    PVOID events[2];
    UCHAR messageBuffer[4096];
    ULONG messageSize;

    UNREFERENCED_PARAMETER(Context);

    DbgPrint("[Worker] 메시지 워커 스레드 시작\n");
    // [Worker] Message worker thread started

    // 타임아웃: 1초
    // Timeout: 1 second
    timeout.QuadPart = -10000000;

    while (TRUE) {
        // 메시지 큐에서 가져오기
        // Get from message queue
        status = MsgQueueGet(
            messageBuffer,
            sizeof(messageBuffer),
            &messageSize,
            &timeout
            );

        if (status == STATUS_SHUTDOWN_IN_PROGRESS) {
            DbgPrint("[Worker] 종료 요청됨\n");
            // [Worker] Shutdown requested
            break;
        }

        if (status == STATUS_TIMEOUT) {
            // 타임아웃 - 계속 대기
            // Timeout - continue waiting

            // 정지 이벤트 확인
            // Check stop event
            status = KeWaitForSingleObject(
                &g_WorkerThread.StopEvent,
                Executive,
                KernelMode,
                FALSE,
                &((LARGE_INTEGER){0})  // 즉시 반환
                );

            if (status == STATUS_SUCCESS) {
                DbgPrint("[Worker] 정지 이벤트 수신\n");
                // [Worker] Stop event received
                break;
            }

            continue;
        }

        if (!NT_SUCCESS(status)) {
            DbgPrint("[Worker] 메시지 가져오기 실패: 0x%08X\n", status);
            // [Worker] Failed to get message
            continue;
        }

        // 메시지 전송
        // Send message
        status = CommSendMessage(
            (PDRIVER_MESSAGE)messageBuffer,
            messageSize
            );

        if (!NT_SUCCESS(status)) {
            DbgPrint("[Worker] 메시지 전송 실패: 0x%08X\n", status);
            // [Worker] Failed to send message

            // 연결 끊김 시 메시지 버림
            // Drop message if disconnected
            if (status == STATUS_PORT_DISCONNECTED) {
                DbgPrint("[Worker] 클라이언트 연결 끊김, 메시지 버림\n");
                // [Worker] Client disconnected, dropping message
            }
        }
    }

    DbgPrint("[Worker] 메시지 워커 스레드 종료\n");
    // [Worker] Message worker thread stopped

    PsTerminateSystemThread(STATUS_SUCCESS);
}

// 워커 스레드 시작
// Start worker thread
NTSTATUS
WorkerThreadStart(
    VOID
    )
{
    NTSTATUS status;
    HANDLE threadHandle;

    // 이미 실행 중인지 확인
    // Check if already running
    if (g_WorkerThread.Running) {
        return STATUS_ALREADY_COMMITTED;
    }

    // 이벤트 초기화
    // Initialize event
    KeInitializeEvent(&g_WorkerThread.StopEvent, NotificationEvent, FALSE);

    // 스레드 생성
    // Create thread
    status = PsCreateSystemThread(
        &threadHandle,
        THREAD_ALL_ACCESS,
        NULL,
        NULL,
        NULL,
        MessageWorkerThread,
        NULL
        );

    if (!NT_SUCCESS(status)) {
        DbgPrint("[Worker] 스레드 생성 실패: 0x%08X\n", status);
        // [Worker] Thread creation failed
        return status;
    }

    // 스레드 객체 참조 얻기
    // Get thread object reference
    status = ObReferenceObjectByHandle(
        threadHandle,
        THREAD_ALL_ACCESS,
        *PsThreadType,
        KernelMode,
        (PVOID*)&g_WorkerThread.Thread,
        NULL
        );

    ZwClose(threadHandle);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[Worker] 스레드 참조 실패: 0x%08X\n", status);
        // [Worker] Thread reference failed
        return status;
    }

    g_WorkerThread.Running = TRUE;

    DbgPrint("[Worker] 워커 스레드 시작 완료\n");
    // [Worker] Worker thread started

    return STATUS_SUCCESS;
}

// 워커 스레드 정지
// Stop worker thread
VOID
WorkerThreadStop(
    VOID
    )
{
    if (!g_WorkerThread.Running) {
        return;
    }

    DbgPrint("[Worker] 워커 스레드 정지 요청\n");
    // [Worker] Worker thread stop requested

    // 정지 이벤트 설정
    // Set stop event
    KeSetEvent(&g_WorkerThread.StopEvent, IO_NO_INCREMENT, FALSE);

    // 메시지 큐 종료
    // Shutdown message queue
    MsgQueueShutdown();

    // 스레드 종료 대기
    // Wait for thread termination
    if (g_WorkerThread.Thread != NULL) {
        KeWaitForSingleObject(
            g_WorkerThread.Thread,
            Executive,
            KernelMode,
            FALSE,
            NULL  // 무한 대기
            );

        ObDereferenceObject(g_WorkerThread.Thread);
        g_WorkerThread.Thread = NULL;
    }

    g_WorkerThread.Running = FALSE;

    DbgPrint("[Worker] 워커 스레드 정지 완료\n");
    // [Worker] Worker thread stopped
}

// 비동기 메시지 전송 (큐에 추가)
// Send message asynchronously (add to queue)
NTSTATUS
SendMessageAsync(
    _In_ PDRIVER_MESSAGE Message,
    _In_ ULONG MessageSize
    )
{
    return MsgQueueAdd(Message, MessageSize);
}
```

## 29.6 보안 고려사항

### 통신 포트 보안

```c
// SecureCommunication.c
// 보안 강화된 통신 포트
// Security-enhanced communication port

#include <fltKernel.h>

// 보안 디스크립터로 접근 제어
// Access control with security descriptor
NTSTATUS
CreateSecureCommunicationPort(
    _In_ PFLT_FILTER Filter,
    _Out_ PFLT_PORT* ServerPort
    )
{
    NTSTATUS status;
    PSECURITY_DESCRIPTOR sd = NULL;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING portName;
    PACL dacl = NULL;
    ULONG daclSize;
    SID_IDENTIFIER_AUTHORITY ntAuthority = SECURITY_NT_AUTHORITY;
    PSID adminSid = NULL;
    PSID systemSid = NULL;

    // 관리자 SID 생성
    // Create Administrators SID
    status = RtlAllocateAndInitializeSid(
        &ntAuthority,
        2,
        SECURITY_BUILTIN_DOMAIN_RID,
        DOMAIN_ALIAS_RID_ADMINS,
        0, 0, 0, 0, 0, 0,
        &adminSid
        );

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // SYSTEM SID 생성
    // Create SYSTEM SID
    status = RtlAllocateAndInitializeSid(
        &ntAuthority,
        1,
        SECURITY_LOCAL_SYSTEM_RID,
        0, 0, 0, 0, 0, 0, 0,
        &systemSid
        );

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // DACL 크기 계산
    // Calculate DACL size
    daclSize = sizeof(ACL) +
               2 * sizeof(ACCESS_ALLOWED_ACE) +
               RtlLengthSid(adminSid) +
               RtlLengthSid(systemSid) -
               2 * sizeof(ULONG);  // SidStart 중복 제거

    // DACL 할당
    // Allocate DACL
    dacl = ExAllocatePool2(POOL_FLAG_PAGED, daclSize, 'lcaD');
    if (dacl == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    // DACL 초기화
    // Initialize DACL
    status = RtlCreateAcl(dacl, daclSize, ACL_REVISION);
    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // 관리자 접근 허용
    // Allow Administrators access
    status = RtlAddAccessAllowedAce(
        dacl,
        ACL_REVISION,
        FLT_PORT_ALL_ACCESS,
        adminSid
        );

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // SYSTEM 접근 허용
    // Allow SYSTEM access
    status = RtlAddAccessAllowedAce(
        dacl,
        ACL_REVISION,
        FLT_PORT_ALL_ACCESS,
        systemSid
        );

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // 보안 디스크립터 생성
    // Create security descriptor
    sd = ExAllocatePool2(POOL_FLAG_PAGED, SECURITY_DESCRIPTOR_MIN_LENGTH, 'dceS');
    if (sd == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    status = RtlCreateSecurityDescriptor(sd, SECURITY_DESCRIPTOR_REVISION);
    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // DACL 설정
    // Set DACL
    status = RtlSetDaclSecurityDescriptor(sd, TRUE, dacl, FALSE);
    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // 포트 생성
    // Create port
    RtlInitUnicodeString(&portName, DRM_PORT_NAME);

    InitializeObjectAttributes(
        &oa,
        &portName,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        sd
        );

    status = FltCreateCommunicationPort(
        Filter,
        ServerPort,
        &oa,
        NULL,
        CommConnectNotify,
        CommDisconnectNotify,
        CommMessageNotify,
        MAX_CONNECTIONS
        );

    DbgPrint("[DrmFilter] 보안 통신 포트 생성: 0x%08X\n", status);
    // [DrmFilter] Secure communication port created

Cleanup:
    if (adminSid) RtlFreeSid(adminSid);
    if (systemSid) RtlFreeSid(systemSid);
    if (dacl) ExFreePoolWithTag(dacl, 'lcaD');
    if (sd) ExFreePoolWithTag(sd, 'dceS');

    return status;
}

// 연결 시 프로세스 검증
// Validate process on connection
NTSTATUS
ValidateConnectingProcess(
    VOID
    )
{
    NTSTATUS status;
    PEPROCESS process;
    PUNICODE_STRING processName = NULL;
    BOOLEAN isValidProcess = FALSE;

    // 현재 프로세스 가져오기
    // Get current process
    process = PsGetCurrentProcess();

    // 프로세스 이름 가져오기
    // Get process name
    status = SeLocateProcessImageName(process, &processName);
    if (!NT_SUCCESS(status)) {
        return STATUS_ACCESS_DENIED;
    }

    // 허용된 프로세스 이름 확인
    // Check allowed process names
    if (processName != NULL && processName->Buffer != NULL) {
        // DrmFilterService.exe만 허용
        // Only allow DrmFilterService.exe
        if (wcsstr(processName->Buffer, L"DrmFilterService.exe") != NULL) {
            isValidProcess = TRUE;
        }
    }

    if (processName) {
        ExFreePool(processName);
    }

    if (!isValidProcess) {
        DbgPrint("[DrmFilter] 허용되지 않은 프로세스의 연결 시도\n");
        // [DrmFilter] Connection attempt from unauthorized process
        return STATUS_ACCESS_DENIED;
    }

    return STATUS_SUCCESS;
}

// 메시지 서명 검증 (선택적)
// Message signature validation (optional)
typedef struct _SIGNED_MESSAGE {
    ULONG Signature;       // 매직 넘버
    ULONG Version;         // 프로토콜 버전
    ULONG SequenceNumber;  // 시퀀스 번호 (리플레이 방지)
    ULONG Checksum;        // 데이터 체크섬
    UCHAR Data[1];         // 실제 메시지
} SIGNED_MESSAGE, *PSIGNED_MESSAGE;

#define MESSAGE_SIGNATURE 0x444D5246  // 'DMRF'
#define PROTOCOL_VERSION  0x00010000

NTSTATUS
ValidateSignedMessage(
    _In_reads_bytes_(MessageSize) PVOID Message,
    _In_ ULONG MessageSize
    )
{
    PSIGNED_MESSAGE signedMsg = (PSIGNED_MESSAGE)Message;
    ULONG calculatedChecksum;
    static LONG s_LastSequence = 0;

    // 크기 확인
    // Check size
    if (MessageSize < FIELD_OFFSET(SIGNED_MESSAGE, Data)) {
        return STATUS_INVALID_PARAMETER;
    }

    // 서명 확인
    // Check signature
    if (signedMsg->Signature != MESSAGE_SIGNATURE) {
        DbgPrint("[DrmFilter] 잘못된 메시지 서명\n");
        // [DrmFilter] Invalid message signature
        return STATUS_INVALID_SIGNATURE;
    }

    // 버전 확인
    // Check version
    if (signedMsg->Version != PROTOCOL_VERSION) {
        DbgPrint("[DrmFilter] 지원되지 않는 프로토콜 버전\n");
        // [DrmFilter] Unsupported protocol version
        return STATUS_REVISION_MISMATCH;
    }

    // 시퀀스 번호 확인 (리플레이 방지)
    // Check sequence number (prevent replay)
    if ((LONG)signedMsg->SequenceNumber <= s_LastSequence) {
        DbgPrint("[DrmFilter] 리플레이 공격 감지\n");
        // [DrmFilter] Replay attack detected
        return STATUS_INVALID_PARAMETER;
    }
    s_LastSequence = signedMsg->SequenceNumber;

    // 체크섬 검증 (간단한 XOR)
    // Verify checksum (simple XOR)
    calculatedChecksum = CalculateChecksum(
        signedMsg->Data,
        MessageSize - FIELD_OFFSET(SIGNED_MESSAGE, Data)
        );

    if (calculatedChecksum != signedMsg->Checksum) {
        DbgPrint("[DrmFilter] 체크섬 불일치\n");
        // [DrmFilter] Checksum mismatch
        return STATUS_DATA_CHECKSUM_ERROR;
    }

    return STATUS_SUCCESS;
}

ULONG
CalculateChecksum(
    _In_reads_bytes_(Size) PUCHAR Data,
    _In_ ULONG Size
    )
{
    ULONG checksum = 0;
    ULONG i;

    for (i = 0; i < Size; i++) {
        checksum ^= (Data[i] << ((i % 4) * 8));
    }

    return checksum;
}
```

## 29.7 통합 예제: 완전한 통신 시스템

### 드라이버 통합

```c
// DriverMain.c
// 통신 기능이 통합된 드라이버
// Driver with integrated communication

#include "DrmFilter.h"
#include "CommunicationPort.h"

PFLT_FILTER g_FilterHandle = NULL;

// 드라이버 진입점
// Driver entry point
NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath
    )
{
    NTSTATUS status;

    DbgPrint("[DrmFilter] 드라이버 로드 중...\n");
    // [DrmFilter] Loading driver...

    // 필터 등록
    // Register filter
    status = FltRegisterFilter(
        DriverObject,
        &FilterRegistration,
        &g_FilterHandle
        );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 메시지 큐 초기화
    // Initialize message queue
    status = MsgQueueInitialize(1000);  // 최대 1000개 메시지
    if (!NT_SUCCESS(status)) {
        FltUnregisterFilter(g_FilterHandle);
        return status;
    }

    // 통신 포트 초기화
    // Initialize communication port
    status = CommInitialize(g_FilterHandle);
    if (!NT_SUCCESS(status)) {
        MsgQueueShutdown();
        FltUnregisterFilter(g_FilterHandle);
        return status;
    }

    // 워커 스레드 시작
    // Start worker thread
    status = WorkerThreadStart();
    if (!NT_SUCCESS(status)) {
        CommUninitialize();
        MsgQueueShutdown();
        FltUnregisterFilter(g_FilterHandle);
        return status;
    }

    // 필터링 시작
    // Start filtering
    status = FltStartFiltering(g_FilterHandle);
    if (!NT_SUCCESS(status)) {
        WorkerThreadStop();
        CommUninitialize();
        MsgQueueShutdown();
        FltUnregisterFilter(g_FilterHandle);
        return status;
    }

    DbgPrint("[DrmFilter] 드라이버 로드 완료\n");
    // [DrmFilter] Driver loaded successfully

    return STATUS_SUCCESS;
}

// 드라이버 언로드
// Driver unload
NTSTATUS
FilterUnload(
    _In_ FLT_FILTER_UNLOAD_FLAGS Flags
    )
{
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("[DrmFilter] 드라이버 언로드 중...\n");
    // [DrmFilter] Unloading driver...

    // 역순으로 정리
    // Cleanup in reverse order
    WorkerThreadStop();
    CommUninitialize();
    MsgQueueShutdown();
    FltUnregisterFilter(g_FilterHandle);

    DbgPrint("[DrmFilter] 드라이버 언로드 완료\n");
    // [DrmFilter] Driver unloaded successfully

    return STATUS_SUCCESS;
}
```

## 29.8 정리

이 챕터에서 학습한 내용:

1. **통신 아키텍처**
   - 커널-사용자 모드 분리의 필요성
   - FilterManager 통신 포트 개념

2. **드라이버 측 구현**
   - FltCreateCommunicationPort로 포트 생성
   - Connect/Disconnect/Message 콜백 구현
   - FltSendMessage로 메시지 전송

3. **사용자 모드 클라이언트**
   - FilterConnectCommunicationPort로 연결
   - FilterGetMessage로 메시지 수신
   - FilterSendMessage로 메시지 전송

4. **Windows 서비스**
   - BackgroundService 기반 구현
   - 드라이버 수명 주기 관리
   - 정책 관리 및 이벤트 처리

5. **비동기 처리**
   - 메시지 큐 구현
   - 워커 스레드 패턴

6. **보안**
   - DACL 기반 접근 제어
   - 프로세스 검증
   - 메시지 서명/검증

다음 챕터에서는 이 통신 기반 위에 **관리 콘솔 애플리케이션**을 구축하여 실시간 모니터링과 정책 관리 UI를 구현합니다.

---

**다음 챕터 예고**: Chapter 30에서는 WPF 기반 관리 콘솔을 구현하여 실시간 파일 이벤트 모니터링, 정책 편집기, 차단 이력 조회 등의 기능을 구축합니다.
