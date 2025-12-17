# 부록 E: 프로젝트 템플릿

## Minifilter 드라이버 프로젝트 시작 템플릿

이 부록은 새로운 Minifilter 프로젝트를 시작할 때 사용할 수 있는 완전한 템플릿을 제공합니다. 프로덕션 준비가 된 코드 구조와 모범 사례를 포함합니다.

---

## E.1 프로젝트 디렉터리 구조

```
MyMinifilter/
├── driver/                      # 드라이버 프로젝트
│   ├── MyMinifilter.vcxproj    # Visual Studio 프로젝트
│   ├── MyMinifilter.inf        # 설치 INF 파일
│   ├── source/                  # 소스 파일
│   │   ├── driver.c            # 드라이버 진입점
│   │   ├── driver.h            # 드라이버 헤더
│   │   ├── callbacks.c         # 콜백 구현
│   │   ├── callbacks.h         # 콜백 헤더
│   │   ├── context.c           # 컨텍스트 관리
│   │   ├── context.h           # 컨텍스트 헤더
│   │   ├── communication.c     # 통신 포트
│   │   ├── communication.h     # 통신 헤더
│   │   ├── utility.c           # 유틸리티 함수
│   │   └── utility.h           # 유틸리티 헤더
│   └── shared/                  # 공유 헤더
│       └── shared.h            # 커널/사용자 공유 정의
├── service/                     # Windows 서비스
│   └── MyService/
├── console/                     # 관리 콘솔
│   └── MyConsole/
├── test/                        # 테스트 프로젝트
│   └── MyMinifilterTest/
├── scripts/                     # 빌드/배포 스크립트
│   ├── build.ps1
│   ├── deploy.ps1
│   └── test.ps1
└── MyMinifilter.sln            # 솔루션 파일
```

---

## E.2 드라이버 헤더 (driver.h)

```c
/*++
Module Name:
    driver.h

Abstract:
    Main driver header file

Environment:
    Kernel mode only
--*/

#pragma once

// 표준 헤더
// Standard headers
#include <fltKernel.h>
#include <dontuse.h>
#include <suppress.h>
#include <ntstrsafe.h>

// 공유 헤더
// Shared header
#include "..\shared\shared.h"

// 드라이버 태그
// Driver tags
#define TAG_GENERAL     'NMYM'  // MyMN - My Minifilter General
#define TAG_CONTEXT     'CMYM'  // MyMC - My Minifilter Context
#define TAG_BUFFER      'BMYM'  // MyMB - My Minifilter Buffer
#define TAG_STRING      'SMYM'  // MyMS - My Minifilter String

// 드라이버 이름
// Driver name
#define DRIVER_NAME         L"MyMinifilter"
#define DRIVER_ALTITUDE     L"385400"
#define DRIVER_PORT_NAME    L"\\MyMinifilterPort"

// 전역 데이터 구조체
// Global data structure
typedef struct _DRIVER_DATA {
    // 필터 핸들
    // Filter handle
    PFLT_FILTER Filter;

    // 통신 포트
    // Communication port
    PFLT_PORT ServerPort;
    PFLT_PORT ClientPort;

    // 드라이버 상태
    // Driver state
    volatile LONG IsRunning;
    volatile LONG ClientConnected;

    // 통계
    // Statistics
    volatile LONG64 FilesScanned;
    volatile LONG64 FilesBlocked;
    volatile LONG64 BytesProcessed;

    // 설정
    // Configuration
    BOOLEAN EnableLogging;
    BOOLEAN EnableBlocking;
    ULONG MaxFileSizeToScan;

    // Lookaside List
    NPAGED_LOOKASIDE_LIST ContextLookaside;
    BOOLEAN LookasideInitialized;

    // 동기화
    // Synchronization
    ERESOURCE ConfigLock;
    BOOLEAN ConfigLockInitialized;

} DRIVER_DATA, *PDRIVER_DATA;

// 전역 변수
// Global variable
extern DRIVER_DATA gDriverData;

// 함수 선언
// Function declarations

// driver.c
DRIVER_INITIALIZE DriverEntry;
NTSTATUS FilterUnload(_In_ FLT_FILTER_UNLOAD_FLAGS Flags);
NTSTATUS InstanceSetup(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_SETUP_FLAGS Flags,
    _In_ DEVICE_TYPE VolumeDeviceType,
    _In_ FLT_FILESYSTEM_TYPE VolumeFilesystemType);
NTSTATUS InstanceQueryTeardown(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_QUERY_TEARDOWN_FLAGS Flags);

// callbacks.c
FLT_PREOP_CALLBACK_STATUS PreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext);
FLT_POSTOP_CALLBACK_STATUS PostCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags);
FLT_PREOP_CALLBACK_STATUS PreWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext);
FLT_PREOP_CALLBACK_STATUS PreSetInformation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext);

// context.c
NTSTATUS InitializeContexts(VOID);
VOID CleanupContexts(VOID);
NTSTATUS GetOrCreateStreamContext(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Out_ PSTREAM_CONTEXT *StreamContext,
    _Out_ PBOOLEAN IsNew);
VOID ContextCleanup(
    _In_ PFLT_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType);

// communication.c
NTSTATUS InitializeCommunication(VOID);
VOID CleanupCommunication(VOID);
NTSTATUS SendNotificationToUser(
    _In_ NOTIFICATION_TYPE Type,
    _In_ PCUNICODE_STRING FilePath,
    _In_ ULONG ProcessId);

// utility.c
BOOLEAN IsProtectedExtension(_In_ PCUNICODE_STRING Extension);
BOOLEAN IsSystemProcess(VOID);
NTSTATUS CopyUnicodeString(
    _Out_ PUNICODE_STRING Dest,
    _In_ PCUNICODE_STRING Src);
VOID FreeUnicodeString(_Inout_ PUNICODE_STRING Str);
```

