# Chapter 24: 파일 보호 정책 설계

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표
- 보호 정책 아키텍처 설계 및 구현
- 정책 저장 및 로드 메커니즘 이해
- 동적 정책 업데이트 시스템 구축

---

## 24.1 보호 대상 식별

### 24.1.1 DRM 보호 시스템의 핵심 요소

효과적인 파일 보호 시스템을 구축하려면 세 가지 핵심 질문에 답해야 합니다:

```
┌─────────────────────────────────────────────────────────────────┐
│                    DRM 보호 시스템 설계 3요소                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐│
│  │  WHAT           │   │  HOW            │   │  WHERE          ││
│  │  무엇을 보호?    │   │  어떻게 보호?    │   │  어디에 저장?    ││
│  ├─────────────────┤   ├─────────────────┤   ├─────────────────┤│
│  │ • 파일 확장자   │   │ • 쓰기 차단     │   │ • 레지스트리    ││
│  │ • 콘텐츠 패턴   │   │ • 이름변경 차단  │   │ • 설정 파일     ││
│  │ • 경로 패턴     │   │ • 삭제 차단     │   │ • 메모리 캐시   ││
│  │ • 메타데이터    │   │ • 복사 제한     │   │ • 파일 스트림   ││
│  └─────────────────┘   └─────────────────┘   └─────────────────┘│
│           │                    │                    │           │
│           └────────────────────┴────────────────────┘           │
│                               │                                 │
│                               v                                 │
│                    ┌─────────────────────┐                     │
│                    │    정책 엔진        │                     │
│                    │   Policy Engine     │                     │
│                    └─────────────────────┘                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 24.1.2 확장자 기반 필터링

가장 단순하고 효율적인 필터링 방법입니다.

```c
// extension_filter.h - 확장자 필터 정의
// extension_filter.h - Extension filter definitions

#pragma once
#include <ntifs.h>

// 최대 확장자 개수
// Maximum extension count
#define MAX_EXTENSIONS 64

// 확장자 필터 구조체
// Extension filter structure
typedef struct _EXTENSION_FILTER {
    ULONG ExtensionCount;                   // 등록된 확장자 수
                                            // Number of registered extensions
    UNICODE_STRING Extensions[MAX_EXTENSIONS];   // 확장자 목록
                                                 // Extension list
    WCHAR ExtensionBuffers[MAX_EXTENSIONS][16];  // 확장자 버퍼
                                                 // Extension buffers
    FAST_MUTEX Lock;                        // 동기화 객체
                                            // Synchronization object
} EXTENSION_FILTER, *PEXTENSION_FILTER;

// 전역 확장자 필터
// Global extension filter
extern EXTENSION_FILTER g_ExtensionFilter;
```

```c
// extension_filter.c - 확장자 필터 구현
// extension_filter.c - Extension filter implementation

#include "extension_filter.h"

// 전역 변수 정의
// Global variable definition
EXTENSION_FILTER g_ExtensionFilter;

// 기본 보호 대상 확장자들
// Default protected extensions
static const WCHAR* g_DefaultExtensions[] = {
    L".txt",
    L".log",
    L".doc",
    L".docx",
    L".xls",
    L".xlsx",
    L".ppt",
    L".pptx",
    L".pdf",
    L".csv",
    NULL
};

// 확장자 필터 초기화
// Initialize extension filter
NTSTATUS ExtensionFilterInitialize(VOID)
{
    RtlZeroMemory(&g_ExtensionFilter, sizeof(EXTENSION_FILTER));
    ExInitializeFastMutex(&g_ExtensionFilter.Lock);

    // 기본 확장자 등록
    // Register default extensions
    for (int i = 0; g_DefaultExtensions[i] != NULL; i++) {
        NTSTATUS status = ExtensionFilterAdd(g_DefaultExtensions[i]);
        if (!NT_SUCCESS(status)) {
            DbgPrint("[Policy] 확장자 추가 실패: %ws (0x%X)\n",
                     g_DefaultExtensions[i], status);
            // Extension add failed
        }
    }

    DbgPrint("[Policy] 확장자 필터 초기화 완료, %lu개 등록\n",
             g_ExtensionFilter.ExtensionCount);
    // Extension filter initialized, N registered

    return STATUS_SUCCESS;
}

// 확장자 필터 정리
// Cleanup extension filter
VOID ExtensionFilterCleanup(VOID)
{
    // FAST_MUTEX는 별도 정리 불필요
    // FAST_MUTEX doesn't need separate cleanup
    g_ExtensionFilter.ExtensionCount = 0;
}

// 확장자 추가
// Add extension
NTSTATUS ExtensionFilterAdd(_In_ PCWSTR Extension)
{
    NTSTATUS status = STATUS_SUCCESS;

    ExAcquireFastMutex(&g_ExtensionFilter.Lock);

    __try {
        if (g_ExtensionFilter.ExtensionCount >= MAX_EXTENSIONS) {
            status = STATUS_INSUFFICIENT_RESOURCES;
            __leave;
        }

        // 중복 검사
        // Check for duplicates
        for (ULONG i = 0; i < g_ExtensionFilter.ExtensionCount; i++) {
            if (_wcsicmp(g_ExtensionFilter.ExtensionBuffers[i], Extension) == 0) {
                status = STATUS_OBJECT_NAME_EXISTS;
                __leave;
            }
        }

        // 새 확장자 추가
        // Add new extension
        ULONG index = g_ExtensionFilter.ExtensionCount;

        // 버퍼에 복사
        // Copy to buffer
        SIZE_T len = wcslen(Extension);
        if (len >= 16) {
            status = STATUS_NAME_TOO_LONG;
            __leave;
        }

        wcscpy_s(g_ExtensionFilter.ExtensionBuffers[index],
                 16,
                 Extension);

        // UNICODE_STRING 초기화
        // Initialize UNICODE_STRING
        RtlInitUnicodeString(
            &g_ExtensionFilter.Extensions[index],
            g_ExtensionFilter.ExtensionBuffers[index]
        );

        g_ExtensionFilter.ExtensionCount++;
    }
    __finally {
        ExReleaseFastMutex(&g_ExtensionFilter.Lock);
    }

    return status;
}

// 확장자 제거
// Remove extension
NTSTATUS ExtensionFilterRemove(_In_ PCWSTR Extension)
{
    NTSTATUS status = STATUS_NOT_FOUND;

    ExAcquireFastMutex(&g_ExtensionFilter.Lock);

    for (ULONG i = 0; i < g_ExtensionFilter.ExtensionCount; i++) {
        if (_wcsicmp(g_ExtensionFilter.ExtensionBuffers[i], Extension) == 0) {
            // 마지막 항목과 교체
            // Swap with last item
            ULONG lastIndex = g_ExtensionFilter.ExtensionCount - 1;

            if (i != lastIndex) {
                wcscpy_s(g_ExtensionFilter.ExtensionBuffers[i],
                         16,
                         g_ExtensionFilter.ExtensionBuffers[lastIndex]);

                RtlInitUnicodeString(
                    &g_ExtensionFilter.Extensions[i],
                    g_ExtensionFilter.ExtensionBuffers[i]
                );
            }

            g_ExtensionFilter.ExtensionCount--;
            status = STATUS_SUCCESS;
            break;
        }
    }

    ExReleaseFastMutex(&g_ExtensionFilter.Lock);

    return status;
}

