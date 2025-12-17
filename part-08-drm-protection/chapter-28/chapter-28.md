# Chapter 28: 클립보드 및 데이터 유출 방지

## 난이도: ⭐⭐⭐⭐⭐ (고급)

## 학습 목표
- 커널 모드에서 클립보드 보호의 근본적 한계 이해
- 파일 복사 감지 및 의심스러운 활동 모니터링
- 사용자 모드 연계를 통한 완전한 DLP 솔루션 설계

---

## 28.1 클립보드 접근 감지의 한계

### 28.1.1 클립보드의 본질

Windows 클립보드는 사용자 모드의 GUI 서브시스템 기능으로, 커널 드라이버에서 직접 접근하거나 모니터링할 수 없습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   클립보드 아키텍처                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    사용자 모드 (Ring 3)                   │  │
│  │                                                          │  │
│  │  ┌──────────┐     ┌──────────────┐     ┌──────────┐     │  │
│  │  │ App A    │ ←──→│  Clipboard   │←───→│ App B    │     │  │
│  │  │ (Ctrl+C) │     │  (User32.dll)│     │ (Ctrl+V) │     │  │
│  │  └──────────┘     └──────────────┘     └──────────┘     │  │
│  │                          │                               │  │
│  │                          │ Win32 API                     │  │
│  │                          │ OpenClipboard()               │  │
│  │                          │ GetClipboardData()            │  │
│  │                          │ SetClipboardData()            │  │
│  │                          │ CloseClipboard()              │  │
│  │                          │                               │  │
│  └──────────────────────────┼───────────────────────────────┘  │
│                             │                                   │
│  ────────────────────────────────────────────────────────────  │
│                     커널/사용자 경계                            │
│                     Kernel/User boundary                        │
│  ────────────────────────────────────────────────────────────  │
│                             │                                   │
│  ┌──────────────────────────▼───────────────────────────────┐  │
│  │                    커널 모드 (Ring 0)                     │  │
│  │                                                          │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │              Minifilter Driver                    │   │  │
│  │  │                                                   │   │  │
│  │  │   ⚠️ 클립보드 접근 불가                           │   │  │
│  │  │   ⚠️ Cannot access clipboard                     │   │  │
│  │  │                                                   │   │  │
│  │  │   ✓ 파일 I/O만 필터링 가능                        │   │  │
│  │  │   ✓ Can only filter file I/O                     │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 28.1.2 커널 모드의 한계

```
┌─────────────────────────────────────────────────────────────────┐
│              커널 모드에서 보호할 수 없는 것들                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 클립보드 복사/붙여넣기                                       │
│     Clipboard copy/paste                                        │
│     ─────────────────────                                       │
│     • Ctrl+C, Ctrl+V 감지 불가                                  │
│       Cannot detect Ctrl+C, Ctrl+V                              │
│     • 드래그 앤 드롭 감지 어려움                                 │
│       Difficult to detect drag and drop                         │
│                                                                 │
│  2. 스크린 캡처                                                  │
│     Screen capture                                              │
│     ─────────────────                                           │
│     • PrintScreen 키 감지 불가                                  │
│       Cannot detect PrintScreen key                             │
│     • 캡처 도구 실행 감지만 가능                                 │
│       Can only detect capture tool execution                    │
│                                                                 │
│  3. 원격 데스크톱 전송                                           │
│     Remote desktop transfer                                     │
│     ────────────────────                                        │
│     • RDP 클립보드 리디렉션                                      │
│       RDP clipboard redirection                                 │
│     • 원격 세션 파일 전송                                        │
│       Remote session file transfer                              │
│                                                                 │
│  4. 네트워크 전송                                                │
│     Network transfer                                            │
│     ─────────────────                                           │
│     • HTTP/HTTPS 업로드                                         │
│       HTTP/HTTPS upload                                         │
│     • 이메일 첨부                                                │
│       Email attachment                                          │
│     • 클라우드 동기화                                            │
│       Cloud synchronization                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 28.1.3 사용자 모드 연계 필요성

```
┌─────────────────────────────────────────────────────────────────┐
│                완전한 DLP 솔루션의 구성 요소                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              사용자 모드 컴포넌트 (필수)                   │  │
│  │                                                          │  │
│  │  ┌────────────────┐  ┌────────────────┐                 │  │
│  │  │ 클립보드 후킹   │  │ 키보드 후킹    │                 │  │
│  │  │ Clipboard Hook │  │ Keyboard Hook  │                 │  │
│  │  │                │  │                │                 │  │
│  │  │ SetClipboard   │  │ PrintScreen    │                 │  │
│  │  │ Viewer()       │  │ 감지           │                 │  │
│  │  └────────────────┘  └────────────────┘                 │  │
│  │                                                          │  │
│  │  ┌────────────────┐  ┌────────────────┐                 │  │
│  │  │ 프로세스 감시  │  │ 네트워크 필터   │                 │  │
│  │  │ Process Watch  │  │ Network Filter │                 │  │
│  │  │                │  │                │                 │  │
│  │  │ 캡처 프로그램  │  │ NDIS/WFP       │                 │  │
│  │  │ 실행 차단      │  │ 드라이버       │                 │  │
│  │  └────────────────┘  └────────────────┘                 │  │
│  │                                                          │  │
│  └───────────────────────────┬──────────────────────────────┘  │
│                              │ 통신 포트                        │
│  ┌───────────────────────────▼──────────────────────────────┐  │
│  │              커널 모드 Minifilter (보완)                  │  │
│  │                                                          │  │
│  │  • 파일 읽기/쓰기/삭제/이름변경 차단                       │  │
│  │    Block file read/write/delete/rename                   │  │
│  │  • 패턴 매칭 (SSN 감지)                                   │  │
│  │    Pattern matching (SSN detection)                      │  │
│  │  • 이벤트 로깅                                            │  │
│  │    Event logging                                         │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 28.2 파일 복사 감지