---

## E.3 드라이버 진입점 (driver.c)

```c
/*++
Module Name:
    driver.c

Abstract:
    Main driver entry point and filter registration

Environment:
    Kernel mode only
--*/

#include "driver.h"
#include "callbacks.h"
#include "context.h"
#include "communication.h"
#include "utility.h"

// 전역 데이터
// Global data
DRIVER_DATA gDriverData = {0};

// 콜백 등록 배열
// Callback registration array
CONST FLT_OPERATION_REGISTRATION Callbacks[] = {
    {
        IRP_MJ_CREATE,
        0,
        PreCreate,
        PostCreate
    },
    {
        IRP_MJ_WRITE,
        FLTFL_OPERATION_REGISTRATION_SKIP_PAGING_IO,
        PreWrite,
        NULL
    },
    {
        IRP_MJ_SET_INFORMATION,
        0,
        PreSetInformation,
        NULL
    },
    { IRP_MJ_OPERATION_END }
};

// 컨텍스트 등록 배열
// Context registration array
CONST FLT_CONTEXT_REGISTRATION ContextRegistration[] = {
    {
        FLT_STREAM_CONTEXT,
        0,
        ContextCleanup,
        sizeof(STREAM_CONTEXT),
        TAG_CONTEXT
    },
    { FLT_CONTEXT_END }
};

// 필터 등록 구조체
// Filter registration structure
CONST FLT_REGISTRATION FilterRegistration = {
    sizeof(FLT_REGISTRATION),           // Size
    FLT_REGISTRATION_VERSION,           // Version
    0,                                  // Flags
    ContextRegistration,                // Context
    Callbacks,                          // Operations
    FilterUnload,                       // Unload
    InstanceSetup,                      // InstanceSetup
    InstanceQueryTeardown,              // InstanceQueryTeardown
    NULL,                               // InstanceTeardownStart
    NULL,                               // InstanceTeardownComplete
    NULL,                               // GenerateFileName
    NULL,                               // NormalizeNameComponent
    NULL,                               // NormalizeContextCleanup
    NULL,                               // TransactionNotification
    NULL                                // NormalizeNameComponentEx
};

/*++
Routine Description:
    Driver entry point

Arguments:
    DriverObject - Pointer to driver object
    RegistryPath - Driver registry path

Return Value:
    NTSTATUS
--*/
NTSTATUS DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;

    UNREFERENCED_PARAMETER(RegistryPath);

    DbgPrint("[%ws] DriverEntry: Loading driver...\n", DRIVER_NAME);

    // 전역 데이터 초기화
    // Initialize global data
    RtlZeroMemory(&gDriverData, sizeof(DRIVER_DATA));
    gDriverData.EnableLogging = TRUE;
    gDriverData.EnableBlocking = TRUE;
    gDriverData.MaxFileSizeToScan = 10 * 1024 * 1024;  // 10MB

    // 리소스 락 초기화
    // Initialize resource lock
    status = ExInitializeResourceLite(&gDriverData.ConfigLock);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[%ws] Failed to initialize config lock: 0x%X\n",
            DRIVER_NAME, status);
        return status;
    }
    gDriverData.ConfigLockInitialized = TRUE;

    // 컨텍스트 초기화
    // Initialize contexts
    status = InitializeContexts();
    if (!NT_SUCCESS(status)) {
        DbgPrint("[%ws] Failed to initialize contexts: 0x%X\n",
            DRIVER_NAME, status);
        goto Cleanup;
    }

    // 필터 등록
    // Register filter
    status = FltRegisterFilter(
        DriverObject,
        &FilterRegistration,
        &gDriverData.Filter);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[%ws] FltRegisterFilter failed: 0x%X\n",
            DRIVER_NAME, status);
        goto Cleanup;
    }

    // 통신 포트 초기화
    // Initialize communication port
    status = InitializeCommunication();
    if (!NT_SUCCESS(status)) {
        DbgPrint("[%ws] Failed to initialize communication: 0x%X\n",
            DRIVER_NAME, status);
        FltUnregisterFilter(gDriverData.Filter);
        goto Cleanup;
    }

    // 필터링 시작
    // Start filtering
    status = FltStartFiltering(gDriverData.Filter);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[%ws] FltStartFiltering failed: 0x%X\n",
            DRIVER_NAME, status);
        CleanupCommunication();
        FltUnregisterFilter(gDriverData.Filter);
        goto Cleanup;
    }

    gDriverData.IsRunning = TRUE;

    DbgPrint("[%ws] Driver loaded successfully\n", DRIVER_NAME);
    return STATUS_SUCCESS;

Cleanup:
    CleanupContexts();

    if (gDriverData.ConfigLockInitialized) {
        ExDeleteResourceLite(&gDriverData.ConfigLock);
        gDriverData.ConfigLockInitialized = FALSE;
    }

    return status;
}

/*++
Routine Description:
    Filter unload routine

Arguments:
    Flags - Unload flags

Return Value:
    NTSTATUS
--*/
NTSTATUS FilterUnload(_In_ FLT_FILTER_UNLOAD_FLAGS Flags)
{
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("[%ws] FilterUnload: Unloading driver...\n", DRIVER_NAME);

    gDriverData.IsRunning = FALSE;

    // 통신 정리
    // Cleanup communication
    CleanupCommunication();

    // 필터 해제
    // Unregister filter
    if (gDriverData.Filter != NULL) {
        FltUnregisterFilter(gDriverData.Filter);
        gDriverData.Filter = NULL;
    }

    // 컨텍스트 정리
    // Cleanup contexts
    CleanupContexts();

    // 리소스 락 정리
    // Cleanup resource lock
    if (gDriverData.ConfigLockInitialized) {
        ExDeleteResourceLite(&gDriverData.ConfigLock);
        gDriverData.ConfigLockInitialized = FALSE;
    }

    DbgPrint("[%ws] Unloaded. Stats: Scanned=%lld, Blocked=%lld\n",
        DRIVER_NAME,
        gDriverData.FilesScanned,
        gDriverData.FilesBlocked);

    return STATUS_SUCCESS;
}

/*++
Routine Description:
    Instance setup callback

Arguments:
    FltObjects - Pointer to FLT_RELATED_OBJECTS
    Flags - Instance setup flags
    VolumeDeviceType - Volume device type
    VolumeFilesystemType - Volume filesystem type

Return Value:
    NTSTATUS
--*/
NTSTATUS InstanceSetup(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_SETUP_FLAGS Flags,
    _In_ DEVICE_TYPE VolumeDeviceType,
    _In_ FLT_FILESYSTEM_TYPE VolumeFilesystemType)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("[%ws] InstanceSetup: DevType=%d, FsType=%d\n",
        DRIVER_NAME, VolumeDeviceType, VolumeFilesystemType);

    // 네트워크 파일 시스템 제외
    // Exclude network file systems
    if (VolumeDeviceType == FILE_DEVICE_NETWORK_FILE_SYSTEM) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    // RAW 파일 시스템 제외
    // Exclude RAW file system
    if (VolumeFilesystemType == FLT_FSTYPE_RAW) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    // CD/DVD 제외
    // Exclude CD/DVD
    if (VolumeDeviceType == FILE_DEVICE_CD_ROM_FILE_SYSTEM) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    return STATUS_SUCCESS;
}

/*++
Routine Description:
    Instance query teardown callback

Arguments:
    FltObjects - Pointer to FLT_RELATED_OBJECTS
    Flags - Query teardown flags

Return Value:
    NTSTATUS
--*/
NTSTATUS InstanceQueryTeardown(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_QUERY_TEARDOWN_FLAGS Flags)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    return STATUS_SUCCESS;
}
```

