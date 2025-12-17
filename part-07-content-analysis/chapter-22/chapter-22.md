# Chapter 22: 정규 표현식과 패턴 매칭 (커널 모드)

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
- 커널 모드 문자열 처리 기법 마스터
- 효율적인 패턴 매칭 알고리즘 이해 및 구현
- 주민등록번호 감지 엔진 구현

---

## 22.1 커널 모드에서의 문자열 처리

### 22.1.1 커널 문자열 처리의 특수성

커널 모드에서의 문자열 처리는 사용자 모드와 근본적으로 다릅니다. C# 개발자에게 익숙한 `string` 타입이나 정규 표현식 라이브러리를 사용할 수 없으며, 제한된 API로 직접 구현해야 합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                 사용자 모드 vs 커널 모드 문자열 처리               │
├────────────────────────────────┬────────────────────────────────┤
│         사용자 모드             │          커널 모드              │
├────────────────────────────────┼────────────────────────────────┤
│ System.String (불변)           │ UNICODE_STRING (가변 참조)      │
│ Regex 클래스 사용 가능          │ 정규 표현식 라이브러리 없음       │
│ GC가 메모리 관리               │ 수동 메모리 관리 필수            │
│ 예외 발생 시 앱 종료           │ 예외 발생 시 BSOD               │
│ StringBuilder 가용            │ 직접 버퍼 관리                  │
│ string.Format() 사용          │ RtlStringCb* 함수들             │
└────────────────────────────────┴────────────────────────────────┘
```

### 22.1.2 UNICODE_STRING 구조체 심화

Windows 커널의 기본 문자열 타입인 UNICODE_STRING을 완전히 이해해야 합니다.

```c
// UNICODE_STRING 구조체 정의
// UNICODE_STRING structure definition
typedef struct _UNICODE_STRING {
    USHORT Length;           // 현재 문자열 길이 (바이트)
                             // Current string length in bytes
    USHORT MaximumLength;    // 버퍼 최대 크기 (바이트)
                             // Maximum buffer size in bytes
    PWCH   Buffer;           // 실제 문자열 버퍼 (NULL 종료 아닐 수 있음!)
                             // Actual string buffer (may NOT be null-terminated!)
} UNICODE_STRING, *PUNICODE_STRING;
```

**C# string과의 비교:**

```csharp
// C#: string은 불변이며 길이 정보를 내부적으로 관리
// C#: string is immutable and manages length internally
string text = "Hello";
int length = text.Length;  // 문자 수 반환
                           // Returns character count

// C#에서 UNICODE_STRING 개념을 표현한다면:
// If UNICODE_STRING concept were expressed in C#:
public struct UnicodeString
{
    public ushort Length;         // 바이트 단위!
                                  // In bytes!
    public ushort MaximumLength;  // 바이트 단위!
                                  // In bytes!
    public unsafe char* Buffer;   // null 종료 보장 안됨
                                  // Not guaranteed null-terminated
}
```

### 22.1.3 UNICODE_STRING 초기화 방법

```c
// 방법 1: 정적 초기화 (컴파일 타임)
// Method 1: Static initialization (compile-time)
UNICODE_STRING staticString = RTL_CONSTANT_STRING(L"Hello");

// 방법 2: 런타임 초기화
// Method 2: Runtime initialization
UNICODE_STRING dynamicString;
RtlInitUnicodeString(&dynamicString, L"Hello World");

// 방법 3: 버퍼 기반 초기화
// Method 3: Buffer-based initialization
WCHAR buffer[256];
UNICODE_STRING bufferString;
bufferString.Buffer = buffer;
bufferString.Length = 0;
bufferString.MaximumLength = sizeof(buffer);

// 방법 4: 동적 할당
// Method 4: Dynamic allocation
UNICODE_STRING allocString;
allocString.MaximumLength = 512;
allocString.Length = 0;
allocString.Buffer = (PWCH)ExAllocatePool2(
    POOL_FLAG_PAGED,
    allocString.MaximumLength,
    'rtSU'
);
if (allocString.Buffer == NULL) {
    return STATUS_INSUFFICIENT_RESOURCES;
}
```

### 22.1.4 문자열 비교 함수들

```c
// 문자열 비교 함수 래퍼
// String comparison wrapper functions

// 대소문자 구분 비교
// Case-sensitive comparison
BOOLEAN StringEquals(
    PCUNICODE_STRING String1,
    PCUNICODE_STRING String2)
{
    return RtlEqualUnicodeString(String1, String2, FALSE);
}

// 대소문자 무시 비교
// Case-insensitive comparison
BOOLEAN StringEqualsIgnoreCase(
    PCUNICODE_STRING String1,
    PCUNICODE_STRING String2)
{
    return RtlEqualUnicodeString(String1, String2, TRUE);
}

// 접두사 확인
// Check prefix
BOOLEAN StringStartsWith(
    PCUNICODE_STRING String,
    PCUNICODE_STRING Prefix,
    BOOLEAN CaseInsensitive)
{
    return RtlPrefixUnicodeString(Prefix, String, CaseInsensitive);
}

// 접미사 확인 (확장자 검사에 유용)
// Check suffix (useful for extension checking)
BOOLEAN StringEndsWith(
    PCUNICODE_STRING String,
    PCUNICODE_STRING Suffix,
    BOOLEAN CaseInsensitive)
{
    return RtlSuffixUnicodeString(Suffix, String, CaseInsensitive);
}

// 사용 예시
// Usage example
NTSTATUS CheckFileExtension(PCUNICODE_STRING FileName)
{
    UNICODE_STRING txtExt = RTL_CONSTANT_STRING(L".txt");
    UNICODE_STRING docExt = RTL_CONSTANT_STRING(L".doc");
    UNICODE_STRING docxExt = RTL_CONSTANT_STRING(L".docx");

    if (StringEndsWith(FileName, &txtExt, TRUE)) {
        DbgPrint("텍스트 파일 감지됨\n");
        // Text file detected
        return STATUS_SUCCESS;
    }

    if (StringEndsWith(FileName, &docExt, TRUE) ||
        StringEndsWith(FileName, &docxExt, TRUE)) {
        DbgPrint("Word 문서 감지됨\n");
        // Word document detected
        return STATUS_SUCCESS;
    }

    return STATUS_NOT_FOUND;
}
```

### 22.1.5 안전한 문자열 함수 (RtlStringCb*)

Windows 커널은 버퍼 오버플로우 방지를 위한 안전한 문자열 함수 제공합니다.

```c
#include <ntstrsafe.h>  // 필수 헤더
                        // Required header

// 안전한 문자열 복사
// Safe string copy
NTSTATUS SafeStringCopy(
    PWCHAR Destination,
    SIZE_T DestinationSize,
    PCWSTR Source)
{
    NTSTATUS status;

    // RtlStringCbCopyW: 바이트 수 기반 복사
    // RtlStringCbCopyW: Byte count-based copy
    status = RtlStringCbCopyW(
        Destination,
        DestinationSize,
        Source
    );

    if (!NT_SUCCESS(status)) {
        // STATUS_BUFFER_OVERFLOW: 잘림 발생
        // STATUS_BUFFER_OVERFLOW: Truncation occurred
        // STATUS_INVALID_PARAMETER: 잘못된 파라미터
        // STATUS_INVALID_PARAMETER: Invalid parameter
        DbgPrint("문자열 복사 실패: 0x%X\n", status);
        // String copy failed
    }

    return status;
}

// 안전한 문자열 연결
// Safe string concatenation
NTSTATUS SafeStringConcat(
    PWCHAR Destination,
    SIZE_T DestinationSize,
    PCWSTR Source)
{
    return RtlStringCbCatW(Destination, DestinationSize, Source);
}

// 안전한 포맷 문자열
// Safe format string
NTSTATUS SafeStringFormat(
    PWCHAR Destination,
    SIZE_T DestinationSize,
    PCWSTR Format,
    ...)
{
    NTSTATUS status;
    va_list args;

    va_start(args, Format);

    status = RtlStringCbVPrintfW(
        Destination,
        DestinationSize,
        Format,
        args
    );

    va_end(args);

    return status;
}