// 확장자 검사
// Check extension
BOOLEAN ExtensionFilterMatch(_In_ PCUNICODE_STRING FileName)
{
    BOOLEAN matched = FALSE;

    // 파일명이 너무 짧으면 확장자 없음
    // No extension if filename too short
    if (FileName->Length < 4) {  // 최소 ".x" = 4바이트
        return FALSE;            // Minimum ".x" = 4 bytes
    }

    ExAcquireFastMutex(&g_ExtensionFilter.Lock);

    for (ULONG i = 0; i < g_ExtensionFilter.ExtensionCount; i++) {
        if (RtlSuffixUnicodeString(
            &g_ExtensionFilter.Extensions[i],
            FileName,
            TRUE))  // 대소문자 무시
                    // Case-insensitive
        {
            matched = TRUE;
            break;
        }
    }

    ExReleaseFastMutex(&g_ExtensionFilter.Lock);

    return matched;
}

// 확장자 추출 유틸리티
// Extension extraction utility
NTSTATUS ExtractFileExtension(
    _In_ PCUNICODE_STRING FileName,
    _Out_ PUNICODE_STRING Extension)
{
    // 뒤에서부터 '.' 찾기
    // Find '.' from the end
    USHORT dotPosition = 0;
    BOOLEAN dotFound = FALSE;

    for (USHORT i = FileName->Length / sizeof(WCHAR); i > 0; i--) {
        WCHAR c = FileName->Buffer[i - 1];

        // 경로 구분자 만나면 중단
        // Stop at path separator
        if (c == L'\\' || c == L'/') {
            break;
        }

        if (c == L'.') {
            dotPosition = i - 1;
            dotFound = TRUE;
            break;
        }
    }

    if (!dotFound) {
        Extension->Length = 0;
        Extension->MaximumLength = 0;
        Extension->Buffer = NULL;
        return STATUS_NOT_FOUND;
    }

    // 확장자 UNICODE_STRING 설정
    // Set extension UNICODE_STRING
    Extension->Buffer = &FileName->Buffer[dotPosition];
    Extension->Length = FileName->Length - (dotPosition * sizeof(WCHAR));
    Extension->MaximumLength = Extension->Length;

    return STATUS_SUCCESS;
}
```

### 24.1.3 경로 패턴 기반 필터링

특정 디렉터리 내의 파일만 보호하는 방식입니다.

```c
// path_filter.h - 경로 패턴 필터 정의
// path_filter.h - Path pattern filter definitions

#pragma once

#define MAX_PATH_PATTERNS 32

// 경로 패턴 매칭 타입
// Path pattern matching types
typedef enum _PATH_MATCH_TYPE {
    PATH_MATCH_EXACT,       // 정확한 일치
                            // Exact match
    PATH_MATCH_PREFIX,      // 접두사 일치 (디렉터리 하위 전체)
                            // Prefix match (all under directory)
    PATH_MATCH_WILDCARD     // 와일드카드 (* 지원)
                            // Wildcard (* supported)
} PATH_MATCH_TYPE;

// 경로 패턴 항목
// Path pattern entry
typedef struct _PATH_PATTERN_ENTRY {
    UNICODE_STRING Pattern;             // 경로 패턴
                                        // Path pattern
    WCHAR PatternBuffer[260];           // 패턴 버퍼
                                        // Pattern buffer
    PATH_MATCH_TYPE MatchType;          // 매칭 타입
                                        // Match type
    BOOLEAN IsInclude;                  // TRUE=포함, FALSE=제외
                                        // TRUE=include, FALSE=exclude
} PATH_PATTERN_ENTRY, *PPATH_PATTERN_ENTRY;

// 경로 패턴 필터
// Path pattern filter
typedef struct _PATH_FILTER {
    ULONG PatternCount;
    PATH_PATTERN_ENTRY Patterns[MAX_PATH_PATTERNS];
    ERESOURCE Lock;
} PATH_FILTER, *PPATH_FILTER;

extern PATH_FILTER g_PathFilter;
```

```c
// path_filter.c - 경로 패턴 필터 구현
// path_filter.c - Path pattern filter implementation

#include "path_filter.h"

PATH_FILTER g_PathFilter;

// 초기화
// Initialize
NTSTATUS PathFilterInitialize(VOID)
{
    RtlZeroMemory(&g_PathFilter, sizeof(PATH_FILTER));

    NTSTATUS status = ExInitializeResourceLite(&g_PathFilter.Lock);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 기본 보호 경로 추가
    // Add default protected paths
    PathFilterAdd(L"\\Device\\HarddiskVolume*\\ProtectedData\\*",
                  PATH_MATCH_WILDCARD, TRUE);
    PathFilterAdd(L"\\Device\\HarddiskVolume*\\Confidential\\",
                  PATH_MATCH_PREFIX, TRUE);

    return STATUS_SUCCESS;
}

// 정리
// Cleanup
VOID PathFilterCleanup(VOID)
{
    ExDeleteResourceLite(&g_PathFilter.Lock);
}

// 경로 패턴 추가
// Add path pattern
NTSTATUS PathFilterAdd(
    _In_ PCWSTR Pattern,
    _In_ PATH_MATCH_TYPE MatchType,
    _In_ BOOLEAN IsInclude)
{
    NTSTATUS status = STATUS_SUCCESS;

    KeEnterCriticalRegion();
    ExAcquireResourceExclusiveLite(&g_PathFilter.Lock, TRUE);

    __try {
        if (g_PathFilter.PatternCount >= MAX_PATH_PATTERNS) {
            status = STATUS_INSUFFICIENT_RESOURCES;
            __leave;
        }

        ULONG index = g_PathFilter.PatternCount;
        PPATH_PATTERN_ENTRY entry = &g_PathFilter.Patterns[index];

        SIZE_T len = wcslen(Pattern);
        if (len >= 260) {
            status = STATUS_NAME_TOO_LONG;
            __leave;
        }

        wcscpy_s(entry->PatternBuffer, 260, Pattern);
        RtlInitUnicodeString(&entry->Pattern, entry->PatternBuffer);
        entry->MatchType = MatchType;
        entry->IsInclude = IsInclude;

        g_PathFilter.PatternCount++;
    }
    __finally {
        ExReleaseResourceLite(&g_PathFilter.Lock);
        KeLeaveCriticalRegion();
    }

    return status;
}

// 간단한 와일드카드 매칭 (*, ? 지원)
// Simple wildcard matching (* and ? supported)
static BOOLEAN WildcardMatch(
    _In_ PCWSTR Pattern,
    _In_ PCWSTR String)
{
    while (*Pattern) {
        if (*Pattern == L'*') {
            Pattern++;

            if (*Pattern == L'\0') {
                return TRUE;  // 마지막 *는 모든 것과 일치
                              // Trailing * matches everything
            }

            while (*String) {
                if (WildcardMatch(Pattern, String)) {
                    return TRUE;
                }
                String++;
            }

            return FALSE;
        }
        else if (*Pattern == L'?' || towlower(*Pattern) == towlower(*String)) {
            Pattern++;
            String++;
        }
        else {
            return FALSE;
        }
    }

    return (*String == L'\0');
}

