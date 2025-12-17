# Chapter 27: 파일 삭제 차단

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
- Windows에서의 다양한 파일 삭제 방법 이해
- 모든 삭제 경로에 대한 완전한 차단 구현
- 휴지통 이동 처리 및 사용자 피드백 제공

---

## 27.1 삭제 작업의 종류

### 27.1.1 Windows의 파일 삭제 메커니즘

Windows에서 파일을 삭제하는 방법은 여러 가지가 있으며, 완전한 보호를 위해서는 모든 경로를 이해하고 차단해야 합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   파일 삭제 방법과 경로                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   사용자 수준 API                         │  │
│  │  DeleteFile()   RemoveDirectory()   SHFileOperation()   │  │
│  │  del 명령어    rmdir 명령어        탐색기 삭제          │  │
│  └───────────────────────────┬──────────────────────────────┘  │
│                              │                                  │
│                              v                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     NT API 수준                           │  │
│  │  NtSetInformationFile(FileDispositionInformation)        │  │
│  │  NtSetInformationFile(FileDispositionInformationEx)      │  │
│  │  NtCreateFile with FILE_DELETE_ON_CLOSE                  │  │
│  └───────────────────────────┬──────────────────────────────┘  │
│                              │                                  │
│                              v                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              커널 I/O Manager → Minifilter                │  │
│  │                                                           │  │
│  │  1. IRP_MJ_SET_INFORMATION                               │  │
│  │     • FileDispositionInformation (class 13)              │  │
│  │     • FileDispositionInformationEx (class 64)            │  │
│  │                                                           │  │
│  │  2. IRP_MJ_CREATE                                        │  │
│  │     • FILE_DELETE_ON_CLOSE 옵션                          │  │
│  │                                                           │  │
│  │  3. IRP_MJ_SET_INFORMATION (이름 변경)                   │  │
│  │     • $Recycle.Bin으로 이동 (휴지통)                      │  │
│  │                                                           │  │
│  │  4. IRP_MJ_CLEANUP (실제 삭제 실행)                       │  │
│  │     • DeletePending이 TRUE인 파일                        │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 27.1.2 FileDispositionInformation 구조체

```c
// 파일 삭제 정보 구조체 (기본)
// File disposition information structure (basic)
typedef struct _FILE_DISPOSITION_INFORMATION {
    BOOLEAN DeleteFile;     // TRUE이면 삭제 예약
                            // If TRUE, mark for deletion
} FILE_DISPOSITION_INFORMATION, *PFILE_DISPOSITION_INFORMATION;

// 확장 삭제 정보 구조체 (Windows 10 RS1 이후)
// Extended disposition information (Windows 10 RS1+)
typedef struct _FILE_DISPOSITION_INFORMATION_EX {
    ULONG Flags;            // 삭제 플래그
                            // Deletion flags
} FILE_DISPOSITION_INFORMATION_EX, *PFILE_DISPOSITION_INFORMATION_EX;

// 확장 플래그 정의
// Extended flag definitions
#define FILE_DISPOSITION_DO_NOT_DELETE              0x00000000
#define FILE_DISPOSITION_DELETE                     0x00000001
#define FILE_DISPOSITION_POSIX_SEMANTICS            0x00000002
#define FILE_DISPOSITION_FORCE_IMAGE_SECTION_CHECK  0x00000004
#define FILE_DISPOSITION_ON_CLOSE                   0x00000008
#define FILE_DISPOSITION_IGNORE_READONLY_ATTRIBUTE  0x00000010
```

### 27.1.3 삭제 감지 포인트 분석