---

## E.4 콜백 구현 (callbacks.c)

```c
/*++
Module Name:
    callbacks.c

Abstract:
    Minifilter callback implementations

Environment:
    Kernel mode only
--*/

#include "driver.h"
#include "callbacks.h"
#include "context.h"
#include "communication.h"
#include "utility.h"

/*++
Routine Description:
    Pre-create callback

Arguments:
    Data - Callback data
    FltObjects - Related objects
    CompletionContext - Completion context

Return Value:
    FLT_PREOP_CALLBACK_STATUS
--*/
FLT_PREOP_CALLBACK_STATUS PreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    FLT_PREOP_CALLBACK_STATUS result = FLT_PREOP_SUCCESS_WITH_CALLBACK;

    UNREFERENCED_PARAMETER(CompletionContext);

    // 드라이버가 실행 중이 아니면 통과
    // Pass if driver is not running
    if (!gDriverData.IsRunning) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 커널 모드 요청 통과
    // Pass kernel mode requests
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 볼륨 열기 통과
    // Pass volume opens
    if (FlagOn(FltObjects->FileObject->Flags, FO_VOLUME_OPEN)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 디렉터리 통과
    // Pass directories
    if (FlagOn(Data->Iopb->Parameters.Create.Options, FILE_DIRECTORY_FILE)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 가져오기
    // Get file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo);

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    status = FltParseFileNameInformation(nameInfo);
    if (!NT_SUCCESS(status)) {
        FltReleaseFileNameInformation(nameInfo);
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 보호 대상 확장자 확인
    // Check protected extension
    if (IsProtectedExtension(&nameInfo->Extension)) {
        ACCESS_MASK desiredAccess =
            Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess;

        // 쓰기/삭제 접근 확인
        // Check write/delete access
        if (desiredAccess & (FILE_WRITE_DATA | FILE_APPEND_DATA | DELETE)) {
            // 시스템 프로세스 허용
            // Allow system processes
            if (!IsSystemProcess()) {
                // 블로킹 활성화 시 차단
                // Block if blocking is enabled
                if (gDriverData.EnableBlocking) {
                    InterlockedIncrement64(&gDriverData.FilesBlocked);

                    // 사용자에게 알림
                    // Notify user
                    SendNotificationToUser(
                        NOTIFICATION_BLOCKED,
                        &nameInfo->Name,
                        HandleToULong(PsGetCurrentProcessId()));

                    FltReleaseFileNameInformation(nameInfo);

                    Data->IoStatus.Status = STATUS_ACCESS_DENIED;
                    Data->IoStatus.Information = 0;
                    return FLT_PREOP_COMPLETE;
                }
            }
        }

        InterlockedIncrement64(&gDriverData.FilesScanned);
    }

    FltReleaseFileNameInformation(nameInfo);
    return result;
}

/*++
Routine Description:
    Post-create callback

Arguments:
    Data - Callback data
    FltObjects - Related objects
    CompletionContext - Completion context
    Flags - Post operation flags

Return Value:
    FLT_POSTOP_CALLBACK_STATUS
--*/
FLT_POSTOP_CALLBACK_STATUS PostCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    PSTREAM_CONTEXT streamCtx = NULL;
    BOOLEAN isNew = FALSE;

    UNREFERENCED_PARAMETER(CompletionContext);

    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 디렉터리 무시
    // Ignore directories
    if (FlagOn(Data->Iopb->Parameters.Create.Options, FILE_DIRECTORY_FILE)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 파일 이름 가져오기
    // Get file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo);

    if (!NT_SUCCESS(status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    FltParseFileNameInformation(nameInfo);

    // 보호 대상 확장자 확인
    // Check protected extension
    if (IsProtectedExtension(&nameInfo->Extension)) {
        // 스트림 컨텍스트 생성/가져오기
        // Create/get stream context
        status = GetOrCreateStreamContext(Data, FltObjects, &streamCtx, &isNew);
        if (NT_SUCCESS(status)) {
            if (isNew) {
                streamCtx->IsProtected = TRUE;
                CopyUnicodeString(&streamCtx->FileName, &nameInfo->Name);
            }
            FltReleaseContext(streamCtx);
        }
    }

    FltReleaseFileNameInformation(nameInfo);
    return FLT_POSTOP_FINISHED_PROCESSING;
}

/*++
Routine Description:
    Pre-write callback

Arguments:
    Data - Callback data
    FltObjects - Related objects
    CompletionContext - Completion context

Return Value:
    FLT_PREOP_CALLBACK_STATUS
--*/
FLT_PREOP_CALLBACK_STATUS PreWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    NTSTATUS status;
    PSTREAM_CONTEXT streamCtx = NULL;

    UNREFERENCED_PARAMETER(Data);
    UNREFERENCED_PARAMETER(CompletionContext);

    if (!gDriverData.IsRunning) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 스트림 컨텍스트 확인
    // Check stream context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&streamCtx);

    if (!NT_SUCCESS(status) || streamCtx == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    if (streamCtx->IsProtected) {
        // 수정 플래그 설정
        // Set modified flag
        streamCtx->WasModified = TRUE;

        InterlockedAdd64(
            &gDriverData.BytesProcessed,
            Data->Iopb->Parameters.Write.Length);
    }

    FltReleaseContext(streamCtx);
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

/*++
Routine Description:
    Pre-set information callback (for delete detection)

Arguments:
    Data - Callback data
    FltObjects - Related objects
    CompletionContext - Completion context

Return Value:
    FLT_PREOP_CALLBACK_STATUS
--*/
FLT_PREOP_CALLBACK_STATUS PreSetInformation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    NTSTATUS status;
    FILE_INFORMATION_CLASS infoClass;
    PSTREAM_CONTEXT streamCtx = NULL;

    UNREFERENCED_PARAMETER(CompletionContext);

    if (!gDriverData.IsRunning) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // 삭제 요청 확인
    // Check delete request
    if (infoClass != FileDispositionInformation &&
        infoClass != FileDispositionInformationEx) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 스트림 컨텍스트 확인
    // Check stream context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&streamCtx);

    if (!NT_SUCCESS(status) || streamCtx == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    if (streamCtx->IsProtected) {
        PFILE_DISPOSITION_INFORMATION dispInfo =
            (PFILE_DISPOSITION_INFORMATION)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        if (dispInfo->DeleteFile && gDriverData.EnableBlocking) {
            FltReleaseContext(streamCtx);

            InterlockedIncrement64(&gDriverData.FilesBlocked);

            Data->IoStatus.Status = STATUS_ACCESS_DENIED;
            return FLT_PREOP_COMPLETE;
        }
    }

    FltReleaseContext(streamCtx);
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## E.5 컨텍스트 관리 (context.c)

```c
/*++
Module Name:
    context.c

Abstract:
    Context management implementation

Environment:
    Kernel mode only
--*/

