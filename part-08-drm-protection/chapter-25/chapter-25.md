# Chapter 25: 파일 수정 차단 구현

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
- 파일 쓰기 작업의 완전한 이해와 차단 구현
- 예외 프로세스 처리 메커니즘 구축
- 차단 이벤트 로깅 시스템 구현

---

## 25.1 쓰기 작업 감지

### 25.1.1 파일 수정의 다양한 경로

Windows에서 파일을 수정하는 방법은 여러 가지가 있으며, 완전한 보호를 위해서는 모든 경로를 차단해야 합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    파일 수정 경로 분석                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐                                               │
│  │ 사용자 앱   │                                               │
│  │ User App    │                                               │
│  └──────┬──────┘                                               │
│         │                                                       │
│         v                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Win32 API Layer                       │   │
│  │  WriteFile()  WriteFileEx()  FlushFileBuffers()         │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             v                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    NT API Layer                          │   │
│  │  NtWriteFile()  NtSetInformationFile()                  │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             v                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               I/O Manager → Minifilter                   │   │
│  │                                                          │   │
│  │  ┌──────────────┐  ┌───────────────────┐  ┌───────────┐ │   │
│  │  │ IRP_MJ_WRITE │  │ IRP_MJ_SET_INFO   │  │ IRP_MJ_   │ │   │
│  │  │              │  │ (EndOfFile,       │  │ FILE_     │ │   │
│  │  │ 직접 쓰기    │  │  AllocationSize,  │  │ SYSTEM_   │ │   │
│  │  │ Direct write │  │  Disposition)     │  │ CONTROL   │ │   │
│  │  └──────────────┘  └───────────────────┘  └───────────┘ │   │
│  │                                                          │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │                                   │
│                             v                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  File System (NTFS)                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 25.1.2 IRP_MJ_WRITE 필터링

가장 기본적인 쓰기 차단 지점입니다.

```c
// write_filter.h - 쓰기 필터 정의
// write_filter.h - Write filter definitions

#pragma once
#include <fltKernel.h>
#include "protection_policy.h"

// 쓰기 필터 콜백 등록 구조체
// Write filter callback registration structure
extern const FLT_OPERATION_REGISTRATION WriteFilterCallbacks[];

// Pre-Write 콜백
// Pre-Write callback
FLT_PREOP_CALLBACK_STATUS
WriteFilterPreWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext);

// Post-Write 콜백
// Post-Write callback
FLT_POSTOP_CALLBACK_STATUS
WriteFilterPostWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags);
```

```c
// write_filter.c - 쓰기 필터 구현
// write_filter.c - Write filter implementation

#include "write_filter.h"
#include "stream_context.h"
#include "audit_log.h"
#include "process_filter.h"

// 콜백 등록
// Callback registration
const FLT_OPERATION_REGISTRATION WriteFilterCallbacks[] = {
    {
        IRP_MJ_WRITE,
        0,
        WriteFilterPreWrite,
        WriteFilterPostWrite
    },
    { IRP_MJ_OPERATION_END }
};

// 쓰기 유형 확인
// Check write type
typedef enum _WRITE_TYPE {
    WRITE_TYPE_NORMAL,          // 일반 쓰기
                                // Normal write
    WRITE_TYPE_PAGING,          // 페이징 I/O (캐시 매니저)
                                // Paging I/O (cache manager)
    WRITE_TYPE_CACHED,          // 캐시된 쓰기
                                // Cached write
    WRITE_TYPE_NON_CACHED       // 비캐시 쓰기
                                // Non-cached write
} WRITE_TYPE;

static WRITE_TYPE GetWriteType(_In_ PFLT_CALLBACK_DATA Data)
{
    PFLT_IO_PARAMETER_BLOCK iopb = Data->Iopb;

    if (FlagOn(iopb->IrpFlags, IRP_PAGING_IO)) {
        return WRITE_TYPE_PAGING;
    }

    if (FlagOn(iopb->IrpFlags, IRP_NOCACHE)) {
        return WRITE_TYPE_NON_CACHED;
    }

    if (FltObjects->FileObject->Flags & FO_CACHE_SUPPORTED) {
        return WRITE_TYPE_CACHED;
    }

    return WRITE_TYPE_NORMAL;
}

// Pre-Write 콜백 구현
// Pre-Write callback implementation
FLT_PREOP_CALLBACK_STATUS
WriteFilterPreWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PSTREAM_CONTEXT streamContext = NULL;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    BOOLEAN shouldBlock = FALSE;
    WRITE_TYPE writeType;

    *CompletionContext = NULL;

    // 1. IRQL 확인 - 높은 IRQL에서는 처리 제한
    // 1. Check IRQL - limited processing at high IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 2. 쓰기 유형 확인
    // 2. Check write type
    writeType = GetWriteType(Data);

    // 페이징 I/O는 캐시 매니저가 발생시킴 - 여기서 차단하면 시스템 불안정
    // Paging I/O is generated by cache manager - blocking here causes instability
    // 대신 원본 쓰기를 차단해야 함
    // Instead, block the original write
    if (writeType == WRITE_TYPE_PAGING) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 3. 스트림 컨텍스트 가져오기
    // 3. Get stream context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        &streamContext
    );

    if (!NT_SUCCESS(status)) {
        // 컨텍스트가 없으면 보호 대상 아님
        // No context means not a protected file
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    __try {
        // 4. 보호 플래그 확인
        // 4. Check protection flags
        if (!streamContext->IsProtected) {
            __leave;
        }

        if (!(streamContext->ProtectionFlags & PROTECT_WRITE)) {
            __leave;
        }

        // 5. 예외 프로세스 확인
        // 5. Check exempt processes
        if (IsExemptProcess(PsGetCurrentProcessId())) {
            DbgPrint("[Write] 예외 프로세스, 쓰기 허용\n");
            // Exempt process, allowing write
            __leave;
        }

        // 6. 차단 결정
        // 6. Decide to block
        shouldBlock = TRUE;

        // 7. 감사 로그 기록
        // 7. Write audit log
        status = FltGetFileNameInformation(
            Data,
            FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
            &nameInfo
        );

        if (NT_SUCCESS(status)) {
            FltParseFileNameInformation(nameInfo);

            AuditLogWrite(
                AUDIT_EVENT_WRITE_BLOCKED,
                &nameInfo->Name,
                PsGetCurrentProcessId(),
                Data->Iopb->Parameters.Write.Length
            );

            FltReleaseFileNameInformation(nameInfo);
            nameInfo = NULL;
        }

        DbgPrint("[Write] 쓰기 차단: 크기=%lu, 오프셋=%llu\n",
                 Data->Iopb->Parameters.Write.Length,
                 Data->Iopb->Parameters.Write.ByteOffset.QuadPart);
        // Write blocked: size, offset
    }
    __finally {
        if (streamContext != NULL) {
            FltReleaseContext(streamContext);
        }

        if (nameInfo != NULL) {
            FltReleaseFileNameInformation(nameInfo);
        }
    }

    if (shouldBlock) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// Post-Write 콜백 (로깅 목적)
// Post-Write callback (for logging)
FLT_POSTOP_CALLBACK_STATUS
WriteFilterPostWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    UNREFERENCED_PARAMETER(Data);
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);
    UNREFERENCED_PARAMETER(Flags);

    // Post 콜백에서는 이미 쓰기가 완료됨
    // In Post callback, write is already completed
    // 여기서는 주로 로깅에 사용
    // Mainly used for logging here

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

### 25.1.3 SetFileInformation 필터링

파일 크기 변경, 파일 덮어쓰기 등도 감지해야 합니다.

```c
// setinfo_filter.c - SetInformation 필터 구현
// setinfo_filter.c - SetInformation filter implementation