```c
// delete_analysis.c - 삭제 감지 분석
// delete_analysis.c - Delete detection analysis

#include <fltKernel.h>

// 삭제 유형
// Delete types
typedef enum _DELETE_TYPE {
    DELETE_TYPE_DISPOSITION,            // FileDispositionInformation
    DELETE_TYPE_DISPOSITION_EX,         // FileDispositionInformationEx
    DELETE_TYPE_ON_CLOSE,               // FILE_DELETE_ON_CLOSE
    DELETE_TYPE_SUPERSEDE,              // FILE_SUPERSEDE (덮어쓰기로 삭제)
                                        // FILE_SUPERSEDE (delete by overwrite)
    DELETE_TYPE_OVERWRITE,              // FILE_OVERWRITE (기존 내용 삭제)
                                        // FILE_OVERWRITE (delete existing content)
    DELETE_TYPE_RECYCLE,                // 휴지통 이동
                                        // Move to Recycle Bin
    DELETE_TYPE_UNKNOWN
} DELETE_TYPE;

// 삭제 정보 구조체
// Delete information structure
typedef struct _DELETE_INFO {
    DELETE_TYPE Type;                   // 삭제 유형
                                        // Delete type
    BOOLEAN DeleteRequested;            // 삭제 요청됨
                                        // Delete requested
    ULONG Flags;                        // 추가 플래그
                                        // Additional flags
    UNICODE_STRING FileName;            // 파일 이름
                                        // File name
} DELETE_INFO, *PDELETE_INFO;

// IRP_MJ_SET_INFORMATION에서 삭제 감지
// Detect delete in IRP_MJ_SET_INFORMATION
BOOLEAN DetectDeleteInSetInfo(
    _In_ PFLT_CALLBACK_DATA Data,
    _Out_ PDELETE_INFO DeleteInfo)
{
    FILE_INFORMATION_CLASS infoClass;
    PVOID infoBuffer;

    RtlZeroMemory(DeleteInfo, sizeof(DELETE_INFO));

    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;
    infoBuffer = Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

    switch (infoClass) {
        case FileDispositionInformation: {
            PFILE_DISPOSITION_INFORMATION dispInfo =
                (PFILE_DISPOSITION_INFORMATION)infoBuffer;

            if (dispInfo->DeleteFile) {
                DeleteInfo->Type = DELETE_TYPE_DISPOSITION;
                DeleteInfo->DeleteRequested = TRUE;
                return TRUE;
            }
            break;
        }

        case FileDispositionInformationEx: {
            PFILE_DISPOSITION_INFORMATION_EX dispInfoEx =
                (PFILE_DISPOSITION_INFORMATION_EX)infoBuffer;

            if (dispInfoEx->Flags & FILE_DISPOSITION_DELETE) {
                DeleteInfo->Type = DELETE_TYPE_DISPOSITION_EX;
                DeleteInfo->DeleteRequested = TRUE;
                DeleteInfo->Flags = dispInfoEx->Flags;
                return TRUE;
            }
            break;
        }

        default:
            break;
    }

    return FALSE;
}

// IRP_MJ_CREATE에서 삭제 감지
// Detect delete in IRP_MJ_CREATE
BOOLEAN DetectDeleteInCreate(
    _In_ PFLT_CALLBACK_DATA Data,
    _Out_ PDELETE_INFO DeleteInfo)
{
    ULONG createOptions;
    ULONG createDisposition;

    RtlZeroMemory(DeleteInfo, sizeof(DELETE_INFO));

    createOptions = Data->Iopb->Parameters.Create.Options & FILE_VALID_OPTION_FLAGS;
    createDisposition = (Data->Iopb->Parameters.Create.Options >> 24) & 0xFF;

    // DELETE_ON_CLOSE 플래그 확인
    // Check DELETE_ON_CLOSE flag
    if (createOptions & FILE_DELETE_ON_CLOSE) {
        DeleteInfo->Type = DELETE_TYPE_ON_CLOSE;
        DeleteInfo->DeleteRequested = TRUE;
        return TRUE;
    }

    // SUPERSEDE 또는 OVERWRITE 확인
    // Check SUPERSEDE or OVERWRITE
    if (createDisposition == FILE_SUPERSEDE) {
        DeleteInfo->Type = DELETE_TYPE_SUPERSEDE;
        DeleteInfo->DeleteRequested = TRUE;  // 기존 파일 삭제됨
                                              // Existing file will be deleted
        return TRUE;
    }

    if (createDisposition == FILE_OVERWRITE ||
        createDisposition == FILE_OVERWRITE_IF) {
        DeleteInfo->Type = DELETE_TYPE_OVERWRITE;
        DeleteInfo->DeleteRequested = TRUE;  // 내용이 삭제됨
                                              // Content will be deleted
        return TRUE;
    }

    return FALSE;
}
```

### 27.1.4 휴지통 이동 감지