#include "driver.h"
#include "context.h"
#include "utility.h"

/*++
Routine Description:
    Initialize context management

Return Value:
    NTSTATUS
--*/
NTSTATUS InitializeContexts(VOID)
{
    ExInitializeNPagedLookasideList(
        &gDriverData.ContextLookaside,
        NULL,
        NULL,
        0,
        sizeof(STREAM_CONTEXT),
        TAG_CONTEXT,
        0);

    gDriverData.LookasideInitialized = TRUE;
    return STATUS_SUCCESS;
}

/*++
Routine Description:
    Cleanup context management
--*/
VOID CleanupContexts(VOID)
{
    if (gDriverData.LookasideInitialized) {
        ExDeleteNPagedLookasideList(&gDriverData.ContextLookaside);
        gDriverData.LookasideInitialized = FALSE;
    }
}

/*++
Routine Description:
    Get or create stream context

Arguments:
    Data - Callback data
    FltObjects - Related objects
    StreamContext - Returned context
    IsNew - Indicates if context was newly created

Return Value:
    NTSTATUS
--*/
NTSTATUS GetOrCreateStreamContext(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Out_ PSTREAM_CONTEXT *StreamContext,
    _Out_ PBOOLEAN IsNew)
{
    NTSTATUS status;
    PSTREAM_CONTEXT context = NULL;
    PSTREAM_CONTEXT existingContext = NULL;

    UNREFERENCED_PARAMETER(Data);

    *StreamContext = NULL;
    *IsNew = FALSE;

    // 기존 컨텍스트 확인
    // Check for existing context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&context);

    if (NT_SUCCESS(status)) {
        *StreamContext = context;
        return STATUS_SUCCESS;
    }

    if (status != STATUS_NOT_FOUND) {
        return status;
    }

    // 새 컨텍스트 할당
    // Allocate new context
    status = FltAllocateContext(
        FltObjects->Filter,
        FLT_STREAM_CONTEXT,
        sizeof(STREAM_CONTEXT),
        NonPagedPoolNx,
        (PFLT_CONTEXT*)&context);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 초기화
    // Initialize
    RtlZeroMemory(context, sizeof(STREAM_CONTEXT));
    context->IsProtected = FALSE;
    context->WasModified = FALSE;

    // 컨텍스트 설정
    // Set context
    status = FltSetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        FLT_SET_CONTEXT_KEEP_IF_EXISTS,
        context,
        (PFLT_CONTEXT*)&existingContext);

    if (status == STATUS_FLT_CONTEXT_ALREADY_DEFINED) {
        // 다른 스레드가 먼저 설정함
        // Another thread set it first
        FltReleaseContext(context);
        *StreamContext = existingContext;
        return STATUS_SUCCESS;
    }

    if (!NT_SUCCESS(status)) {
        FltReleaseContext(context);
        return status;
    }

    *StreamContext = context;
    *IsNew = TRUE;
    return STATUS_SUCCESS;
}