### 28.2.1 복사 작업의 I/O 패턴

파일 복사는 일련의 읽기-쓰기 작업으로 수행됩니다. 이 패턴을 분석하여 복사를 감지할 수 있습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    파일 복사의 I/O 패턴                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  원본 파일 (C:\Protected\file.docx)                            │
│  Source file                                                    │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  1. IRP_MJ_CREATE (원본)                               │    │
│  │     DesiredAccess: FILE_READ_DATA                      │    │
│  │     ShareAccess: FILE_SHARE_READ                       │    │
│  │                                                        │    │
│  │  2. IRP_MJ_READ (순차적 또는 전체)                      │    │
│  │     Offset: 0, Length: FileSize                        │    │
│  │     또는 청크 단위 읽기                                 │    │
│  │     Or chunk-based read                                │    │
│  │                                                        │    │
│  │  3. IRP_MJ_CLEANUP / IRP_MJ_CLOSE                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              │ 데이터 전달                       │
│                              │ Data transfer                    │
│                              v                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  4. IRP_MJ_CREATE (대상, 새 파일 또는 덮어쓰기)         │    │
│  │     DesiredAccess: FILE_WRITE_DATA                     │    │
│  │     Disposition: FILE_CREATE / FILE_OVERWRITE_IF       │    │
│  │                                                        │    │
│  │  5. IRP_MJ_WRITE (데이터 기록)                         │    │
│  │     원본에서 읽은 데이터 기록                           │    │
│  │     Write data read from source                        │    │
│  │                                                        │    │
│  │  6. IRP_MJ_SET_INFORMATION                             │    │
│  │     FileBasicInformation (시간 복사)                   │    │
│  │     FileEndOfFileInformation (크기 설정)               │    │
│  │                                                        │    │
│  │  7. IRP_MJ_CLEANUP / IRP_MJ_CLOSE                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
│  대상 파일 (D:\Unprotected\file.docx)                          │
│  Target file                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 28.2.2 읽기 모니터링 구현

```c
// read_monitor.h - 읽기 모니터링 정의
// read_monitor.h - Read monitoring definitions

#pragma once
#include <fltKernel.h>

// 읽기 통계 구조체
// Read statistics structure
typedef struct _READ_STATISTICS {
    LONGLONG TotalBytesRead;        // 총 읽기 바이트
                                    // Total bytes read
    ULONG ReadCount;                // 읽기 횟수
                                    // Read count
    LARGE_INTEGER FirstReadTime;    // 첫 읽기 시간
                                    // First read time
    LARGE_INTEGER LastReadTime;     // 마지막 읽기 시간
                                    // Last read time
    BOOLEAN SequentialRead;         // 순차 읽기 여부
                                    // Is sequential read
    LONGLONG LastOffset;            // 마지막 읽기 오프셋
                                    // Last read offset
} READ_STATISTICS, *PREAD_STATISTICS;

// 의심스러운 읽기 패턴
// Suspicious read patterns
typedef enum _SUSPICIOUS_READ_TYPE {
    SUSPICIOUS_READ_NONE,           // 정상
                                    // Normal
    SUSPICIOUS_READ_FULL_FILE,      // 전체 파일 읽기
                                    // Full file read
    SUSPICIOUS_READ_RAPID_SEQUENCE, // 빠른 순차 읽기
                                    // Rapid sequential read
    SUSPICIOUS_READ_LARGE_CHUNK,    // 대용량 청크 읽기
                                    // Large chunk read
    SUSPICIOUS_READ_MULTIPLE_COPIES // 여러 복사본 생성 패턴
                                    // Multiple copy pattern
} SUSPICIOUS_READ_TYPE;

// 함수 선언
// Function declarations
FLT_PREOP_CALLBACK_STATUS
ReadMonitorPreRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext);

FLT_POSTOP_CALLBACK_STATUS
ReadMonitorPostRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags);
```