```c
// recycle_bin.c - 휴지통 이동 감지
// recycle_bin.c - Recycle Bin move detection

#include <fltKernel.h>

// 휴지통 경로 패턴
// Recycle Bin path patterns
static const UNICODE_STRING g_RecycleBinPatterns[] = {
    RTL_CONSTANT_STRING(L"$Recycle.Bin"),
    RTL_CONSTANT_STRING(L"$RECYCLE.BIN"),
    RTL_CONSTANT_STRING(L"RECYCLER"),           // 이전 Windows 버전
                                                // Older Windows versions
    RTL_CONSTANT_STRING(L"RECYCLED"),           // FAT 파일시스템
                                                // FAT filesystem
};
#define RECYCLE_PATTERN_COUNT (sizeof(g_RecycleBinPatterns) / sizeof(g_RecycleBinPatterns[0]))

// 휴지통 경로인지 확인
// Check if path is Recycle Bin
BOOLEAN IsRecycleBinPath(_In_ PCUNICODE_STRING Path)
{
    if (Path == NULL || Path->Length == 0) {
        return FALSE;
    }

    for (ULONG i = 0; i < RECYCLE_PATTERN_COUNT; i++) {
        // 경로에 휴지통 패턴이 포함되어 있는지 확인
        // Check if path contains Recycle Bin pattern
        PWCHAR found = NULL;
        WCHAR upperPath[520];

        // 대문자 변환
        // Convert to uppercase
        ULONG copyLen = min(Path->Length / sizeof(WCHAR), 519);
        RtlCopyMemory(upperPath, Path->Buffer, copyLen * sizeof(WCHAR));
        upperPath[copyLen] = L'\0';
        _wcsupr(upperPath);

        WCHAR upperPattern[64];
        copyLen = min(g_RecycleBinPatterns[i].Length / sizeof(WCHAR), 63);
        RtlCopyMemory(upperPattern, g_RecycleBinPatterns[i].Buffer,
                      copyLen * sizeof(WCHAR));
        upperPattern[copyLen] = L'\0';
        _wcsupr(upperPattern);

        if (wcsstr(upperPath, upperPattern) != NULL) {
            return TRUE;
        }
    }

    return FALSE;
}

// 이름 변경이 휴지통 이동인지 확인
// Check if rename is Recycle Bin move
BOOLEAN IsRecycleBinMove(
    _In_ PCUNICODE_STRING SourcePath,
    _In_ PCUNICODE_STRING TargetPath)
{
    // 원본이 휴지통이 아니고, 대상이 휴지통인 경우
    // Source is not Recycle Bin, but target is Recycle Bin
    if (!IsRecycleBinPath(SourcePath) && IsRecycleBinPath(TargetPath)) {
        DbgPrint("[Delete] 휴지통 이동 감지: %wZ → %wZ\n",
                 SourcePath, TargetPath);
        // Recycle Bin move detected
        return TRUE;
    }

    return FALSE;
}

// 이름 변경 콜백에서 휴지통 이동 차단
// Block Recycle Bin move in rename callback
FLT_PREOP_CALLBACK_STATUS
RecycleBinRenameCheck(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ PCUNICODE_STRING SourcePath,
    _In_ PROTECTION_FLAGS ProtectionFlags)
{
    PFILE_RENAME_INFORMATION renameInfo;
    UNICODE_STRING targetPath;
    WCHAR targetBuffer[520];

    UNREFERENCED_PARAMETER(FltObjects);

    // 삭제 차단 플래그가 없으면 패스
    // Pass if no delete block flag
    if (!(ProtectionFlags & PROTECT_DELETE)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 이름 변경 정보 가져오기
    // Get rename information
    FILE_INFORMATION_CLASS infoClass =
        Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    if (infoClass != FileRenameInformation &&
        infoClass != FileRenameInformationEx) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    renameInfo = (PFILE_RENAME_INFORMATION)
        Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

    // 대상 경로 구성
    // Build target path
    ULONG copyLen = min(renameInfo->FileNameLength, sizeof(targetBuffer) - sizeof(WCHAR));
    RtlCopyMemory(targetBuffer, renameInfo->FileName, copyLen);
    targetBuffer[copyLen / sizeof(WCHAR)] = L'\0';

    RtlInitUnicodeString(&targetPath, targetBuffer);

    // 휴지통 이동인지 확인
    // Check if Recycle Bin move
    if (IsRecycleBinMove(SourcePath, &targetPath)) {
        DbgPrint("[Delete] 휴지통 이동 차단: %wZ\n", SourcePath);
        // Recycle Bin move blocked

        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 27.2 삭제 차단 구현

### 27.2.1 통합 삭제 차단 모듈

```c
// delete_blocker.h - 삭제 차단 모듈 정의
// delete_blocker.h - Delete blocker module definitions

#pragma once
#include <fltKernel.h>
#include "protection_policy.h"