// 사용 예시
// Usage example
VOID DemoSafeStrings(VOID)
{
    WCHAR buffer[128];
    NTSTATUS status;

    // 복사
    // Copy
    status = RtlStringCbCopyW(
        buffer,
        sizeof(buffer),
        L"파일 경로: "
        // "File path: "
    );

    if (NT_SUCCESS(status)) {
        // 연결
        // Concatenate
        status = RtlStringCbCatW(
            buffer,
            sizeof(buffer),
            L"C:\\Test\\Document.txt"
        );
    }

    if (NT_SUCCESS(status)) {
        DbgPrint("%ws\n", buffer);
    }
}
```

### 22.1.6 바이트 배열에서 문자열 검색

커널에서 파일 콘텐츠를 스캔할 때는 바이트 배열 기반 검색이 필요합니다.

```c
// 바이트 배열에서 패턴 찾기
// Find pattern in byte array
BOOLEAN FindBytePattern(
    _In_ PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _In_ PUCHAR Pattern,
    _In_ ULONG PatternLength,
    _Out_opt_ PULONG FoundOffset)
{
    if (PatternLength == 0 || PatternLength > BufferLength) {
        return FALSE;
    }

    // 단순 검색 알고리즘
    // Simple search algorithm
    for (ULONG i = 0; i <= BufferLength - PatternLength; i++) {
        BOOLEAN found = TRUE;

        for (ULONG j = 0; j < PatternLength; j++) {
            if (Buffer[i + j] != Pattern[j]) {
                found = FALSE;
                break;
            }
        }

        if (found) {
            if (FoundOffset != NULL) {
                *FoundOffset = i;
            }
            return TRUE;
        }
    }

    return FALSE;
}

// 유니코드 문자열에서 ASCII 패턴 찾기
// Find ASCII pattern in Unicode string
BOOLEAN FindAsciiInUnicode(
    _In_ PWCHAR Buffer,
    _In_ ULONG CharCount,
    _In_ PCSTR AsciiPattern,
    _Out_opt_ PULONG FoundOffset)
{
    ULONG patternLen = (ULONG)strlen(AsciiPattern);

    if (patternLen == 0 || patternLen > CharCount) {
        return FALSE;
    }

    for (ULONG i = 0; i <= CharCount - patternLen; i++) {
        BOOLEAN found = TRUE;

        for (ULONG j = 0; j < patternLen; j++) {
            // 유니코드 문자의 하위 바이트만 비교
            // Compare only lower byte of Unicode character
            if ((CHAR)Buffer[i + j] != AsciiPattern[j]) {
                found = FALSE;
                break;
            }
        }

        if (found) {
            if (FoundOffset != NULL) {
                *FoundOffset = i;
            }
            return TRUE;
        }
    }

    return FALSE;
}
```

---

## 22.2 패턴 매칭 알고리즘

### 22.2.1 알고리즘 선택 기준

커널 모드에서 패턴 매칭 알고리즘 선택 시 고려사항:

```
┌─────────────────────────────────────────────────────────────────┐
│                    패턴 매칭 알고리즘 비교                        │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│   알고리즘    │  시간 복잡도  │  공간 복잡도  │     적합한 경우     │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ Naive        │ O(nm)        │ O(1)         │ 짧은 패턴, 단순 구현 │
│              │              │              │ Short pattern       │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ KMP          │ O(n+m)       │ O(m)         │ 반복 패턴 많을 때    │
│              │              │              │ Many repeating      │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ Boyer-Moore  │ O(n/m) best  │ O(σ+m)       │ 긴 패턴, 대용량     │
│              │ O(nm) worst  │              │ Long pattern        │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ FSM 기반     │ O(n)         │ O(m²)        │ 고정 패턴 반복 검색  │
│ (DFA/NFA)    │              │              │ Repeated search     │
└──────────────┴──────────────┴──────────────┴────────────────────┘

n = 텍스트 길이, m = 패턴 길이, σ = 알파벳 크기
n = text length, m = pattern length, σ = alphabet size
```

### 22.2.2 Naive 알고리즘 (커널 최적화 버전)

주민등록번호처럼 고정 길이 패턴에는 Naive 알고리즘도 효율적입니다.

```c
// 최적화된 Naive 검색 (SSN 전용)
// Optimized Naive search (SSN-specific)
typedef struct _SEARCH_RESULT {
    ULONG Offset;       // 발견 위치
                        // Found position
    BOOLEAN IsValid;    // 유효성 검증 통과 여부
                        // Validation passed
} SEARCH_RESULT, *PSEARCH_RESULT;

ULONG NaiveSSNSearch(
    _In_ PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _Out_writes_(MaxResults) PSEARCH_RESULT Results,
    _In_ ULONG MaxResults)
{
    ULONG resultCount = 0;

    // SSN 형식: NNNNNN-NNNNNNN (14 바이트)
    // SSN format: NNNNNN-NNNNNNN (14 bytes)
    const ULONG SSN_LENGTH = 14;
    const ULONG HYPHEN_POS = 6;

    if (BufferLength < SSN_LENGTH) {
        return 0;
    }

    // 하이픈 위치를 먼저 찾아 검색 최적화
    // Optimize by finding hyphen first
    for (ULONG i = HYPHEN_POS; i <= BufferLength - SSN_LENGTH + HYPHEN_POS; i++) {
        // 하이픈 확인 (빠른 필터링)
        // Check hyphen (fast filtering)
        if (Buffer[i] != '-') {
            continue;
        }

        ULONG start = i - HYPHEN_POS;

        // 앞 6자리 숫자 확인
        // Check first 6 digits
        BOOLEAN validPrefix = TRUE;
        for (ULONG j = 0; j < 6; j++) {
            if (Buffer[start + j] < '0' || Buffer[start + j] > '9') {
                validPrefix = FALSE;
                break;
            }
        }

        if (!validPrefix) {
            continue;
        }

        // 뒤 7자리 숫자 확인
        // Check last 7 digits
        BOOLEAN validSuffix = TRUE;
        for (ULONG j = 7; j < 14; j++) {
            if (Buffer[start + j] < '0' || Buffer[start + j] > '9') {
                validSuffix = FALSE;
                break;
            }
        }

        if (!validSuffix) {
            continue;
        }

        // 결과 저장
        // Save result
        if (resultCount < MaxResults) {
            Results[resultCount].Offset = start;
            Results[resultCount].IsValid = TRUE;  // 후속 유효성 검증 필요
                                                  // Needs further validation
            resultCount++;
        }
    }

    return resultCount;
}
```

### 22.2.3 Boyer-Moore 알고리즘 개요

대용량 파일 스캔에서는 Boyer-Moore가 효과적입니다.

```c
// Boyer-Moore Bad Character 테이블 생성
// Boyer-Moore Bad Character table generation
#define ALPHABET_SIZE 256

typedef struct _BM_PATTERN {
    PUCHAR Pattern;
    ULONG PatternLength;
    LONG BadChar[ALPHABET_SIZE];
} BM_PATTERN, *PBM_PATTERN;

VOID InitBoyerMoore(
    _Out_ PBM_PATTERN Bm,
    _In_ PUCHAR Pattern,
    _In_ ULONG PatternLength)
{
    Bm->Pattern = Pattern;
    Bm->PatternLength = PatternLength;

    // Bad Character 테이블 초기화
    // Initialize Bad Character table
    for (int i = 0; i < ALPHABET_SIZE; i++) {
        Bm->BadChar[i] = -1;
    }

    // 패턴에 있는 문자의 마지막 위치 기록
    // Record last position of characters in pattern
    for (ULONG i = 0; i < PatternLength; i++) {
        Bm->BadChar[Pattern[i]] = i;
    }
}

BOOLEAN BoyerMooreSearch(
    _In_ PBM_PATTERN Bm,
    _In_ PUCHAR Text,
    _In_ ULONG TextLength,
    _Out_opt_ PULONG FoundOffset)
{
    if (Bm->PatternLength > TextLength) {
        return FALSE;
    }

    LONG shift = 0;
    LONG patLen = (LONG)Bm->PatternLength;
    LONG textLen = (LONG)TextLength;

    while (shift <= textLen - patLen) {
        LONG j = patLen - 1;

        // 패턴 끝에서부터 비교
        // Compare from end of pattern
        while (j >= 0 && Bm->Pattern[j] == Text[shift + j]) {
            j--;
        }

        if (j < 0) {
            // 패턴 발견!
            // Pattern found!
            if (FoundOffset != NULL) {
                *FoundOffset = (ULONG)shift;
            }
            return TRUE;
        }

        // Bad Character 규칙으로 이동
        // Shift using Bad Character rule
        LONG badCharShift = j - Bm->BadChar[Text[shift + j]];
        shift += max(1, badCharShift);
    }

    return FALSE;
}
```

### 22.2.4 상태 기계 기반 매칭

고정 패턴을 반복 검색할 때는 DFA(결정적 유한 오토마타)가 효율적입니다.

```c
// SSN 패턴용 상태 기계
// State machine for SSN pattern
typedef enum _SSN_STATE {
    SSN_START = 0,
    SSN_DIGIT_1,        // 첫 번째 숫자
                        // First digit
    SSN_DIGIT_2,
    SSN_DIGIT_3,
    SSN_DIGIT_4,
    SSN_DIGIT_5,
    SSN_DIGIT_6,
    SSN_HYPHEN,         // 하이픈
                        // Hyphen
    SSN_DIGIT_7,
    SSN_DIGIT_8,
    SSN_DIGIT_9,
    SSN_DIGIT_10,
    SSN_DIGIT_11,
    SSN_DIGIT_12,
    SSN_DIGIT_13,
    SSN_COMPLETE,       // 완료
                        // Complete
    SSN_STATE_COUNT
} SSN_STATE;