// 경로 필터 매칭
// Path filter matching
BOOLEAN PathFilterMatch(_In_ PCUNICODE_STRING FilePath)
{
    BOOLEAN shouldProtect = FALSE;
    WCHAR pathBuffer[260];

    // NULL 종료 문자열로 변환
    // Convert to null-terminated string
    if (FilePath->Length >= sizeof(pathBuffer)) {
        return FALSE;
    }

    RtlCopyMemory(pathBuffer, FilePath->Buffer, FilePath->Length);
    pathBuffer[FilePath->Length / sizeof(WCHAR)] = L'\0';

    KeEnterCriticalRegion();
    ExAcquireResourceSharedLite(&g_PathFilter.Lock, TRUE);

    for (ULONG i = 0; i < g_PathFilter.PatternCount; i++) {
        PPATH_PATTERN_ENTRY entry = &g_PathFilter.Patterns[i];
        BOOLEAN matched = FALSE;

        switch (entry->MatchType) {
            case PATH_MATCH_EXACT:
                matched = (RtlCompareUnicodeString(
                    FilePath, &entry->Pattern, TRUE) == 0);
                break;

            case PATH_MATCH_PREFIX:
                matched = RtlPrefixUnicodeString(
                    &entry->Pattern, FilePath, TRUE);
                break;

            case PATH_MATCH_WILDCARD:
                matched = WildcardMatch(entry->PatternBuffer, pathBuffer);
                break;
        }

        if (matched) {
            shouldProtect = entry->IsInclude;
            // 마지막 매칭 패턴이 결과를 결정 (exclude가 우선)
            // Last matching pattern determines result (exclude takes priority)
        }
    }

    ExReleaseResourceLite(&g_PathFilter.Lock);
    KeLeaveCriticalRegion();

    return shouldProtect;
}
```

### 24.1.4 콘텐츠 기반 필터링

파일 내용을 검사하여 보호 여부를 결정합니다 (Chapter 22의 SSN 감지와 연계).

```c
// content_filter.h - 콘텐츠 기반 필터 정의
// content_filter.h - Content-based filter definitions

#pragma once

#include "ssn_detector.h"

// 콘텐츠 분류 결과
// Content classification result
typedef enum _CONTENT_CLASSIFICATION {
    CONTENT_NORMAL,             // 일반 콘텐츠
                                // Normal content
    CONTENT_CONTAINS_SSN,       // 주민등록번호 포함
                                // Contains SSN
    CONTENT_CONTAINS_CARD,      // 신용카드 번호 포함
                                // Contains credit card number
    CONTENT_CONFIDENTIAL        // 기밀 정보 포함
                                // Contains confidential information
} CONTENT_CLASSIFICATION;

// 콘텐츠 검사 결과
// Content scan result
typedef struct _CONTENT_SCAN_RESULT {
    CONTENT_CLASSIFICATION Classification;
    ULONG SensitiveDataCount;   // 발견된 민감 정보 개수
                                // Number of sensitive data found
    BOOLEAN RequiresProtection; // 보호 필요 여부
                                // Protection required
} CONTENT_SCAN_RESULT, *PCONTENT_SCAN_RESULT;

// 콘텐츠 검사 함수
// Content scan function
NTSTATUS ScanContentForSensitiveData(
    _In_reads_bytes_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength,
    _Out_ PCONTENT_SCAN_RESULT Result);
```

```c
// content_filter.c - 콘텐츠 기반 필터 구현
// content_filter.c - Content-based filter implementation

#include "content_filter.h"

// 전역 SSN 감지기
// Global SSN detector
extern SSN_DETECTOR g_SsnDetector;

// 기밀 키워드 (예시)
// Confidential keywords (example)
static const CHAR* g_ConfidentialKeywords[] = {
    "CONFIDENTIAL",
    "SECRET",
    "INTERNAL ONLY",
    "기밀",
    "대외비",
    NULL
};

// 키워드 검사
// Keyword check
static BOOLEAN ContainsConfidentialKeyword(
    _In_reads_bytes_(Length) PUCHAR Buffer,
    _In_ ULONG Length)
{
    for (int i = 0; g_ConfidentialKeywords[i] != NULL; i++) {
        ULONG keywordLen = (ULONG)strlen(g_ConfidentialKeywords[i]);

        for (ULONG j = 0; j + keywordLen <= Length; j++) {
            if (RtlCompareMemory(&Buffer[j],
                                  g_ConfidentialKeywords[i],
                                  keywordLen) == keywordLen) {
                return TRUE;
            }
        }
    }

    return FALSE;
}

// 콘텐츠 스캔
// Content scan
NTSTATUS ScanContentForSensitiveData(
    _In_reads_bytes_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength,
    _Out_ PCONTENT_SCAN_RESULT Result)
{
    SSN_SCAN_RESULT ssnResult;
    NTSTATUS status;

    RtlZeroMemory(Result, sizeof(CONTENT_SCAN_RESULT));
    Result->Classification = CONTENT_NORMAL;
    Result->RequiresProtection = FALSE;

    if (Buffer == NULL || BufferLength == 0) {
        return STATUS_SUCCESS;
    }

    // 1. 주민등록번호 검사
    // 1. SSN scan
    status = SsnScanBuffer(&g_SsnDetector, Buffer, BufferLength, &ssnResult);

    if (NT_SUCCESS(status) && ssnResult.ValidatedCount > 0) {
        Result->Classification = CONTENT_CONTAINS_SSN;
        Result->SensitiveDataCount = ssnResult.ValidatedCount;
        Result->RequiresProtection = TRUE;

        DbgPrint("[Content] SSN 발견: %lu개\n", ssnResult.ValidatedCount);
        // SSN found: N items
        return STATUS_SUCCESS;
    }

    // 2. 기밀 키워드 검사
    // 2. Confidential keyword check
    if (ContainsConfidentialKeyword((PUCHAR)Buffer, BufferLength)) {
        Result->Classification = CONTENT_CONFIDENTIAL;
        Result->SensitiveDataCount = 1;
        Result->RequiresProtection = TRUE;

        DbgPrint("[Content] 기밀 키워드 발견\n");
        // Confidential keyword found
        return STATUS_SUCCESS;
    }

    // 추가 검사 (신용카드 번호 등) 여기에 구현
    // Additional checks (credit card numbers, etc.) implement here

    return STATUS_SUCCESS;
}
```

---

## 24.2 보호 동작 정의

### 24.2.1 보호 플래그 시스템

```c
// protection_flags.h - 보호 플래그 정의
// protection_flags.h - Protection flags definitions

#pragma once

// 보호 플래그 (비트 필드)
// Protection flags (bit field)
typedef enum _PROTECTION_FLAGS {
    PROTECT_NONE            = 0x0000,   // 보호 없음
                                        // No protection
    PROTECT_READ            = 0x0001,   // 읽기 제한 (거의 사용 안 함)
                                        // Read restriction (rarely used)
    PROTECT_WRITE           = 0x0002,   // 쓰기 차단
                                        // Block write
    PROTECT_APPEND          = 0x0004,   // 추가 쓰기만 허용
                                        // Allow append only
    PROTECT_RENAME          = 0x0008,   // 이름 변경 차단
                                        // Block rename
    PROTECT_DELETE          = 0x0010,   // 삭제 차단
                                        // Block delete
    PROTECT_MOVE            = 0x0020,   // 이동 차단 (이름변경과 유사)
                                        // Block move (similar to rename)
    PROTECT_COPY_OUT        = 0x0040,   // 보호 영역 외부로 복사 차단
                                        // Block copy outside protected area
    PROTECT_EXECUTE         = 0x0080,   // 실행 차단
                                        // Block execution
    PROTECT_PRINT           = 0x0100,   // 인쇄 차단 (사용자 모드 연동 필요)
                                        // Block printing (needs user mode)
    PROTECT_SCREENSHOT      = 0x0200,   // 스크린샷 차단 (사용자 모드 연동)
                                        // Block screenshot (needs user mode)

    // 복합 플래그
    // Composite flags
    PROTECT_MODIFY          = PROTECT_WRITE | PROTECT_APPEND,
    PROTECT_FILE_OPS        = PROTECT_RENAME | PROTECT_DELETE | PROTECT_MOVE,
    PROTECT_STANDARD        = PROTECT_WRITE | PROTECT_FILE_OPS,
    PROTECT_STRICT          = PROTECT_STANDARD | PROTECT_COPY_OUT,
    PROTECT_ALL             = 0xFFFF
} PROTECTION_FLAGS;