/*++
Routine Description:
    Context cleanup callback

Arguments:
    Context - Context to cleanup
    ContextType - Type of context

--*/
VOID ContextCleanup(
    _In_ PFLT_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType)
{
    PSTREAM_CONTEXT streamCtx;

    UNREFERENCED_PARAMETER(ContextType);

    streamCtx = (PSTREAM_CONTEXT)Context;

    if (streamCtx->FileName.Buffer != NULL) {
        FreeUnicodeString(&streamCtx->FileName);
    }
}
```

---

## E.6 통신 포트 (communication.c)

```c
/*++
Module Name:
    communication.c

Abstract:
    User mode communication implementation

Environment:
    Kernel mode only
--*/

#include "driver.h"
#include "communication.h"

// 전방 선언
// Forward declarations
NTSTATUS ClientConnect(
    _In_ PFLT_PORT ClientPort,
    _In_opt_ PVOID ServerPortCookie,
    _In_reads_bytes_opt_(SizeOfContext) PVOID ConnectionContext,
    _In_ ULONG SizeOfContext,
    _Outptr_result_maybenull_ PVOID *ConnectionPortCookie);

VOID ClientDisconnect(_In_opt_ PVOID ConnectionCookie);

NTSTATUS ClientMessage(
    _In_opt_ PVOID PortCookie,
    _In_reads_bytes_opt_(InputBufferLength) PVOID InputBuffer,
    _In_ ULONG InputBufferLength,
    _Out_writes_bytes_to_opt_(OutputBufferLength, *ReturnOutputBufferLength) PVOID OutputBuffer,
    _In_ ULONG OutputBufferLength,
    _Out_ PULONG ReturnOutputBufferLength);

/*++
Routine Description:
    Initialize communication port

Return Value:
    NTSTATUS
--*/
NTSTATUS InitializeCommunication(VOID)
{
    NTSTATUS status;
    UNICODE_STRING portName;
    OBJECT_ATTRIBUTES oa;
    PSECURITY_DESCRIPTOR sd = NULL;

    DbgPrint("[%ws] Creating communication port...\n", DRIVER_NAME);

    // 보안 설명자 생성
    // Create security descriptor
    status = FltBuildDefaultSecurityDescriptor(&sd, FLT_PORT_ALL_ACCESS);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[%ws] FltBuildDefaultSecurityDescriptor failed: 0x%X\n",
            DRIVER_NAME, status);
        return status;
    }

    RtlInitUnicodeString(&portName, DRIVER_PORT_NAME);

    InitializeObjectAttributes(
        &oa,
        &portName,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        sd);

    status = FltCreateCommunicationPort(
        gDriverData.Filter,
        &gDriverData.ServerPort,
        &oa,
        NULL,
        ClientConnect,
        ClientDisconnect,
        ClientMessage,
        1);  // 단일 클라이언트

    FltFreeSecurityDescriptor(sd);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[%ws] FltCreateCommunicationPort failed: 0x%X\n",
            DRIVER_NAME, status);
        return status;
    }

    DbgPrint("[%ws] Communication port created\n", DRIVER_NAME);
    return STATUS_SUCCESS;
}