typedef struct _SSN_FSM {
    SSN_STATE CurrentState;
    ULONG StartOffset;      // 패턴 시작 위치
                            // Pattern start position
    UCHAR Digits[13];       // 감지된 숫자들
                            // Detected digits
    ULONG DigitIndex;
} SSN_FSM, *PSSN_FSM;

VOID ResetSSNFsm(PSSN_FSM Fsm)
{
    Fsm->CurrentState = SSN_START;
    Fsm->StartOffset = 0;
    Fsm->DigitIndex = 0;
}

// 상태 전이 함수
// State transition function
SSN_STATE TransitionSSN(SSN_STATE Current, UCHAR Input)
{
    BOOLEAN isDigit = (Input >= '0' && Input <= '9');
    BOOLEAN isHyphen = (Input == '-');

    switch (Current) {
        case SSN_START:
        case SSN_DIGIT_1:
        case SSN_DIGIT_2:
        case SSN_DIGIT_3:
        case SSN_DIGIT_4:
        case SSN_DIGIT_5:
            return isDigit ? (SSN_STATE)(Current + 1) : SSN_START;

        case SSN_DIGIT_6:
            return isHyphen ? SSN_HYPHEN :
                   (isDigit ? SSN_DIGIT_1 : SSN_START);

        case SSN_HYPHEN:
        case SSN_DIGIT_7:
        case SSN_DIGIT_8:
        case SSN_DIGIT_9:
        case SSN_DIGIT_10:
        case SSN_DIGIT_11:
        case SSN_DIGIT_12:
            return isDigit ? (SSN_STATE)(Current + 1) :
                   (isDigit ? SSN_DIGIT_1 : SSN_START);

        case SSN_DIGIT_13:
            return SSN_COMPLETE;

        default:
            return SSN_START;
    }
}

// FSM 기반 SSN 검색
// FSM-based SSN search
ULONG ScanWithFSM(
    _In_ PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _Out_writes_(MaxMatches) PULONG MatchOffsets,
    _In_ ULONG MaxMatches)
{
    SSN_FSM fsm;
    ResetSSNFsm(&fsm);

    ULONG matchCount = 0;

    for (ULONG i = 0; i < BufferLength && matchCount < MaxMatches; i++) {
        UCHAR c = Buffer[i];
        SSN_STATE prevState = fsm.CurrentState;
        fsm.CurrentState = TransitionSSN(fsm.CurrentState, c);

        // 시작 위치 기록
        // Record start position
        if (prevState == SSN_START && fsm.CurrentState == SSN_DIGIT_1) {
            fsm.StartOffset = i;
        }

        // 완료 상태 도달
        // Reached complete state
        if (fsm.CurrentState == SSN_COMPLETE) {
            MatchOffsets[matchCount++] = fsm.StartOffset;
            ResetSSNFsm(&fsm);

            // 현재 문자가 숫자면 새 패턴 시작 가능
            // If current char is digit, may start new pattern
            if (c >= '0' && c <= '9') {
                fsm.CurrentState = SSN_DIGIT_1;
                fsm.StartOffset = i;
            }
        }
    }

    return matchCount;
}
```

---

## 22.3 주민등록번호 패턴 정의

### 22.3.1 주민등록번호 형식 분석

주민등록번호는 13자리 숫자로 구성되며, 각 자릿수에 의미가 있습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   주민등록번호 형식 분석                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     Y Y M M D D - G N N N N N C                                │
│     │ │ │ │ │ │   │ │ │ │ │ │ │                                │
│     └─┴─┴─┴─┴─┘   │ └─┴─┴─┴─┴─┘                                │
│         │         │       │                                     │
│    생년월일 (6)   성별(1) 일련번호+검증(6)                        │
│    Birth date    Gender  Serial + Check digit                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ 위치 │ 범위       │ 설명                                         │
├──────┼────────────┼──────────────────────────────────────────────┤
│ YY   │ 00-99      │ 출생년도 뒤 2자리 / Last 2 digits of year   │
│ MM   │ 01-12      │ 출생월 / Birth month                        │
│ DD   │ 01-31      │ 출생일 / Birth day                          │
│ G    │ 1-4 (9,0)  │ 성별 코드 / Gender code                     │
│      │ 1,3 = 남   │ Male                                        │
│      │ 2,4 = 여   │ Female                                      │
│      │ 1,2 = 1900s│ Born in 1900s                               │
│      │ 3,4 = 2000s│ Born in 2000s                               │
│ NNNNN│ 00001-     │ 출생신고 일련번호 / Birth registration seq   │
│ C    │ 0-9        │ 검증 숫자 / Check digit                     │
└──────┴────────────┴──────────────────────────────────────────────┘
```

### 22.3.2 검증 알고리즘 구현

```c
// 주민등록번호 유효성 검증 모듈
// SSN validation module

// 윤년 확인
// Check leap year
static BOOLEAN IsLeapYear(int year)
{
    return (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0);
}

// 월별 일수
// Days per month
static int DaysInMonth(int year, int month)
{
    static const int days[] = {0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};

    if (month == 2 && IsLeapYear(year)) {
        return 29;
    }

    return days[month];
}

// 성별 코드로부터 4자리 연도 계산
// Calculate 4-digit year from gender code
static int CalculateFullYear(int yearPart, int genderCode)
{
    switch (genderCode) {
        case 1:
        case 2:
            return 1900 + yearPart;
        case 3:
        case 4:
            return 2000 + yearPart;
        case 9:
        case 0:
            return 1800 + yearPart;  // 외국인 또는 1800년대
                                      // Foreigner or 1800s
        default:
            return 0;  // 잘못된 코드
                       // Invalid code
    }
}

// 날짜 유효성 검증
// Date validation
typedef struct _SSN_DATE_INFO {
    int Year;
    int Month;
    int Day;
    int GenderCode;
    BOOLEAN IsMale;
} SSN_DATE_INFO, *PSSN_DATE_INFO;

BOOLEAN ValidateSSNDate(
    _In_reads_(13) const UCHAR* Digits,
    _Out_opt_ PSSN_DATE_INFO DateInfo)
{
    // 각 자릿수를 정수로 변환
    // Convert each digit to integer
    int yearPart = (Digits[0] - '0') * 10 + (Digits[1] - '0');
    int month = (Digits[2] - '0') * 10 + (Digits[3] - '0');
    int day = (Digits[4] - '0') * 10 + (Digits[5] - '0');
    int genderCode = Digits[6] - '0';

    // 월 검증 (1-12)
    // Month validation (1-12)
    if (month < 1 || month > 12) {
        return FALSE;
    }

    // 성별 코드 검증
    // Gender code validation
    if (genderCode < 1 || genderCode > 4) {
        // 9, 0은 외국인용이므로 일단 제외
        // 9, 0 are for foreigners, excluded for now
        return FALSE;
    }

    // 4자리 연도 계산
    // Calculate 4-digit year
    int fullYear = CalculateFullYear(yearPart, genderCode);

    // 일 검증
    // Day validation
    int maxDays = DaysInMonth(fullYear, month);
    if (day < 1 || day > maxDays) {
        return FALSE;
    }

    // 미래 날짜 검증 (선택적)
    // Future date validation (optional)
    // 현재 연도보다 이후면 유효하지 않음
    // Invalid if after current year

    if (DateInfo != NULL) {
        DateInfo->Year = fullYear;
        DateInfo->Month = month;
        DateInfo->Day = day;
        DateInfo->GenderCode = genderCode;
        DateInfo->IsMale = (genderCode == 1 || genderCode == 3);
    }

    return TRUE;
}

// 검증 숫자 계산
// Calculate check digit
BOOLEAN ValidateSSNCheckDigit(_In_reads_(13) const UCHAR* Digits)
{
    // 가중치 배열
    // Weight array
    static const int weights[] = {2, 3, 4, 5, 6, 7, 8, 9, 2, 3, 4, 5};

    int sum = 0;

    // 처음 12자리에 가중치 적용
    // Apply weights to first 12 digits
    for (int i = 0; i < 12; i++) {
        sum += (Digits[i] - '0') * weights[i];
    }

    // 검증 숫자 계산: (11 - (sum % 11)) % 10
    // Check digit calculation
    int expectedCheckDigit = (11 - (sum % 11)) % 10;
    int actualCheckDigit = Digits[12] - '0';

    return (expectedCheckDigit == actualCheckDigit);
}

// 전체 유효성 검증
// Complete validation
typedef struct _SSN_VALIDATION_RESULT {
    BOOLEAN IsValid;
    BOOLEAN DateValid;
    BOOLEAN CheckDigitValid;
    SSN_DATE_INFO DateInfo;
    NTSTATUS ErrorCode;
} SSN_VALIDATION_RESULT, *PSSN_VALIDATION_RESULT;

VOID ValidateSSN(
    _In_reads_(13) const UCHAR* Digits,
    _Out_ PSSN_VALIDATION_RESULT Result)
{
    RtlZeroMemory(Result, sizeof(SSN_VALIDATION_RESULT));

    // 1. 모든 문자가 숫자인지 확인
    // 1. Check all characters are digits
    for (int i = 0; i < 13; i++) {
        if (Digits[i] < '0' || Digits[i] > '9') {
            Result->IsValid = FALSE;
            Result->ErrorCode = STATUS_INVALID_PARAMETER;
            return;
        }
    }

    // 2. 날짜 검증
    // 2. Date validation
    Result->DateValid = ValidateSSNDate(Digits, &Result->DateInfo);

    // 3. 검증 숫자 확인
    // 3. Check digit validation
    Result->CheckDigitValid = ValidateSSNCheckDigit(Digits);

    // 최종 결과
    // Final result
    Result->IsValid = Result->DateValid && Result->CheckDigitValid;
    Result->ErrorCode = Result->IsValid ? STATUS_SUCCESS : STATUS_DATA_ERROR;
}
```