// 삭제 차단 결과
// Delete blocking result
typedef enum _DELETE_BLOCK_RESULT {
    DELETE_ALLOWED,             // 삭제 허용
                                // Delete allowed
    DELETE_BLOCKED_POLICY,      // 정책에 의해 차단
                                // Blocked by policy
    DELETE_BLOCKED_PROTECTED,   // 보호된 파일이므로 차단
                                // Blocked because file is protected
    DELETE_BLOCKED_RECYCLE,     // 휴지통 이동 차단
                                // Recycle Bin move blocked
    DELETE_BLOCKED_SYSTEM       // 시스템 파일이므로 차단
                                // Blocked because system file
} DELETE_BLOCK_RESULT;

// 삭제 차단 콜백
// Delete blocking callbacks
FLT_PREOP_CALLBACK_STATUS
DeleteBlockerPreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext);

FLT_PREOP_CALLBACK_STATUS
DeleteBlockerPreSetInfo(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext);
```

```c
// delete_blocker.c - 삭제 차단 모듈 구현
// delete_blocker.c - Delete blocker module implementation

#include "delete_blocker.h"
#include "audit_log.h"
#include "user_notification.h"

// 시스템 파일/디렉터리 확인
// Check system file/directory
BOOLEAN IsSystemFile(_In_ PCUNICODE_STRING FilePath)
{
    static const UNICODE_STRING systemPaths[] = {
        RTL_CONSTANT_STRING(L"\\Windows\\"),
        RTL_CONSTANT_STRING(L"\\System32\\"),
        RTL_CONSTANT_STRING(L"\\SysWOW64\\"),
        RTL_CONSTANT_STRING(L"\\Program Files\\"),
        RTL_CONSTANT_STRING(L"\\Program Files (x86)\\"),
    };

    for (ULONG i = 0; i < sizeof(systemPaths)/sizeof(systemPaths[0]); i++) {
        WCHAR* found = wcsstr(FilePath->Buffer, systemPaths[i].Buffer);
        if (found != NULL) {
            return TRUE;
        }
    }

    return FALSE;
}

// 삭제 차단 결정
// Decide delete blocking
DELETE_BLOCK_RESULT ShouldBlockDelete(
    _In_ PCUNICODE_STRING FilePath,
    _In_ HANDLE ProcessId,
    _In_ DELETE_TYPE DeleteType)
{
    PROTECTION_FLAGS protectionFlags;

    // 예외 프로세스 확인
    // Check exempt process
    if (IsExemptProcess(ProcessId)) {
        return DELETE_ALLOWED;
    }

    // 시스템 파일은 항상 허용 (시스템이 관리)
    // Always allow system files (managed by system)
    if (IsSystemFile(FilePath)) {
        return DELETE_ALLOWED;
    }

    // 정책 평가
    // Evaluate policy
    protectionFlags = PolicyEvaluate(
        FilePath,
        NULL,
        0,
        ProcessId
    );

    // 삭제 차단 플래그 확인
    // Check delete block flag
    if (protectionFlags & PROTECT_DELETE) {
        return DELETE_BLOCKED_POLICY;
    }

    return DELETE_ALLOWED;
}