```c
// read_monitor.c - 읽기 모니터링 구현
// read_monitor.c - Read monitoring implementation

#include "read_monitor.h"
#include "stream_context.h"
#include "audit_log.h"

// 대용량 읽기 임계값
// Large read threshold
#define LARGE_READ_THRESHOLD    (1024 * 1024)   // 1MB
#define SUSPICIOUS_READ_RATIO   0.9             // 파일의 90% 이상 읽기

// 컨텍스트에서 읽기 통계 가져오기
// Get read statistics from context
static PREAD_STATISTICS GetReadStats(_In_ PSTREAM_CONTEXT Context)
{
    // 실제 구현에서는 Stream Context 내에 READ_STATISTICS 포함
    // In real implementation, include READ_STATISTICS in Stream Context
    return &Context->ReadStats;
}

// 의심스러운 읽기 분석
// Analyze suspicious read
SUSPICIOUS_READ_TYPE AnalyzeReadPattern(
    _In_ PREAD_STATISTICS Stats,
    _In_ LONGLONG FileSize,
    _In_ ULONG CurrentReadLength)
{
    // 전체 파일 읽기 감지
    // Detect full file read
    if (Stats->TotalBytesRead >= (LONGLONG)(FileSize * SUSPICIOUS_READ_RATIO)) {
        return SUSPICIOUS_READ_FULL_FILE;
    }

    // 대용량 청크 읽기
    // Large chunk read
    if (CurrentReadLength >= LARGE_READ_THRESHOLD) {
        return SUSPICIOUS_READ_LARGE_CHUNK;
    }

    // 빠른 순차 읽기 (짧은 시간 내 많은 읽기)
    // Rapid sequential read (many reads in short time)
    LARGE_INTEGER currentTime;
    KeQuerySystemTime(&currentTime);

    if (Stats->ReadCount > 10) {
        LONGLONG elapsedMs = (currentTime.QuadPart - Stats->FirstReadTime.QuadPart) / 10000;
        if (elapsedMs > 0 && Stats->ReadCount / elapsedMs > 10) {
            return SUSPICIOUS_READ_RAPID_SEQUENCE;
        }
    }

    return SUSPICIOUS_READ_NONE;
}

// Pre-Read 콜백
// Pre-Read callback
FLT_PREOP_CALLBACK_STATUS
ReadMonitorPreRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PSTREAM_CONTEXT streamContext = NULL;

    *CompletionContext = NULL;

    // 페이징 I/O는 스킵
    // Skip paging I/O
    if (FlagOn(Data->Iopb->IrpFlags, IRP_PAGING_IO)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
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

    // 보호된 파일만 모니터링
    // Only monitor protected files
    if (streamContext->IsProtected) {
        *CompletionContext = streamContext;
        return FLT_PREOP_SUCCESS_WITH_CALLBACK;
    }

    FltReleaseContext(streamContext);
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// Post-Read 콜백
// Post-Read callback
FLT_POSTOP_CALLBACK_STATUS
ReadMonitorPostRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    PSTREAM_CONTEXT streamContext = (PSTREAM_CONTEXT)CompletionContext;
    PREAD_STATISTICS readStats;
    SUSPICIOUS_READ_TYPE suspiciousType;
    ULONG bytesRead;
    LONGLONG readOffset;

    UNREFERENCED_PARAMETER(FltObjects);

    if (CompletionContext == NULL) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // DRAINING 상태면 패스
    // Pass if DRAINING
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        FltReleaseContext(streamContext);
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 읽기 성공 확인
    // Check read success
    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        FltReleaseContext(streamContext);
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    bytesRead = (ULONG)Data->IoStatus.Information;
    readOffset = Data->Iopb->Parameters.Read.ByteOffset.QuadPart;

    // 읽기 통계 업데이트
    // Update read statistics
    readStats = GetReadStats(streamContext);

    KeEnterCriticalRegion();
    ExAcquireResourceExclusiveLite(&streamContext->Lock, TRUE);

    if (readStats->ReadCount == 0) {
        KeQuerySystemTime(&readStats->FirstReadTime);
    }

    readStats->ReadCount++;
    readStats->TotalBytesRead += bytesRead;
    KeQuerySystemTime(&readStats->LastReadTime);

    // 순차 읽기 확인
    // Check sequential read
    if (readOffset == readStats->LastOffset + bytesRead ||
        readStats->LastOffset == 0) {
        readStats->SequentialRead = TRUE;
    } else {
        readStats->SequentialRead = FALSE;
    }

    readStats->LastOffset = readOffset;

    ExReleaseResourceLite(&streamContext->Lock);
    KeLeaveCriticalRegion();

    // 의심스러운 패턴 분석
    // Analyze suspicious patterns
    suspiciousType = AnalyzeReadPattern(
        readStats,
        streamContext->FileSize.QuadPart,
        bytesRead
    );

    if (suspiciousType != SUSPICIOUS_READ_NONE) {
        DbgPrint("[ReadMonitor] 의심스러운 읽기 감지:\n");
        // Suspicious read detected
        DbgPrint("[ReadMonitor]   파일: %wZ\n", &streamContext->FileName);
        // File
        DbgPrint("[ReadMonitor]   타입: %d, 총 읽기: %lld\n",
                 suspiciousType, readStats->TotalBytesRead);
        // Type, Total read

        // 감사 로그
        // Audit log
        AuditLogWrite(
            AUDIT_EVENT_SUSPICIOUS_READ,
            &streamContext->FileName,
            PsGetCurrentProcessId(),
            (ULONG)suspiciousType
        );

        // 선택적으로 읽기 차단 가능
        // Can optionally block read
        // 단, Post 콜백에서는 이미 읽기가 완료됨
        // However, in Post callback, read is already complete
    }

    FltReleaseContext(streamContext);

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

### 28.2.3 읽기 차단 정책

민감한 파일의 대용량 읽기를 차단하는 방법입니다.

```c
// read_blocker.c - 읽기 차단 구현
// read_blocker.c - Read blocking implementation