// 보호 수준 사전 정의
// Predefined protection levels
typedef enum _PROTECTION_LEVEL {
    PROTECTION_LEVEL_NONE,          // 보호 없음
                                    // No protection
    PROTECTION_LEVEL_LOW,           // 낮음: 삭제만 차단
                                    // Low: block delete only
    PROTECTION_LEVEL_MEDIUM,        // 중간: 수정, 삭제 차단
                                    // Medium: block modify, delete
    PROTECTION_LEVEL_HIGH,          // 높음: 수정, 삭제, 이름변경 차단
                                    // High: block modify, delete, rename
    PROTECTION_LEVEL_MAXIMUM        // 최대: 모든 변경 차단
                                    // Maximum: block all changes
} PROTECTION_LEVEL;

// 보호 수준을 플래그로 변환
// Convert protection level to flags
__forceinline PROTECTION_FLAGS GetProtectionFlags(PROTECTION_LEVEL Level)
{
    switch (Level) {
        case PROTECTION_LEVEL_LOW:
            return PROTECT_DELETE;

        case PROTECTION_LEVEL_MEDIUM:
            return PROTECT_WRITE | PROTECT_DELETE;

        case PROTECTION_LEVEL_HIGH:
            return PROTECT_STANDARD;

        case PROTECTION_LEVEL_MAXIMUM:
            return PROTECT_STRICT;

        default:
            return PROTECT_NONE;
    }
}
```

### 24.2.2 보호 정책 구조체

```c
// protection_policy.h - 보호 정책 정의
// protection_policy.h - Protection policy definitions

#pragma once

#include "protection_flags.h"

// 정책 식별자
// Policy identifier
typedef ULONG POLICY_ID;

#define INVALID_POLICY_ID   0
#define DEFAULT_POLICY_ID   1

// 개별 정책 항목
// Individual policy entry
typedef struct _PROTECTION_POLICY_ENTRY {
    POLICY_ID PolicyId;             // 정책 ID
                                    // Policy ID
    PROTECTION_FLAGS Flags;         // 보호 플래그
                                    // Protection flags
    UNICODE_STRING Description;     // 정책 설명
                                    // Policy description
    WCHAR DescBuffer[128];          // 설명 버퍼
                                    // Description buffer

    // 적용 조건
    // Application conditions
    struct {
        BOOLEAN UseExtensionFilter; // 확장자 필터 사용
                                    // Use extension filter
        BOOLEAN UsePathFilter;      // 경로 필터 사용
                                    // Use path filter
        BOOLEAN UseContentFilter;   // 콘텐츠 필터 사용
                                    // Use content filter
    } Conditions;

    // 예외 처리
    // Exception handling
    struct {
        BOOLEAN AllowSystemProcesses;   // 시스템 프로세스 허용
                                        // Allow system processes
        BOOLEAN AllowAdministrators;    // 관리자 허용
                                        // Allow administrators
        UNICODE_STRING ExcludedProcesses[8];  // 제외 프로세스 목록
                                              // Excluded process list
        ULONG ExcludedProcessCount;
    } Exceptions;

    // 감사 설정
    // Audit settings
    struct {
        BOOLEAN LogAllAttempts;         // 모든 시도 기록
                                        // Log all attempts
        BOOLEAN LogBlockedOnly;         // 차단된 것만 기록
                                        // Log blocked only
        BOOLEAN NotifyUser;             // 사용자에게 알림
                                        // Notify user
    } Audit;

} PROTECTION_POLICY_ENTRY, *PPROTECTION_POLICY_ENTRY;

// 정책 매니저
// Policy manager
typedef struct _POLICY_MANAGER {
    ULONG PolicyCount;                  // 정책 개수
                                        // Policy count
    PPROTECTION_POLICY_ENTRY Policies;  // 정책 배열
                                        // Policy array
    ERESOURCE Lock;                     // 동기화 객체
                                        // Synchronization object
    POLICY_ID NextPolicyId;             // 다음 정책 ID
                                        // Next policy ID
    BOOLEAN Initialized;                // 초기화 여부
                                        // Initialized flag
} POLICY_MANAGER, *PPOLICY_MANAGER;

// 전역 정책 매니저
// Global policy manager
extern POLICY_MANAGER g_PolicyManager;

// 함수 선언
// Function declarations
NTSTATUS PolicyManagerInitialize(VOID);
VOID PolicyManagerCleanup(VOID);
NTSTATUS PolicyCreate(_In_ PPROTECTION_POLICY_ENTRY Template, _Out_ PPOLICY_ID PolicyId);
NTSTATUS PolicyDelete(_In_ POLICY_ID PolicyId);
NTSTATUS PolicyUpdate(_In_ POLICY_ID PolicyId, _In_ PPROTECTION_POLICY_ENTRY NewPolicy);
PPROTECTION_POLICY_ENTRY PolicyFind(_In_ POLICY_ID PolicyId);
PROTECTION_FLAGS PolicyEvaluate(
    _In_ PCUNICODE_STRING FilePath,
    _In_opt_ PVOID FileContent,
    _In_ ULONG ContentLength,
    _In_ HANDLE ProcessId);
```

### 24.2.3 정책 매니저 구현

```c
// policy_manager.c - 정책 매니저 구현
// policy_manager.c - Policy manager implementation

#include "protection_policy.h"
#include "extension_filter.h"
#include "path_filter.h"
#include "content_filter.h"

// 전역 정책 매니저
// Global policy manager
POLICY_MANAGER g_PolicyManager;

// 최대 정책 수
// Maximum policies
#define MAX_POLICIES 64