#include <fltKernel.h>
#include "stream_context.h"
#include "audit_log.h"

// SetInformation 콜백
// SetInformation callbacks
FLT_PREOP_CALLBACK_STATUS
SetInfoFilterPreOperation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PSTREAM_CONTEXT streamContext = NULL;
    FILE_INFORMATION_CLASS infoClass;
    BOOLEAN shouldBlock = FALSE;

    *CompletionContext = NULL;

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 정보 클래스 확인
    // Check information class
    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // 쓰기 관련 정보 클래스만 확인
    // Only check write-related information classes
    switch (infoClass) {
        case FileEndOfFileInformation:      // 파일 크기 변경
                                            // File size change
        case FileAllocationInformation:     // 할당 크기 변경
                                            // Allocation size change
        case FileValidDataLengthInformation: // 유효 데이터 길이 변경
                                             // Valid data length change
            break;

        default:
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 스트림 컨텍스트 가져오기
    // Get stream context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        &streamContext
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    __try {
        // 보호 확인
        // Check protection
        if (!streamContext->IsProtected) {
            __leave;
        }

        if (!(streamContext->ProtectionFlags & PROTECT_WRITE)) {
            __leave;
        }

        // 예외 프로세스 확인
        // Check exempt process
        if (IsExemptProcess(PsGetCurrentProcessId())) {
            __leave;
        }

        // 크기 변경 정보 로깅
        // Log size change information
        switch (infoClass) {
            case FileEndOfFileInformation: {
                PFILE_END_OF_FILE_INFORMATION eofInfo =
                    Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

                DbgPrint("[SetInfo] EndOfFile 변경 시도: %llu\n",
                         eofInfo->EndOfFile.QuadPart);
                // EndOfFile change attempt
                break;
            }

            case FileAllocationInformation: {
                PFILE_ALLOCATION_INFORMATION allocInfo =
                    Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

                DbgPrint("[SetInfo] Allocation 변경 시도: %llu\n",
                         allocInfo->AllocationSize.QuadPart);
                // Allocation change attempt
                break;
            }

            default:
                break;
        }

        shouldBlock = TRUE;
    }
    __finally {
        FltReleaseContext(streamContext);
    }

    if (shouldBlock) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### 25.1.4 캐시 매니저 고려사항

```c
// cache_considerations.c - 캐시 매니저 관련 고려사항
// cache_considerations.c - Cache manager considerations

#include <fltKernel.h>

/*
 * Windows 캐시 매니저 동작 이해
 * Understanding Windows Cache Manager Behavior
 *
 * 1. 일반 쓰기 흐름 (Non-cached Write가 아닌 경우)
 *    Normal write flow (when not Non-cached Write)
 *
 *    App WriteFile() → NT IRP_MJ_WRITE → Cache Manager → Memory
 *                                                    ↓
 *                      (나중에) Lazy Writer → IRP_MJ_WRITE (Paging I/O)
 *                      (later)
 *
 * 2. 차단 시점
 *    Blocking point
 *
 *    - 원본 IRP_MJ_WRITE를 차단해야 함 (Paging I/O 아닌 것)
 *      Block original IRP_MJ_WRITE (not Paging I/O)
 *
 *    - Paging I/O를 차단하면 캐시 매니저 오류 발생
 *      Blocking Paging I/O causes cache manager errors
 *
 * 3. 캐시 무효화 필요한 경우
 *    When cache invalidation is needed
 *
 *    - 보호 상태가 변경될 때 기존 캐시 무효화 필요
 *      Need to invalidate existing cache when protection state changes
 */

// 파일 캐시 무효화 (보호 활성화 시)
// Invalidate file cache (when enabling protection)
NTSTATUS InvalidateFileCache(
    _In_ PFLT_INSTANCE Instance,
    _In_ PFILE_OBJECT FileObject)
{
    NTSTATUS status;

    // 캐시가 있는지 확인
    // Check if cache exists
    if (!CcIsFileCached(FileObject)) {
        return STATUS_SUCCESS;
    }

    __try {
        // 섹션 객체 포인터 가져오기
        // Get section object pointers
        PSECTION_OBJECT_POINTERS sectionPointers =
            FileObject->SectionObjectPointer;

        if (sectionPointers == NULL) {
            return STATUS_SUCCESS;
        }

        // 캐시 플러시
        // Flush cache
        CcFlushCache(sectionPointers, NULL, 0, NULL);

        // 캐시 퍼지 (데이터 매핑)
        // Purge cache (data mapping)
        CcPurgeCacheSection(sectionPointers, NULL, 0, FALSE);

        DbgPrint("[Cache] 파일 캐시 무효화 완료\n");
        // File cache invalidated

        status = STATUS_SUCCESS;
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        status = GetExceptionCode();
        DbgPrint("[Cache] 캐시 무효화 예외: 0x%X\n", status);
        // Cache invalidation exception
    }

    return status;
}

// 안전한 캐시 작업을 위한 래퍼
// Wrapper for safe cache operations
NTSTATUS SafeFlushCache(
    _In_ PFILE_OBJECT FileObject,
    _Out_opt_ PIO_STATUS_BLOCK IoStatus)
{
    IO_STATUS_BLOCK localIoStatus;
    PSECTION_OBJECT_POINTERS sectionPointers;

    if (IoStatus == NULL) {
        IoStatus = &localIoStatus;
    }

    RtlZeroMemory(IoStatus, sizeof(IO_STATUS_BLOCK));

    // NULL 체크
    // NULL check
    if (FileObject == NULL ||
        FileObject->SectionObjectPointer == NULL) {
        return STATUS_SUCCESS;
    }

    sectionPointers = FileObject->SectionObjectPointer;

    // 데이터 섹션 존재 확인
    // Check data section exists
    if (sectionPointers->DataSectionObject == NULL &&
        sectionPointers->SharedCacheMap == NULL) {
        return STATUS_SUCCESS;
    }

    __try {
        CcFlushCache(sectionPointers, NULL, 0, IoStatus);
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        IoStatus->Status = GetExceptionCode();
    }

    return IoStatus->Status;
}
```

---

## 25.2 쓰기 차단 구현

### 25.2.1 스트림 컨텍스트 활용

효율적인 보호 상태 확인을 위해 스트림 컨텍스트를 사용합니다.

```c
// stream_context.h - 스트림 컨텍스트 정의
// stream_context.h - Stream context definitions

#pragma once
#include <fltKernel.h>
#include "protection_flags.h"

// 스트림 컨텍스트 구조체
// Stream context structure
typedef struct _STREAM_CONTEXT {
    // 보호 상태
    // Protection state
    BOOLEAN IsProtected;            // 보호 활성화 여부
                                    // Is protection enabled
    PROTECTION_FLAGS ProtectionFlags;   // 보호 플래그
                                        // Protection flags
    POLICY_ID PolicyId;             // 적용된 정책 ID
                                    // Applied policy ID

    // 파일 정보
    // File information
    UNICODE_STRING FileName;        // 파일 이름
                                    // File name
    WCHAR FileNameBuffer[260];      // 파일 이름 버퍼
                                    // File name buffer
    LARGE_INTEGER FileSize;         // 파일 크기
                                    // File size

    // 통계
    // Statistics
    LONG BlockedWriteCount;         // 차단된 쓰기 횟수
                                    // Blocked write count
    LONG BlockedDeleteCount;        // 차단된 삭제 횟수
                                    // Blocked delete count

    // 동기화
    // Synchronization
    ERESOURCE Lock;                 // 컨텍스트 잠금
                                    // Context lock
    BOOLEAN LockInitialized;        // 잠금 초기화 여부
                                    // Lock initialized flag
} STREAM_CONTEXT, *PSTREAM_CONTEXT;

// 컨텍스트 관리 함수
// Context management functions
NTSTATUS StreamContextCreate(
    _In_ PFLT_FILTER Filter,
    _Out_ PSTREAM_CONTEXT* Context);

VOID StreamContextCleanup(
    _In_ PFLT_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType);

NTSTATUS StreamContextSetProtection(
    _In_ PSTREAM_CONTEXT Context,
    _In_ BOOLEAN IsProtected,
    _In_ PROTECTION_FLAGS Flags,
    _In_ POLICY_ID PolicyId);
```

```c
// stream_context.c - 스트림 컨텍스트 구현
// stream_context.c - Stream context implementation

#include "stream_context.h"

// 컨텍스트 정의
// Context definition
const FLT_CONTEXT_REGISTRATION ContextRegistration[] = {
    {
        FLT_STREAM_CONTEXT,
        0,
        StreamContextCleanup,
        sizeof(STREAM_CONTEXT),
        'xtCS',
        NULL,
        NULL,
        NULL
    },
    { FLT_CONTEXT_END }
};

// 컨텍스트 생성
// Create context
NTSTATUS StreamContextCreate(
    _In_ PFLT_FILTER Filter,
    _Out_ PSTREAM_CONTEXT* Context)
{
    NTSTATUS status;
    PSTREAM_CONTEXT newContext = NULL;

    *Context = NULL;

    status = FltAllocateContext(
        Filter,
        FLT_STREAM_CONTEXT,
        sizeof(STREAM_CONTEXT),
        PagedPool,
        &newContext
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 초기화
    // Initialize
    RtlZeroMemory(newContext, sizeof(STREAM_CONTEXT));

    status = ExInitializeResourceLite(&newContext->Lock);
    if (!NT_SUCCESS(status)) {
        FltReleaseContext(newContext);
        return status;
    }

    newContext->LockInitialized = TRUE;
    newContext->IsProtected = FALSE;
    newContext->ProtectionFlags = PROTECT_NONE;
    newContext->PolicyId = INVALID_POLICY_ID;

    *Context = newContext;

    return STATUS_SUCCESS;
}

// 컨텍스트 정리 콜백
// Context cleanup callback
VOID StreamContextCleanup(
    _In_ PFLT_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType)
{
    PSTREAM_CONTEXT streamContext = (PSTREAM_CONTEXT)Context;

    UNREFERENCED_PARAMETER(ContextType);

    if (streamContext->LockInitialized) {
        ExDeleteResourceLite(&streamContext->Lock);
        streamContext->LockInitialized = FALSE;
    }
}

// 보호 설정
// Set protection
NTSTATUS StreamContextSetProtection(
    _In_ PSTREAM_CONTEXT Context,
    _In_ BOOLEAN IsProtected,
    _In_ PROTECTION_FLAGS Flags,
    _In_ POLICY_ID PolicyId)
{
    KeEnterCriticalRegion();
    ExAcquireResourceExclusiveLite(&Context->Lock, TRUE);

    Context->IsProtected = IsProtected;
    Context->ProtectionFlags = Flags;
    Context->PolicyId = PolicyId;

    ExReleaseResourceLite(&Context->Lock);
    KeLeaveCriticalRegion();

    DbgPrint("[Context] 보호 설정 변경: Protected=%d, Flags=0x%04X\n",
             IsProtected, Flags);
    // Protection setting changed

    return STATUS_SUCCESS;
}

// 컨텍스트 가져오기 또는 생성
// Get or create context
NTSTATUS StreamContextGetOrCreate(
    _In_ PFLT_FILTER Filter,
    _In_ PFLT_INSTANCE Instance,
    _In_ PFILE_OBJECT FileObject,
    _Out_ PSTREAM_CONTEXT* Context,
    _Out_opt_ PBOOLEAN ContextCreated)
{
    NTSTATUS status;
    PSTREAM_CONTEXT existingContext = NULL;
    PSTREAM_CONTEXT newContext = NULL;
    BOOLEAN created = FALSE;

    *Context = NULL;
    if (ContextCreated != NULL) {
        *ContextCreated = FALSE;
    }

    // 기존 컨텍스트 확인
    // Check for existing context
    status = FltGetStreamContext(Instance, FileObject, &existingContext);

    if (NT_SUCCESS(status)) {
        *Context = existingContext;
        return STATUS_SUCCESS;
    }

    if (status != STATUS_NOT_FOUND) {
        return status;
    }

    // 새 컨텍스트 생성
    // Create new context
    status = StreamContextCreate(Filter, &newContext);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 컨텍스트 설정 시도
    // Try to set context
    status = FltSetStreamContext(
        Instance,
        FileObject,
        FLT_SET_CONTEXT_KEEP_IF_EXISTS,
        newContext,
        &existingContext
    );

    if (status == STATUS_FLT_CONTEXT_ALREADY_DEFINED) {
        // 다른 스레드가 먼저 설정함
        // Another thread set it first
        FltReleaseContext(newContext);
        *Context = existingContext;
        return STATUS_SUCCESS;
    }

    if (!NT_SUCCESS(status)) {
        FltReleaseContext(newContext);
        return status;
    }

    created = TRUE;
    *Context = newContext;

    if (ContextCreated != NULL) {
        *ContextCreated = created;
    }

    return STATUS_SUCCESS;
}
```

### 25.2.2 사용자 알림 메커니즘

```c
// user_notification.h - 사용자 알림 정의
// user_notification.h - User notification definitions

#pragma once

// 알림 메시지 타입
// Notification message types
typedef enum _NOTIFICATION_TYPE {
    NOTIFY_WRITE_BLOCKED,           // 쓰기 차단됨
                                    // Write blocked
    NOTIFY_DELETE_BLOCKED,          // 삭제 차단됨
                                    // Delete blocked
    NOTIFY_RENAME_BLOCKED,          // 이름변경 차단됨
                                    // Rename blocked
    NOTIFY_PROTECTION_ENABLED,      // 보호 활성화됨
                                    // Protection enabled
    NOTIFY_PROTECTION_DISABLED      // 보호 비활성화됨
                                    // Protection disabled
} NOTIFICATION_TYPE;

// 알림 메시지 구조체
// Notification message structure
typedef struct _USER_NOTIFICATION {
    NOTIFICATION_TYPE Type;         // 알림 타입
                                    // Notification type
    ULONG ProcessId;                // 프로세스 ID
                                    // Process ID
    WCHAR FileName[260];            // 파일 이름
                                    // File name
    WCHAR ProcessName[64];          // 프로세스 이름
                                    // Process name
    LARGE_INTEGER Timestamp;        // 타임스탬프
                                    // Timestamp
    ULONG AdditionalInfo;           // 추가 정보 (쓰기 크기 등)
                                    // Additional info (write size, etc.)
} USER_NOTIFICATION, *PUSER_NOTIFICATION;

// 알림 큐 관련 함수
// Notification queue functions
NTSTATUS NotificationQueueInitialize(VOID);
VOID NotificationQueueCleanup(VOID);
NTSTATUS NotificationSend(_In_ PUSER_NOTIFICATION Notification);
NTSTATUS NotificationReceive(_Out_ PUSER_NOTIFICATION Notification, _In_ ULONG TimeoutMs);
```

```c
// user_notification.c - 사용자 알림 구현
// user_notification.c - User notification implementation

#include "user_notification.h"
#include <ntifs.h>

#define MAX_NOTIFICATIONS 256

// 알림 큐
// Notification queue
typedef struct _NOTIFICATION_QUEUE {
    USER_NOTIFICATION Items[MAX_NOTIFICATIONS];
    ULONG Head;                     // 읽기 위치
                                    // Read position
    ULONG Tail;                     // 쓰기 위치
                                    // Write position
    ULONG Count;                    // 현재 항목 수
                                    // Current item count
    KEVENT DataAvailableEvent;      // 데이터 대기 이벤트
                                    // Data available event
    FAST_MUTEX Lock;                // 동기화
                                    // Synchronization
    BOOLEAN Initialized;
} NOTIFICATION_QUEUE, *PNOTIFICATION_QUEUE;

static NOTIFICATION_QUEUE g_NotificationQueue;

// 초기화
// Initialize
NTSTATUS NotificationQueueInitialize(VOID)
{
    RtlZeroMemory(&g_NotificationQueue, sizeof(NOTIFICATION_QUEUE));

    KeInitializeEvent(&g_NotificationQueue.DataAvailableEvent,
                      NotificationEvent, FALSE);

    ExInitializeFastMutex(&g_NotificationQueue.Lock);

    g_NotificationQueue.Head = 0;
    g_NotificationQueue.Tail = 0;
    g_NotificationQueue.Count = 0;
    g_NotificationQueue.Initialized = TRUE;

    DbgPrint("[Notify] 알림 큐 초기화 완료\n");
    // Notification queue initialized

    return STATUS_SUCCESS;
}

// 정리
// Cleanup
VOID NotificationQueueCleanup(VOID)
{
    g_NotificationQueue.Initialized = FALSE;

    // 대기 중인 스레드 깨우기
    // Wake up waiting threads
    KeSetEvent(&g_NotificationQueue.DataAvailableEvent, IO_NO_INCREMENT, FALSE);
}

// 알림 전송
// Send notification
NTSTATUS NotificationSend(_In_ PUSER_NOTIFICATION Notification)
{
    if (!g_NotificationQueue.Initialized) {
        return STATUS_NOT_INITIALIZED;
    }

    ExAcquireFastMutex(&g_NotificationQueue.Lock);

    // 큐가 가득 찼는지 확인
    // Check if queue is full
    if (g_NotificationQueue.Count >= MAX_NOTIFICATIONS) {
        // 가장 오래된 항목 삭제
        // Remove oldest item
        g_NotificationQueue.Head = (g_NotificationQueue.Head + 1) % MAX_NOTIFICATIONS;
        g_NotificationQueue.Count--;

        DbgPrint("[Notify] 큐 오버플로우, 오래된 알림 삭제\n");
        // Queue overflow, removing old notification
    }

    // 새 알림 추가
    // Add new notification
    RtlCopyMemory(
        &g_NotificationQueue.Items[g_NotificationQueue.Tail],
        Notification,
        sizeof(USER_NOTIFICATION)
    );

    g_NotificationQueue.Tail = (g_NotificationQueue.Tail + 1) % MAX_NOTIFICATIONS;
    g_NotificationQueue.Count++;

    ExReleaseFastMutex(&g_NotificationQueue.Lock);

    // 대기 중인 스레드 깨우기
    // Wake up waiting threads
    KeSetEvent(&g_NotificationQueue.DataAvailableEvent, IO_NO_INCREMENT, FALSE);

    return STATUS_SUCCESS;
}

// 알림 수신 (사용자 모드 IOCTL에서 호출)
// Receive notification (called from user mode IOCTL)
NTSTATUS NotificationReceive(
    _Out_ PUSER_NOTIFICATION Notification,
    _In_ ULONG TimeoutMs)
{
    NTSTATUS status;
    LARGE_INTEGER timeout;

    if (!g_NotificationQueue.Initialized) {
        return STATUS_NOT_INITIALIZED;
    }

    // 타임아웃 설정
    // Set timeout
    timeout.QuadPart = -((LONGLONG)TimeoutMs * 10000);  // 밀리초 → 100나노초
                                                         // ms → 100ns

    // 데이터 대기
    // Wait for data
    status = KeWaitForSingleObject(
        &g_NotificationQueue.DataAvailableEvent,
        Executive,
        KernelMode,
        FALSE,
        TimeoutMs > 0 ? &timeout : NULL
    );

    if (status == STATUS_TIMEOUT) {
        return STATUS_TIMEOUT;
    }

    ExAcquireFastMutex(&g_NotificationQueue.Lock);

    if (g_NotificationQueue.Count == 0) {
        // 이벤트가 설정됐지만 데이터 없음 (종료 신호일 수 있음)
        // Event was set but no data (might be shutdown signal)
        ExReleaseFastMutex(&g_NotificationQueue.Lock);

        if (!g_NotificationQueue.Initialized) {
            return STATUS_SHUTDOWN_IN_PROGRESS;
        }

        // 이벤트 리셋
        // Reset event
        KeClearEvent(&g_NotificationQueue.DataAvailableEvent);
        return STATUS_NO_MORE_ENTRIES;
    }

    // 알림 가져오기
    // Get notification
    RtlCopyMemory(
        Notification,
        &g_NotificationQueue.Items[g_NotificationQueue.Head],
        sizeof(USER_NOTIFICATION)
    );

    g_NotificationQueue.Head = (g_NotificationQueue.Head + 1) % MAX_NOTIFICATIONS;
    g_NotificationQueue.Count--;

    // 큐가 비었으면 이벤트 리셋
    // Reset event if queue is empty
    if (g_NotificationQueue.Count == 0) {
        KeClearEvent(&g_NotificationQueue.DataAvailableEvent);
    }

    ExReleaseFastMutex(&g_NotificationQueue.Lock);

    return STATUS_SUCCESS;
}

// 쓰기 차단 알림 헬퍼
// Write blocked notification helper
VOID NotifyWriteBlocked(
    _In_ PCUNICODE_STRING FileName,
    _In_ HANDLE ProcessId,
    _In_ ULONG WriteSize)
{
    USER_NOTIFICATION notification;

    RtlZeroMemory(&notification, sizeof(notification));

    notification.Type = NOTIFY_WRITE_BLOCKED;
    notification.ProcessId = HandleToULong(ProcessId);
    notification.AdditionalInfo = WriteSize;
    KeQuerySystemTime(&notification.Timestamp);

    // 파일 이름 복사
    // Copy file name
    if (FileName != NULL && FileName->Length > 0) {
        ULONG copyLen = min(FileName->Length / sizeof(WCHAR), 259);
        RtlCopyMemory(notification.FileName, FileName->Buffer,
                      copyLen * sizeof(WCHAR));
        notification.FileName[copyLen] = L'\0';
    }

    // 프로세스 이름 가져오기
    // Get process name
    PEPROCESS process;
    if (NT_SUCCESS(PsLookupProcessByProcessId(ProcessId, &process))) {
        PCHAR imageName = PsGetProcessImageFileName(process);
        if (imageName != NULL) {
            // ASCII → Unicode 변환
            // ASCII → Unicode conversion
            for (int i = 0; i < 63 && imageName[i] != '\0'; i++) {
                notification.ProcessName[i] = (WCHAR)imageName[i];
            }
        }
        ObDereferenceObject(process);
    }

    NotificationSend(&notification);
}
```

### 25.2.3 감사 로그 기록

```c
// audit_log.h - 감사 로그 정의
// audit_log.h - Audit log definitions

#pragma once

// 감사 이벤트 타입
// Audit event types
typedef enum _AUDIT_EVENT_TYPE {
    AUDIT_EVENT_WRITE_BLOCKED,      // 쓰기 차단
                                    // Write blocked
    AUDIT_EVENT_WRITE_ALLOWED,      // 쓰기 허용
                                    // Write allowed
    AUDIT_EVENT_DELETE_BLOCKED,     // 삭제 차단
                                    // Delete blocked
    AUDIT_EVENT_RENAME_BLOCKED,     // 이름변경 차단
                                    // Rename blocked
    AUDIT_EVENT_POLICY_APPLIED,     // 정책 적용
                                    // Policy applied
    AUDIT_EVENT_POLICY_VIOLATION    // 정책 위반
                                    // Policy violation
} AUDIT_EVENT_TYPE;

// 감사 로그 기록
// Write audit log
NTSTATUS AuditLogWrite(
    _In_ AUDIT_EVENT_TYPE EventType,
    _In_ PCUNICODE_STRING FileName,
    _In_ HANDLE ProcessId,
    _In_ ULONG AdditionalData);

// 감사 로그 초기화/정리
// Audit log initialize/cleanup
NTSTATUS AuditLogInitialize(VOID);
VOID AuditLogCleanup(VOID);
```

```c
// audit_log.c - 감사 로그 구현
// audit_log.c - Audit log implementation

#include "audit_log.h"
#include <ntifs.h>

// 로그 파일 핸들
// Log file handle
static HANDLE g_LogFileHandle = NULL;
static FAST_MUTEX g_LogMutex;
static BOOLEAN g_LogInitialized = FALSE;

// 로그 파일 경로
// Log file path
#define LOG_FILE_PATH L"\\SystemRoot\\DrmFilter.log"

// 초기화
// Initialize
NTSTATUS AuditLogInitialize(VOID)
{
    NTSTATUS status;
    UNICODE_STRING filePath = RTL_CONSTANT_STRING(LOG_FILE_PATH);
    OBJECT_ATTRIBUTES objAttr;
    IO_STATUS_BLOCK ioStatus;

    ExInitializeFastMutex(&g_LogMutex);

    InitializeObjectAttributes(
        &objAttr,
        &filePath,
        OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE,
        NULL,
        NULL
    );

    status = ZwCreateFile(
        &g_LogFileHandle,
        FILE_APPEND_DATA | SYNCHRONIZE,
        &objAttr,
        &ioStatus,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        FILE_SHARE_READ,
        FILE_OPEN_IF,
        FILE_SYNCHRONOUS_IO_NONALERT | FILE_NON_DIRECTORY_FILE,
        NULL,
        0
    );

    if (NT_SUCCESS(status)) {
        g_LogInitialized = TRUE;
        DbgPrint("[Audit] 감사 로그 초기화 완료: %wZ\n", &filePath);
        // Audit log initialized
    } else {
        DbgPrint("[Audit] 로그 파일 생성 실패: 0x%X\n", status);
        // Log file creation failed
    }

    return status;
}

// 정리
// Cleanup
VOID AuditLogCleanup(VOID)
{
    if (g_LogFileHandle != NULL) {
        ZwClose(g_LogFileHandle);
        g_LogFileHandle = NULL;
    }

    g_LogInitialized = FALSE;
}

// 이벤트 타입 문자열
// Event type string
static const CHAR* GetEventTypeString(AUDIT_EVENT_TYPE Type)
{
    switch (Type) {
        case AUDIT_EVENT_WRITE_BLOCKED:     return "WRITE_BLOCKED";
        case AUDIT_EVENT_WRITE_ALLOWED:     return "WRITE_ALLOWED";
        case AUDIT_EVENT_DELETE_BLOCKED:    return "DELETE_BLOCKED";
        case AUDIT_EVENT_RENAME_BLOCKED:    return "RENAME_BLOCKED";
        case AUDIT_EVENT_POLICY_APPLIED:    return "POLICY_APPLIED";
        case AUDIT_EVENT_POLICY_VIOLATION:  return "POLICY_VIOLATION";
        default:                            return "UNKNOWN";
    }
}

// 로그 기록
// Write log
NTSTATUS AuditLogWrite(
    _In_ AUDIT_EVENT_TYPE EventType,
    _In_ PCUNICODE_STRING FileName,
    _In_ HANDLE ProcessId,
    _In_ ULONG AdditionalData)
{
    NTSTATUS status;
    CHAR logBuffer[512];
    LARGE_INTEGER systemTime, localTime;
    TIME_FIELDS timeFields;
    IO_STATUS_BLOCK ioStatus;

    if (!g_LogInitialized || g_LogFileHandle == NULL) {
        return STATUS_NOT_INITIALIZED;
    }

    // 타임스탬프 가져오기
    // Get timestamp
    KeQuerySystemTime(&systemTime);
    ExSystemTimeToLocalTime(&systemTime, &localTime);
    RtlTimeToTimeFields(&localTime, &timeFields);

    // 로그 항목 포맷
    // Format log entry
    status = RtlStringCbPrintfA(
        logBuffer,
        sizeof(logBuffer),
        "[%04d-%02d-%02d %02d:%02d:%02d] %s PID=%lu File=%wZ Data=%lu\r\n",
        timeFields.Year,
        timeFields.Month,
        timeFields.Day,
        timeFields.Hour,
        timeFields.Minute,
        timeFields.Second,
        GetEventTypeString(EventType),
        HandleToULong(ProcessId),
        FileName,
        AdditionalData
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 파일에 쓰기
    // Write to file
    ExAcquireFastMutex(&g_LogMutex);

    status = ZwWriteFile(
        g_LogFileHandle,
        NULL,
        NULL,
        NULL,
        &ioStatus,
        logBuffer,
        (ULONG)strlen(logBuffer),
        NULL,
        NULL
    );

    ExReleaseFastMutex(&g_LogMutex);

    return status;
}
```

---

## 25.3 [실습] 특정 확장자 쓰기 차단

### 25.3.1 완전한 쓰기 차단 드라이버

```c
// WriteBlocker.c - 쓰기 차단 Minifilter 드라이버
// WriteBlocker.c - Write Blocking Minifilter Driver

#include <fltKernel.h>
#include <dontuse.h>
#include <suppress.h>
#include <ntstrsafe.h>

#pragma prefast(disable:__WARNING_ENCODE_MEMBER_FUNCTION_POINTER, "Not valid for kernel mode drivers")

// 전역 변수
// Global variables
PFLT_FILTER g_FilterHandle = NULL;

// 보호 대상 확장자
// Protected extensions
static const UNICODE_STRING g_ProtectedExtensions[] = {
    RTL_CONSTANT_STRING(L".txt"),
    RTL_CONSTANT_STRING(L".log"),
    RTL_CONSTANT_STRING(L".docx"),
    RTL_CONSTANT_STRING(L".xlsx"),
};
#define PROTECTED_EXT_COUNT (sizeof(g_ProtectedExtensions) / sizeof(g_ProtectedExtensions[0]))

// 예외 프로세스
// Exempt processes
static const CHAR* g_ExemptProcesses[] = {
    "notepad.exe",      // 메모장 (테스트용)
                        // Notepad (for testing)
    "SYSTEM",           // 시스템 프로세스
                        // System process
    NULL
};

// 확장자 검사
// Check extension
BOOLEAN IsProtectedExtension(_In_ PCUNICODE_STRING FileName)
{
    for (ULONG i = 0; i < PROTECTED_EXT_COUNT; i++) {
        if (RtlSuffixUnicodeString(&g_ProtectedExtensions[i], FileName, TRUE)) {
            return TRUE;
        }
    }
    return FALSE;
}

// 예외 프로세스 확인
// Check exempt process
BOOLEAN IsExemptProcess(_In_ HANDLE ProcessId)
{
    PEPROCESS process = NULL;
    PCHAR imageName;
    BOOLEAN isExempt = FALSE;

    // System 프로세스 (PID 4)
    // System process (PID 4)
    if (HandleToULong(ProcessId) <= 4) {
        return TRUE;
    }

    if (!NT_SUCCESS(PsLookupProcessByProcessId(ProcessId, &process))) {
        return FALSE;
    }

    imageName = PsGetProcessImageFileName(process);

    if (imageName != NULL) {
        for (int i = 0; g_ExemptProcesses[i] != NULL; i++) {
            if (_stricmp(imageName, g_ExemptProcesses[i]) == 0) {
                isExempt = TRUE;
                break;
            }
        }
    }

    ObDereferenceObject(process);

    return isExempt;
}

// Pre-Create 콜백 - 쓰기 의도 확인 및 차단
// Pre-Create callback - Check write intent and block
FLT_PREOP_CALLBACK_STATUS
PreCreateCallback(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    ACCESS_MASK desiredAccess;
    ULONG createDisposition;
    BOOLEAN shouldBlock = FALSE;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 커널 모드 요청은 허용
    // Allow kernel mode requests
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 쓰기 관련 액세스 확인
    // Check write-related access
    desiredAccess = Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess;
    createDisposition = (Data->Iopb->Parameters.Create.Options >> 24) & 0xFF;

    // 쓰기/수정 의도가 없으면 패스
    // Pass if no write/modify intent
    if (!(desiredAccess & (FILE_WRITE_DATA | FILE_APPEND_DATA | DELETE | FILE_WRITE_ATTRIBUTES))) {
        // 덮어쓰기 Disposition도 확인
        // Also check overwrite disposition
        if (createDisposition != FILE_SUPERSEDE &&
            createDisposition != FILE_OVERWRITE &&
            createDisposition != FILE_OVERWRITE_IF) {
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
        }
    }

    // 파일 이름 가져오기
    // Get file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    status = FltParseFileNameInformation(nameInfo);
    if (!NT_SUCCESS(status)) {
        FltReleaseFileNameInformation(nameInfo);
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 보호 대상 확장자인지 확인
    // Check if protected extension
    if (IsProtectedExtension(&nameInfo->Name)) {
        // 예외 프로세스 확인
        // Check exempt process
        if (!IsExemptProcess(PsGetCurrentProcessId())) {
            shouldBlock = TRUE;

            DbgPrint("[WriteBlocker] 쓰기 의도 차단: %wZ (PID: %lu)\n",
                     &nameInfo->Name,
                     HandleToULong(PsGetCurrentProcessId()));
            // Write intent blocked
        }
    }

    FltReleaseFileNameInformation(nameInfo);

    if (shouldBlock) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// Pre-Write 콜백
// Pre-Write callback
FLT_PREOP_CALLBACK_STATUS
PreWriteCallback(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    BOOLEAN shouldBlock = FALSE;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // 페이징 I/O는 허용 (캐시 매니저)
    // Allow paging I/O (cache manager)
    if (FlagOn(Data->Iopb->IrpFlags, IRP_PAGING_IO)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 커널 모드 요청은 허용
    // Allow kernel mode requests
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 예외 프로세스 확인 (먼저)
    // Check exempt process (first)
    if (IsExemptProcess(PsGetCurrentProcessId())) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 가져오기
    // Get file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

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
    if (IsProtectedExtension(&nameInfo->Name)) {
        shouldBlock = TRUE;

        DbgPrint("[WriteBlocker] 쓰기 차단: %wZ, 크기=%lu (PID: %lu)\n",
                 &nameInfo->Name,
                 Data->Iopb->Parameters.Write.Length,
                 HandleToULong(PsGetCurrentProcessId()));
        // Write blocked, size, PID
    }

    FltReleaseFileNameInformation(nameInfo);

    if (shouldBlock) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// Pre-SetInformation 콜백
// Pre-SetInformation callback
FLT_PREOP_CALLBACK_STATUS
PreSetInformationCallback(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    FILE_INFORMATION_CLASS infoClass;
    BOOLEAN shouldBlock = FALSE;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // 크기 변경 관련 클래스만 확인
    // Only check size change related classes
    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    if (infoClass != FileEndOfFileInformation &&
        infoClass != FileAllocationInformation &&
        infoClass != FileValidDataLengthInformation) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 예외 프로세스 확인
    // Check exempt process
    if (IsExemptProcess(PsGetCurrentProcessId())) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 가져오기
    // Get file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

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
    if (IsProtectedExtension(&nameInfo->Name)) {
        shouldBlock = TRUE;

        DbgPrint("[WriteBlocker] 크기 변경 차단: %wZ, 클래스=%d (PID: %lu)\n",
                 &nameInfo->Name,
                 infoClass,
                 HandleToULong(PsGetCurrentProcessId()));
        // Size change blocked, class, PID
    }

    FltReleaseFileNameInformation(nameInfo);

    if (shouldBlock) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 콜백 등록
// Callback registration
const FLT_OPERATION_REGISTRATION Callbacks[] = {
    {
        IRP_MJ_CREATE,
        0,
        PreCreateCallback,
        NULL
    },
    {
        IRP_MJ_WRITE,
        0,
        PreWriteCallback,
        NULL
    },
    {
        IRP_MJ_SET_INFORMATION,
        0,
        PreSetInformationCallback,
        NULL
    },
    { IRP_MJ_OPERATION_END }
};

// 필터 언로드
// Filter unload
NTSTATUS
FilterUnload(
    _In_ FLT_FILTER_UNLOAD_FLAGS Flags)
{
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("[WriteBlocker] 드라이버 언로드\n");
    // Driver unloading

    FltUnregisterFilter(g_FilterHandle);

    return STATUS_SUCCESS;
}

// 인스턴스 설정
// Instance setup
NTSTATUS
InstanceSetup(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_SETUP_FLAGS Flags,
    _In_ DEVICE_TYPE VolumeDeviceType,
    _In_ FLT_FILESYSTEM_TYPE VolumeFilesystemType)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    // 네트워크 리디렉터와 CDFS 제외
    // Exclude network redirectors and CDFS
    if (VolumeDeviceType == FILE_DEVICE_NETWORK_FILE_SYSTEM) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    if (VolumeFilesystemType == FLT_FSTYPE_CDFS) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    DbgPrint("[WriteBlocker] 볼륨에 연결됨 (타입: %d)\n", VolumeFilesystemType);
    // Attached to volume (type)

    return STATUS_SUCCESS;
}

// 필터 등록
// Filter registration
const FLT_REGISTRATION FilterRegistration = {
    sizeof(FLT_REGISTRATION),           // Size
    FLT_REGISTRATION_VERSION,           // Version
    0,                                  // Flags
    NULL,                               // Context
    Callbacks,                          // Operation callbacks
    FilterUnload,                       // FilterUnload
    InstanceSetup,                      // InstanceSetup
    NULL,                               // InstanceQueryTeardown
    NULL,                               // InstanceTeardownStart
    NULL,                               // InstanceTeardownComplete
    NULL,                               // GenerateFileName
    NULL,                               // NormalizeNameComponent
    NULL,                               // NormalizeContextCleanup
    NULL,                               // TransactionNotification
    NULL                                // NormalizeNameComponentEx
};

// 드라이버 진입점
// Driver entry point
NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;

    UNREFERENCED_PARAMETER(RegistryPath);

    DbgPrint("[WriteBlocker] 드라이버 시작\n");
    // Driver starting

    // 필터 등록
    // Register filter
    status = FltRegisterFilter(
        DriverObject,
        &FilterRegistration,
        &g_FilterHandle
    );

    if (!NT_SUCCESS(status)) {
        DbgPrint("[WriteBlocker] 필터 등록 실패: 0x%X\n", status);
        // Filter registration failed
        return status;
    }

    // 필터링 시작
    // Start filtering
    status = FltStartFiltering(g_FilterHandle);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[WriteBlocker] 필터링 시작 실패: 0x%X\n", status);
        // Filtering start failed
        FltUnregisterFilter(g_FilterHandle);
        return status;
    }

    DbgPrint("[WriteBlocker] 필터링 시작됨\n");
    // Filtering started

    return STATUS_SUCCESS;
}
```

### 25.3.2 INF 파일

```inf
;;;
;;; WriteBlocker.inf
;;;

[Version]
Signature   = "$Windows NT$"
Class       = "ActivityMonitor"
ClassGuid   = {b86dff51-a31e-4bac-b3cf-e8cfe75c9fc2}
Provider    = %ProviderString%
DriverVer   = 01/01/2024,1.0.0.0
CatalogFile = WriteBlocker.cat

[DestinationDirs]
DefaultDestDir = 12
WriteBlocker.DriverFiles = 12

[DefaultInstall.NTamd64]
OptionDesc  = %ServiceDescription%
CopyFiles   = WriteBlocker.DriverFiles

[DefaultInstall.NTamd64.Services]
AddService  = %ServiceName%,,WriteBlocker.Service

[DefaultUninstall.NTamd64]
DelFiles    = WriteBlocker.DriverFiles

[DefaultUninstall.NTamd64.Services]
DelService  = %ServiceName%,0x200

[WriteBlocker.Service]
DisplayName    = %ServiceName%
Description    = %ServiceDescription%
ServiceBinary  = %12%\WriteBlocker.sys
ServiceType    = 2
StartType      = 3
ErrorControl   = 1
LoadOrderGroup = "FSFilter Activity Monitor"
AddReg         = WriteBlocker.AddRegistry

[WriteBlocker.AddRegistry]
HKR,,"SupportedFeatures",0x00010001,0x3
HKR,"Instances","DefaultInstance",0x00000000,%DefaultInstance%
HKR,"Instances\"%Instance1.Name%,"Altitude",0x00000000,%Instance1.Altitude%
HKR,"Instances\"%Instance1.Name%,"Flags",0x00010001,%Instance1.Flags%

[WriteBlocker.DriverFiles]
WriteBlocker.sys

[SourceDisksFiles]
WriteBlocker.sys = 1,,

[SourceDisksNames]
1 = %DiskId1%,,,

[Strings]
ProviderString      = "My Company"
ServiceDescription  = "Write Blocker Minifilter Driver"
ServiceName         = "WriteBlocker"
DiskId1             = "WriteBlocker Installation Disk"
DefaultInstance     = "WriteBlocker Instance"
Instance1.Name      = "WriteBlocker Instance"
Instance1.Altitude  = "370000"
Instance1.Flags     = 0x0
```

---

## 25.4 요약

### 핵심 내용 정리

| 항목 | 내용 |
|------|------|
| **쓰기 차단 지점** | IRP_MJ_CREATE, IRP_MJ_WRITE, IRP_MJ_SET_INFORMATION |
| **페이징 I/O** | 차단하지 않음 - 원본 요청을 차단해야 함 |
| **예외 처리** | 시스템 프로세스, 지정된 프로세스 허용 |
| **알림 시스템** | 큐 기반 비동기 알림 메커니즘 |
| **감사 로그** | 파일 기반 이벤트 기록 |

### .NET 개발자 참고 사항

```
┌─────────────────────────────────────────────────────────────────┐
│               C# vs C 파일 I/O 차단 비교                         │
├─────────────────────────────────────────────────────────────────┤
│ C# (사용자 모드)                 │ C (커널 모드)                  │
├──────────────────────────────────┼────────────────────────────────┤
│ FileSystemWatcher               │ Minifilter 콜백                │
│ FileStream.Write 예외 처리       │ Pre-operation에서 차단         │
│ Process.GetProcessById          │ PsLookupProcessByProcessId    │
│ Queue<T> 또는 Channel<T>        │ 링 버퍼 + 수동 동기화          │
│ EventLog 또는 NLog              │ ZwWriteFile + 수동 포맷       │
│ Task.WhenAny (비동기)            │ KeWaitForSingleObject         │
└──────────────────────────────────┴────────────────────────────────┘
```

### 다음 챕터 예고

Chapter 26에서는 **파일 이름 변경 및 이동 차단**을 다룹니다. IRP_MJ_SET_INFORMATION의 FileRenameInformation을 통해 파일 이름 변경과 이동을 감지하고 차단하는 방법을 구현합니다.

---

## 참고 자료

- [Microsoft Docs: IRP_MJ_WRITE](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/irp-mj-write)
- [Microsoft Docs: IRP_MJ_SET_INFORMATION](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/irp-mj-set-information)
- [Microsoft Docs: Cache Manager](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/cache-manager-support-routines)
- [Windows Internals: File System Filter Drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/)
