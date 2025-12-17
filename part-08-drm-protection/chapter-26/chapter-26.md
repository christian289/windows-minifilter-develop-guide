# Chapter 26: 파일 이름 변경 및 이동 차단

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
- 파일 이름 변경 작업의 완전한 감지
- 파일 이동(다른 디렉터리로)의 감지 및 차단
- 하드 링크 생성을 통한 우회 방지

---

## 26.1 이름 변경 작업 감지

### 26.1.1 Windows에서의 이름 변경 메커니즘

Windows에서 파일 이름 변경은 다양한 방법으로 수행됩니다. 모든 경로를 이해해야 완전한 차단이 가능합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   파일 이름 변경/이동 경로                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐                                           │
│  │ Win32 API       │                                           │
│  │ MoveFile()      │                                           │
│  │ MoveFileEx()    │                                           │
│  │ ReplaceFile()   │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│           v                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    NT API                                │   │
│  │ NtSetInformationFile(FileRenameInformation)             │   │
│  │ NtSetInformationFile(FileRenameInformationEx)           │   │
│  │ NtSetInformationFile(FileLinkInformation)               │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               │                                 │
│                               v                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              IRP_MJ_SET_INFORMATION                      │   │
│  │                                                          │   │
│  │  FileRenameInformation (class 10)                       │   │
│  │  • 동일 볼륨 이름 변경/이동                               │   │
│  │  • Same-volume rename/move                               │   │
│  │                                                          │   │
│  │  FileRenameInformationEx (class 65)                     │   │
│  │  • 확장 플래그 지원 (POSIX 스타일 등)                     │   │
│  │  • Extended flags support (POSIX style, etc.)           │   │
│  │                                                          │   │
│  │  FileLinkInformation (class 11)                         │   │
│  │  • 하드 링크 생성                                        │   │
│  │  • Hard link creation                                    │   │
│  │                                                          │   │
│  │  FileShortNameInformation (class 40)                    │   │
│  │  • 8.3 짧은 이름 변경                                    │   │
│  │  • 8.3 short name change                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         다른 볼륨 이동 (복사 + 삭제)                       │   │
│  │         Cross-volume move (copy + delete)                │   │
│  │                                                          │   │
│  │  IRP_MJ_CREATE (대상)  →  IRP_MJ_READ (원본)             │   │
│  │                        →  IRP_MJ_WRITE (대상)            │   │
│  │                        →  삭제 (원본)                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 26.1.2 FileRenameInformation 구조체

```c
// 이름 변경 정보 구조체
// Rename information structure
typedef struct _FILE_RENAME_INFORMATION {
    BOOLEAN ReplaceIfExists;    // 대상이 존재하면 교체
                                // Replace if target exists
    HANDLE  RootDirectory;      // 상대 경로의 기준 디렉터리
                                // Root directory for relative path
    ULONG   FileNameLength;     // 파일 이름 길이 (바이트)
                                // File name length in bytes
    WCHAR   FileName[1];        // 새 파일 이름 (가변 길이)
                                // New file name (variable length)
} FILE_RENAME_INFORMATION, *PFILE_RENAME_INFORMATION;

// Windows 10 1709부터 추가된 확장 구조체
// Extended structure added from Windows 10 1709
typedef struct _FILE_RENAME_INFORMATION_EX {
    ULONG   Flags;              // 확장 플래그
                                // Extended flags
    HANDLE  RootDirectory;
    ULONG   FileNameLength;
    WCHAR   FileName[1];
} FILE_RENAME_INFORMATION_EX, *PFILE_RENAME_INFORMATION_EX;

// 확장 플래그 정의
// Extended flag definitions
#define FILE_RENAME_REPLACE_IF_EXISTS       0x00000001
#define FILE_RENAME_POSIX_SEMANTICS         0x00000002
#define FILE_RENAME_SUPPRESS_PIN_STATE_INH  0x00000004
#define FILE_RENAME_SUPPRESS_STORAGE_RESV_INH 0x00000008
#define FILE_RENAME_NO_INCREASE_AVAILABLE_SPACE 0x00000010
#define FILE_RENAME_NO_DECREASE_AVAILABLE_SPACE 0x00000020
#define FILE_RENAME_IGNORE_READONLY_ATTRIBUTE   0x00000040
```

### 26.1.3 이름 변경 감지 구현

```c
// rename_filter.h - 이름 변경 필터 정의
// rename_filter.h - Rename filter definitions

#pragma once
#include <fltKernel.h>

// 이름 변경 정보
// Rename information
typedef struct _RENAME_INFO {
    UNICODE_STRING SourceName;      // 원본 파일 이름
                                    // Source file name
    UNICODE_STRING TargetName;      // 대상 파일 이름
                                    // Target file name
    BOOLEAN ReplaceIfExists;        // 대상 존재 시 교체
                                    // Replace if target exists
    BOOLEAN IsCrossDirectory;       // 다른 디렉터리로 이동
                                    // Moving to different directory
    BOOLEAN IsSameVolume;           // 동일 볼륨 내
                                    // Within same volume
} RENAME_INFO, *PRENAME_INFO;

// 함수 선언
// Function declarations
FLT_PREOP_CALLBACK_STATUS
RenameFilterPreSetInformation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext);

NTSTATUS
GetRenameInfo(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Out_ PRENAME_INFO RenameInfo);
```