// 초기화
// Initialize
NTSTATUS PolicyManagerInitialize(VOID)
{
    NTSTATUS status;

    RtlZeroMemory(&g_PolicyManager, sizeof(POLICY_MANAGER));

    status = ExInitializeResourceLite(&g_PolicyManager.Lock);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 정책 배열 할당
    // Allocate policy array
    g_PolicyManager.Policies = ExAllocatePool2(
        POOL_FLAG_PAGED,
        sizeof(PROTECTION_POLICY_ENTRY) * MAX_POLICIES,
        'ploP'
    );

    if (g_PolicyManager.Policies == NULL) {
        ExDeleteResourceLite(&g_PolicyManager.Lock);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    RtlZeroMemory(g_PolicyManager.Policies,
                  sizeof(PROTECTION_POLICY_ENTRY) * MAX_POLICIES);

    g_PolicyManager.NextPolicyId = DEFAULT_POLICY_ID;
    g_PolicyManager.Initialized = TRUE;

    // 기본 정책 생성
    // Create default policy
    PROTECTION_POLICY_ENTRY defaultPolicy;
    RtlZeroMemory(&defaultPolicy, sizeof(defaultPolicy));

    defaultPolicy.Flags = PROTECT_STANDARD;
    wcscpy_s(defaultPolicy.DescBuffer, 128, L"기본 보호 정책");
    // "Default protection policy"
    RtlInitUnicodeString(&defaultPolicy.Description, defaultPolicy.DescBuffer);
    defaultPolicy.Conditions.UseExtensionFilter = TRUE;
    defaultPolicy.Conditions.UsePathFilter = TRUE;
    defaultPolicy.Conditions.UseContentFilter = TRUE;
    defaultPolicy.Exceptions.AllowSystemProcesses = TRUE;
    defaultPolicy.Audit.LogBlockedOnly = TRUE;

    POLICY_ID defaultId;
    PolicyCreate(&defaultPolicy, &defaultId);

    DbgPrint("[Policy] 정책 매니저 초기화 완료\n");
    // Policy manager initialized

    return STATUS_SUCCESS;
}

// 정리
// Cleanup
VOID PolicyManagerCleanup(VOID)
{
    if (!g_PolicyManager.Initialized) {
        return;
    }

    if (g_PolicyManager.Policies != NULL) {
        ExFreePoolWithTag(g_PolicyManager.Policies, 'ploP');
        g_PolicyManager.Policies = NULL;
    }

    ExDeleteResourceLite(&g_PolicyManager.Lock);
    g_PolicyManager.Initialized = FALSE;

    DbgPrint("[Policy] 정책 매니저 정리 완료\n");
    // Policy manager cleaned up
}

// 정책 생성
// Create policy
NTSTATUS PolicyCreate(
    _In_ PPROTECTION_POLICY_ENTRY Template,
    _Out_ PPOLICY_ID PolicyId)
{
    NTSTATUS status = STATUS_SUCCESS;

    *PolicyId = INVALID_POLICY_ID;

    KeEnterCriticalRegion();
    ExAcquireResourceExclusiveLite(&g_PolicyManager.Lock, TRUE);

    __try {
        if (g_PolicyManager.PolicyCount >= MAX_POLICIES) {
            status = STATUS_QUOTA_EXCEEDED;
            __leave;
        }

        // 새 정책 슬롯 찾기
        // Find new policy slot
        PPROTECTION_POLICY_ENTRY newEntry = NULL;
        for (ULONG i = 0; i < MAX_POLICIES; i++) {
            if (g_PolicyManager.Policies[i].PolicyId == INVALID_POLICY_ID) {
                newEntry = &g_PolicyManager.Policies[i];
                break;
            }
        }

        if (newEntry == NULL) {
            status = STATUS_INSUFFICIENT_RESOURCES;
            __leave;
        }

        // 정책 복사
        // Copy policy
        RtlCopyMemory(newEntry, Template, sizeof(PROTECTION_POLICY_ENTRY));
        newEntry->PolicyId = g_PolicyManager.NextPolicyId++;

        // 문자열 포인터 재설정
        // Reset string pointers
        RtlInitUnicodeString(&newEntry->Description, newEntry->DescBuffer);

        g_PolicyManager.PolicyCount++;
        *PolicyId = newEntry->PolicyId;

        DbgPrint("[Policy] 정책 생성됨: ID=%lu, 플래그=0x%04X\n",
                 newEntry->PolicyId, newEntry->Flags);
        // Policy created: ID, Flags
    }
    __finally {
        ExReleaseResourceLite(&g_PolicyManager.Lock);
        KeLeaveCriticalRegion();
    }

    return status;
}

// 정책 삭제
// Delete policy
NTSTATUS PolicyDelete(_In_ POLICY_ID PolicyId)
{
    NTSTATUS status = STATUS_NOT_FOUND;

    if (PolicyId == DEFAULT_POLICY_ID) {
        return STATUS_ACCESS_DENIED;  // 기본 정책 삭제 불가
                                      // Cannot delete default policy
    }

    KeEnterCriticalRegion();
    ExAcquireResourceExclusiveLite(&g_PolicyManager.Lock, TRUE);

    for (ULONG i = 0; i < MAX_POLICIES; i++) {
        if (g_PolicyManager.Policies[i].PolicyId == PolicyId) {
            RtlZeroMemory(&g_PolicyManager.Policies[i],
                          sizeof(PROTECTION_POLICY_ENTRY));
            g_PolicyManager.PolicyCount--;
            status = STATUS_SUCCESS;
            break;
        }
    }

    ExReleaseResourceLite(&g_PolicyManager.Lock);
    KeLeaveCriticalRegion();

    return status;
}

// 정책 찾기
// Find policy
PPROTECTION_POLICY_ENTRY PolicyFind(_In_ POLICY_ID PolicyId)
{
    PPROTECTION_POLICY_ENTRY found = NULL;

    KeEnterCriticalRegion();
    ExAcquireResourceSharedLite(&g_PolicyManager.Lock, TRUE);

    for (ULONG i = 0; i < MAX_POLICIES; i++) {
        if (g_PolicyManager.Policies[i].PolicyId == PolicyId) {
            found = &g_PolicyManager.Policies[i];
            break;
        }
    }

    ExReleaseResourceLite(&g_PolicyManager.Lock);
    KeLeaveCriticalRegion();

    return found;
}

// 시스템 프로세스 확인
// Check system process
static BOOLEAN IsSystemProcess(HANDLE ProcessId)
{
    // System(4)과 주요 시스템 프로세스 확인
    // Check System(4) and major system processes
    if ((ULONG_PTR)ProcessId <= 4) {
        return TRUE;
    }

    // 추가 시스템 프로세스 검사 로직
    // Additional system process check logic
    // (실제 구현에서는 프로세스 이름 검사 등)
    // (In real implementation, check process name, etc.)

    return FALSE;
}

// 정책 평가 - 핵심 함수
// Policy evaluation - core function
PROTECTION_FLAGS PolicyEvaluate(
    _In_ PCUNICODE_STRING FilePath,
    _In_opt_ PVOID FileContent,
    _In_ ULONG ContentLength,
    _In_ HANDLE ProcessId)
{
    PROTECTION_FLAGS resultFlags = PROTECT_NONE;
    BOOLEAN anyPolicyMatched = FALSE;

    KeEnterCriticalRegion();
    ExAcquireResourceSharedLite(&g_PolicyManager.Lock, TRUE);

    for (ULONG i = 0; i < MAX_POLICIES; i++) {
        PPROTECTION_POLICY_ENTRY policy = &g_PolicyManager.Policies[i];

        if (policy->PolicyId == INVALID_POLICY_ID) {
            continue;
        }

        // 예외 처리 먼저 확인
        // Check exceptions first
        if (policy->Exceptions.AllowSystemProcesses &&
            IsSystemProcess(ProcessId)) {
            continue;
        }

        // 조건 평가
        // Evaluate conditions
        BOOLEAN conditionsMet = FALSE;

        if (policy->Conditions.UseExtensionFilter) {
            if (ExtensionFilterMatch(FilePath)) {
                conditionsMet = TRUE;
            }
        }

        if (policy->Conditions.UsePathFilter) {
            if (PathFilterMatch(FilePath)) {
                conditionsMet = TRUE;
            }
        }

        if (policy->Conditions.UseContentFilter &&
            FileContent != NULL && ContentLength > 0) {
            CONTENT_SCAN_RESULT contentResult;
            if (NT_SUCCESS(ScanContentForSensitiveData(
                FileContent, ContentLength, &contentResult))) {
                if (contentResult.RequiresProtection) {
                    conditionsMet = TRUE;
                }
            }
        }

        // 조건 충족 시 플래그 적용
        // Apply flags if conditions met
        if (conditionsMet) {
            resultFlags |= policy->Flags;
            anyPolicyMatched = TRUE;
        }
    }

    ExReleaseResourceLite(&g_PolicyManager.Lock);
    KeLeaveCriticalRegion();

    if (anyPolicyMatched) {
        DbgPrint("[Policy] 파일 보호 적용: %wZ, 플래그=0x%04X\n",
                 FilePath, resultFlags);
        // File protection applied: path, flags
    }

    return resultFlags;
}
```

---

## 24.3 정책 저장소 설계

### 24.3.1 레지스트리 기반 정책 저장

```c
// policy_registry.h - 레지스트리 기반 정책 저장
// policy_registry.h - Registry-based policy storage

#pragma once

// 레지스트리 경로
// Registry path
#define POLICY_REGISTRY_PATH \
    L"\\Registry\\Machine\\SYSTEM\\CurrentControlSet\\Services\\DrmFilter\\Policy"

// 레지스트리에서 정책 로드
// Load policy from registry
NTSTATUS LoadPoliciesFromRegistry(VOID);

// 레지스트리에 정책 저장
// Save policy to registry
NTSTATUS SavePoliciesToRegistry(VOID);
```

```c
// policy_registry.c - 레지스트리 정책 저장 구현
// policy_registry.c - Registry policy storage implementation

#include "policy_registry.h"
#include "protection_policy.h"

// 레지스트리 값 읽기 헬퍼
// Registry value read helper
static NTSTATUS ReadRegistryValue(
    _In_ HANDLE KeyHandle,
    _In_ PCWSTR ValueName,
    _Out_writes_bytes_(BufferSize) PVOID Buffer,
    _In_ ULONG BufferSize,
    _Out_opt_ PULONG ResultSize)
{
    NTSTATUS status;
    UNICODE_STRING valueName;
    ULONG resultLength;
    PKEY_VALUE_PARTIAL_INFORMATION valueInfo;
    ULONG valueInfoSize;

    RtlInitUnicodeString(&valueName, ValueName);

    // 값 크기 먼저 확인
    // First check value size
    status = ZwQueryValueKey(
        KeyHandle,
        &valueName,
        KeyValuePartialInformation,
        NULL,
        0,
        &resultLength
    );

    if (status != STATUS_BUFFER_TOO_SMALL &&
        status != STATUS_BUFFER_OVERFLOW) {
        return status;
    }

    // 버퍼 할당
    // Allocate buffer
    valueInfoSize = resultLength;
    valueInfo = ExAllocatePool2(POOL_FLAG_PAGED, valueInfoSize, 'geRV');

    if (valueInfo == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // 값 읽기
    // Read value
    status = ZwQueryValueKey(
        KeyHandle,
        &valueName,
        KeyValuePartialInformation,
        valueInfo,
        valueInfoSize,
        &resultLength
    );

    if (NT_SUCCESS(status)) {
        ULONG copySize = min(valueInfo->DataLength, BufferSize);
        RtlCopyMemory(Buffer, valueInfo->Data, copySize);

        if (ResultSize != NULL) {
            *ResultSize = copySize;
        }
    }

    ExFreePoolWithTag(valueInfo, 'geRV');

    return status;
}

// 레지스트리 값 쓰기 헬퍼
// Registry value write helper
static NTSTATUS WriteRegistryValue(
    _In_ HANDLE KeyHandle,
    _In_ PCWSTR ValueName,
    _In_ ULONG ValueType,
    _In_reads_bytes_(DataSize) PVOID Data,
    _In_ ULONG DataSize)
{
    UNICODE_STRING valueName;
    RtlInitUnicodeString(&valueName, ValueName);

    return ZwSetValueKey(
        KeyHandle,
        &valueName,
        0,
        ValueType,
        Data,
        DataSize
    );
}

// 레지스트리에서 정책 로드
// Load policies from registry
NTSTATUS LoadPoliciesFromRegistry(VOID)
{
    NTSTATUS status;
    HANDLE keyHandle = NULL;
    OBJECT_ATTRIBUTES objAttr;
    UNICODE_STRING keyPath = RTL_CONSTANT_STRING(POLICY_REGISTRY_PATH);

    InitializeObjectAttributes(
        &objAttr,
        &keyPath,
        OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE,
        NULL,
        NULL
    );

    status = ZwOpenKey(&keyHandle, KEY_READ, &objAttr);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[Policy] 레지스트리 키 열기 실패: 0x%X (기본값 사용)\n", status);
        // Registry key open failed (using defaults)
        return STATUS_SUCCESS;  // 기본 정책 사용
                                // Use default policy
    }

    __try {
        // 확장자 목록 로드
        // Load extension list
        WCHAR extensionBuffer[512];
        ULONG bytesRead;

        status = ReadRegistryValue(
            keyHandle,
            L"ProtectedExtensions",
            extensionBuffer,
            sizeof(extensionBuffer),
            &bytesRead
        );

        if (NT_SUCCESS(status) && bytesRead > 0) {
            // 멀티스트링 파싱 (NULL로 구분)
            // Parse multi-string (NULL-separated)
            PWCHAR current = extensionBuffer;

            while (*current != L'\0') {
                ExtensionFilterAdd(current);
                current += wcslen(current) + 1;
            }

            DbgPrint("[Policy] 확장자 목록 로드됨\n");
            // Extension list loaded
        }

        // 보호 수준 로드
        // Load protection level
        ULONG protectionLevel = PROTECTION_LEVEL_HIGH;

        status = ReadRegistryValue(
            keyHandle,
            L"ProtectionLevel",
            &protectionLevel,
            sizeof(protectionLevel),
            NULL
        );

        if (NT_SUCCESS(status)) {
            // 기본 정책 업데이트
            // Update default policy
            PPROTECTION_POLICY_ENTRY defaultPolicy = PolicyFind(DEFAULT_POLICY_ID);
            if (defaultPolicy != NULL) {
                defaultPolicy->Flags = GetProtectionFlags((PROTECTION_LEVEL)protectionLevel);
            }

            DbgPrint("[Policy] 보호 수준 로드됨: %lu\n", protectionLevel);
            // Protection level loaded
        }

        // 경로 패턴 로드
        // Load path patterns
        WCHAR pathBuffer[1024];

        status = ReadRegistryValue(
            keyHandle,
            L"ProtectedPaths",
            pathBuffer,
            sizeof(pathBuffer),
            &bytesRead
        );

        if (NT_SUCCESS(status) && bytesRead > 0) {
            PWCHAR current = pathBuffer;

            while (*current != L'\0') {
                PathFilterAdd(current, PATH_MATCH_PREFIX, TRUE);
                current += wcslen(current) + 1;
            }

            DbgPrint("[Policy] 경로 패턴 로드됨\n");
            // Path patterns loaded
        }

        status = STATUS_SUCCESS;
    }
    __finally {
        if (keyHandle != NULL) {
            ZwClose(keyHandle);
        }
    }

    return status;
}

// 레지스트리에 정책 저장
// Save policies to registry
NTSTATUS SavePoliciesToRegistry(VOID)
{
    NTSTATUS status;
    HANDLE keyHandle = NULL;
    OBJECT_ATTRIBUTES objAttr;
    UNICODE_STRING keyPath = RTL_CONSTANT_STRING(POLICY_REGISTRY_PATH);
    ULONG disposition;

    InitializeObjectAttributes(
        &objAttr,
        &keyPath,
        OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE,
        NULL,
        NULL
    );

    // 키 생성 또는 열기
    // Create or open key
    status = ZwCreateKey(
        &keyHandle,
        KEY_WRITE,
        &objAttr,
        0,
        NULL,
        REG_OPTION_NON_VOLATILE,
        &disposition
    );

    if (!NT_SUCCESS(status)) {
        DbgPrint("[Policy] 레지스트리 키 생성 실패: 0x%X\n", status);
        // Registry key creation failed
        return status;
    }

    __try {
        // 확장자 목록 저장 (멀티스트링)
        // Save extension list (multi-string)
        WCHAR extensionBuffer[512];
        ULONG offset = 0;

        ExAcquireFastMutex(&g_ExtensionFilter.Lock);

        for (ULONG i = 0; i < g_ExtensionFilter.ExtensionCount; i++) {
            SIZE_T len = wcslen(g_ExtensionFilter.ExtensionBuffers[i]);

            if (offset + len + 2 < 512) {
                wcscpy_s(&extensionBuffer[offset], 512 - offset,
                         g_ExtensionFilter.ExtensionBuffers[i]);
                offset += (ULONG)(len + 1);
            }
        }

        ExReleaseFastMutex(&g_ExtensionFilter.Lock);

        extensionBuffer[offset] = L'\0';  // 이중 NULL 종료
                                          // Double NULL termination

        status = WriteRegistryValue(
            keyHandle,
            L"ProtectedExtensions",
            REG_MULTI_SZ,
            extensionBuffer,
            (offset + 1) * sizeof(WCHAR)
        );

        if (!NT_SUCCESS(status)) {
            DbgPrint("[Policy] 확장자 저장 실패: 0x%X\n", status);
            // Extension save failed
        }

        // 보호 수준 저장
        // Save protection level
        PPROTECTION_POLICY_ENTRY defaultPolicy = PolicyFind(DEFAULT_POLICY_ID);
        if (defaultPolicy != NULL) {
            ULONG level = PROTECTION_LEVEL_HIGH;  // 플래그에서 역산 필요
                                                  // Need to reverse from flags

            status = WriteRegistryValue(
                keyHandle,
                L"ProtectionLevel",
                REG_DWORD,
                &level,
                sizeof(level)
            );
        }

        DbgPrint("[Policy] 정책 저장 완료\n");
        // Policy saved
        status = STATUS_SUCCESS;
    }
    __finally {
        if (keyHandle != NULL) {
            ZwClose(keyHandle);
        }
    }

    return status;
}
```

### 24.3.2 동적 정책 업데이트

```c
// policy_update.h - 동적 정책 업데이트
// policy_update.h - Dynamic policy updates

#pragma once

// 정책 변경 알림 타입
// Policy change notification type
typedef enum _POLICY_CHANGE_TYPE {
    POLICY_CHANGE_ADD,          // 정책 추가
                                // Policy added
    POLICY_CHANGE_REMOVE,       // 정책 제거
                                // Policy removed
    POLICY_CHANGE_MODIFY,       // 정책 수정
                                // Policy modified
    POLICY_CHANGE_RELOAD        // 전체 재로드
                                // Full reload
} POLICY_CHANGE_TYPE;

// 정책 변경 콜백
// Policy change callback
typedef VOID (*POLICY_CHANGE_CALLBACK)(
    _In_ POLICY_CHANGE_TYPE ChangeType,
    _In_opt_ POLICY_ID PolicyId,
    _In_opt_ PVOID Context);

// 콜백 등록
// Register callback
NTSTATUS RegisterPolicyChangeCallback(
    _In_ POLICY_CHANGE_CALLBACK Callback,
    _In_opt_ PVOID Context);

// 콜백 해제
// Unregister callback
VOID UnregisterPolicyChangeCallback(
    _In_ POLICY_CHANGE_CALLBACK Callback);

// 정책 변경 알림 발송
// Notify policy change
VOID NotifyPolicyChange(
    _In_ POLICY_CHANGE_TYPE ChangeType,
    _In_opt_ POLICY_ID PolicyId);
```

```c
// policy_update.c - 동적 정책 업데이트 구현
// policy_update.c - Dynamic policy update implementation

#include "policy_update.h"

#define MAX_CALLBACKS 8

// 콜백 항목
// Callback entry
typedef struct _CALLBACK_ENTRY {
    POLICY_CHANGE_CALLBACK Callback;
    PVOID Context;
    BOOLEAN InUse;
} CALLBACK_ENTRY, *PCALLBACK_ENTRY;

// 콜백 관리자
// Callback manager
static struct {
    CALLBACK_ENTRY Callbacks[MAX_CALLBACKS];
    FAST_MUTEX Lock;
    BOOLEAN Initialized;
} g_CallbackManager;

// 초기화
// Initialize
NTSTATUS PolicyUpdateInitialize(VOID)
{
    RtlZeroMemory(&g_CallbackManager, sizeof(g_CallbackManager));
    ExInitializeFastMutex(&g_CallbackManager.Lock);
    g_CallbackManager.Initialized = TRUE;
    return STATUS_SUCCESS;
}

// 정리
// Cleanup
VOID PolicyUpdateCleanup(VOID)
{
    g_CallbackManager.Initialized = FALSE;
}

// 콜백 등록
// Register callback
NTSTATUS RegisterPolicyChangeCallback(
    _In_ POLICY_CHANGE_CALLBACK Callback,
    _In_opt_ PVOID Context)
{
    NTSTATUS status = STATUS_INSUFFICIENT_RESOURCES;

    if (!g_CallbackManager.Initialized) {
        return STATUS_NOT_INITIALIZED;
    }

    ExAcquireFastMutex(&g_CallbackManager.Lock);

    for (int i = 0; i < MAX_CALLBACKS; i++) {
        if (!g_CallbackManager.Callbacks[i].InUse) {
            g_CallbackManager.Callbacks[i].Callback = Callback;
            g_CallbackManager.Callbacks[i].Context = Context;
            g_CallbackManager.Callbacks[i].InUse = TRUE;
            status = STATUS_SUCCESS;
            break;
        }
    }

    ExReleaseFastMutex(&g_CallbackManager.Lock);

    return status;
}

// 콜백 해제
// Unregister callback
VOID UnregisterPolicyChangeCallback(_In_ POLICY_CHANGE_CALLBACK Callback)
{
    if (!g_CallbackManager.Initialized) {
        return;
    }

    ExAcquireFastMutex(&g_CallbackManager.Lock);

    for (int i = 0; i < MAX_CALLBACKS; i++) {
        if (g_CallbackManager.Callbacks[i].InUse &&
            g_CallbackManager.Callbacks[i].Callback == Callback) {
            g_CallbackManager.Callbacks[i].InUse = FALSE;
            g_CallbackManager.Callbacks[i].Callback = NULL;
            g_CallbackManager.Callbacks[i].Context = NULL;
            break;
        }
    }

    ExReleaseFastMutex(&g_CallbackManager.Lock);
}

// 정책 변경 알림
// Notify policy change
VOID NotifyPolicyChange(
    _In_ POLICY_CHANGE_TYPE ChangeType,
    _In_opt_ POLICY_ID PolicyId)
{
    if (!g_CallbackManager.Initialized) {
        return;
    }

    // 콜백 목록 복사 (잠금 최소화)
    // Copy callback list (minimize lock time)
    CALLBACK_ENTRY localCallbacks[MAX_CALLBACKS];

    ExAcquireFastMutex(&g_CallbackManager.Lock);
    RtlCopyMemory(localCallbacks, g_CallbackManager.Callbacks,
                  sizeof(localCallbacks));
    ExReleaseFastMutex(&g_CallbackManager.Lock);

    // 콜백 호출 (잠금 없이)
    // Call callbacks (without lock)
    for (int i = 0; i < MAX_CALLBACKS; i++) {
        if (localCallbacks[i].InUse && localCallbacks[i].Callback != NULL) {
            __try {
                localCallbacks[i].Callback(
                    ChangeType,
                    PolicyId,
                    localCallbacks[i].Context
                );
            }
            __except (EXCEPTION_EXECUTE_HANDLER) {
                DbgPrint("[Policy] 콜백 예외 발생: 0x%X\n", GetExceptionCode());
                // Callback exception occurred
            }
        }
    }

    DbgPrint("[Policy] 정책 변경 알림: 타입=%d, ID=%lu\n",
             ChangeType, PolicyId);
    // Policy change notification: type, ID
}

// 사용자 모드 IOCTL 처리 (정책 업데이트)
// User mode IOCTL handler (policy update)
NTSTATUS HandlePolicyUpdateIoctl(
    _In_ PIRP Irp,
    _In_ PIO_STACK_LOCATION IoStackLocation)
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID inputBuffer = Irp->AssociatedIrp.SystemBuffer;
    ULONG inputLength = IoStackLocation->Parameters.DeviceIoControl.InputBufferLength;
    ULONG controlCode = IoStackLocation->Parameters.DeviceIoControl.IoControlCode;

    switch (controlCode) {
        case IOCTL_ADD_EXTENSION: {
            if (inputLength < sizeof(WCHAR) * 2) {
                status = STATUS_BUFFER_TOO_SMALL;
                break;
            }

            status = ExtensionFilterAdd((PCWSTR)inputBuffer);
            if (NT_SUCCESS(status)) {
                NotifyPolicyChange(POLICY_CHANGE_MODIFY, DEFAULT_POLICY_ID);
            }
            break;
        }

        case IOCTL_REMOVE_EXTENSION: {
            if (inputLength < sizeof(WCHAR) * 2) {
                status = STATUS_BUFFER_TOO_SMALL;
                break;
            }

            status = ExtensionFilterRemove((PCWSTR)inputBuffer);
            if (NT_SUCCESS(status)) {
                NotifyPolicyChange(POLICY_CHANGE_MODIFY, DEFAULT_POLICY_ID);
            }
            break;
        }

        case IOCTL_RELOAD_POLICY: {
            status = LoadPoliciesFromRegistry();
            if (NT_SUCCESS(status)) {
                NotifyPolicyChange(POLICY_CHANGE_RELOAD, INVALID_POLICY_ID);
            }
            break;
        }

        default:
            status = STATUS_INVALID_DEVICE_REQUEST;
            break;
    }

    return status;
}
```

---

## 24.4 Minifilter 통합

### 24.4.1 정책 기반 콜백 구현

```c
// drm_filter.c - DRM Minifilter 정책 통합
// drm_filter.c - DRM Minifilter policy integration