### 22.3.3 다양한 표기 형식 처리

실제 문서에서는 다양한 형식으로 주민등록번호가 표기됩니다.

```c
// 지원하는 SSN 표기 형식들
// Supported SSN notation formats
typedef enum _SSN_FORMAT {
    SSN_FORMAT_STANDARD,       // 123456-1234567
    SSN_FORMAT_NO_HYPHEN,      // 1234561234567
    SSN_FORMAT_SPACES,         // 123456 1234567
    SSN_FORMAT_MASKED_LAST4,   // 123456-123****
    SSN_FORMAT_MASKED_LAST6,   // 123456-1******
    SSN_FORMAT_MASKED_BACK7,   // 123456-*******
    SSN_FORMAT_COUNT
} SSN_FORMAT;

// 형식 정규화 (모든 형식을 13자리 숫자로 변환)
// Format normalization (convert all formats to 13 digits)
NTSTATUS NormalizeSSN(
    _In_ PUCHAR Input,
    _In_ ULONG InputLength,
    _Out_writes_(13) PUCHAR NormalizedDigits,
    _Out_ PSSN_FORMAT DetectedFormat,
    _Out_ PBOOLEAN ContainsMask)
{
    *ContainsMask = FALSE;
    *DetectedFormat = SSN_FORMAT_STANDARD;

    ULONG digitIndex = 0;
    BOOLEAN foundSeparator = FALSE;
    ULONG separatorPos = 0;

    for (ULONG i = 0; i < InputLength && digitIndex < 13; i++) {
        UCHAR c = Input[i];

        if (c >= '0' && c <= '9') {
            NormalizedDigits[digitIndex++] = c;
        }
        else if (c == '-' || c == ' ') {
            if (!foundSeparator) {
                foundSeparator = TRUE;
                separatorPos = digitIndex;

                if (c == ' ') {
                    *DetectedFormat = SSN_FORMAT_SPACES;
                }
            }
        }
        else if (c == '*' || c == 'X' || c == 'x') {
            // 마스킹된 문자
            // Masked character
            *ContainsMask = TRUE;
            NormalizedDigits[digitIndex++] = '*';
        }
        else {
            // 예상치 못한 문자
            // Unexpected character
            return STATUS_INVALID_PARAMETER;
        }
    }

    // 13자리 확인
    // Verify 13 digits
    if (digitIndex != 13) {
        return STATUS_INVALID_PARAMETER;
    }

    // 구분자가 없으면 NO_HYPHEN 형식
    // If no separator, it's NO_HYPHEN format
    if (!foundSeparator) {
        *DetectedFormat = SSN_FORMAT_NO_HYPHEN;
    }

    // 마스킹 형식 판별
    // Determine masking format
    if (*ContainsMask) {
        ULONG maskCount = 0;
        ULONG firstMaskPos = 13;

        for (ULONG i = 0; i < 13; i++) {
            if (NormalizedDigits[i] == '*') {
                maskCount++;
                if (i < firstMaskPos) {
                    firstMaskPos = i;
                }
            }
        }

        if (maskCount == 7 && firstMaskPos == 6) {
            *DetectedFormat = SSN_FORMAT_MASKED_BACK7;
        }
        else if (maskCount == 6 && firstMaskPos == 7) {
            *DetectedFormat = SSN_FORMAT_MASKED_LAST6;
        }
        else if (maskCount == 4 && firstMaskPos == 9) {
            *DetectedFormat = SSN_FORMAT_MASKED_LAST4;
        }
    }

    return STATUS_SUCCESS;
}
```

---

## 22.4 [실습] 주민등록번호 감지 엔진 구현

### 22.4.1 감지 엔진 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                    SSN 감지 엔진 아키텍처                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ 파일 버퍼     │ -> │  패턴 검색    │ -> │  후보 목록    │      │
│  │ File Buffer  │    │ Pattern Scan │    │  Candidates  │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                 │               │
│                                                 v               │
│                      ┌──────────────────────────────────────┐   │
│                      │           유효성 검증 파이프라인        │   │
│                      │      Validation Pipeline             │   │
│                      ├──────────────────────────────────────┤   │
│                      │ 1. 형식 정규화   Format Normalization │   │
│                      │ 2. 날짜 검증     Date Validation      │   │
│                      │ 3. 검증자리 확인  Check Digit         │   │
│                      │ 4. 컨텍스트 분석  Context Analysis    │   │
│                      └───────────────────┬──────────────────┘   │
│                                          │                      │
│                                          v                      │
│                      ┌──────────────────────────────────────┐   │
│                      │         결과 목록 (검증된 SSN)         │   │
│                      │       Results (Validated SSN)        │   │
│                      └──────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 22.4.2 완전한 감지 엔진 구현

```c
// ssn_detector.h - 주민등록번호 감지 엔진 헤더
// ssn_detector.h - SSN Detection Engine Header

#pragma once
#include <ntifs.h>
#include <ntstrsafe.h>

// 최대 감지 개수
// Maximum detections per scan
#define MAX_SSN_MATCHES 100

// 매치 정보 구조체
// Match information structure
typedef struct _SSN_MATCH {
    ULONG Offset;               // 버퍼 내 위치
                                // Position in buffer
    ULONG Length;               // 원본 표기 길이
                                // Original notation length
    UCHAR OriginalText[20];     // 원본 텍스트
                                // Original text
    UCHAR NormalizedDigits[14]; // 정규화된 13자리 + NULL
                                // Normalized 13 digits + NULL
    SSN_FORMAT Format;          // 감지된 형식
                                // Detected format
    BOOLEAN IsMasked;           // 마스킹 여부
                                // Is masked
    BOOLEAN IsFullyValid;       // 완전 유효성 검증 통과
                                // Fully validated
    SSN_VALIDATION_RESULT Validation;  // 검증 상세 결과
                                       // Detailed validation result
} SSN_MATCH, *PSSN_MATCH;

// 스캔 결과 구조체
// Scan result structure
typedef struct _SSN_SCAN_RESULT {
    ULONG TotalCandidates;      // 후보 개수
                                // Number of candidates
    ULONG ValidatedCount;       // 검증 통과 개수
                                // Number validated
    ULONG MaskedCount;          // 마스킹된 개수
                                // Number masked
    SSN_MATCH Matches[MAX_SSN_MATCHES];
} SSN_SCAN_RESULT, *PSSN_SCAN_RESULT;

// 감지 엔진 컨텍스트
// Detection engine context
typedef struct _SSN_DETECTOR {
    BOOLEAN EnableStrictValidation;    // 엄격한 검증 모드
                                        // Strict validation mode
    BOOLEAN DetectMaskedSSN;           // 마스킹된 SSN도 감지
                                        // Also detect masked SSN
    BOOLEAN IgnoreNoHyphen;            // 하이픈 없는 형식 무시
                                        // Ignore format without hyphen
    ULONG MinContextLength;            // 컨텍스트 분석 최소 길이
                                        // Minimum context length
} SSN_DETECTOR, *PSSN_DETECTOR;

// 함수 선언
// Function declarations
NTSTATUS SsnDetectorInitialize(_Out_ PSSN_DETECTOR Detector);
VOID SsnDetectorCleanup(_In_ PSSN_DETECTOR Detector);

NTSTATUS SsnScanBuffer(
    _In_ PSSN_DETECTOR Detector,
    _In_reads_bytes_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength,
    _Out_ PSSN_SCAN_RESULT Result);

NTSTATUS SsnScanUnicodeBuffer(
    _In_ PSSN_DETECTOR Detector,
    _In_reads_(CharCount) PWCHAR Buffer,
    _In_ ULONG CharCount,
    _Out_ PSSN_SCAN_RESULT Result);
```