/*++
Routine Description:
    Cleanup communication port
--*/
VOID CleanupCommunication(VOID)
{
    if (gDriverData.ServerPort != NULL) {
        FltCloseCommunicationPort(gDriverData.ServerPort);
        gDriverData.ServerPort = NULL;
    }
}

/*++
Routine Description:
    Client connect callback
--*/
NTSTATUS ClientConnect(
    _In_ PFLT_PORT ClientPort,
    _In_opt_ PVOID ServerPortCookie,
    _In_reads_bytes_opt_(SizeOfContext) PVOID ConnectionContext,
    _In_ ULONG SizeOfContext,
    _Outptr_result_maybenull_ PVOID *ConnectionPortCookie)
{
    UNREFERENCED_PARAMETER(ServerPortCookie);
    UNREFERENCED_PARAMETER(ConnectionContext);
    UNREFERENCED_PARAMETER(SizeOfContext);
    UNREFERENCED_PARAMETER(ConnectionPortCookie);

    DbgPrint("[%ws] Client connecting...\n", DRIVER_NAME);

    if (gDriverData.ClientPort != NULL) {
        DbgPrint("[%ws] Client already connected\n", DRIVER_NAME);
        return STATUS_ALREADY_REGISTERED;
    }

    gDriverData.ClientPort = ClientPort;
    gDriverData.ClientConnected = TRUE;

    DbgPrint("[%ws] Client connected\n", DRIVER_NAME);
    return STATUS_SUCCESS;
}

/*++
Routine Description:
    Client disconnect callback
--*/
VOID ClientDisconnect(_In_opt_ PVOID ConnectionCookie)
{
    UNREFERENCED_PARAMETER(ConnectionCookie);

    DbgPrint("[%ws] Client disconnecting...\n", DRIVER_NAME);

    FltCloseClientPort(gDriverData.Filter, &gDriverData.ClientPort);
    gDriverData.ClientPort = NULL;
    gDriverData.ClientConnected = FALSE;

    DbgPrint("[%ws] Client disconnected\n", DRIVER_NAME);
}

/*++
Routine Description:
    Client message callback
--*/
NTSTATUS ClientMessage(
    _In_opt_ PVOID PortCookie,
    _In_reads_bytes_opt_(InputBufferLength) PVOID InputBuffer,
    _In_ ULONG InputBufferLength,
    _Out_writes_bytes_to_opt_(OutputBufferLength, *ReturnOutputBufferLength) PVOID OutputBuffer,
    _In_ ULONG OutputBufferLength,
    _Out_ PULONG ReturnOutputBufferLength)
{
    PCOMMAND_MESSAGE cmd;
    PCOMMAND_RESPONSE response;

    UNREFERENCED_PARAMETER(PortCookie);

    if (InputBuffer == NULL || InputBufferLength < sizeof(COMMAND_MESSAGE)) {
        return STATUS_INVALID_PARAMETER;
    }

    cmd = (PCOMMAND_MESSAGE)InputBuffer;
    response = (PCOMMAND_RESPONSE)OutputBuffer;

    switch (cmd->Command) {
    case CMD_GET_STATS:
        if (OutputBufferLength >= sizeof(COMMAND_RESPONSE)) {
            response->Status = STATUS_SUCCESS;
            response->Stats.FilesScanned = gDriverData.FilesScanned;
            response->Stats.FilesBlocked = gDriverData.FilesBlocked;
            response->Stats.BytesProcessed = gDriverData.BytesProcessed;
            *ReturnOutputBufferLength = sizeof(COMMAND_RESPONSE);
        }
        break;

    case CMD_SET_CONFIG:
        ExAcquireResourceExclusiveLite(&gDriverData.ConfigLock, TRUE);
        gDriverData.EnableLogging = cmd->Config.EnableLogging;
        gDriverData.EnableBlocking = cmd->Config.EnableBlocking;
        ExReleaseResourceLite(&gDriverData.ConfigLock);

        if (OutputBufferLength >= sizeof(COMMAND_RESPONSE)) {
            response->Status = STATUS_SUCCESS;
            *ReturnOutputBufferLength = sizeof(COMMAND_RESPONSE);
        }
        break;

    default:
        return STATUS_INVALID_PARAMETER;
    }

    return STATUS_SUCCESS;
}