#include <fltKernel.h>

// 읽기 차단 정책
// Read blocking policy
typedef struct _READ_BLOCK_POLICY {
    BOOLEAN BlockFullFileRead;      // 전체 파일 읽기 차단
                                    // Block full file read
    BOOLEAN BlockLargeRead;         // 대용량 읽기 차단
                                    // Block large read
    ULONG MaxSingleReadSize;        // 최대 단일 읽기 크기
                                    // Max single read size
    ULONG MaxTotalReadSize;         // 최대 총 읽기 크기
                                    // Max total read size
    BOOLEAN AllowAuthorizedApps;    // 인가된 앱 허용
                                    // Allow authorized apps
} READ_BLOCK_POLICY, *PREAD_BLOCK_POLICY;

// 기본 정책
// Default policy
static READ_BLOCK_POLICY g_ReadPolicy = {
    .BlockFullFileRead = FALSE,     // 기본적으로 읽기는 허용
                                    // Allow read by default
    .BlockLargeRead = FALSE,
    .MaxSingleReadSize = 0,         // 0 = 제한 없음
                                    // 0 = no limit
    .MaxTotalReadSize = 0,
    .AllowAuthorizedApps = TRUE
};

// 읽기 차단 여부 결정
// Decide whether to block read
BOOLEAN ShouldBlockRead(
    _In_ PSTREAM_CONTEXT Context,
    _In_ ULONG ReadLength,
    _In_ HANDLE ProcessId)
{
    // 보호 플래그에 PROTECT_READ가 없으면 허용
    // Allow if no PROTECT_READ flag
    if (!(Context->ProtectionFlags & PROTECT_READ)) {
        return FALSE;
    }

    // 인가된 앱 확인
    // Check authorized app
    if (g_ReadPolicy.AllowAuthorizedApps) {
        if (IsAuthorizedApp(ProcessId)) {
            return FALSE;
        }
    }

    // 대용량 읽기 차단
    // Block large read
    if (g_ReadPolicy.BlockLargeRead &&
        g_ReadPolicy.MaxSingleReadSize > 0 &&
        ReadLength > g_ReadPolicy.MaxSingleReadSize) {
        DbgPrint("[ReadBlock] 대용량 읽기 차단: %lu > %lu\n",
                 ReadLength, g_ReadPolicy.MaxSingleReadSize);
        // Large read blocked
        return TRUE;
    }

    // 총 읽기량 초과 차단
    // Block if total read exceeded
    if (g_ReadPolicy.MaxTotalReadSize > 0) {
        PREAD_STATISTICS stats = GetReadStats(Context);
        if (stats->TotalBytesRead + ReadLength > g_ReadPolicy.MaxTotalReadSize) {
            DbgPrint("[ReadBlock] 총 읽기량 초과: %lld + %lu > %lu\n",
                     stats->TotalBytesRead, ReadLength, g_ReadPolicy.MaxTotalReadSize);
            // Total read exceeded
            return TRUE;
        }
    }

    return FALSE;
}

// 인가된 앱 확인
// Check authorized app
BOOLEAN IsAuthorizedApp(_In_ HANDLE ProcessId)
{
    // 실제 구현에서는 프로세스 서명 또는 해시 검증
    // In real implementation, verify process signature or hash
    PEPROCESS process;
    PCHAR imageName;
    BOOLEAN authorized = FALSE;

    static const CHAR* authorizedApps[] = {
        "winword.exe",      // Microsoft Word
        "excel.exe",        // Microsoft Excel
        "acrobat.exe",      // Adobe Acrobat
        NULL
    };

    if (NT_SUCCESS(PsLookupProcessByProcessId(ProcessId, &process))) {
        imageName = PsGetProcessImageFileName(process);

        if (imageName != NULL) {
            for (int i = 0; authorizedApps[i] != NULL; i++) {
                if (_stricmp(imageName, authorizedApps[i]) == 0) {
                    authorized = TRUE;
                    break;
                }
            }
        }

        ObDereferenceObject(process);
    }

    return authorized;
}