```c
// ssn_detector.c - 주민등록번호 감지 엔진 구현
// ssn_detector.c - SSN Detection Engine Implementation

#include "ssn_detector.h"

// 엔진 초기화
// Initialize engine
NTSTATUS SsnDetectorInitialize(_Out_ PSSN_DETECTOR Detector)
{
    RtlZeroMemory(Detector, sizeof(SSN_DETECTOR));

    // 기본 설정
    // Default settings
    Detector->EnableStrictValidation = TRUE;
    Detector->DetectMaskedSSN = TRUE;
    Detector->IgnoreNoHyphen = FALSE;
    Detector->MinContextLength = 0;

    return STATUS_SUCCESS;
}

// 엔진 정리
// Cleanup engine
VOID SsnDetectorCleanup(_In_ PSSN_DETECTOR Detector)
{
    UNREFERENCED_PARAMETER(Detector);
    // 현재 동적 할당 없음
    // Currently no dynamic allocations
}

// ASCII 버퍼에서 SSN 후보 검색
// Search SSN candidates in ASCII buffer
static ULONG FindSSNCandidates(
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _Out_writes_(MaxCandidates) PSSN_MATCH Candidates,
    _In_ ULONG MaxCandidates,
    _In_ PSSN_DETECTOR Detector)
{
    ULONG candidateCount = 0;

    if (BufferLength < 13) {
        return 0;
    }

    for (ULONG i = 0; i <= BufferLength - 13 && candidateCount < MaxCandidates; i++) {
        UCHAR c = Buffer[i];

        // 숫자로 시작하는지 확인
        // Check if starts with digit
        if (c < '0' || c > '9') {
            continue;
        }

        // 하이픈 형식 체크 (NNNNNN-NNNNNNN)
        // Check hyphen format
        if (i + 13 < BufferLength && Buffer[i + 6] == '-') {
            // 14바이트 형식 확인
            // Verify 14-byte format
            BOOLEAN valid = TRUE;

            for (int j = 0; j < 6 && valid; j++) {
                if (Buffer[i + j] < '0' || Buffer[i + j] > '9') {
                    valid = FALSE;
                }
            }

            for (int j = 7; j < 14 && valid; j++) {
                UCHAR ch = Buffer[i + j];
                // 숫자 또는 마스킹 문자
                // Digit or masking character
                if (!((ch >= '0' && ch <= '9') ||
                      ch == '*' || ch == 'X' || ch == 'x')) {
                    valid = FALSE;
                }
            }

            if (valid) {
                PSSN_MATCH match = &Candidates[candidateCount];
                match->Offset = i;
                match->Length = 14;
                RtlCopyMemory(match->OriginalText, &Buffer[i], 14);
                match->OriginalText[14] = '\0';

                candidateCount++;
                i += 13;  // 다음 위치로 스킵
                          // Skip to next position
                continue;
            }
        }

        // 하이픈 없는 형식 체크 (13자리 연속 숫자)
        // Check no-hyphen format (13 consecutive digits)
        if (!Detector->IgnoreNoHyphen) {
            BOOLEAN valid = TRUE;

            for (int j = 0; j < 13 && valid; j++) {
                if (Buffer[i + j] < '0' || Buffer[i + j] > '9') {
                    valid = FALSE;
                }
            }

            // 앞뒤 문자가 숫자가 아닌지 확인 (경계 체크)
            // Check boundary (prev/next chars not digits)
            if (valid) {
                if (i > 0 && Buffer[i - 1] >= '0' && Buffer[i - 1] <= '9') {
                    valid = FALSE;
                }
                if (i + 13 < BufferLength &&
                    Buffer[i + 13] >= '0' && Buffer[i + 13] <= '9') {
                    valid = FALSE;
                }
            }

            if (valid) {
                PSSN_MATCH match = &Candidates[candidateCount];
                match->Offset = i;
                match->Length = 13;
                RtlCopyMemory(match->OriginalText, &Buffer[i], 13);
                match->OriginalText[13] = '\0';

                candidateCount++;
                i += 12;
            }
        }
    }

    return candidateCount;
}

// 후보 검증 및 정규화
// Validate and normalize candidates
static VOID ValidateCandidates(
    _Inout_updates_(CandidateCount) PSSN_MATCH Candidates,
    _In_ ULONG CandidateCount,
    _In_ PSSN_DETECTOR Detector)
{
    for (ULONG i = 0; i < CandidateCount; i++) {
        PSSN_MATCH match = &Candidates[i];

        // 정규화
        // Normalize
        NTSTATUS status = NormalizeSSN(
            match->OriginalText,
            match->Length,
            match->NormalizedDigits,
            &match->Format,
            &match->IsMasked
        );

        if (!NT_SUCCESS(status)) {
            match->IsFullyValid = FALSE;
            continue;
        }

        match->NormalizedDigits[13] = '\0';

        // 마스킹된 SSN 처리
        // Handle masked SSN
        if (match->IsMasked) {
            if (!Detector->DetectMaskedSSN) {
                match->IsFullyValid = FALSE;
                continue;
            }

            // 마스킹된 부분은 검증 불가, 가용 부분만 검증
            // Masked parts can't be validated, check available parts
            match->IsFullyValid = ValidateSSNDate(
                match->NormalizedDigits,
                &match->Validation.DateInfo
            );
            match->Validation.CheckDigitValid = FALSE;  // 확인 불가
                                                        // Cannot verify
            continue;
        }

        // 완전 검증
        // Full validation
        ValidateSSN(match->NormalizedDigits, &match->Validation);

        if (Detector->EnableStrictValidation) {
            match->IsFullyValid = match->Validation.IsValid;
        } else {
            // 완화된 검증: 날짜만 유효하면 통과
            // Relaxed validation: pass if only date is valid
            match->IsFullyValid = match->Validation.DateValid;
        }
    }
}

// 메인 스캔 함수
// Main scan function
NTSTATUS SsnScanBuffer(
    _In_ PSSN_DETECTOR Detector,
    _In_reads_bytes_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength,
    _Out_ PSSN_SCAN_RESULT Result)
{
    NTSTATUS status = STATUS_SUCCESS;

    RtlZeroMemory(Result, sizeof(SSN_SCAN_RESULT));

    if (Buffer == NULL || BufferLength == 0) {
        return STATUS_INVALID_PARAMETER;
    }

    // 후보 검색
    // Find candidates
    Result->TotalCandidates = FindSSNCandidates(
        (PUCHAR)Buffer,
        BufferLength,
        Result->Matches,
        MAX_SSN_MATCHES,
        Detector
    );

    if (Result->TotalCandidates == 0) {
        return STATUS_SUCCESS;
    }

    // 후보 검증
    // Validate candidates
    ValidateCandidates(
        Result->Matches,
        Result->TotalCandidates,
        Detector
    );

    // 통계 계산
    // Calculate statistics
    for (ULONG i = 0; i < Result->TotalCandidates; i++) {
        if (Result->Matches[i].IsFullyValid) {
            Result->ValidatedCount++;
        }
        if (Result->Matches[i].IsMasked) {
            Result->MaskedCount++;
        }
    }

    return status;
}

// 유니코드 버퍼 스캔 (UTF-16)
// Scan Unicode buffer (UTF-16)
NTSTATUS SsnScanUnicodeBuffer(
    _In_ PSSN_DETECTOR Detector,
    _In_reads_(CharCount) PWCHAR Buffer,
    _In_ ULONG CharCount,
    _Out_ PSSN_SCAN_RESULT Result)
{
    NTSTATUS status;
    PUCHAR asciiBuffer = NULL;

    RtlZeroMemory(Result, sizeof(SSN_SCAN_RESULT));

    if (Buffer == NULL || CharCount == 0) {
        return STATUS_INVALID_PARAMETER;
    }

    // ASCII 변환 버퍼 할당
    // Allocate ASCII conversion buffer
    asciiBuffer = (PUCHAR)ExAllocatePool2(
        POOL_FLAG_PAGED,
        CharCount + 1,
        'nsSS'
    );

    if (asciiBuffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // 유니코드 -> ASCII 변환 (하위 바이트만)
    // Unicode -> ASCII conversion (lower byte only)
    for (ULONG i = 0; i < CharCount; i++) {
        WCHAR wc = Buffer[i];

        // ASCII 범위의 문자만 유지
        // Keep only ASCII-range characters
        if (wc <= 0x7F) {
            asciiBuffer[i] = (UCHAR)wc;
        } else {
            asciiBuffer[i] = ' ';  // 비-ASCII는 공백으로
                                   // Non-ASCII becomes space
        }
    }
    asciiBuffer[CharCount] = '\0';

    // ASCII 버퍼 스캔
    // Scan ASCII buffer
    status = SsnScanBuffer(
        Detector,
        asciiBuffer,
        CharCount,
        Result
    );

    // 오프셋 조정 (원본 유니코드 기준)
    // Adjust offsets (based on original Unicode)
    // 여기서는 1:1 매핑이므로 조정 불필요
    // Here 1:1 mapping so no adjustment needed

    ExFreePoolWithTag(asciiBuffer, 'nsSS');

    return status;
}
```