// Pre-Create 콜백 (DELETE_ON_CLOSE 차단)
// Pre-Create callback (block DELETE_ON_CLOSE)
FLT_PREOP_CALLBACK_STATUS
DeleteBlockerPreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    DELETE_INFO deleteInfo;
    DELETE_BLOCK_RESULT blockResult;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 삭제 감지
    // Detect delete
    if (!DetectDeleteInCreate(Data, &deleteInfo)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // SUPERSEDE/OVERWRITE의 경우 쓰기 차단에서 처리될 수 있음
    // SUPERSEDE/OVERWRITE may be handled in write blocking
    if (deleteInfo.Type == DELETE_TYPE_SUPERSEDE ||
        deleteInfo.Type == DELETE_TYPE_OVERWRITE) {
        // 여기서도 차단하거나, 쓰기 차단에 위임
        // Block here too, or delegate to write blocking
        // return FLT_PREOP_SUCCESS_NO_CALLBACK;
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

    FltParseFileNameInformation(nameInfo);

    // 차단 결정
    // Decide blocking
    blockResult = ShouldBlockDelete(
        &nameInfo->Name,
        PsGetCurrentProcessId(),
        deleteInfo.Type
    );

    if (blockResult != DELETE_ALLOWED) {
        DbgPrint("[Delete] DELETE_ON_CLOSE 차단:\n");
        // DELETE_ON_CLOSE blocked
        DbgPrint("[Delete]   파일: %wZ\n", &nameInfo->Name);
        // File
        DbgPrint("[Delete]   이유: %d, PID: %lu\n",
                 blockResult, HandleToULong(PsGetCurrentProcessId()));
        // Reason, PID

        // 감사 로그
        // Audit log
        AuditLogWrite(
            AUDIT_EVENT_DELETE_BLOCKED,
            &nameInfo->Name,
            PsGetCurrentProcessId(),
            deleteInfo.Type
        );

        FltReleaseFileNameInformation(nameInfo);

        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    FltReleaseFileNameInformation(nameInfo);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// Pre-SetInformation 콜백 (Disposition 차단)
// Pre-SetInformation callback (block Disposition)
FLT_PREOP_CALLBACK_STATUS
DeleteBlockerPreSetInfo(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    FILE_INFORMATION_CLASS infoClass;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    DELETE_INFO deleteInfo;
    DELETE_BLOCK_RESULT blockResult;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // 정보 클래스 확인
    // Check information class
    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // Disposition 클래스만 처리
    // Only handle Disposition classes
    if (infoClass != FileDispositionInformation &&
        infoClass != FileDispositionInformationEx) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 삭제 감지
    // Detect delete
    if (!DetectDeleteInSetInfo(Data, &deleteInfo)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 삭제 요청이 아니면 패스
    // Pass if not delete request
    if (!deleteInfo.DeleteRequested) {
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

    FltParseFileNameInformation(nameInfo);

    // 차단 결정
    // Decide blocking
    blockResult = ShouldBlockDelete(
        &nameInfo->Name,
        PsGetCurrentProcessId(),
        deleteInfo.Type
    );

    if (blockResult != DELETE_ALLOWED) {
        DbgPrint("[Delete] Disposition 삭제 차단:\n");
        // Disposition delete blocked
        DbgPrint("[Delete]   파일: %wZ\n", &nameInfo->Name);
        // File
        DbgPrint("[Delete]   타입: %d, 이유: %d, PID: %lu\n",
                 deleteInfo.Type,
                 blockResult,
                 HandleToULong(PsGetCurrentProcessId()));
        // Type, Reason, PID

        // 감사 로그
        // Audit log
        AuditLogWrite(
            AUDIT_EVENT_DELETE_BLOCKED,
            &nameInfo->Name,
            PsGetCurrentProcessId(),
            deleteInfo.Type
        );

        // 사용자 알림
        // User notification
        NotifyDeleteBlocked(&nameInfo->Name, PsGetCurrentProcessId());

        FltReleaseFileNameInformation(nameInfo);

        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    FltReleaseFileNameInformation(nameInfo);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 삭제 차단 사용자 알림
// User notification for delete block
VOID NotifyDeleteBlocked(
    _In_ PCUNICODE_STRING FileName,
    _In_ HANDLE ProcessId)
{
    USER_NOTIFICATION notification;

    RtlZeroMemory(&notification, sizeof(notification));

    notification.Type = NOTIFY_DELETE_BLOCKED;
    notification.ProcessId = HandleToULong(ProcessId);
    KeQuerySystemTime(&notification.Timestamp);

    // 파일 이름 복사
    // Copy file name
    if (FileName != NULL && FileName->Length > 0) {
        ULONG copyLen = min(FileName->Length / sizeof(WCHAR), 259);
        RtlCopyMemory(notification.FileName, FileName->Buffer,
                      copyLen * sizeof(WCHAR));
        notification.FileName[copyLen] = L'\0';
    }

    NotificationSend(&notification);
}
```

### 27.2.2 DELETE_ON_CLOSE 플래그 제거

차단 대신 DELETE_ON_CLOSE 플래그를 제거하는 방법도 있습니다.

```c
// delete_flag_removal.c - DELETE_ON_CLOSE 플래그 제거
// delete_flag_removal.c - DELETE_ON_CLOSE flag removal

#include <fltKernel.h>

// DELETE_ON_CLOSE 플래그 제거 (Post-Create에서)
// Remove DELETE_ON_CLOSE flag (in Post-Create)
FLT_POSTOP_CALLBACK_STATUS
RemoveDeleteOnCloseFlag(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    UNREFERENCED_PARAMETER(CompletionContext);

    // 성공한 요청만 처리
    // Only handle successful requests
    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // DRAINING 상태면 패스
    // Pass if DRAINING
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // FileObject에서 DELETE_ON_CLOSE 플래그 확인
    // Check DELETE_ON_CLOSE flag in FileObject
    if (FltObjects->FileObject != NULL) {
        // FO_DELETE_ON_CLOSE 플래그 제거
        // Remove FO_DELETE_ON_CLOSE flag
        // 주의: 직접 수정은 권장되지 않음, 여기서는 개념만 설명
        // Note: Direct modification not recommended, just explaining concept

        /*
        if (FlagOn(FltObjects->FileObject->Flags, FO_DELETE_ON_CLOSE)) {
            // 플래그 제거
            // Remove flag
            ClearFlag(FltObjects->FileObject->Flags, FO_DELETE_ON_CLOSE);

            DbgPrint("[Delete] DELETE_ON_CLOSE 플래그 제거됨\n");
            // DELETE_ON_CLOSE flag removed
        }
        */

        // 실제로는 Pre-Create에서 차단하는 것이 더 안전
        // Actually, blocking in Pre-Create is safer
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## 27.3 [실습] 보호된 파일 삭제 방지

### 27.3.1 완전한 삭제 차단 드라이버

```c
// DeleteBlocker.c - 삭제 차단 Minifilter 드라이버
// DeleteBlocker.c - Delete Blocking Minifilter Driver

#include <fltKernel.h>
#include <ntstrsafe.h>

#pragma prefast(disable:__WARNING_ENCODE_MEMBER_FUNCTION_POINTER)

// 전역 변수
// Global variables
PFLT_FILTER g_FilterHandle = NULL;

// 보호 대상 확장자
// Protected extensions
static const UNICODE_STRING g_ProtectedExtensions[] = {
    RTL_CONSTANT_STRING(L".txt"),
    RTL_CONSTANT_STRING(L".doc"),
    RTL_CONSTANT_STRING(L".docx"),
    RTL_CONSTANT_STRING(L".xls"),
    RTL_CONSTANT_STRING(L".xlsx"),
    RTL_CONSTANT_STRING(L".pdf"),
};
#define PROTECTED_EXT_COUNT (sizeof(g_ProtectedExtensions) / sizeof(g_ProtectedExtensions[0]))

// 휴지통 패턴
// Recycle Bin patterns
static const WCHAR* g_RecycleBinPatterns[] = {
    L"$RECYCLE.BIN",
    L"$Recycle.Bin",
    L"RECYCLER",
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

// 휴지통 경로 확인
// Check Recycle Bin path
BOOLEAN IsRecycleBinPath(_In_ PCUNICODE_STRING Path)
{
    WCHAR upperPath[520];
    ULONG copyLen;

    if (Path == NULL || Path->Length == 0) {
        return FALSE;
    }

    copyLen = min(Path->Length / sizeof(WCHAR), 519);
    RtlCopyMemory(upperPath, Path->Buffer, copyLen * sizeof(WCHAR));
    upperPath[copyLen] = L'\0';
    _wcsupr(upperPath);

    for (int i = 0; g_RecycleBinPatterns[i] != NULL; i++) {
        if (wcsstr(upperPath, g_RecycleBinPatterns[i]) != NULL) {
            return TRUE;
        }
    }

    return FALSE;
}

// 예외 프로세스 확인
// Check exempt process
BOOLEAN IsExemptProcess(VOID)
{
    HANDLE processId = PsGetCurrentProcessId();

    // System 프로세스
    // System process
    if (HandleToULong(processId) <= 4) {
        return TRUE;
    }

    PEPROCESS process = PsGetCurrentProcess();
    PCHAR imageName = PsGetProcessImageFileName(process);

    if (imageName != NULL) {
        // 일부 시스템 프로세스 허용
        // Allow some system processes
        if (_stricmp(imageName, "System") == 0 ||
            _stricmp(imageName, "svchost.exe") == 0) {
            return TRUE;
        }
    }

    return FALSE;
}

// Pre-Create 콜백 (DELETE_ON_CLOSE, SUPERSEDE 차단)
// Pre-Create callback (block DELETE_ON_CLOSE, SUPERSEDE)
FLT_PREOP_CALLBACK_STATUS
PreCreateCallback(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    ULONG createOptions;
    ULONG createDisposition;
    BOOLEAN shouldBlock = FALSE;
    BOOLEAN isDeleteOperation = FALSE;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 예외 프로세스
    // Exempt process
    if (IsExemptProcess()) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    createOptions = Data->Iopb->Parameters.Create.Options & FILE_VALID_OPTION_FLAGS;
    createDisposition = (Data->Iopb->Parameters.Create.Options >> 24) & 0xFF;

    // 삭제 관련 작업 확인
    // Check delete-related operations
    if (createOptions & FILE_DELETE_ON_CLOSE) {
        isDeleteOperation = TRUE;
    }

    if (createDisposition == FILE_SUPERSEDE) {
        isDeleteOperation = TRUE;
    }

    if (!isDeleteOperation) {
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

    FltParseFileNameInformation(nameInfo);

    // 보호 대상 확인
    // Check protection target
    if (IsProtectedExtension(&nameInfo->Name)) {
        shouldBlock = TRUE;

        DbgPrint("[DeleteBlocker] DELETE_ON_CLOSE/SUPERSEDE 차단:\n");
        // DELETE_ON_CLOSE/SUPERSEDE blocked
        DbgPrint("[DeleteBlocker]   파일: %wZ\n", &nameInfo->Name);
        // File
        DbgPrint("[DeleteBlocker]   옵션: 0x%08X, Disposition: %lu\n",
                 createOptions, createDisposition);
        // Options, Disposition
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
    FILE_INFORMATION_CLASS infoClass;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    BOOLEAN shouldBlock = FALSE;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 예외 프로세스
    // Exempt process
    if (IsExemptProcess()) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 1. FileDisposition 처리
    // 1. Handle FileDisposition
    if (infoClass == FileDispositionInformation) {
        PFILE_DISPOSITION_INFORMATION dispInfo =
            (PFILE_DISPOSITION_INFORMATION)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        if (!dispInfo->DeleteFile) {
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
        }

        status = FltGetFileNameInformation(
            Data,
            FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
            &nameInfo
        );

        if (NT_SUCCESS(status)) {
            FltParseFileNameInformation(nameInfo);

            if (IsProtectedExtension(&nameInfo->Name)) {
                shouldBlock = TRUE;

                DbgPrint("[DeleteBlocker] FileDisposition 삭제 차단:\n");
                // FileDisposition delete blocked
                DbgPrint("[DeleteBlocker]   파일: %wZ\n", &nameInfo->Name);
                // File
            }

            FltReleaseFileNameInformation(nameInfo);
        }
    }
    // 2. FileDispositionEx 처리
    // 2. Handle FileDispositionEx
    else if (infoClass == FileDispositionInformationEx) {
        PFILE_DISPOSITION_INFORMATION_EX dispInfoEx =
            (PFILE_DISPOSITION_INFORMATION_EX)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        if (!(dispInfoEx->Flags & FILE_DISPOSITION_DELETE)) {
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
        }

        status = FltGetFileNameInformation(
            Data,
            FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
            &nameInfo
        );

        if (NT_SUCCESS(status)) {
            FltParseFileNameInformation(nameInfo);

            if (IsProtectedExtension(&nameInfo->Name)) {
                shouldBlock = TRUE;

                DbgPrint("[DeleteBlocker] FileDispositionEx 삭제 차단:\n");
                // FileDispositionEx delete blocked
                DbgPrint("[DeleteBlocker]   파일: %wZ, 플래그: 0x%X\n",
                         &nameInfo->Name, dispInfoEx->Flags);
                // File, Flags
            }

            FltReleaseFileNameInformation(nameInfo);
        }
    }
    // 3. 휴지통 이동 처리
    // 3. Handle Recycle Bin move
    else if (infoClass == FileRenameInformation ||
             infoClass == FileRenameInformationEx) {
        PFILE_RENAME_INFORMATION renameInfo =
            (PFILE_RENAME_INFORMATION)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        // 대상 경로 확인
        // Check target path
        WCHAR targetBuffer[520];
        UNICODE_STRING targetPath;

        ULONG copyLen = min(renameInfo->FileNameLength,
                            sizeof(targetBuffer) - sizeof(WCHAR));
        RtlCopyMemory(targetBuffer, renameInfo->FileName, copyLen);
        targetBuffer[copyLen / sizeof(WCHAR)] = L'\0';

        RtlInitUnicodeString(&targetPath, targetBuffer);

        // 휴지통으로 이동하는지 확인
        // Check if moving to Recycle Bin
        if (IsRecycleBinPath(&targetPath)) {
            status = FltGetFileNameInformation(
                Data,
                FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
                &nameInfo
            );

            if (NT_SUCCESS(status)) {
                FltParseFileNameInformation(nameInfo);

                // 원본이 휴지통이 아닌 경우만 차단
                // Only block if source is not Recycle Bin
                if (!IsRecycleBinPath(&nameInfo->Name) &&
                    IsProtectedExtension(&nameInfo->Name)) {
                    shouldBlock = TRUE;

                    DbgPrint("[DeleteBlocker] 휴지통 이동 차단:\n");
                    // Recycle Bin move blocked
                    DbgPrint("[DeleteBlocker]   원본: %wZ\n", &nameInfo->Name);
                    // Source
                    DbgPrint("[DeleteBlocker]   대상: %ws\n", targetBuffer);
                    // Target
                }

                FltReleaseFileNameInformation(nameInfo);
            }
        }
    }

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
        IRP_MJ_SET_INFORMATION,
        0,
        PreSetInformationCallback,
        NULL
    },
    { IRP_MJ_OPERATION_END }
};

// 필터 언로드
// Filter unload
NTSTATUS FilterUnload(_In_ FLT_FILTER_UNLOAD_FLAGS Flags)
{
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("[DeleteBlocker] 드라이버 언로드\n");
    // Driver unloading

    FltUnregisterFilter(g_FilterHandle);

    return STATUS_SUCCESS;
}

// 인스턴스 설정
// Instance setup
NTSTATUS InstanceSetup(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_SETUP_FLAGS Flags,
    _In_ DEVICE_TYPE VolumeDeviceType,
    _In_ FLT_FILESYSTEM_TYPE VolumeFilesystemType)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    if (VolumeDeviceType == FILE_DEVICE_NETWORK_FILE_SYSTEM) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    if (VolumeFilesystemType == FLT_FSTYPE_CDFS) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    DbgPrint("[DeleteBlocker] 볼륨에 연결됨\n");
    // Attached to volume

    return STATUS_SUCCESS;
}

// 필터 등록
// Filter registration
const FLT_REGISTRATION FilterRegistration = {
    sizeof(FLT_REGISTRATION),
    FLT_REGISTRATION_VERSION,
    0,
    NULL,
    Callbacks,
    FilterUnload,
    InstanceSetup,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL
};

// 드라이버 진입점
// Driver entry point
NTSTATUS DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;

    UNREFERENCED_PARAMETER(RegistryPath);

    DbgPrint("[DeleteBlocker] 드라이버 시작\n");
    // Driver starting

    status = FltRegisterFilter(
        DriverObject,
        &FilterRegistration,
        &g_FilterHandle
    );

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DeleteBlocker] 필터 등록 실패: 0x%X\n", status);
        // Filter registration failed
        return status;
    }

    status = FltStartFiltering(g_FilterHandle);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DeleteBlocker] 필터링 시작 실패: 0x%X\n", status);
        // Filtering start failed
        FltUnregisterFilter(g_FilterHandle);
        return status;
    }

    DbgPrint("[DeleteBlocker] 필터링 시작됨\n");
    // Filtering started

    return STATUS_SUCCESS;
}
```

---

## 27.4 요약

### 핵심 내용 정리

| 항목 | 내용 |
|------|------|
| **삭제 방법** | FileDisposition, DELETE_ON_CLOSE, SUPERSEDE |
| **휴지통 이동** | FileRenameInformation으로 감지 |
| **차단 포인트** | Pre-Create, Pre-SetInformation |
| **반환 값** | STATUS_ACCESS_DENIED |
| **우회 방지** | 모든 삭제 경로 동시 차단 필요 |

### .NET 개발자 참고 사항

```
┌─────────────────────────────────────────────────────────────────┐
│              C# vs C 파일 삭제 처리 비교                         │
├─────────────────────────────────────────────────────────────────┤
│ C# (사용자 모드)                 │ C (커널 모드)                  │
├──────────────────────────────────┼────────────────────────────────┤
│ File.Delete()                    │ FileDispositionInformation    │
│ FileInfo.Delete()                │ IRP_MJ_SET_INFORMATION        │
│ Directory.Delete()               │ FileDispositionInformationEx  │
│ FileShare.Delete                 │ DELETE_ON_CLOSE 플래그        │
│ FileStream(..., Delete)          │ IRP_MJ_CREATE 옵션            │
│ Shell RecycleBin                 │ FileRenameInfo + $Recycle.Bin │
└──────────────────────────────────┴────────────────────────────────┘
```

### 다음 챕터 예고

Chapter 28에서는 **클립보드 및 데이터 유출 방지**를 다룹니다. 커널 모드에서 클립보드 접근 감지의 한계와 파일 복사 감지, 사용자 모드 컴포넌트와의 연동 방법을 학습합니다.

---

## 참고 자료

- [Microsoft Docs: FILE_DISPOSITION_INFORMATION](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/ns-ntddk-_file_disposition_information)
- [Microsoft Docs: IRP_MJ_SET_INFORMATION](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/irp-mj-set-information)
- [Microsoft Docs: File Deletion](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/file-deletion)