// Pre-Read 콜백 (차단 버전)
// Pre-Read callback (blocking version)
FLT_PREOP_CALLBACK_STATUS
PreReadWithBlocking(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PSTREAM_CONTEXT streamContext = NULL;
    ULONG readLength;

    *CompletionContext = NULL;

    // 페이징 I/O는 스킵
    // Skip paging I/O
    if (FlagOn(Data->Iopb->IrpFlags, IRP_PAGING_IO)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
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

    // 보호된 파일 확인
    // Check protected file
    if (!streamContext->IsProtected) {
        FltReleaseContext(streamContext);
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    readLength = Data->Iopb->Parameters.Read.Length;

    // 차단 여부 결정
    // Decide blocking
    if (ShouldBlockRead(streamContext, readLength, PsGetCurrentProcessId())) {
        DbgPrint("[ReadBlock] 읽기 차단: %wZ\n", &streamContext->FileName);
        // Read blocked

        FltReleaseContext(streamContext);

        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    FltReleaseContext(streamContext);
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 28.3 사용자 모드 컴포넌트 연동

### 28.3.1 통합 DLP 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                   통합 DLP 솔루션 아키텍처                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               사용자 모드 서비스 (.NET)                   │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ DLP Service (Windows Service)                      │  │  │
│  │  │                                                    │  │  │
│  │  │  ┌──────────────┐  ┌──────────────┐               │  │  │
│  │  │  │ Clipboard    │  │ Screenshot   │               │  │  │
│  │  │  │ Monitor      │  │ Blocker      │               │  │  │
│  │  │  │              │  │              │               │  │  │
│  │  │  │ Hook chain   │  │ Hook chain   │               │  │  │
│  │  │  └──────────────┘  └──────────────┘               │  │  │
│  │  │                                                    │  │  │
│  │  │  ┌──────────────┐  ┌──────────────┐               │  │  │
│  │  │  │ Process      │  │ Network      │               │  │  │
│  │  │  │ Monitor      │  │ Monitor      │               │  │  │
│  │  │  │              │  │              │               │  │  │
│  │  │  │ ETW / WMI    │  │ WFP callback │               │  │  │
│  │  │  └──────────────┘  └──────────────┘               │  │  │
│  │  │                                                    │  │  │
│  │  │  ┌──────────────────────────────────────────────┐ │  │  │
│  │  │  │ Policy Manager & Event Aggregator            │ │  │  │
│  │  │  └──────────────────────────────────────────────┘ │  │  │
│  │  └─────────────────────────┬──────────────────────────┘  │  │
│  │                            │                              │  │
│  └────────────────────────────┼──────────────────────────────┘  │
│                               │ FltSendMessage                  │
│                               │ FilterGetMessage                │
│  ┌────────────────────────────▼──────────────────────────────┐  │
│  │               커널 모드 Minifilter                         │  │
│  │                                                           │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐ │  │
│  │  │ File Access │ │ Content     │ │ Event Notification  │ │  │
│  │  │ Control     │ │ Scanner     │ │ (to User Mode)      │ │  │
│  │  └─────────────┘ └─────────────┘ └─────────────────────┘ │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 28.3.2 사용자 모드 클립보드 모니터링

```csharp
// ClipboardMonitor.cs - 클립보드 모니터링 (.NET)
// ClipboardMonitor.cs - Clipboard Monitoring (.NET)

using System;
using System.Runtime.InteropServices;
using System.Windows.Forms;

namespace DlpService
{
    /// <summary>
    /// 클립보드 모니터링 클래스
    /// Clipboard Monitoring Class
    /// </summary>
    public sealed class ClipboardMonitor : IDisposable
    {
        private readonly IntPtr _hwnd;
        private IntPtr _nextClipboardViewer;
        private bool _disposed;

        // Win32 API 선언
        // Win32 API declarations
        [DllImport("user32.dll")]
        private static extern IntPtr SetClipboardViewer(IntPtr hWndNewViewer);

        [DllImport("user32.dll")]
        private static extern bool ChangeClipboardChain(IntPtr hWndRemove, IntPtr hWndNewNext);

        [DllImport("user32.dll")]
        private static extern IntPtr SendMessage(IntPtr hWnd, int Msg, IntPtr wParam, IntPtr lParam);

        private const int WM_DRAWCLIPBOARD = 0x0308;
        private const int WM_CHANGECBCHAIN = 0x030D;

        public event EventHandler<ClipboardEventArgs>? ClipboardChanged;

        public ClipboardMonitor(IntPtr windowHandle)
        {
            _hwnd = windowHandle;
            _nextClipboardViewer = SetClipboardViewer(_hwnd);

            Console.WriteLine("클립보드 모니터링 시작");
            // Clipboard monitoring started
        }

        /// <summary>
        /// 클립보드 변경 처리
        /// Handle clipboard change
        /// </summary>
        public void ProcessClipboardChange()
        {
            try
            {
                // 클립보드 내용 확인
                // Check clipboard content
                if (Clipboard.ContainsText())
                {
                    var text = Clipboard.GetText();

                    // SSN 패턴 검사
                    // Check SSN pattern
                    if (ContainsSsnPattern(text))
                    {
                        Console.WriteLine("경고: 클립보드에 주민등록번호 패턴 감지!");
                        // Warning: SSN pattern detected in clipboard!

                        // 클립보드 비우기 (선택적)
                        // Clear clipboard (optional)
                        // Clipboard.Clear();

                        ClipboardChanged?.Invoke(this, new ClipboardEventArgs
                        {
                            Type = ClipboardEventType.SensitiveDataDetected,
                            DataType = "SSN",
                            ProcessId = GetClipboardOwnerProcessId()
                        });
                    }
                }

                // 파일 복사 감지
                // Detect file copy
                if (Clipboard.ContainsFileDropList())
                {
                    var files = Clipboard.GetFileDropList();

                    Console.WriteLine($"클립보드에 {files.Count}개 파일 감지");
                    // Files detected in clipboard

                    ClipboardChanged?.Invoke(this, new ClipboardEventArgs
                    {
                        Type = ClipboardEventType.FileCopy,
                        DataType = "Files",
                        FileCount = files.Count
                    });
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"클립보드 처리 오류: {ex.Message}");
                // Clipboard processing error
            }

            // 체인의 다음 뷰어에 전달
            // Pass to next viewer in chain
            if (_nextClipboardViewer != IntPtr.Zero)
            {
                SendMessage(_nextClipboardViewer, WM_DRAWCLIPBOARD, IntPtr.Zero, IntPtr.Zero);
            }
        }

        /// <summary>
        /// SSN 패턴 확인
        /// Check SSN pattern
        /// </summary>
        private static bool ContainsSsnPattern(string text)
        {
            // 간단한 패턴 매칭
            // Simple pattern matching
            // 실제 구현에서는 더 정교한 검증 필요
            // In real implementation, more sophisticated validation needed

            var pattern = @"\d{6}-[1-4]\d{6}";
            return System.Text.RegularExpressions.Regex.IsMatch(text, pattern);
        }

        /// <summary>
        /// 클립보드 소유 프로세스 ID 가져오기
        /// Get clipboard owner process ID
        /// </summary>
        private static uint GetClipboardOwnerProcessId()
        {
            var hwnd = GetClipboardOwner();
            if (hwnd == IntPtr.Zero) return 0;

            GetWindowThreadProcessId(hwnd, out uint processId);
            return processId;
        }

        [DllImport("user32.dll")]
        private static extern IntPtr GetClipboardOwner();

        [DllImport("user32.dll")]
        private static extern uint GetWindowThreadProcessId(IntPtr hWnd, out uint processId);

        /// <summary>
        /// 윈도우 메시지 처리
        /// Process window message
        /// </summary>
        public void ProcessWindowMessage(int msg, IntPtr wParam, IntPtr lParam)
        {
            switch (msg)
            {
                case WM_DRAWCLIPBOARD:
                    ProcessClipboardChange();
                    break;

                case WM_CHANGECBCHAIN:
                    if (wParam == _nextClipboardViewer)
                    {
                        _nextClipboardViewer = lParam;
                    }
                    else if (_nextClipboardViewer != IntPtr.Zero)
                    {
                        SendMessage(_nextClipboardViewer, msg, wParam, lParam);
                    }
                    break;
            }
        }

        public void Dispose()
        {
            if (_disposed) return;
            _disposed = true;

            ChangeClipboardChain(_hwnd, _nextClipboardViewer);

            Console.WriteLine("클립보드 모니터링 종료");
            // Clipboard monitoring stopped
        }
    }

    /// <summary>
    /// 클립보드 이벤트 인자
    /// Clipboard event arguments
    /// </summary>
    public sealed class ClipboardEventArgs : EventArgs
    {
        public ClipboardEventType Type { get; init; }
        public string? DataType { get; init; }
        public uint ProcessId { get; init; }
        public int FileCount { get; init; }
    }

    /// <summary>
    /// 클립보드 이벤트 타입
    /// Clipboard event type
    /// </summary>
    public enum ClipboardEventType
    {
        Normal,
        SensitiveDataDetected,
        FileCopy,
        ImageCopy
    }
}
```

### 28.3.3 커널-사용자 모드 통신

```c
// dlp_communication.h - DLP 통신 프로토콜
// dlp_communication.h - DLP Communication Protocol

#pragma once

// 메시지 타입
// Message types
typedef enum _DLP_MESSAGE_TYPE {
    // 커널 → 사용자 모드
    // Kernel → User mode
    DLP_MSG_FILE_ACCESS,            // 파일 접근 이벤트
                                    // File access event
    DLP_MSG_SUSPICIOUS_READ,        // 의심스러운 읽기
                                    // Suspicious read
    DLP_MSG_SENSITIVE_DATA,         // 민감 정보 감지
                                    // Sensitive data detected
    DLP_MSG_POLICY_VIOLATION,       // 정책 위반
                                    // Policy violation

    // 사용자 모드 → 커널
    // User mode → Kernel
    DLP_MSG_POLICY_UPDATE,          // 정책 업데이트
                                    // Policy update
    DLP_MSG_PROCESS_AUTHORIZE,      // 프로세스 인가
                                    // Process authorization
    DLP_MSG_QUERY_STATUS            // 상태 쿼리
                                    // Status query
} DLP_MESSAGE_TYPE;

// 파일 접근 이벤트 메시지
// File access event message
typedef struct _DLP_FILE_ACCESS_MSG {
    DLP_MESSAGE_TYPE Type;          // = DLP_MSG_FILE_ACCESS
    ULONG ProcessId;                // 프로세스 ID
                                    // Process ID
    ULONG AccessType;               // 접근 유형 (읽기/쓰기/삭제)
                                    // Access type (read/write/delete)
    ULONG DataSize;                 // 데이터 크기
                                    // Data size
    WCHAR FilePath[260];            // 파일 경로
                                    // File path
    WCHAR ProcessName[64];          // 프로세스 이름
                                    // Process name
    LARGE_INTEGER Timestamp;        // 타임스탬프
                                    // Timestamp
} DLP_FILE_ACCESS_MSG, *PDLP_FILE_ACCESS_MSG;

// 민감 정보 감지 메시지
// Sensitive data detection message
typedef struct _DLP_SENSITIVE_DATA_MSG {
    DLP_MESSAGE_TYPE Type;          // = DLP_MSG_SENSITIVE_DATA
    ULONG ProcessId;
    ULONG DataType;                 // 데이터 타입 (SSN, 카드번호 등)
                                    // Data type (SSN, card number, etc.)
    ULONG MatchCount;               // 매치 개수
                                    // Match count
    WCHAR FilePath[260];
    LARGE_INTEGER Timestamp;
} DLP_SENSITIVE_DATA_MSG, *PDLP_SENSITIVE_DATA_MSG;
```

```c
// dlp_kernel_comm.c - 커널 측 통신 구현
// dlp_kernel_comm.c - Kernel-side communication implementation

#include "dlp_communication.h"
#include <fltKernel.h>

// 전역 변수
// Global variables
static PFLT_PORT g_ServerPort = NULL;
static PFLT_PORT g_ClientPort = NULL;
static PFLT_FILTER g_Filter = NULL;

// 이벤트 전송 (비동기)
// Send event (asynchronous)
NTSTATUS SendDlpEvent(
    _In_ DLP_MESSAGE_TYPE EventType,
    _In_ PVOID EventData,
    _In_ ULONG EventDataSize)
{
    NTSTATUS status;

    if (g_ClientPort == NULL) {
        return STATUS_PORT_DISCONNECTED;
    }

    // 메시지 전송 (응답 불필요)
    // Send message (no response needed)
    status = FltSendMessage(
        g_Filter,
        &g_ClientPort,
        EventData,
        EventDataSize,
        NULL,           // 응답 버퍼 없음
                        // No response buffer
        NULL,           // 응답 크기 없음
                        // No response size
        NULL            // 타임아웃 없음
                        // No timeout
    );

    return status;
}

// 파일 접근 이벤트 전송
// Send file access event
VOID NotifyFileAccess(
    _In_ PCUNICODE_STRING FilePath,
    _In_ HANDLE ProcessId,
    _In_ ULONG AccessType,
    _In_ ULONG DataSize)
{
    DLP_FILE_ACCESS_MSG msg;

    RtlZeroMemory(&msg, sizeof(msg));
    msg.Type = DLP_MSG_FILE_ACCESS;
    msg.ProcessId = HandleToULong(ProcessId);
    msg.AccessType = AccessType;
    msg.DataSize = DataSize;

    // 파일 경로 복사
    // Copy file path
    if (FilePath != NULL && FilePath->Length > 0) {
        ULONG copyLen = min(FilePath->Length / sizeof(WCHAR), 259);
        RtlCopyMemory(msg.FilePath, FilePath->Buffer, copyLen * sizeof(WCHAR));
    }

    // 프로세스 이름 가져오기
    // Get process name
    PEPROCESS process;
    if (NT_SUCCESS(PsLookupProcessByProcessId(ProcessId, &process))) {
        PCHAR imageName = PsGetProcessImageFileName(process);
        if (imageName != NULL) {
            for (int i = 0; i < 63 && imageName[i] != '\0'; i++) {
                msg.ProcessName[i] = (WCHAR)imageName[i];
            }
        }
        ObDereferenceObject(process);
    }

    KeQuerySystemTime(&msg.Timestamp);

    // 이벤트 전송
    // Send event
    SendDlpEvent(DLP_MSG_FILE_ACCESS, &msg, sizeof(msg));
}

// 민감 정보 감지 이벤트 전송
// Send sensitive data detection event
VOID NotifySensitiveDataFound(
    _In_ PCUNICODE_STRING FilePath,
    _In_ HANDLE ProcessId,
    _In_ ULONG DataType,
    _In_ ULONG MatchCount)
{
    DLP_SENSITIVE_DATA_MSG msg;

    RtlZeroMemory(&msg, sizeof(msg));
    msg.Type = DLP_MSG_SENSITIVE_DATA;
    msg.ProcessId = HandleToULong(ProcessId);
    msg.DataType = DataType;
    msg.MatchCount = MatchCount;

    if (FilePath != NULL && FilePath->Length > 0) {
        ULONG copyLen = min(FilePath->Length / sizeof(WCHAR), 259);
        RtlCopyMemory(msg.FilePath, FilePath->Buffer, copyLen * sizeof(WCHAR));
    }

    KeQuerySystemTime(&msg.Timestamp);

    SendDlpEvent(DLP_MSG_SENSITIVE_DATA, &msg, sizeof(msg));
}
```

---

## 28.4 요약

### 핵심 내용 정리

| 항목 | 내용 |
|------|------|
| **클립보드 한계** | 커널에서 직접 접근 불가, 사용자 모드 연계 필수 |
| **읽기 모니터링** | 의심스러운 패턴 감지로 복사 시도 추적 |
| **완전한 DLP** | 커널 + 사용자 모드 + 네트워크 필터 조합 필요 |
| **통신** | FltSendMessage/FilterGetMessage로 이벤트 교환 |

### .NET 개발자 참고 사항

```
┌─────────────────────────────────────────────────────────────────┐
│                DLP 솔루션 컴포넌트 역할 분담                     │
├─────────────────────────────────────────────────────────────────┤
│ .NET 사용자 모드 서비스             │ C 커널 Minifilter          │
├─────────────────────────────────────┼───────────────────────────┤
│ • 클립보드 모니터링                 │ • 파일 I/O 차단           │
│   SetClipboardViewer               │   IRP 필터링              │
│                                    │                           │
│ • 스크린샷 차단                     │ • 패턴 매칭               │
│   키보드 후킹                       │   SSN 감지                │
│                                    │                           │
│ • 네트워크 감시                     │ • 이벤트 생성             │
│   WFP 드라이버 연동                 │   FltSendMessage          │
│                                    │                           │
│ • UI 알림                          │ • 정책 수신               │
│   WPF/WinForms                     │   FilterReplyMessage      │
│                                    │                           │
│ • 정책 관리                        │ • 감사 로깅               │
│   데이터베이스/설정                 │   파일/ETW                │
└─────────────────────────────────────┴───────────────────────────┘
```

### Part 8 완료!

Part 8 "DRM 및 파일 보호 기능 구현"을 완료했습니다. 이 Part에서 다룬 내용:

1. **Chapter 24**: 파일 보호 정책 설계
2. **Chapter 25**: 파일 수정 차단 구현
3. **Chapter 26**: 파일 이름 변경 및 이동 차단
4. **Chapter 27**: 파일 삭제 차단
5. **Chapter 28**: 클립보드 및 데이터 유출 방지

### 다음 Part 예고

**Part 9: 사용자 모드 서비스 연동**에서는 이 챕터에서 언급한 커널-사용자 모드 통신을 본격적으로 구현합니다. FltCreateCommunicationPort, FltSendMessage, FilterConnectCommunicationPort 등을 사용한 완전한 통신 시스템을 구축합니다.

---

## 참고 자료

- [Microsoft Docs: Clipboard Functions](https://docs.microsoft.com/en-us/windows/win32/dataxchg/clipboard-functions)
- [Microsoft Docs: Filter Manager Communication](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/communication-between-user-mode-and-kernel-mode)
- [Data Loss Prevention (DLP) Best Practices](https://docs.microsoft.com/en-us/microsoft-365/compliance/dlp-overview-plan-for-dlp)
- [Windows Filtering Platform](https://docs.microsoft.com/en-us/windows/win32/fwp/windows-filtering-platform-start-page)