### 22.4.3 Minifilter 통합

```c
// ssn_minifilter.c - Minifilter와 SSN 감지 엔진 통합
// ssn_minifilter.c - Minifilter and SSN Detection Engine Integration

#include <fltKernel.h>
#include "ssn_detector.h"

// 전역 감지 엔진
// Global detection engine
SSN_DETECTOR g_SsnDetector;

// 드라이버 초기화 시 호출
// Called during driver initialization
NTSTATUS InitializeSsnDetection(VOID)
{
    NTSTATUS status;

    status = SsnDetectorInitialize(&g_SsnDetector);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 옵션 설정
    // Configure options
    g_SsnDetector.EnableStrictValidation = TRUE;
    g_SsnDetector.DetectMaskedSSN = TRUE;

    DbgPrint("[SSN] 감지 엔진 초기화 완료\n");
    // Detection engine initialized

    return STATUS_SUCCESS;
}

// 드라이버 언로드 시 호출
// Called during driver unload
VOID CleanupSsnDetection(VOID)
{
    SsnDetectorCleanup(&g_SsnDetector);

    DbgPrint("[SSN] 감지 엔진 정리 완료\n");
    // Detection engine cleaned up
}

// 파일 버퍼 스캔 및 결과 처리
// Scan file buffer and handle results
NTSTATUS ScanFileContent(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_reads_bytes_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength)
{
    NTSTATUS status;
    SSN_SCAN_RESULT scanResult;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;

    UNREFERENCED_PARAMETER(FltObjects);

    // 버퍼 스캔
    // Scan buffer
    status = SsnScanBuffer(
        &g_SsnDetector,
        Buffer,
        BufferLength,
        &scanResult
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // SSN 발견됨
    // SSN detected
    if (scanResult.ValidatedCount > 0) {
        // 파일명 가져오기
        // Get file name
        status = FltGetFileNameInformation(
            Data,
            FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
            &nameInfo
        );

        if (NT_SUCCESS(status)) {
            FltParseFileNameInformation(nameInfo);

            DbgPrint("[SSN] 주민등록번호 감지!\n");
            // SSN detected!
            DbgPrint("[SSN]   파일: %wZ\n", &nameInfo->Name);
            // File
            DbgPrint("[SSN]   발견 개수: %lu (마스킹: %lu)\n",
                     scanResult.ValidatedCount,
                     scanResult.MaskedCount);
            // Found count (Masked count)

            // 상세 정보 출력
            // Print detailed info
            for (ULONG i = 0; i < scanResult.TotalCandidates; i++) {
                PSSN_MATCH match = &scanResult.Matches[i];

                if (!match->IsFullyValid) {
                    continue;
                }

                DbgPrint("[SSN]   [%lu] 오프셋 0x%08X: %s (형식: %d)\n",
                         i + 1,
                         match->Offset,
                         match->IsMasked ? "(마스킹됨)" : "(유효)",
                         // (masked) : (valid)
                         match->Format);
            }

            FltReleaseFileNameInformation(nameInfo);
        }

        // 이벤트 로깅 또는 차단 로직 수행
        // Perform event logging or blocking logic
        // 여기서는 감지만 수행
        // Here only detection is performed
    }

    return STATUS_SUCCESS;
}

// Write 콜백에서 SSN 스캔
// Scan SSN in Write callback
FLT_PREOP_CALLBACK_STATUS PreWriteWithSsnScan(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID* CompletionContext)
{
    NTSTATUS status;
    PVOID buffer = NULL;
    ULONG bufferLength;
    PFLT_IO_PARAMETER_BLOCK iopb = Data->Iopb;

    *CompletionContext = NULL;

    // IRP_MJ_WRITE 확인
    // Verify IRP_MJ_WRITE
    if (iopb->MajorFunction != IRP_MJ_WRITE) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // IRQL 확인
    // Check IRQL
    if (KeGetCurrentIrql() > APC_LEVEL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 버퍼 길이 확인
    // Check buffer length
    bufferLength = iopb->Parameters.Write.Length;
    if (bufferLength == 0 || bufferLength > (1024 * 1024)) {
        // 0이거나 1MB 초과면 스킵
        // Skip if 0 or over 1MB
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 버퍼 접근
    // Access buffer
    if (iopb->Parameters.Write.MdlAddress != NULL) {
        buffer = MmGetSystemAddressForMdlSafe(
            iopb->Parameters.Write.MdlAddress,
            NormalPagePriority | MdlMappingNoExecute
        );
    } else {
        buffer = iopb->Parameters.Write.WriteBuffer;
    }

    if (buffer == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // SSN 스캔 수행
    // Perform SSN scan
    __try {
        status = ScanFileContent(
            Data,
            FltObjects,
            buffer,
            bufferLength
        );

        if (!NT_SUCCESS(status)) {
            DbgPrint("[SSN] 스캔 오류: 0x%X\n", status);
            // Scan error
        }
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        DbgPrint("[SSN] 예외 발생: 0x%X\n", GetExceptionCode());
        // Exception occurred
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### 22.4.4 성능 최적화 전략

```c
// 성능 최적화된 SSN 검색
// Performance-optimized SSN search

// 1. 빠른 필터링 (Quick Filtering)
// 숫자와 하이픈이 없는 영역은 빠르게 스킵
// Quickly skip regions without digits and hyphens
static ULONG QuickFilterScan(
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _Out_writes_(MaxRegions) PULONG RegionOffsets,
    _Out_writes_(MaxRegions) PULONG RegionLengths,
    _In_ ULONG MaxRegions)
{
    ULONG regionCount = 0;
    ULONG regionStart = 0;
    BOOLEAN inRegion = FALSE;

    // 16바이트 청크 단위로 빠른 스캔
    // Fast scan in 16-byte chunks
    for (ULONG i = 0; i < BufferLength; i++) {
        BOOLEAN hasCandidate = FALSE;

        // 숫자 또는 하이픈 체크
        // Check for digit or hyphen
        UCHAR c = Buffer[i];
        if ((c >= '0' && c <= '9') || c == '-') {
            hasCandidate = TRUE;
        }

        if (hasCandidate && !inRegion) {
            // 관심 영역 시작
            // Start region of interest
            regionStart = (i >= 6) ? (i - 6) : 0;
            inRegion = TRUE;
        }
        else if (!hasCandidate && inRegion) {
            // 관심 영역 종료
            // End region of interest
            if (regionCount < MaxRegions) {
                RegionOffsets[regionCount] = regionStart;
                RegionLengths[regionCount] = min(i - regionStart + 14,
                                                  BufferLength - regionStart);
                regionCount++;
            }
            inRegion = FALSE;
        }
    }

    // 마지막 영역 처리
    // Handle last region
    if (inRegion && regionCount < MaxRegions) {
        RegionOffsets[regionCount] = regionStart;
        RegionLengths[regionCount] = BufferLength - regionStart;
        regionCount++;
    }

    return regionCount;
}