/*++
Routine Description:
    Send notification to user mode

Arguments:
    Type - Notification type
    FilePath - File path
    ProcessId - Process ID

Return Value:
    NTSTATUS
--*/
NTSTATUS SendNotificationToUser(
    _In_ NOTIFICATION_TYPE Type,
    _In_ PCUNICODE_STRING FilePath,
    _In_ ULONG ProcessId)
{
    NOTIFICATION_MESSAGE notification;
    LARGE_INTEGER timeout;

    if (gDriverData.ClientPort == NULL) {
        return STATUS_PORT_DISCONNECTED;
    }

    RtlZeroMemory(&notification, sizeof(notification));
    notification.Type = Type;
    notification.ProcessId = ProcessId;

    // 파일 경로 복사
    // Copy file path
    ULONG copyLength = min(
        FilePath->Length,
        sizeof(notification.FilePath) - sizeof(WCHAR));
    RtlCopyMemory(notification.FilePath, FilePath->Buffer, copyLength);

    // 타임아웃 설정 (100ms)
    // Set timeout (100ms)
    timeout.QuadPart = -1000000LL;

    return FltSendMessage(
        gDriverData.Filter,
        &gDriverData.ClientPort,
        &notification,
        sizeof(notification),
        NULL,
        NULL,
        &timeout);
}
```

---

## E.7 공유 헤더 (shared.h)

```c
/*++
Module Name:
    shared.h

Abstract:
    Shared definitions between kernel and user mode

Environment:
    Kernel and User mode
--*/

#pragma once

// 알림 유형
// Notification types
typedef enum _NOTIFICATION_TYPE {
    NOTIFICATION_BLOCKED = 1,
    NOTIFICATION_MODIFIED = 2,
    NOTIFICATION_DELETED = 3
} NOTIFICATION_TYPE;

// 명령 유형
// Command types
typedef enum _COMMAND_TYPE {
    CMD_GET_STATS = 1,
    CMD_SET_CONFIG = 2,
    CMD_GET_CONFIG = 3
} COMMAND_TYPE;

// 통계 구조체
// Statistics structure
typedef struct _DRIVER_STATS {
    LONG64 FilesScanned;
    LONG64 FilesBlocked;
    LONG64 BytesProcessed;
} DRIVER_STATS, *PDRIVER_STATS;

// 설정 구조체
// Configuration structure
typedef struct _DRIVER_CONFIG {
    BOOLEAN EnableLogging;
    BOOLEAN EnableBlocking;
    ULONG MaxFileSizeToScan;
} DRIVER_CONFIG, *PDRIVER_CONFIG;

// 명령 메시지
// Command message
typedef struct _COMMAND_MESSAGE {
    COMMAND_TYPE Command;
    union {
        DRIVER_CONFIG Config;
    };
} COMMAND_MESSAGE, *PCOMMAND_MESSAGE;

// 명령 응답
// Command response
typedef struct _COMMAND_RESPONSE {
    NTSTATUS Status;
    union {
        DRIVER_STATS Stats;
        DRIVER_CONFIG Config;
    };
} COMMAND_RESPONSE, *PCOMMAND_RESPONSE;

// 알림 메시지
// Notification message
typedef struct _NOTIFICATION_MESSAGE {
    NOTIFICATION_TYPE Type;
    ULONG ProcessId;
    WCHAR FilePath[260];
} NOTIFICATION_MESSAGE, *PNOTIFICATION_MESSAGE;

// 스트림 컨텍스트
// Stream context
typedef struct _STREAM_CONTEXT {
    BOOLEAN IsProtected;
    BOOLEAN WasModified;
    UNICODE_STRING FileName;
} STREAM_CONTEXT, *PSTREAM_CONTEXT;
```

---

## E.8 INF 파일 템플릿

```inf
;;;
;;; MyMinifilter
;;;

[Version]
Signature   = "$WINDOWS NT$"
Class       = "ActivityMonitor"
ClassGuid   = {b86dff51-a31e-4bac-b3cf-e8cfe75c9fc2}
Provider    = %ProviderName%
DriverVer   = 01/01/2025,1.0.0.0
CatalogFile = MyMinifilter.cat
PnpLockdown = 1

[SourceDisksFiles]
MyMinifilter.sys = 1,,

[SourceDisksNames]
1 = %DiskName%,,,

[DestinationDirs]
DefaultDestDir          = 12
MyMinifilter.DriverFiles = 12

[DefaultInstall.NTAMD64]
OptionDesc = %ServiceDescription%
CopyFiles  = MyMinifilter.DriverFiles

[DefaultInstall.NTAMD64.Services]
AddService = %ServiceName%,,MyMinifilter.Service

[DefaultUninstall.NTAMD64]
LegacyUninstall = 1
DelFiles        = MyMinifilter.DriverFiles

[DefaultUninstall.NTAMD64.Services]
DelService = %ServiceName%,0x200

[MyMinifilter.Service]
DisplayName    = %ServiceDisplayName%
Description    = %ServiceDescription%
ServiceBinary  = %12%\%DriverName%.sys
ServiceType    = 2
StartType      = 3
ErrorControl   = 1
LoadOrderGroup = "FSFilter Activity Monitor"
AddReg         = MyMinifilter.AddRegistry

[MyMinifilter.AddRegistry]
HKR,"Instances","DefaultInstance",0x00000000,%DefaultInstance%
HKR,"Instances\"%Instance1.Name%,"Altitude",0x00000000,%Instance1.Altitude%
HKR,"Instances\"%Instance1.Name%,"Flags",0x00010001,%Instance1.Flags%

[MyMinifilter.DriverFiles]
%DriverName%.sys