```c
// rename_filter.c - 이름 변경 필터 구현
// rename_filter.c - Rename filter implementation

#include "rename_filter.h"
#include "protection_policy.h"
#include "audit_log.h"

// 이름 변경 정보 추출
// Extract rename information
NTSTATUS
GetRenameInfo(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Out_ PRENAME_INFO RenameInfo)
{
    NTSTATUS status;
    PFILE_RENAME_INFORMATION renameInfo = NULL;
    PFILE_RENAME_INFORMATION_EX renameInfoEx = NULL;
    FILE_INFORMATION_CLASS infoClass;
    PFLT_FILE_NAME_INFORMATION sourceNameInfo = NULL;
    PFLT_FILE_NAME_INFORMATION targetNameInfo = NULL;

    RtlZeroMemory(RenameInfo, sizeof(RENAME_INFO));

    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // 정보 버퍼 가져오기
    // Get information buffer
    if (infoClass == FileRenameInformation) {
        renameInfo = (PFILE_RENAME_INFORMATION)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        RenameInfo->ReplaceIfExists = renameInfo->ReplaceIfExists;
        RenameInfo->TargetName.Buffer = renameInfo->FileName;
        RenameInfo->TargetName.Length = (USHORT)renameInfo->FileNameLength;
        RenameInfo->TargetName.MaximumLength = RenameInfo->TargetName.Length;
    }
    else if (infoClass == FileRenameInformationEx) {
        renameInfoEx = (PFILE_RENAME_INFORMATION_EX)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        RenameInfo->ReplaceIfExists =
            (renameInfoEx->Flags & FILE_RENAME_REPLACE_IF_EXISTS) != 0;
        RenameInfo->TargetName.Buffer = renameInfoEx->FileName;
        RenameInfo->TargetName.Length = (USHORT)renameInfoEx->FileNameLength;
        RenameInfo->TargetName.MaximumLength = RenameInfo->TargetName.Length;
    }
    else {
        return STATUS_INVALID_PARAMETER;
    }

    // 원본 파일 이름 가져오기
    // Get source file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &sourceNameInfo
    );

    if (NT_SUCCESS(status)) {
        FltParseFileNameInformation(sourceNameInfo);
        RenameInfo->SourceName = sourceNameInfo->Name;
        FltReleaseFileNameInformation(sourceNameInfo);
    }

    // 대상 파일 이름 분석 (전체 경로 구성)
    // Analyze target file name (construct full path)
    // RootDirectory가 NULL이면 절대 경로
    // If RootDirectory is NULL, it's an absolute path
    if (renameInfo != NULL && renameInfo->RootDirectory != NULL) {
        // 상대 경로 처리
        // Handle relative path
        RenameInfo->IsCrossDirectory = TRUE;
    }
    else if (RenameInfo->TargetName.Length > 0 &&
             RenameInfo->TargetName.Buffer[0] == L'\\') {
        // 절대 경로 - 디렉터리 이동일 수 있음
        // Absolute path - might be directory move
        RenameInfo->IsCrossDirectory = TRUE;
    }
    else {
        // 동일 디렉터리 내 이름 변경
        // Rename within same directory
        RenameInfo->IsCrossDirectory = FALSE;
    }

    RenameInfo->IsSameVolume = TRUE;  // 다른 볼륨 이동은 별도 처리 필요
                                      // Cross-volume move needs separate handling

    return STATUS_SUCCESS;
}

// Pre-SetInformation 콜백 (이름 변경)
// Pre-SetInformation callback (rename)
FLT_PREOP_CALLBACK_STATUS
RenameFilterPreSetInformation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    FILE_INFORMATION_CLASS infoClass;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    PROTECTION_FLAGS protectionFlags;
    BOOLEAN shouldBlock = FALSE;
    RENAME_INFO renameInfo;

    *CompletionContext = NULL;

    // 정보 클래스 확인
    // Check information class
    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // 이름 변경 관련 클래스만 처리
    // Only handle rename-related classes
    if (infoClass != FileRenameInformation &&
        infoClass != FileRenameInformationEx) {
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

    FltParseFileNameInformation(nameInfo);

    // 정책 평가
    // Evaluate policy
    protectionFlags = PolicyEvaluate(
        &nameInfo->Name,
        NULL,
        0,
        PsGetCurrentProcessId()
    );

    if (protectionFlags & PROTECT_RENAME) {
        shouldBlock = TRUE;

        // 상세 정보 추출
        // Extract detailed info
        status = GetRenameInfo(Data, FltObjects, &renameInfo);

        if (NT_SUCCESS(status)) {
            DbgPrint("[Rename] 이름 변경 차단:\n");
            // Rename blocked
            DbgPrint("[Rename]   원본: %wZ\n", &nameInfo->Name);
            // Source
            DbgPrint("[Rename]   대상: %wZ\n", &renameInfo.TargetName);
            // Target
            DbgPrint("[Rename]   교체: %s, 디렉터리변경: %s\n",
                     renameInfo.ReplaceIfExists ? "예" : "아니오",
                     // Yes : No
                     renameInfo.IsCrossDirectory ? "예" : "아니오");
                     // Yes : No
        }

        // 감사 로그
        // Audit log
        AuditLogWrite(
            AUDIT_EVENT_RENAME_BLOCKED,
            &nameInfo->Name,
            PsGetCurrentProcessId(),
            infoClass
        );
    }

    FltReleaseFileNameInformation(nameInfo);

    if (shouldBlock) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### 26.1.4 하드 링크 생성 감지

하드 링크를 통해 보호된 파일을 우회하려는 시도를 차단합니다.

```c
// hardlink_filter.c - 하드 링크 필터 구현
// hardlink_filter.c - Hard link filter implementation