// 2. SIMD 힌트를 활용한 검색 (컴파일러 최적화)
// Search using SIMD hints (compiler optimization)
#pragma optimize("gt", on)  // 전역 최적화 활성화
                            // Enable global optimization

static BOOLEAN FastDigitCheck(PUCHAR Buffer, ULONG Length)
{
    // 컴파일러가 SIMD 명령으로 최적화할 수 있도록
    // Allow compiler to optimize with SIMD instructions
    for (ULONG i = 0; i < Length; i++) {
        UCHAR c = Buffer[i];
        if (c < '0' || c > '9') {
            return FALSE;
        }
    }
    return TRUE;
}

#pragma optimize("", on)

// 3. 룩업 테이블 활용
// Utilize lookup tables
static const UCHAR g_DigitTable[256] = {
    // 0-47: 숫자 아님 (Non-digits)
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    // 48-57: '0'-'9'
    1,1,1,1,1,1,1,1,1,1,
    // 58-255: 숫자 아님 (Non-digits)
    0,0,0,0,0,0, // 58-63
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, // 64-79
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, // 80-95
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, // 96-111
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, // 112-127
    // 128-255: Non-ASCII
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0
};

__forceinline BOOLEAN IsDigitFast(UCHAR c)
{
    return g_DigitTable[c];
}

// 4. 캐시 친화적 스캔
// Cache-friendly scan
#define SCAN_BLOCK_SIZE 4096  // L1 캐시 친화적 크기
                              // L1 cache-friendly size

NTSTATUS CacheFriendlyScan(
    _In_ PSSN_DETECTOR Detector,
    _In_reads_bytes_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength,
    _Out_ PSSN_SCAN_RESULT Result)
{
    PUCHAR buffer = (PUCHAR)Buffer;
    ULONG offset = 0;

    RtlZeroMemory(Result, sizeof(SSN_SCAN_RESULT));

    while (offset < BufferLength) {
        ULONG blockSize = min(SCAN_BLOCK_SIZE, BufferLength - offset);

        // 블록 단위로 스캔
        // Scan block by block
        SSN_SCAN_RESULT blockResult;

        NTSTATUS status = SsnScanBuffer(
            Detector,
            buffer + offset,
            blockSize,
            &blockResult
        );

        if (NT_SUCCESS(status)) {
            // 결과 병합
            // Merge results
            for (ULONG i = 0; i < blockResult.TotalCandidates; i++) {
                if (Result->TotalCandidates < MAX_SSN_MATCHES) {
                    Result->Matches[Result->TotalCandidates] =
                        blockResult.Matches[i];
                    Result->Matches[Result->TotalCandidates].Offset += offset;
                    Result->TotalCandidates++;

                    if (blockResult.Matches[i].IsFullyValid) {
                        Result->ValidatedCount++;
                    }
                    if (blockResult.Matches[i].IsMasked) {
                        Result->MaskedCount++;
                    }
                }
            }
        }

        // 경계 처리: 이전 블록 끝에서 14바이트 오버랩
        // Boundary handling: 14-byte overlap from previous block end
        if (offset > 0) {
            offset += blockSize - 14;
        } else {
            offset += blockSize;
        }
    }

    return STATUS_SUCCESS;
}
```

### 22.4.5 오탐 최소화 전략

```c
// 오탐(False Positive) 최소화
// Minimize False Positives

// 컨텍스트 분석 - SSN 주변 텍스트 확인
// Context analysis - check text around SSN
typedef enum _SSN_CONTEXT_TYPE {
    SSN_CONTEXT_UNKNOWN,        // 알 수 없음 / Unknown
    SSN_CONTEXT_LIKELY_SSN,     // SSN일 가능성 높음 / Likely SSN
    SSN_CONTEXT_PHONE_NUMBER,   // 전화번호로 의심 / Suspected phone number
    SSN_CONTEXT_ACCOUNT,        // 계좌번호로 의심 / Suspected account number
    SSN_CONTEXT_DATE,           // 날짜로 의심 / Suspected date
} SSN_CONTEXT_TYPE;

// 컨텍스트 키워드
// Context keywords
static const CHAR* g_SsnKeywords[] = {
    "주민", "등록", "번호", "resident", "registration", "ssn",
    "생년월일", "birth", "jumin", NULL
};

static const CHAR* g_PhoneKeywords[] = {
    "전화", "연락처", "phone", "tel", "mobile", "휴대폰", NULL
};

static const CHAR* g_AccountKeywords[] = {
    "계좌", "account", "bank", "은행", NULL
};

// 키워드 검색
// Keyword search
static BOOLEAN ContainsKeyword(
    _In_reads_bytes_(Length) PUCHAR Buffer,
    _In_ ULONG Length,
    _In_ const CHAR** Keywords)
{
    for (int i = 0; Keywords[i] != NULL; i++) {
        if (FindBytePattern(
            Buffer, Length,
            (PUCHAR)Keywords[i],
            (ULONG)strlen(Keywords[i]),
            NULL)) {
            return TRUE;
        }
    }
    return FALSE;
}

// 컨텍스트 분석
// Context analysis
SSN_CONTEXT_TYPE AnalyzeContext(
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _In_ ULONG MatchOffset)
{
    // 분석 범위: 매치 전후 64바이트
    // Analysis range: 64 bytes before and after match
    const ULONG CONTEXT_SIZE = 64;

    ULONG startOffset = (MatchOffset >= CONTEXT_SIZE) ?
                        (MatchOffset - CONTEXT_SIZE) : 0;
    ULONG endOffset = min(MatchOffset + 14 + CONTEXT_SIZE, BufferLength);
    ULONG contextLength = endOffset - startOffset;

    PUCHAR contextBuffer = Buffer + startOffset;

    // SSN 관련 키워드 체크
    // Check SSN-related keywords
    if (ContainsKeyword(contextBuffer, contextLength, g_SsnKeywords)) {
        return SSN_CONTEXT_LIKELY_SSN;
    }

    // 전화번호 키워드 체크
    // Check phone number keywords
    if (ContainsKeyword(contextBuffer, contextLength, g_PhoneKeywords)) {
        return SSN_CONTEXT_PHONE_NUMBER;
    }

    // 계좌번호 키워드 체크
    // Check account number keywords
    if (ContainsKeyword(contextBuffer, contextLength, g_AccountKeywords)) {
        return SSN_CONTEXT_ACCOUNT;
    }

    return SSN_CONTEXT_UNKNOWN;
}

// 향상된 검증 (컨텍스트 포함)
// Enhanced validation (with context)
BOOLEAN ValidateWithContext(
    _In_ PSSN_MATCH Match,
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength)
{
    // 기본 검증 통과 필수
    // Basic validation required
    if (!Match->IsFullyValid) {
        return FALSE;
    }

    // 컨텍스트 분석
    // Context analysis
    SSN_CONTEXT_TYPE contextType = AnalyzeContext(
        Buffer,
        BufferLength,
        Match->Offset
    );

    switch (contextType) {
        case SSN_CONTEXT_LIKELY_SSN:
            // SSN일 가능성 높음 - 통과
            // Likely SSN - pass
            return TRUE;

        case SSN_CONTEXT_PHONE_NUMBER:
        case SSN_CONTEXT_ACCOUNT:
            // 다른 종류의 번호로 의심 - 거부
            // Suspected other type of number - reject
            DbgPrint("[SSN] 컨텍스트 분석: %d, 오탐 가능성\n", contextType);
            // Context analysis: possible false positive
            return FALSE;

        case SSN_CONTEXT_UNKNOWN:
        default:
            // 알 수 없음 - 추가 검증 필요
            // Unknown - needs additional validation
            // 여기서는 검증자리 확인 결과에 따라 결정
            // Decide based on check digit validation
            return Match->Validation.CheckDigitValid;
    }
}
```

---

## 22.5 테스트 및 검증

### 22.5.1 단위 테스트

```c
// ssn_test.c - SSN 감지 엔진 테스트
// ssn_test.c - SSN Detection Engine Tests