[Strings]
ProviderName        = "My Company"
ServiceName         = "MyMinifilter"
ServiceDisplayName  = "My Minifilter Driver"
ServiceDescription  = "File system minifilter driver"
DriverName          = "MyMinifilter"
DiskName            = "MyMinifilter Installation Disk"
DefaultInstance     = "MyMinifilter Instance"
Instance1.Name      = "MyMinifilter Instance"
Instance1.Altitude  = "385400"
Instance1.Flags     = 0x0
```

---

## E.9 빌드 스크립트 (build.ps1)

```powershell
# build.ps1 - 빌드 스크립트
# Build script

param(
    [ValidateSet('Debug', 'Release')]
    [string]$Configuration = 'Debug',

    [ValidateSet('x64', 'ARM64')]
    [string]$Platform = 'x64',

    [switch]$Clean,
    [switch]$Sign
)

$ErrorActionPreference = 'Stop'
$SolutionPath = Join-Path $PSScriptRoot '..\MyMinifilter.sln'

# Visual Studio 환경 설정
# Setup Visual Studio environment
$vsWhere = "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe"
$vsPath = & $vsWhere -latest -property installationPath
$msbuild = Join-Path $vsPath 'MSBuild\Current\Bin\MSBuild.exe'

if (-not (Test-Path $msbuild)) {
    throw "MSBuild not found"
}

# 클린 빌드
# Clean build
if ($Clean) {
    Write-Host "Cleaning solution..." -ForegroundColor Yellow
    & $msbuild $SolutionPath /t:Clean /p:Configuration=$Configuration /p:Platform=$Platform
}

# 빌드
# Build
Write-Host "Building $Configuration|$Platform..." -ForegroundColor Green
& $msbuild $SolutionPath /t:Build /p:Configuration=$Configuration /p:Platform=$Platform /v:minimal

if ($LASTEXITCODE -ne 0) {
    throw "Build failed"
}

# 서명 (선택적)
# Sign (optional)
if ($Sign) {
    $driverPath = Join-Path $PSScriptRoot "..\driver\$Platform\$Configuration\MyMinifilter.sys"

    if (Test-Path $driverPath) {
        Write-Host "Signing driver..." -ForegroundColor Yellow
        signtool sign /a /t http://timestamp.digicert.com /fd SHA256 $driverPath

        if ($LASTEXITCODE -ne 0) {
            throw "Signing failed"
        }
    }
}

Write-Host "Build completed successfully!" -ForegroundColor Green
```

---

## E.10 배포 스크립트 (deploy.ps1)

```powershell
# deploy.ps1 - 배포 스크립트
# Deployment script

param(
    [ValidateSet('Debug', 'Release')]
    [string]$Configuration = 'Debug',

    [string]$TargetComputer = 'localhost',
    [switch]$Uninstall
)

$ErrorActionPreference = 'Stop'
$DriverName = 'MyMinifilter'
$BasePath = Join-Path $PSScriptRoot '..\driver\x64\' $Configuration

if ($Uninstall) {
    Write-Host "Uninstalling $DriverName..." -ForegroundColor Yellow

    # 드라이버 분리
    # Detach driver
    fltmc detach $DriverName C: 2>$null

    # 드라이버 언로드
    # Unload driver
    fltmc unload $DriverName 2>$null

    # 서비스 삭제
    # Delete service
    sc.exe delete $DriverName 2>$null

    # 파일 삭제
    # Delete files
    $driverFile = "C:\Windows\System32\drivers\$DriverName.sys"
    if (Test-Path $driverFile) {
        Remove-Item $driverFile -Force
    }

    Write-Host "Uninstalled successfully" -ForegroundColor Green
    return
}

# 설치
# Install
Write-Host "Installing $DriverName..." -ForegroundColor Green

# 파일 복사
# Copy files
$sourceDriver = Join-Path $BasePath "$DriverName.sys"
$targetDriver = "C:\Windows\System32\drivers\$DriverName.sys"

if (-not (Test-Path $sourceDriver)) {
    throw "Driver file not found: $sourceDriver"
}

# 기존 드라이버 언로드
# Unload existing driver
fltmc unload $DriverName 2>$null

# 파일 복사
# Copy file
Copy-Item $sourceDriver $targetDriver -Force

# INF 설치
# Install INF
$infPath = Join-Path $BasePath "$DriverName.inf"
if (Test-Path $infPath) {
    pnputil /add-driver $infPath /install
}

# 드라이버 로드
# Load driver
fltmc load $DriverName

if ($LASTEXITCODE -ne 0) {
    throw "Failed to load driver"
}

# 인스턴스 확인
# Verify instances
fltmc instances -f $DriverName

Write-Host "Installation completed successfully!" -ForegroundColor Green
```

---

## 요약

이 프로젝트 템플릿은 프로덕션 준비가 된 Minifilter 드라이버를 개발하기 위한 완전한 시작점을 제공합니다:

1. **모듈화된 구조**: 기능별로 분리된 소스 파일
2. **적절한 오류 처리**: 모든 경로에서 리소스 정리
3. **성능 최적화**: 조기 반환, 캐시 사용
4. **사용자 통신**: 완전한 통신 포트 구현
5. **빌드/배포 자동화**: PowerShell 스크립트

이 템플릿을 기반으로 DRM 보호, 안티바이러스, 백업 등 다양한 파일 시스템 기능을 구현할 수 있습니다.