#include <fltKernel.h>
#include "protection_policy.h"

// Pre-Create 콜백에서 정책 평가
// Evaluate policy in Pre-Create callback
FLT_PREOP_CALLBACK_STATUS PreCreateWithPolicy(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    PROTECTION_FLAGS protectionFlags;
    ACCESS_MASK desiredAccess;
    ULONG createDisposition;

    UNREFERENCED_PARAMETER(FltObjects);
    *CompletionContext = NULL;

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 액세스 정보 가져오기
    // Get access information
    desiredAccess = Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess;
    createDisposition = (Data->Iopb->Parameters.Create.Options >> 24) & 0xFF;

    // 파일명 가져오기
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

    // 정책 평가 (콘텐츠 없이 - Create 단계)
    // Evaluate policy (without content - Create phase)
    protectionFlags = PolicyEvaluate(
        &nameInfo->Name,
        NULL,   // 콘텐츠 아직 없음
                // No content yet
        0,
        PsGetCurrentProcessId()
    );

    FltReleaseFileNameInformation(nameInfo);

    if (protectionFlags == PROTECT_NONE) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 보호 플래그에 따른 차단 결정
    // Decide blocking based on protection flags

    // 쓰기 관련 액세스 차단
    // Block write-related access
    if ((protectionFlags & PROTECT_WRITE) &&
        (desiredAccess & (FILE_WRITE_DATA | FILE_APPEND_DATA))) {
        DbgPrint("[DRM] 쓰기 차단: 플래그=0x%04X\n", protectionFlags);
        // Write blocked: flags

        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    // 삭제 액세스 차단
    // Block delete access
    if ((protectionFlags & PROTECT_DELETE) &&
        (desiredAccess & DELETE)) {
        // FILE_SUPERSEDE, FILE_OVERWRITE도 확인
        // Also check FILE_SUPERSEDE, FILE_OVERWRITE
        if (createDisposition == FILE_SUPERSEDE ||
            createDisposition == FILE_OVERWRITE ||
            createDisposition == FILE_OVERWRITE_IF) {
            DbgPrint("[DRM] 덮어쓰기/삭제 차단\n");
            // Overwrite/delete blocked

            Data->IoStatus.Status = STATUS_ACCESS_DENIED;
            Data->IoStatus.Information = 0;
            return FLT_PREOP_COMPLETE;
        }
    }

    // 컨텍스트에 플래그 저장 (후속 콜백용)
    // Store flags in context (for subsequent callbacks)
    // (실제 구현에서는 Stream Context 사용)
    // (In real implementation, use Stream Context)

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

---

## 24.5 요약

### 핵심 내용 정리

| 항목 | 내용 |
|------|------|
| **보호 대상 식별** | 확장자, 경로 패턴, 콘텐츠 기반 필터링 |
| **보호 플래그** | 비트 필드로 다중 보호 옵션 조합 가능 |
| **정책 저장** | 레지스트리 기반 영속적 저장 |
| **동적 업데이트** | 콜백 메커니즘으로 런타임 정책 변경 지원 |

### .NET 개발자 참고 사항

```
┌─────────────────────────────────────────────────────────────────┐
│                 C# vs C 정책 시스템 비교                         │
├─────────────────────────────────────────────────────────────────┤
│ C# (사용자 모드 애플리케이션)    │ C (커널 드라이버)               │
├─────────────────────────────────┼─────────────────────────────────┤
│ [Flags] enum                    │ typedef enum (비트 필드)        │
│ Microsoft.Win32.Registry        │ ZwOpenKey/ZwQueryValueKey       │
│ List<Policy>                    │ 정적 배열 + 수동 관리           │
│ event PolicyChanged             │ 콜백 함수 포인터                │
│ lock 키워드                     │ ERESOURCE / FAST_MUTEX         │
│ 자동 직렬화 (JSON/XML)          │ 수동 바이너리 변환              │
└─────────────────────────────────┴─────────────────────────────────┘
```

### 다음 챕터 예고

Chapter 25에서는 **파일 수정 차단 구현**을 다룹니다. IRP_MJ_WRITE 콜백을 활용하여 보호된 파일에 대한 쓰기 작업을 차단하는 방법을 구현합니다.

---

## 참고 자료

- [Microsoft Docs: Filter Manager Concepts](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/filter-manager-concepts)
- [Microsoft Docs: Registry Programming](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/registry-key-object-routines)
- [Windows Driver Security Best Practices](https://docs.microsoft.com/en-us/windows-hardware/drivers/driversecurity/)