#include "ssn_detector.h"

// 테스트 케이스 구조체
// Test case structure
typedef struct _SSN_TEST_CASE {
    const char* Description;    // 테스트 설명
                                // Test description
    const char* Input;          // 입력 데이터
                                // Input data
    ULONG ExpectedCount;        // 예상 감지 개수
                                // Expected detection count
    BOOLEAN ExpectedValid;      // 예상 유효성
                                // Expected validity
} SSN_TEST_CASE, *PSSN_TEST_CASE;

// 테스트 케이스 배열
// Test case array
static SSN_TEST_CASE g_TestCases[] = {
    // 유효한 SSN (체크섬 계산된 예시)
    // Valid SSN (example with calculated checksum)
    {
        "유효한 주민등록번호 (하이픈 있음)",
        // "Valid SSN (with hyphen)"
        "주민등록번호: 880101-1234561",
        1,
        TRUE
    },
    {
        "유효한 주민등록번호 (하이픈 없음)",
        // "Valid SSN (without hyphen)"
        "번호는 8801011234561입니다",
        1,
        TRUE
    },
    {
        "마스킹된 주민등록번호",
        // "Masked SSN"
        "주민번호: 880101-1******",
        1,
        TRUE  // 마스킹은 부분 검증
              // Masked is partially validated
    },
    {
        "여러 개의 SSN",
        // "Multiple SSNs"
        "A: 880101-1234561, B: 900202-2345678",
        2,
        TRUE
    },
    {
        "무효한 월 (13월)",
        // "Invalid month (13)"
        "881301-1234567",
        0,  // 날짜 검증 실패로 감지 안됨
            // Not detected due to date validation failure
        FALSE
    },
    {
        "무효한 일 (32일)",
        // "Invalid day (32)"
        "880132-1234567",
        0,
        FALSE
    },
    {
        "전화번호 (오탐 테스트)",
        // "Phone number (false positive test)"
        "전화번호: 010-1234-5678",
        0,
        FALSE
    },
    {
        "빈 문자열",
        // "Empty string"
        "",
        0,
        FALSE
    },
    {NULL, NULL, 0, FALSE}  // 종료 마커 / End marker
};

// 테스트 실행
// Run tests
NTSTATUS RunSsnTests(VOID)
{
    SSN_DETECTOR detector;
    NTSTATUS status;
    ULONG passCount = 0;
    ULONG failCount = 0;

    DbgPrint("[SSN Test] 테스트 시작\n");
    // Test starting

    status = SsnDetectorInitialize(&detector);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[SSN Test] 초기화 실패: 0x%X\n", status);
        // Initialization failed
        return status;
    }

    for (int i = 0; g_TestCases[i].Description != NULL; i++) {
        PSSN_TEST_CASE test = &g_TestCases[i];
        SSN_SCAN_RESULT result;

        DbgPrint("[SSN Test] %d: %s\n", i + 1, test->Description);

        status = SsnScanBuffer(
            &detector,
            (PVOID)test->Input,
            (ULONG)strlen(test->Input),
            &result
        );

        BOOLEAN passed = TRUE;

        if (!NT_SUCCESS(status)) {
            DbgPrint("  스캔 실패: 0x%X\n", status);
            // Scan failed
            passed = FALSE;
        }
        else if (result.ValidatedCount != test->ExpectedCount) {
            DbgPrint("  개수 불일치: 예상 %lu, 실제 %lu\n",
                     test->ExpectedCount, result.ValidatedCount);
            // Count mismatch: expected, actual
            passed = FALSE;
        }

        if (passed) {
            DbgPrint("  통과\n");
            // Passed
            passCount++;
        } else {
            DbgPrint("  실패\n");
            // Failed
            failCount++;
        }
    }

    SsnDetectorCleanup(&detector);

    DbgPrint("[SSN Test] 결과: %lu 통과, %lu 실패\n", passCount, failCount);
    // Result: passed, failed

    return (failCount == 0) ? STATUS_SUCCESS : STATUS_UNSUCCESSFUL;
}
```

### 22.5.2 성능 벤치마크

```c
// 성능 측정
// Performance measurement
VOID BenchmarkSsnDetection(VOID)
{
    SSN_DETECTOR detector;
    SSN_SCAN_RESULT result;
    LARGE_INTEGER startTime, endTime, frequency;

    SsnDetectorInitialize(&detector);

    // 테스트 데이터 생성 (1MB)
    // Generate test data (1MB)
    const ULONG TEST_SIZE = 1024 * 1024;
    PUCHAR testBuffer = ExAllocatePool2(
        POOL_FLAG_PAGED,
        TEST_SIZE,
        'tseT'
    );

    if (testBuffer == NULL) {
        return;
    }

    // 랜덤 데이터 + 주기적 SSN 삽입
    // Random data + periodic SSN insertion
    for (ULONG i = 0; i < TEST_SIZE; i++) {
        testBuffer[i] = (UCHAR)('A' + (i % 26));
    }

    // 1000바이트마다 SSN 삽입
    // Insert SSN every 1000 bytes
    const char* sampleSsn = "880101-1234561";
    for (ULONG i = 0; i + 14 < TEST_SIZE; i += 1000) {
        RtlCopyMemory(&testBuffer[i], sampleSsn, 14);
    }

    // 타이밍 측정
    // Timing measurement
    KeQueryPerformanceCounter(&frequency);
    KeQueryPerformanceCounter(&startTime);

    SsnScanBuffer(&detector, testBuffer, TEST_SIZE, &result);

    KeQueryPerformanceCounter(&endTime);

    LONGLONG elapsedTicks = endTime.QuadPart - startTime.QuadPart;
    LONGLONG elapsedMs = (elapsedTicks * 1000) / frequency.QuadPart;

    DbgPrint("[SSN Benchmark] 1MB 스캔 완료\n");
    // 1MB scan complete
    DbgPrint("[SSN Benchmark]   소요 시간: %lld ms\n", elapsedMs);
    // Elapsed time
    DbgPrint("[SSN Benchmark]   발견 개수: %lu\n", result.ValidatedCount);
    // Found count
    DbgPrint("[SSN Benchmark]   처리 속도: %.2f MB/s\n",
             (double)TEST_SIZE / (elapsedMs + 1) * 1000 / (1024 * 1024));
    // Processing speed

    ExFreePoolWithTag(testBuffer, 'tseT');
    SsnDetectorCleanup(&detector);
}
```

---

## 22.6 요약

### 핵심 내용 정리

| 항목 | 내용 |
|------|------|
| **UNICODE_STRING** | 커널 기본 문자열 타입, NULL 종료 보장 없음 |
| **RtlStringCb*** | 버퍼 오버플로우 방지 안전 문자열 함수 |
| **패턴 매칭** | Naive, Boyer-Moore, FSM 기반 알고리즘 |
| **SSN 검증** | 날짜 유효성 + 검증자리 계산으로 오탐 최소화 |
| **성능 최적화** | 룩업 테이블, 캐시 친화적 스캔, 빠른 필터링 |

### .NET 개발자 참고 사항

```
┌─────────────────────────────────────────────────────────────────┐
│                    C# vs C 패턴 매칭 비교                        │
├─────────────────────────────────────────────────────────────────┤
│ C# (사용자 모드)              │ C (커널 모드)                    │
├───────────────────────────────┼─────────────────────────────────┤
│ Regex.IsMatch(text, pattern)  │ 직접 구현한 FSM 또는 알고리즘    │
│ string.Contains(substring)    │ FindBytePattern() 구현          │
│ GC가 메모리 관리              │ ExAllocatePool2/ExFreePool      │
│ Exception 처리                │ NTSTATUS 반환 + __try/__except  │
│ LINQ 쿼리                     │ 수동 루프 및 조건문              │
└───────────────────────────────┴─────────────────────────────────┘
```

### 다음 챕터 예고

Chapter 23에서는 **Office 문서 파싱**을 다룹니다. Office Open XML 형식의 구조를 이해하고, 커널에서 ZIP 컨테이너를 처리하는 전략과 텍스트 추출 구현을 학습합니다.

---

## 참고 자료

- [Microsoft Docs: UNICODE_STRING structure](https://docs.microsoft.com/en-us/windows/win32/api/ntdef/ns-ntdef-_unicode_string)
- [Microsoft Docs: Safe String Functions](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntstrsafe/)
- [String Matching Algorithms](https://en.wikipedia.org/wiki/String-searching_algorithm)
- [Boyer-Moore Algorithm](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm)