#include <fltKernel.h>
#include "protection_policy.h"

// 하드 링크 정보 구조체
// Hard link information structure
typedef struct _FILE_LINK_INFORMATION {
    BOOLEAN ReplaceIfExists;    // 대상이 존재하면 교체
                                // Replace if target exists
    HANDLE  RootDirectory;      // 상대 경로의 기준 디렉터리
                                // Root directory for relative path
    ULONG   FileNameLength;     // 링크 이름 길이
                                // Link name length
    WCHAR   FileName[1];        // 새 링크 이름
                                // New link name
} FILE_LINK_INFORMATION, *PFILE_LINK_INFORMATION;

// 하드 링크 차단
// Block hard link
FLT_PREOP_CALLBACK_STATUS
HardLinkFilterPre(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    FILE_INFORMATION_CLASS infoClass;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    PFILE_LINK_INFORMATION linkInfo;
    PROTECTION_FLAGS protectionFlags;
    BOOLEAN shouldBlock = FALSE;

    *CompletionContext = NULL;

    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // 하드 링크 생성만 처리
    // Only handle hard link creation
    if (infoClass != FileLinkInformation &&
        infoClass != FileLinkInformationEx) {
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

    FltParseFileNameInformation(nameInfo);

    // 정책 평가 - 원본 파일이 보호 대상인지
    // Policy evaluation - check if source file is protected
    protectionFlags = PolicyEvaluate(
        &nameInfo->Name,
        NULL,
        0,
        PsGetCurrentProcessId()
    );

    if (protectionFlags & PROTECT_RENAME) {
        shouldBlock = TRUE;

        linkInfo = (PFILE_LINK_INFORMATION)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        DbgPrint("[HardLink] 하드 링크 생성 차단:\n");
        // Hard link creation blocked
        DbgPrint("[HardLink]   원본: %wZ\n", &nameInfo->Name);
        // Source
        DbgPrint("[HardLink]   링크 이름 길이: %lu\n", linkInfo->FileNameLength);
        // Link name length

        AuditLogWrite(
            AUDIT_EVENT_RENAME_BLOCKED,
            &nameInfo->Name,
            PsGetCurrentProcessId(),
            FileLinkInformation
        );
    }

    FltReleaseFileNameInformation(nameInfo);

    if (shouldBlock) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 26.2 파일 이동 감지

### 26.2.1 동일 볼륨 이동 vs 다른 볼륨 이동

```
┌─────────────────────────────────────────────────────────────────┐
│                  파일 이동 시나리오 분석                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  동일 볼륨 이동 (Same-Volume Move)                              │
│  ─────────────────────────────────────                          │
│  C:\Folder1\file.txt → C:\Folder2\file.txt                     │
│                                                                 │
│  • IRP_MJ_SET_INFORMATION (FileRenameInformation)              │
│  • 파일 데이터 이동 없음 (메타데이터만 변경)                      │
│    No data movement (only metadata change)                      │
│  • 단일 I/O 작업으로 완료                                        │
│    Completes in single I/O operation                            │
│                                                                 │
│  ┌────────┐   FileRenameInfo   ┌────────┐                      │
│  │ Source │ ───────────────────→ │ Target │                      │
│  │ File   │   (한 번에 처리)    │ Path   │                      │
│  └────────┘   (processed once) └────────┘                      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  다른 볼륨 이동 (Cross-Volume Move)                             │
│  ─────────────────────────────────────                          │
│  C:\Folder\file.txt → D:\Folder\file.txt                       │
│                                                                 │
│  • 복사 + 삭제 시퀀스로 구현                                     │
│    Implemented as copy + delete sequence                        │
│  • 여러 I/O 작업으로 구성                                        │
│    Composed of multiple I/O operations                          │
│                                                                 │
│  ┌────────┐                    ┌────────┐                      │
│  │ Source │   1. CREATE        │ Target │                      │
│  │ (C:)   │ ───────────────────→ │ (D:)   │                      │
│  └────┬───┘                    └────┬───┘                      │
│       │         2. READ             │                           │
│       ├────────────────────────────→│                           │
│       │         3. WRITE            │                           │
│       ├────────────────────────────→│                           │
│       │         4. DELETE           │                           │
│       └──────────────────────────────                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 26.2.2 이동 대상 경로 분석

```c
// move_detection.c - 이동 감지 구현
// move_detection.c - Move detection implementation

#include <fltKernel.h>

// 경로 정보 구조체
// Path information structure
typedef struct _PATH_INFO {
    UNICODE_STRING FullPath;        // 전체 경로
                                    // Full path
    UNICODE_STRING Volume;          // 볼륨 (C:, D: 등)
                                    // Volume (C:, D:, etc.)
    UNICODE_STRING Directory;       // 디렉터리
                                    // Directory
    UNICODE_STRING FileName;        // 파일명
                                    // File name
    UNICODE_STRING Extension;       // 확장자
                                    // Extension
} PATH_INFO, *PPATH_INFO;

// 경로 파싱
// Parse path
NTSTATUS ParsePath(
    _In_ PCUNICODE_STRING FullPath,
    _Out_ PPATH_INFO PathInfo)
{
    RtlZeroMemory(PathInfo, sizeof(PATH_INFO));

    PathInfo->FullPath = *FullPath;

    // 볼륨 추출 (예: \Device\HarddiskVolume1)
    // Extract volume (e.g., \Device\HarddiskVolume1)
    // 실제 구현에서는 FltGetVolumeName 등 사용
    // In real implementation, use FltGetVolumeName, etc.

    // 디렉터리와 파일명 분리
    // Separate directory and filename
    for (USHORT i = FullPath->Length / sizeof(WCHAR); i > 0; i--) {
        if (FullPath->Buffer[i - 1] == L'\\') {
            // 파일명
            // Filename
            PathInfo->FileName.Buffer = &FullPath->Buffer[i];
            PathInfo->FileName.Length = FullPath->Length - (i * sizeof(WCHAR));
            PathInfo->FileName.MaximumLength = PathInfo->FileName.Length;

            // 디렉터리
            // Directory
            PathInfo->Directory.Buffer = FullPath->Buffer;
            PathInfo->Directory.Length = (i - 1) * sizeof(WCHAR);
            PathInfo->Directory.MaximumLength = PathInfo->Directory.Length;

            break;
        }
    }

    // 확장자 추출
    // Extract extension
    for (USHORT i = PathInfo->FileName.Length / sizeof(WCHAR); i > 0; i--) {
        if (PathInfo->FileName.Buffer[i - 1] == L'.') {
            PathInfo->Extension.Buffer = &PathInfo->FileName.Buffer[i - 1];
            PathInfo->Extension.Length = PathInfo->FileName.Length -
                                         ((i - 1) * sizeof(WCHAR));
            PathInfo->Extension.MaximumLength = PathInfo->Extension.Length;
            break;
        }
    }

    return STATUS_SUCCESS;
}

// 이동인지 이름변경인지 판별
// Determine if it's a move or rename
typedef enum _OPERATION_TYPE {
    OPERATION_RENAME,           // 이름만 변경 (동일 디렉터리)
                                // Name change only (same directory)
    OPERATION_MOVE_SAME_VOLUME, // 동일 볼륨 이동
                                // Same volume move
    OPERATION_MOVE_CROSS_VOLUME // 다른 볼륨 이동 (감지 어려움)
                                // Cross volume move (difficult to detect)
} OPERATION_TYPE;

OPERATION_TYPE DetermineOperationType(
    _In_ PCUNICODE_STRING SourcePath,
    _In_ PCUNICODE_STRING TargetPath)
{
    PATH_INFO sourceInfo, targetInfo;

    ParsePath(SourcePath, &sourceInfo);
    ParsePath(TargetPath, &targetInfo);

    // 볼륨 비교 (간단한 비교)
    // Compare volumes (simple comparison)
    // 실제 구현에서는 FltGetVolumeFromInstance 등으로 확인
    // In real implementation, verify with FltGetVolumeFromInstance, etc.

    // 디렉터리 비교
    // Compare directories
    if (RtlEqualUnicodeString(&sourceInfo.Directory, &targetInfo.Directory, TRUE)) {
        return OPERATION_RENAME;
    }

    return OPERATION_MOVE_SAME_VOLUME;
}

// 이동 차단 정책 확인
// Check move blocking policy
BOOLEAN ShouldBlockMove(
    _In_ PCUNICODE_STRING SourcePath,
    _In_ PCUNICODE_STRING TargetPath,
    _In_ PROTECTION_FLAGS Flags)
{
    // PROTECT_MOVE 플래그 확인
    // Check PROTECT_MOVE flag
    if (!(Flags & PROTECT_MOVE)) {
        // PROTECT_RENAME만 있으면 동일 디렉터리 이름변경만 차단
        // If only PROTECT_RENAME, only block same-directory rename
        if (Flags & PROTECT_RENAME) {
            OPERATION_TYPE opType = DetermineOperationType(SourcePath, TargetPath);
            return (opType == OPERATION_RENAME);
        }
        return FALSE;
    }

    // 모든 이동/이름변경 차단
    // Block all move/rename
    return TRUE;
}
```

### 26.2.3 보호 영역 이탈 감지

보호된 디렉터리에서 비보호 영역으로 이동하는 것을 차단합니다.

```c
// area_protection.c - 영역 보호 구현
// area_protection.c - Area protection implementation

#include <fltKernel.h>

// 보호 영역 정의
// Protected area definitions
typedef struct _PROTECTED_AREA {
    UNICODE_STRING Path;            // 보호 경로
                                    // Protected path
    WCHAR PathBuffer[260];          // 경로 버퍼
                                    // Path buffer
    BOOLEAN BlockExitOnly;          // 이탈만 차단 (진입은 허용)
                                    // Block exit only (allow entry)
    BOOLEAN BlockEntryOnly;         // 진입만 차단 (이탈은 허용)
                                    // Block entry only (allow exit)
} PROTECTED_AREA, *PPROTECTED_AREA;

#define MAX_PROTECTED_AREAS 16

static PROTECTED_AREA g_ProtectedAreas[MAX_PROTECTED_AREAS];
static ULONG g_ProtectedAreaCount = 0;
static FAST_MUTEX g_AreaLock;

// 초기화
// Initialize
NTSTATUS ProtectedAreaInitialize(VOID)
{
    RtlZeroMemory(g_ProtectedAreas, sizeof(g_ProtectedAreas));
    ExInitializeFastMutex(&g_AreaLock);
    g_ProtectedAreaCount = 0;
    return STATUS_SUCCESS;
}

// 보호 영역 추가
// Add protected area
NTSTATUS ProtectedAreaAdd(
    _In_ PCWSTR Path,
    _In_ BOOLEAN BlockExitOnly,
    _In_ BOOLEAN BlockEntryOnly)
{
    NTSTATUS status = STATUS_SUCCESS;

    ExAcquireFastMutex(&g_AreaLock);

    if (g_ProtectedAreaCount >= MAX_PROTECTED_AREAS) {
        status = STATUS_INSUFFICIENT_RESOURCES;
    }
    else {
        PPROTECTED_AREA area = &g_ProtectedAreas[g_ProtectedAreaCount];

        wcscpy_s(area->PathBuffer, 260, Path);
        RtlInitUnicodeString(&area->Path, area->PathBuffer);
        area->BlockExitOnly = BlockExitOnly;
        area->BlockEntryOnly = BlockEntryOnly;

        g_ProtectedAreaCount++;
    }

    ExReleaseFastMutex(&g_AreaLock);

    return status;
}

// 경로가 보호 영역 내인지 확인
// Check if path is within protected area
BOOLEAN IsInProtectedArea(
    _In_ PCUNICODE_STRING Path,
    _Out_opt_ PPROTECTED_AREA* MatchedArea)
{
    BOOLEAN inArea = FALSE;

    ExAcquireFastMutex(&g_AreaLock);

    for (ULONG i = 0; i < g_ProtectedAreaCount; i++) {
        if (RtlPrefixUnicodeString(&g_ProtectedAreas[i].Path, Path, TRUE)) {
            inArea = TRUE;
            if (MatchedArea != NULL) {
                *MatchedArea = &g_ProtectedAreas[i];
            }
            break;
        }
    }

    ExReleaseFastMutex(&g_AreaLock);

    return inArea;
}

// 이동이 허용되는지 확인
// Check if move is allowed
typedef enum _MOVE_DECISION {
    MOVE_ALLOW,             // 허용
                            // Allow
    MOVE_BLOCK_EXIT,        // 보호 영역 이탈 차단
                            // Block exit from protected area
    MOVE_BLOCK_ENTRY,       // 보호 영역 진입 차단
                            // Block entry to protected area
    MOVE_BLOCK_BOTH         // 이동 자체 차단
                            // Block move itself
} MOVE_DECISION;

MOVE_DECISION CheckMovePermission(
    _In_ PCUNICODE_STRING SourcePath,
    _In_ PCUNICODE_STRING TargetPath)
{
    PPROTECTED_AREA sourceArea = NULL;
    PPROTECTED_AREA targetArea = NULL;
    BOOLEAN sourceInProtected = IsInProtectedArea(SourcePath, &sourceArea);
    BOOLEAN targetInProtected = IsInProtectedArea(TargetPath, &targetArea);

    // 보호 영역 → 비보호 영역 (이탈)
    // Protected area → Unprotected area (exit)
    if (sourceInProtected && !targetInProtected) {
        if (sourceArea != NULL && sourceArea->BlockExitOnly) {
            DbgPrint("[Area] 보호 영역 이탈 차단: %wZ → %wZ\n",
                     SourcePath, TargetPath);
            // Protected area exit blocked
            return MOVE_BLOCK_EXIT;
        }
    }

    // 비보호 영역 → 보호 영역 (진입)
    // Unprotected area → Protected area (entry)
    if (!sourceInProtected && targetInProtected) {
        if (targetArea != NULL && targetArea->BlockEntryOnly) {
            DbgPrint("[Area] 보호 영역 진입 차단: %wZ → %wZ\n",
                     SourcePath, TargetPath);
            // Protected area entry blocked
            return MOVE_BLOCK_ENTRY;
        }
    }

    return MOVE_ALLOW;
}
```

---

## 26.3 [실습] 보호된 파일 이름 변경 차단

### 26.3.1 완전한 이름 변경/이동 차단 드라이버

```c
// RenameBlocker.c - 이름 변경/이동 차단 Minifilter
// RenameBlocker.c - Rename/Move Blocking Minifilter

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
    RTL_CONSTANT_STRING(L".docx"),
    RTL_CONSTANT_STRING(L".xlsx"),
    RTL_CONSTANT_STRING(L".pdf"),
};
#define PROTECTED_EXT_COUNT (sizeof(g_ProtectedExtensions) / sizeof(g_ProtectedExtensions[0]))

// 보호 디렉터리 (예시)
// Protected directories (example)
static const UNICODE_STRING g_ProtectedDirs[] = {
    RTL_CONSTANT_STRING(L"\\Protected\\"),
    RTL_CONSTANT_STRING(L"\\Confidential\\"),
};
#define PROTECTED_DIR_COUNT (sizeof(g_ProtectedDirs) / sizeof(g_ProtectedDirs[0]))

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

// 보호 디렉터리 검사
// Check protected directory
BOOLEAN IsInProtectedDirectory(_In_ PCUNICODE_STRING FilePath)
{
    WCHAR upperPath[520];
    UNICODE_STRING upperPathStr;

    // 대문자 변환을 위한 복사
    // Copy for uppercase conversion
    if (FilePath->Length >= sizeof(upperPath)) {
        return FALSE;
    }

    RtlCopyMemory(upperPath, FilePath->Buffer, FilePath->Length);
    upperPath[FilePath->Length / sizeof(WCHAR)] = L'\0';

    // 대문자 변환
    // Convert to uppercase
    _wcsupr(upperPath);

    upperPathStr.Buffer = upperPath;
    upperPathStr.Length = FilePath->Length;
    upperPathStr.MaximumLength = sizeof(upperPath);

    for (ULONG i = 0; i < PROTECTED_DIR_COUNT; i++) {
        // 경로에 보호 디렉터리가 포함되어 있는지 확인
        // Check if path contains protected directory
        PWCHAR found = wcsstr(upperPath,
                              g_ProtectedDirs[i].Buffer);
        if (found != NULL) {
            return TRUE;
        }
    }

    return FALSE;
}

// 예외 프로세스 확인
// Check exempt process
BOOLEAN IsExemptProcess(VOID)
{
    PEPROCESS process = PsGetCurrentProcess();
    PCHAR imageName = PsGetProcessImageFileName(process);

    if (imageName != NULL) {
        // 시스템 프로세스 허용
        // Allow system processes
        if (_stricmp(imageName, "System") == 0) {
            return TRUE;
        }
        // explorer.exe의 일부 작업 허용 (선택적)
        // Allow some explorer.exe operations (optional)
        // if (_stricmp(imageName, "explorer.exe") == 0) {
        //     return TRUE;
        // }
    }

    return FALSE;
}

// 대상 경로 추출
// Extract target path
NTSTATUS GetTargetPath(
    _In_ PFLT_CALLBACK_DATA Data,
    _Out_ PUNICODE_STRING TargetPath,
    _In_ ULONG BufferSize,
    _Out_ PWCHAR Buffer)
{
    FILE_INFORMATION_CLASS infoClass;
    PFILE_RENAME_INFORMATION renameInfo;

    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    if (infoClass == FileRenameInformation ||
        infoClass == FileRenameInformationEx) {
        renameInfo = (PFILE_RENAME_INFORMATION)
            Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

        ULONG copyLen = min(renameInfo->FileNameLength, BufferSize - sizeof(WCHAR));
        RtlCopyMemory(Buffer, renameInfo->FileName, copyLen);
        Buffer[copyLen / sizeof(WCHAR)] = L'\0';

        RtlInitUnicodeString(TargetPath, Buffer);
        return STATUS_SUCCESS;
    }

    return STATUS_INVALID_PARAMETER;
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
    WCHAR targetBuffer[260];
    UNICODE_STRING targetPath;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // 정보 클래스 확인
    // Check information class
    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // 처리 대상 클래스
    // Classes to handle
    switch (infoClass) {
        case FileRenameInformation:
        case FileRenameInformationEx:
        case FileLinkInformation:
        case FileLinkInformationEx:
        case FileShortNameInformation:
            break;

        default:
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

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

    // 원본 파일 이름 가져오기
    // Get source file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    FltParseFileNameInformation(nameInfo);

    // 보호 조건 확인
    // Check protection conditions
    BOOLEAN sourceProtected =
        IsProtectedExtension(&nameInfo->Name) ||
        IsInProtectedDirectory(&nameInfo->Name);

    if (sourceProtected) {
        shouldBlock = TRUE;

        // 대상 경로도 로깅
        // Also log target path
        RtlZeroMemory(targetBuffer, sizeof(targetBuffer));

        if (infoClass == FileRenameInformation ||
            infoClass == FileRenameInformationEx) {
            GetTargetPath(Data, &targetPath, sizeof(targetBuffer), targetBuffer);

            DbgPrint("[RenameBlocker] 이름 변경/이동 차단:\n");
            // Rename/move blocked
            DbgPrint("[RenameBlocker]   원본: %wZ\n", &nameInfo->Name);
            // Source
            DbgPrint("[RenameBlocker]   대상: %ws\n", targetBuffer);
            // Target
            DbgPrint("[RenameBlocker]   클래스: %d, PID: %lu\n",
                     infoClass,
                     HandleToULong(PsGetCurrentProcessId()));
            // Class, PID
        }
        else if (infoClass == FileLinkInformation ||
                 infoClass == FileLinkInformationEx) {
            DbgPrint("[RenameBlocker] 하드 링크 생성 차단:\n");
            // Hard link creation blocked
            DbgPrint("[RenameBlocker]   원본: %wZ\n", &nameInfo->Name);
            // Source
        }
        else if (infoClass == FileShortNameInformation) {
            DbgPrint("[RenameBlocker] 짧은 이름 변경 차단:\n");
            // Short name change blocked
            DbgPrint("[RenameBlocker]   파일: %wZ\n", &nameInfo->Name);
            // File
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

// 콜백 등록
// Callback registration
const FLT_OPERATION_REGISTRATION Callbacks[] = {
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

    DbgPrint("[RenameBlocker] 드라이버 언로드\n");
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

    if (VolumeFilesystemType == FLT_FSTYPE_CDFS ||
        VolumeFilesystemType == FLT_FSTYPE_RDPDR) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    DbgPrint("[RenameBlocker] 볼륨에 연결됨\n");
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

    DbgPrint("[RenameBlocker] 드라이버 시작\n");
    // Driver starting

    status = FltRegisterFilter(
        DriverObject,
        &FilterRegistration,
        &g_FilterHandle
    );

    if (!NT_SUCCESS(status)) {
        DbgPrint("[RenameBlocker] 필터 등록 실패: 0x%X\n", status);
        // Filter registration failed
        return status;
    }

    status = FltStartFiltering(g_FilterHandle);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[RenameBlocker] 필터링 시작 실패: 0x%X\n", status);
        // Filtering start failed
        FltUnregisterFilter(g_FilterHandle);
        return status;
    }

    DbgPrint("[RenameBlocker] 필터링 시작됨\n");
    // Filtering started

    return STATUS_SUCCESS;
}
```

### 26.3.2 테스트 시나리오

```
┌─────────────────────────────────────────────────────────────────┐
│                      테스트 시나리오                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 기본 이름 변경 차단 테스트                                   │
│     Basic rename blocking test                                  │
│     ─────────────────────────                                   │
│     rename test.txt test2.txt                                   │
│     → STATUS_ACCESS_DENIED 예상                                 │
│       Expected STATUS_ACCESS_DENIED                             │
│                                                                 │
│  2. 디렉터리 이동 차단 테스트                                    │
│     Directory move blocking test                                │
│     ─────────────────────────                                   │
│     move C:\Protected\doc.txt C:\Temp\doc.txt                  │
│     → STATUS_ACCESS_DENIED 예상                                 │
│       Expected STATUS_ACCESS_DENIED                             │
│                                                                 │
│  3. 하드 링크 생성 차단 테스트                                   │
│     Hard link creation blocking test                            │
│     ─────────────────────────────                               │
│     mklink /H link.txt original.txt                            │
│     → STATUS_ACCESS_DENIED 예상 (보호된 파일의 경우)             │
│       Expected STATUS_ACCESS_DENIED (for protected files)       │
│                                                                 │
│  4. 짧은 이름 변경 차단 테스트                                   │
│     Short name change blocking test                             │
│     ─────────────────────────────                               │
│     fsutil file setshortname file.txt F~1.TXT                  │
│     → STATUS_ACCESS_DENIED 예상                                 │
│       Expected STATUS_ACCESS_DENIED                             │
│                                                                 │
│  5. 허용 프로세스 테스트                                        │
│     Exempt process test                                         │
│     ──────────────────────                                      │
│     예외 프로세스에서 rename 실행                                │
│     Execute rename from exempt process                          │
│     → 성공 예상                                                  │
│       Expected success                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 26.4 우회 방지 전략

### 26.4.1 알려진 우회 기법과 대응

```c
// anti_bypass.c - 우회 방지 구현
// anti_bypass.c - Anti-bypass implementation

#include <fltKernel.h>

/*
 * 우회 기법과 대응 방법
 * Bypass techniques and countermeasures
 *
 * 1. 볼륨 간 복사 후 삭제
 *    Cross-volume copy then delete
 *    → 삭제 차단 (Chapter 27)과 함께 사용
 *      Use together with delete blocking (Chapter 27)
 *
 * 2. 임시 파일로 저장 후 이름 변경
 *    Save to temp file then rename
 *    → 임시 파일 패턴 감지
 *      Detect temp file patterns
 *
 * 3. 프로세스 주입으로 예외 프로세스 사용
 *    Process injection to use exempt process
 *    → 컨텍스트 기반 검증 강화
 *      Strengthen context-based verification
 *
 * 4. 직접 NTFS 메타데이터 수정
 *    Direct NTFS metadata modification
 *    → IRP_MJ_FILE_SYSTEM_CONTROL 필터링
 *      Filter IRP_MJ_FILE_SYSTEM_CONTROL
 *
 * 5. 심볼릭 링크 사용
 *    Using symbolic links
 *    → 심볼릭 링크 생성도 차단
 *      Also block symbolic link creation
 */

// 임시 파일 패턴 감지
// Detect temp file patterns
BOOLEAN IsTempFilePattern(_In_ PCUNICODE_STRING FileName)
{
    static const UNICODE_STRING tempPatterns[] = {
        RTL_CONSTANT_STRING(L".tmp"),
        RTL_CONSTANT_STRING(L".temp"),
        RTL_CONSTANT_STRING(L"~$"),       // Office 임시 파일
                                          // Office temp file
        RTL_CONSTANT_STRING(L".~"),
    };

    for (ULONG i = 0; i < sizeof(tempPatterns)/sizeof(tempPatterns[0]); i++) {
        // 접미사 확인
        // Check suffix
        if (RtlSuffixUnicodeString(&tempPatterns[i], FileName, TRUE)) {
            return TRUE;
        }

        // 접두사 확인 (파일명 부분에서)
        // Check prefix (in filename part)
        // ...
    }

    // 임시 디렉터리 확인
    // Check temp directory
    UNICODE_STRING tempDir = RTL_CONSTANT_STRING(L"\\TEMP\\");
    UNICODE_STRING localTemp = RTL_CONSTANT_STRING(L"\\Local\\Temp\\");

    if (wcsstr(FileName->Buffer, tempDir.Buffer) != NULL ||
        wcsstr(FileName->Buffer, localTemp.Buffer) != NULL) {
        return TRUE;
    }

    return FALSE;
}

// 의심스러운 이름 변경 패턴 감지
// Detect suspicious rename patterns
BOOLEAN IsSuspiciousRename(
    _In_ PCUNICODE_STRING SourceName,
    _In_ PCUNICODE_STRING TargetName)
{
    // 1. 확장자 변경으로 보호 우회 시도
    // 1. Attempting bypass via extension change
    UNICODE_STRING sourceExt = {0}, targetExt = {0};

    // 확장자 추출
    // Extract extensions
    for (USHORT i = SourceName->Length / sizeof(WCHAR); i > 0; i--) {
        if (SourceName->Buffer[i-1] == L'.') {
            sourceExt.Buffer = &SourceName->Buffer[i-1];
            sourceExt.Length = SourceName->Length - ((i-1) * sizeof(WCHAR));
            break;
        }
    }

    for (USHORT i = TargetName->Length / sizeof(WCHAR); i > 0; i--) {
        if (TargetName->Buffer[i-1] == L'.') {
            targetExt.Buffer = &TargetName->Buffer[i-1];
            targetExt.Length = TargetName->Length - ((i-1) * sizeof(WCHAR));
            break;
        }
    }

    // 보호된 확장자 → 비보호 확장자로 변경 시도
    // Attempting to change from protected to unprotected extension
    if (sourceExt.Length > 0 && targetExt.Length > 0) {
        if (IsProtectedExtension(&sourceExt) &&
            !IsProtectedExtension(&targetExt)) {
            DbgPrint("[AntiBypass] 의심스러운 확장자 변경 감지: %wZ → %wZ\n",
                     &sourceExt, &targetExt);
            // Suspicious extension change detected
            return TRUE;
        }
    }

    // 2. 임시 파일로 이름 변경
    // 2. Renaming to temp file
    if (IsTempFilePattern(TargetName)) {
        DbgPrint("[AntiBypass] 임시 파일로 이름 변경 시도 감지\n");
        // Temp file rename attempt detected
        return TRUE;
    }

    return FALSE;
}
```

---

## 26.5 요약

### 핵심 내용 정리

| 항목 | 내용 |
|------|------|
| **이름 변경 클래스** | FileRenameInformation, FileRenameInformationEx |
| **하드 링크** | FileLinkInformation으로 감지 |
| **동일 볼륨 이동** | 이름 변경과 동일한 IRP로 처리 |
| **다른 볼륨 이동** | 복사+삭제로 구현됨, 삭제 차단 필요 |
| **우회 방지** | 확장자 변경, 임시 파일 패턴 감지 |

### .NET 개발자 참고 사항

```
┌─────────────────────────────────────────────────────────────────┐
│           C# vs C 파일 이름 변경 처리 비교                       │
├─────────────────────────────────────────────────────────────────┤
│ C# (사용자 모드)                 │ C (커널 모드)                  │
├──────────────────────────────────┼────────────────────────────────┤
│ File.Move()                      │ FileRenameInformation 차단    │
│ FileInfo.MoveTo()                │ IRP_MJ_SET_INFORMATION 콜백   │
│ Directory.Move()                 │ 동일 볼륨/다른 볼륨 구분 필요  │
│ File.CreateHardLink (P/Invoke)   │ FileLinkInformation 차단      │
│ Path.GetDirectoryName()          │ 직접 경로 파싱                │
│ try-catch 예외 처리              │ NTSTATUS 반환                 │
└──────────────────────────────────┴────────────────────────────────┘
```

### 다음 챕터 예고

Chapter 27에서는 **파일 삭제 차단**을 다룹니다. 다양한 삭제 방법(직접 삭제, Disposition 설정, 휴지통 이동 등)을 모두 감지하고 차단하는 방법을 구현합니다.

---

## 참고 자료

- [Microsoft Docs: FILE_RENAME_INFORMATION](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntifs/ns-ntifs-_file_rename_information)
- [Microsoft Docs: IRP_MJ_SET_INFORMATION (Rename)](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/irp-mj-set-information)
- [Microsoft Docs: Hard Links and Junctions](https://docs.microsoft.com/en-us/windows/win32/fileio/hard-links-and-junctions)
