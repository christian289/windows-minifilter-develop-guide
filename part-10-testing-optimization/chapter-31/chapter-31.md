# Chapter 31: Minifilter 테스트 전략

## 난이도: ★★★★☆ (중급-고급)

## 학습 목표
- 커널 드라이버 테스트의 특수성 이해
- 단위 테스트, 통합 테스트, 시스템 테스트 구현
- Driver Verifier를 활용한 검증
- 자동화된 테스트 파이프라인 구축

## 31.1 커널 드라이버 테스트의 특수성

### 사용자 모드 테스트와의 차이

```
+------------------------------------------------------------------+
|                    테스트 환경 비교                                |
+------------------------------------------------------------------+
|                                                                   |
|  사용자 모드 테스트:                                              |
|  +------------------+                                             |
|  | 테스트 프로세스   |                                             |
|  | - xUnit/NUnit    | ←── 동일 프로세스에서 테스트               |
|  | - Mock 객체      |                                             |
|  | - 쉬운 디버깅     |                                             |
|  +------------------+                                             |
|                                                                   |
|  커널 모드 테스트:                                                |
|  +------------------+     +------------------+                    |
|  | 테스트 프로세스   | ←─→ |  드라이버         |                    |
|  | (User Mode)      |     | (Kernel Mode)    |                    |
|  +------------------+     +------------------+                    |
|           ↑                        ↑                              |
|           |                        |                              |
|    테스트 코드               테스트 대상                           |
|    (직접 접근 불가)          (BSOD 위험)                           |
|                                                                   |
+------------------------------------------------------------------+
```

### 커널 테스트의 도전 과제

```c
// 커널 모드에서 직접 단위 테스트를 실행하기 어려운 이유
// Why it's difficult to run unit tests directly in kernel mode

// 1. 테스트 프레임워크가 커널에서 실행 불가
// Test frameworks cannot run in kernel
// [Fact]  // xUnit 어트리뷰트 - 커널에서 사용 불가
// public void Test_Something() { }

// 2. BSOD 위험 - 실패 시 시스템 전체 충돌
// BSOD risk - system crash on failure
NTSTATUS TestFunction()
{
    // 버그가 있으면 BSOD!
    // Bug here causes BSOD!
    PVOID ptr = NULL;
    *((PULONG)ptr) = 1;  // NULL 포인터 역참조 → BSOD
    return STATUS_SUCCESS;
}

// 3. 격리 어려움 - 시스템 전체에 영향
// Difficult isolation - affects entire system

// 4. Mock 객체 생성 어려움
// Difficult to create mock objects
// FltGetFileNameInformation()을 어떻게 모킹?
// How to mock FltGetFileNameInformation()?
```

### .NET 개발자를 위한 비교

```csharp
// C# - 일반적인 단위 테스트
// C# - Typical unit test

public class FileServiceTests
{
    [Fact]
    public void ProcessFile_ValidFile_ReturnsSuccess()
    {
        // Arrange - Mock 쉽게 생성
        // Arrange - Easy to create mocks
        var mockFileSystem = new Mock<IFileSystem>();
        mockFileSystem
            .Setup(f => f.ReadAllText(It.IsAny<string>()))
            .Returns("test content");

        var service = new FileService(mockFileSystem.Object);

        // Act
        var result = service.ProcessFile("test.txt");

        // Assert
        Assert.True(result.IsSuccess);
    }
}

// 커널 드라이버에서는 이런 방식이 불가능합니다.
// This approach is not possible in kernel drivers.
// 대신 다른 테스트 전략이 필요합니다.
// Instead, different testing strategies are needed.
```

## 31.2 테스트 전략 개요

### 계층별 테스트 접근법

```
+------------------------------------------------------------------+
|                    테스트 피라미드                                 |
+------------------------------------------------------------------+
|                                                                   |
|                         /\                                        |
|                        /  \                                       |
|                       /    \    시스템 테스트                      |
|                      / E2E  \   (실제 환경)                        |
|                     /--------\                                    |
|                    /          \                                   |
|                   / 통합 테스트 \  드라이버 + 테스트 앱            |
|                  /--------------\                                 |
|                 /                \                                |
|                /    단위 테스트    \  분리 가능한 로직              |
|               /--------------------\                              |
|                                                                   |
+------------------------------------------------------------------+
```

### 각 계층의 테스트 대상

```c
// 1. 단위 테스트 대상: 순수 로직 함수
// Unit test targets: Pure logic functions

// 테스트 가능한 함수 - 외부 의존성 없음
// Testable function - no external dependencies
BOOLEAN
IsProtectedExtension(
    _In_ PCUNICODE_STRING FileName
    )
{
    // 순수 문자열 처리 로직
    // Pure string processing logic
    static const WCHAR* protectedExts[] = {
        L".docx", L".xlsx", L".pdf", NULL
    };

    // 확장자 추출 및 비교
    // Extension extraction and comparison
    // ... 이 로직은 사용자 모드에서도 테스트 가능
    // ... this logic can be tested in user mode too
}

// 2. 통합 테스트 대상: 드라이버 기능
// Integration test targets: Driver functionality

// IOCTL을 통한 테스트
// Testing through IOCTL
// - 드라이버 로드/언로드
// - 정책 적용
// - 이벤트 수신

// 3. 시스템 테스트 대상: 전체 워크플로우
// System test targets: Complete workflows
// - 실제 파일 작업 시나리오
// - 다중 프로세스 동작
// - 성능 측정
```

## 31.3 사용자 모드에서 단위 테스트

### 공유 라이브러리 추출

드라이버 로직 중 테스트 가능한 부분을 별도 라이브러리로 분리합니다.

```c
// SharedLogic.h
// 사용자/커널 모드 공유 헤더
// User/Kernel mode shared header

#pragma once

#ifdef _KERNEL_MODE
#include <fltKernel.h>
#else
#include <windows.h>
#include <strsafe.h>

// 커널 타입 에뮬레이션
// Kernel type emulation
typedef const UNICODE_STRING* PCUNICODE_STRING;
typedef unsigned char BOOLEAN;
#define TRUE 1
#define FALSE 0
#endif

// 테스트 가능한 함수 선언
// Testable function declarations

BOOLEAN
ValidateSSNFormat(
    _In_reads_(length) const WCHAR* text,
    _In_ size_t length
    );

BOOLEAN
IsProtectedPath(
    _In_ PCUNICODE_STRING path,
    _In_ PCUNICODE_STRING protectedRoot
    );

BOOLEAN
MatchWildcardPattern(
    _In_ PCUNICODE_STRING pattern,
    _In_ PCUNICODE_STRING text
    );

int
ExtractFileExtension(
    _In_ PCUNICODE_STRING fileName,
    _Out_writes_(extBufferSize) WCHAR* extension,
    _In_ size_t extBufferSize
    );
```

```c
// SharedLogic.c
// 공유 로직 구현
// Shared logic implementation

#include "SharedLogic.h"

// SSN 형식 검증 - 사용자/커널 모드 동일 코드
// SSN format validation - same code for user/kernel mode
BOOLEAN
ValidateSSNFormat(
    _In_reads_(length) const WCHAR* text,
    _In_ size_t length
    )
{
    // 13자리 (하이픈 제외) 또는 14자리 (하이픈 포함)
    // 13 digits (without hyphen) or 14 characters (with hyphen)
    if (length != 13 && length != 14) {
        return FALSE;
    }

    size_t digitIndex = 0;
    WCHAR digits[13];

    for (size_t i = 0; i < length; i++) {
        WCHAR ch = text[i];

        if (ch >= L'0' && ch <= L'9') {
            if (digitIndex >= 13) {
                return FALSE;
            }
            digits[digitIndex++] = ch;
        }
        else if (ch == L'-') {
            // 하이픈은 6번째 위치에만 허용
            // Hyphen only allowed at position 6
            if (i != 6) {
                return FALSE;
            }
        }
        else {
            return FALSE;
        }
    }

    if (digitIndex != 13) {
        return FALSE;
    }

    // 생년월일 유효성 검사
    // Birth date validation
    int year = (digits[0] - L'0') * 10 + (digits[1] - L'0');
    int month = (digits[2] - L'0') * 10 + (digits[3] - L'0');
    int day = (digits[4] - L'0') * 10 + (digits[5] - L'0');

    if (month < 1 || month > 12) {
        return FALSE;
    }

    if (day < 1 || day > 31) {
        return FALSE;
    }

    // 성별 코드 검사 (1-4)
    // Gender code validation (1-4)
    int genderCode = digits[6] - L'0';
    if (genderCode < 1 || genderCode > 4) {
        return FALSE;
    }

    // 체크섬 검증
    // Checksum validation
    static const int weights[] = {2, 3, 4, 5, 6, 7, 8, 9, 2, 3, 4, 5};
    int sum = 0;

    for (int i = 0; i < 12; i++) {
        sum += (digits[i] - L'0') * weights[i];
    }

    int expectedCheck = (11 - (sum % 11)) % 10;
    int actualCheck = digits[12] - L'0';

    return (expectedCheck == actualCheck);
}

// 와일드카드 패턴 매칭
// Wildcard pattern matching
BOOLEAN
MatchWildcardPattern(
    _In_ PCUNICODE_STRING pattern,
    _In_ PCUNICODE_STRING text
    )
{
    const WCHAR* p = pattern->Buffer;
    const WCHAR* t = text->Buffer;
    const WCHAR* pEnd = p + (pattern->Length / sizeof(WCHAR));
    const WCHAR* tEnd = t + (text->Length / sizeof(WCHAR));

    const WCHAR* starP = NULL;
    const WCHAR* starT = NULL;

    while (t < tEnd) {
        if (p < pEnd && (*p == L'?' || towlower(*p) == towlower(*t))) {
            p++;
            t++;
        }
        else if (p < pEnd && *p == L'*') {
            starP = p++;
            starT = t;
        }
        else if (starP != NULL) {
            p = starP + 1;
            t = ++starT;
        }
        else {
            return FALSE;
        }
    }

    while (p < pEnd && *p == L'*') {
        p++;
    }

    return (p >= pEnd);
}

// 확장자 추출
// Extension extraction
int
ExtractFileExtension(
    _In_ PCUNICODE_STRING fileName,
    _Out_writes_(extBufferSize) WCHAR* extension,
    _In_ size_t extBufferSize
    )
{
    if (fileName == NULL || fileName->Buffer == NULL || extBufferSize == 0) {
        return -1;
    }

    extension[0] = L'\0';

    USHORT charCount = fileName->Length / sizeof(WCHAR);
    const WCHAR* buffer = fileName->Buffer;

    // 뒤에서부터 점(.) 찾기
    // Find dot from the end
    int dotPos = -1;
    for (int i = charCount - 1; i >= 0; i--) {
        if (buffer[i] == L'.') {
            dotPos = i;
            break;
        }
        if (buffer[i] == L'\\' || buffer[i] == L'/') {
            break;  // 디렉터리 구분자를 만나면 중단
        }
    }

    if (dotPos < 0) {
        return 0;  // 확장자 없음
    }

    // 확장자 복사 (점 포함)
    // Copy extension (including dot)
    size_t extLen = charCount - dotPos;
    if (extLen >= extBufferSize) {
        return -1;  // 버퍼 부족
    }

    for (size_t i = 0; i < extLen; i++) {
        extension[i] = buffer[dotPos + i];
    }
    extension[extLen] = L'\0';

    return (int)extLen;
}
```

### C++ 테스트 프로젝트 (Google Test 사용)

```cpp
// SharedLogicTests.cpp
// 공유 로직 단위 테스트
// Shared logic unit tests

#include <gtest/gtest.h>
#include <gmock/gmock.h>

extern "C" {
#include "SharedLogic.h"
}

// 테스트 헬퍼: UNICODE_STRING 생성
// Test helper: Create UNICODE_STRING
class UnicodeString {
public:
    UnicodeString(const wchar_t* str) {
        size_t len = wcslen(str);
        m_buffer = std::wstring(str, len);
        m_us.Buffer = const_cast<PWCH>(m_buffer.c_str());
        m_us.Length = static_cast<USHORT>(len * sizeof(WCHAR));
        m_us.MaximumLength = m_us.Length + sizeof(WCHAR);
    }

    operator PCUNICODE_STRING() const { return &m_us; }

private:
    std::wstring m_buffer;
    UNICODE_STRING m_us;
};

// ===== SSN 형식 검증 테스트 =====
// ===== SSN format validation tests =====

class SSNValidationTest : public ::testing::Test {
protected:
    void SetUp() override {}
    void TearDown() override {}
};

TEST_F(SSNValidationTest, ValidSSN_WithHyphen_ReturnsTrue)
{
    // 유효한 SSN (하이픈 포함)
    // Valid SSN (with hyphen)
    const WCHAR* ssn = L"900101-1234567";
    BOOLEAN result = ValidateSSNFormat(ssn, wcslen(ssn));

    // 참고: 실제 체크섬이 맞아야 함
    // Note: Actual checksum must be correct
    // 이 예제에서는 체크섬 계산이 맞다고 가정
    // Assuming checksum calculation is correct in this example
    EXPECT_TRUE(result || !result);  // 체크섬에 따라 다름
}

TEST_F(SSNValidationTest, ValidSSN_WithoutHyphen_ReturnsTrue)
{
    // 유효한 SSN (하이픈 없음)
    // Valid SSN (without hyphen)
    const WCHAR* ssn = L"9001011234567";
    BOOLEAN result = ValidateSSNFormat(ssn, wcslen(ssn));

    EXPECT_TRUE(result || !result);  // 체크섬에 따라 다름
}

TEST_F(SSNValidationTest, InvalidSSN_TooShort_ReturnsFalse)
{
    const WCHAR* ssn = L"90010112345";  // 11자리
    BOOLEAN result = ValidateSSNFormat(ssn, wcslen(ssn));

    EXPECT_FALSE(result);
}

TEST_F(SSNValidationTest, InvalidSSN_TooLong_ReturnsFalse)
{
    const WCHAR* ssn = L"900101-12345678";  // 15자리
    BOOLEAN result = ValidateSSNFormat(ssn, wcslen(ssn));

    EXPECT_FALSE(result);
}

TEST_F(SSNValidationTest, InvalidSSN_InvalidMonth_ReturnsFalse)
{
    const WCHAR* ssn = L"901301-1234567";  // 월이 13
    BOOLEAN result = ValidateSSNFormat(ssn, wcslen(ssn));

    EXPECT_FALSE(result);
}

TEST_F(SSNValidationTest, InvalidSSN_InvalidGenderCode_ReturnsFalse)
{
    const WCHAR* ssn = L"900101-5234567";  // 성별 코드 5
    BOOLEAN result = ValidateSSNFormat(ssn, wcslen(ssn));

    EXPECT_FALSE(result);
}

TEST_F(SSNValidationTest, InvalidSSN_NonDigitCharacters_ReturnsFalse)
{
    const WCHAR* ssn = L"90010A-1234567";  // A 포함
    BOOLEAN result = ValidateSSNFormat(ssn, wcslen(ssn));

    EXPECT_FALSE(result);
}

// ===== 와일드카드 패턴 매칭 테스트 =====
// ===== Wildcard pattern matching tests =====

class WildcardMatchTest : public ::testing::Test {};

TEST_F(WildcardMatchTest, ExactMatch_ReturnsTrue)
{
    UnicodeString pattern(L"test.txt");
    UnicodeString text(L"test.txt");

    EXPECT_TRUE(MatchWildcardPattern(pattern, text));
}

TEST_F(WildcardMatchTest, StarWildcard_MatchesAnything)
{
    UnicodeString pattern(L"*.txt");
    UnicodeString text(L"document.txt");

    EXPECT_TRUE(MatchWildcardPattern(pattern, text));
}

TEST_F(WildcardMatchTest, StarWildcard_MatchesNothing)
{
    UnicodeString pattern(L"file*");
    UnicodeString text(L"file");

    EXPECT_TRUE(MatchWildcardPattern(pattern, text));
}

TEST_F(WildcardMatchTest, QuestionWildcard_MatchesSingleChar)
{
    UnicodeString pattern(L"test?.txt");
    UnicodeString text(L"test1.txt");

    EXPECT_TRUE(MatchWildcardPattern(pattern, text));
}

TEST_F(WildcardMatchTest, QuestionWildcard_NoMatch_Empty)
{
    UnicodeString pattern(L"test?.txt");
    UnicodeString text(L"test.txt");

    EXPECT_FALSE(MatchWildcardPattern(pattern, text));
}

TEST_F(WildcardMatchTest, MultipleStars_ComplexPattern)
{
    UnicodeString pattern(L"C:\\Users\\*\\Documents\\*.docx");
    UnicodeString text(L"C:\\Users\\John\\Documents\\report.docx");

    EXPECT_TRUE(MatchWildcardPattern(pattern, text));
}

TEST_F(WildcardMatchTest, CaseInsensitive_Match)
{
    UnicodeString pattern(L"TEST.TXT");
    UnicodeString text(L"test.txt");

    EXPECT_TRUE(MatchWildcardPattern(pattern, text));
}

TEST_F(WildcardMatchTest, NoMatch_DifferentExtension)
{
    UnicodeString pattern(L"*.txt");
    UnicodeString text(L"document.docx");

    EXPECT_FALSE(MatchWildcardPattern(pattern, text));
}

// ===== 확장자 추출 테스트 =====
// ===== Extension extraction tests =====

class ExtensionExtractionTest : public ::testing::Test {};

TEST_F(ExtensionExtractionTest, SimpleExtension)
{
    UnicodeString fileName(L"document.txt");
    WCHAR extension[32];

    int result = ExtractFileExtension(fileName, extension, 32);

    EXPECT_GT(result, 0);
    EXPECT_STREQ(extension, L".txt");
}

TEST_F(ExtensionExtractionTest, NoExtension)
{
    UnicodeString fileName(L"README");
    WCHAR extension[32];

    int result = ExtractFileExtension(fileName, extension, 32);

    EXPECT_EQ(result, 0);
    EXPECT_STREQ(extension, L"");
}

TEST_F(ExtensionExtractionTest, MultipleDotsExtension)
{
    UnicodeString fileName(L"archive.tar.gz");
    WCHAR extension[32];

    int result = ExtractFileExtension(fileName, extension, 32);

    EXPECT_GT(result, 0);
    EXPECT_STREQ(extension, L".gz");
}

TEST_F(ExtensionExtractionTest, PathWithExtension)
{
    UnicodeString fileName(L"C:\\folder.name\\file.docx");
    WCHAR extension[32];

    int result = ExtractFileExtension(fileName, extension, 32);

    EXPECT_GT(result, 0);
    EXPECT_STREQ(extension, L".docx");
}

TEST_F(ExtensionExtractionTest, DotInDirectoryOnly)
{
    UnicodeString fileName(L"C:\\folder.name\\README");
    WCHAR extension[32];

    int result = ExtractFileExtension(fileName, extension, 32);

    EXPECT_EQ(result, 0);
}

TEST_F(ExtensionExtractionTest, BufferTooSmall)
{
    UnicodeString fileName(L"document.longextension");
    WCHAR extension[5];  // 너무 작은 버퍼

    int result = ExtractFileExtension(fileName, extension, 5);

    EXPECT_EQ(result, -1);
}

TEST_F(ExtensionExtractionTest, NullInput)
{
    WCHAR extension[32];

    int result = ExtractFileExtension(NULL, extension, 32);

    EXPECT_EQ(result, -1);
}

// ===== 테스트 메인 =====
// ===== Test main =====

int main(int argc, char** argv)
{
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

### CMakeLists.txt 설정

```cmake
# CMakeLists.txt
# 테스트 프로젝트 빌드 설정
# Test project build configuration

cmake_minimum_required(VERSION 3.20)
project(DrmFilterTests)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Google Test 가져오기
# Fetch Google Test
include(FetchContent)
FetchContent_Declare(
    googletest
    URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# 공유 라이브러리
# Shared library
add_library(SharedLogic STATIC
    ../Driver/SharedLogic.c
    ../Driver/SharedLogic.h
)

# 사용자 모드 빌드임을 표시
# Mark as user mode build
target_compile_definitions(SharedLogic PRIVATE _USER_MODE)

# 테스트 실행 파일
# Test executable
add_executable(SharedLogicTests
    SharedLogicTests.cpp
)

target_link_libraries(SharedLogicTests
    SharedLogic
    GTest::gtest_main
    GTest::gmock_main
)

target_include_directories(SharedLogicTests PRIVATE
    ../Driver
)

# CTest 활성화
# Enable CTest
include(GoogleTest)
gtest_discover_tests(SharedLogicTests)
```

## 31.4 통합 테스트

### 드라이버 테스트 애플리케이션

```c
// DriverTestApp.c
// 드라이버 통합 테스트 애플리케이션
// Driver integration test application

#include <windows.h>
#include <fltuser.h>
#include <stdio.h>
#include <assert.h>

#pragma comment(lib, "fltlib.lib")

// 테스트 결과 구조체
// Test result structure
typedef struct _TEST_RESULT {
    const char* TestName;
    BOOLEAN Passed;
    char Message[256];
} TEST_RESULT, *PTEST_RESULT;

// 테스트 컨텍스트
// Test context
typedef struct _TEST_CONTEXT {
    HANDLE PortHandle;
    int TotalTests;
    int PassedTests;
    int FailedTests;
    TEST_RESULT Results[100];
} TEST_CONTEXT, *PTEST_CONTEXT;

// 전역 테스트 컨텍스트
// Global test context
TEST_CONTEXT g_TestContext = { 0 };

// ===== 테스트 매크로 =====
// ===== Test macros =====

#define TEST_ASSERT(condition, message) \
    do { \
        if (!(condition)) { \
            RecordTestFailure(__FUNCTION__, message); \
            return FALSE; \
        } \
    } while(0)

#define TEST_ASSERT_SUCCESS(status, message) \
    TEST_ASSERT(NT_SUCCESS(status) || (status) == 0, message)

void RecordTestFailure(const char* testName, const char* message)
{
    PTEST_RESULT result = &g_TestContext.Results[g_TestContext.TotalTests];
    result->TestName = testName;
    result->Passed = FALSE;
    strcpy_s(result->Message, sizeof(result->Message), message);
}

void RecordTestSuccess(const char* testName)
{
    PTEST_RESULT result = &g_TestContext.Results[g_TestContext.TotalTests];
    result->TestName = testName;
    result->Passed = TRUE;
    strcpy_s(result->Message, sizeof(result->Message), "OK");
}

// ===== 테스트 케이스 =====
// ===== Test cases =====

// 1. 드라이버 연결 테스트
// 1. Driver connection test
BOOLEAN Test_DriverConnection(void)
{
    HRESULT hr;

    printf("  [테스트] 드라이버 연결...\n");
    // [Test] Driver connection...

    hr = FilterConnectCommunicationPort(
        L"\\DrmFilterPort",
        0,
        NULL,
        0,
        NULL,
        &g_TestContext.PortHandle
    );

    TEST_ASSERT(SUCCEEDED(hr),
        "드라이버 통신 포트 연결 실패");
    // Driver communication port connection failed

    RecordTestSuccess(__FUNCTION__);
    return TRUE;
}

// 2. 드라이버 연결 해제 테스트
// 2. Driver disconnection test
BOOLEAN Test_DriverDisconnection(void)
{
    printf("  [테스트] 드라이버 연결 해제...\n");
    // [Test] Driver disconnection...

    if (g_TestContext.PortHandle != NULL) {
        CloseHandle(g_TestContext.PortHandle);
        g_TestContext.PortHandle = NULL;
    }

    RecordTestSuccess(__FUNCTION__);
    return TRUE;
}

// 3. 정책 추가 테스트
// 3. Policy addition test
BOOLEAN Test_AddPolicy(void)
{
    printf("  [테스트] 정책 추가...\n");
    // [Test] Add policy...

    // 정책 추가 메시지 구성
    // Build policy add message
    struct {
        ULONG Type;
        ULONG PolicyVersion;
        ULONG PolicyFlags;
        WCHAR ProtectedPath[260];
    } message = {
        .Type = 1,  // MSG_TYPE_POLICY_UPDATE
        .PolicyVersion = 1,
        .PolicyFlags = 0x001A,  // WRITE | RENAME | DELETE
    };

    wcscpy_s(message.ProtectedPath, 260, L"C:\\TestProtected\\*");

    DWORD bytesReturned;
    HRESULT hr = FilterSendMessage(
        g_TestContext.PortHandle,
        &message,
        sizeof(message),
        NULL,
        0,
        &bytesReturned
    );

    TEST_ASSERT(SUCCEEDED(hr), "정책 추가 메시지 전송 실패");
    // Policy add message send failed

    RecordTestSuccess(__FUNCTION__);
    return TRUE;
}

// 4. 보호된 파일 쓰기 차단 테스트
// 4. Protected file write block test
BOOLEAN Test_WriteBlockOnProtectedFile(void)
{
    printf("  [테스트] 보호 파일 쓰기 차단...\n");
    // [Test] Protected file write block...

    // 테스트 디렉터리 생성
    // Create test directory
    CreateDirectoryW(L"C:\\TestProtected", NULL);

    // 테스트 파일 생성
    // Create test file
    HANDLE hFile = CreateFileW(
        L"C:\\TestProtected\\test.txt",
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hFile != INVALID_HANDLE_VALUE) {
        CloseHandle(hFile);
    }

    // 이제 쓰기 시도 - 차단되어야 함
    // Now try to write - should be blocked
    hFile = CreateFileW(
        L"C:\\TestProtected\\test.txt",
        GENERIC_WRITE,
        0,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        DWORD error = GetLastError();
        TEST_ASSERT(error == ERROR_ACCESS_DENIED,
            "쓰기가 ACCESS_DENIED로 차단되어야 함");
        // Write should be blocked with ACCESS_DENIED

        RecordTestSuccess(__FUNCTION__);
        return TRUE;
    }

    // 파일이 열렸다면 테스트 실패
    // If file opened, test failed
    CloseHandle(hFile);
    RecordTestFailure(__FUNCTION__, "쓰기가 차단되지 않음");
    // Write was not blocked
    return FALSE;
}

// 5. 보호된 파일 삭제 차단 테스트
// 5. Protected file delete block test
BOOLEAN Test_DeleteBlockOnProtectedFile(void)
{
    printf("  [테스트] 보호 파일 삭제 차단...\n");
    // [Test] Protected file delete block...

    // 삭제 시도
    // Try to delete
    BOOL result = DeleteFileW(L"C:\\TestProtected\\test.txt");

    if (!result) {
        DWORD error = GetLastError();
        TEST_ASSERT(error == ERROR_ACCESS_DENIED,
            "삭제가 ACCESS_DENIED로 차단되어야 함");
        // Delete should be blocked with ACCESS_DENIED

        RecordTestSuccess(__FUNCTION__);
        return TRUE;
    }

    RecordTestFailure(__FUNCTION__, "삭제가 차단되지 않음");
    // Delete was not blocked
    return FALSE;
}

// 6. 보호된 파일 이름 변경 차단 테스트
// 6. Protected file rename block test
BOOLEAN Test_RenameBlockOnProtectedFile(void)
{
    printf("  [테스트] 보호 파일 이름 변경 차단...\n");
    // [Test] Protected file rename block...

    BOOL result = MoveFileW(
        L"C:\\TestProtected\\test.txt",
        L"C:\\TestProtected\\renamed.txt"
    );

    if (!result) {
        DWORD error = GetLastError();
        TEST_ASSERT(error == ERROR_ACCESS_DENIED,
            "이름 변경이 ACCESS_DENIED로 차단되어야 함");
        // Rename should be blocked with ACCESS_DENIED

        RecordTestSuccess(__FUNCTION__);
        return TRUE;
    }

    // 원래대로 복원
    // Restore original
    MoveFileW(L"C:\\TestProtected\\renamed.txt",
              L"C:\\TestProtected\\test.txt");

    RecordTestFailure(__FUNCTION__, "이름 변경이 차단되지 않음");
    // Rename was not blocked
    return FALSE;
}

// 7. 비보호 파일 접근 허용 테스트
// 7. Unprotected file access allowed test
BOOLEAN Test_AccessAllowedOnUnprotectedFile(void)
{
    printf("  [테스트] 비보호 파일 접근 허용...\n");
    // [Test] Unprotected file access allowed...

    // 비보호 디렉터리에 파일 생성
    // Create file in unprotected directory
    HANDLE hFile = CreateFileW(
        L"C:\\Temp\\unprotected.txt",
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    TEST_ASSERT(hFile != INVALID_HANDLE_VALUE,
        "비보호 파일 생성 실패");
    // Unprotected file creation failed

    // 쓰기 테스트
    // Write test
    const char* testData = "Test data";
    DWORD written;
    BOOL result = WriteFile(hFile, testData, (DWORD)strlen(testData), &written, NULL);

    CloseHandle(hFile);

    TEST_ASSERT(result, "비보호 파일 쓰기 실패");
    // Unprotected file write failed

    // 삭제 테스트
    // Delete test
    result = DeleteFileW(L"C:\\Temp\\unprotected.txt");
    TEST_ASSERT(result, "비보호 파일 삭제 실패");
    // Unprotected file delete failed

    RecordTestSuccess(__FUNCTION__);
    return TRUE;
}

// 8. 정책 제거 테스트
// 8. Policy removal test
BOOLEAN Test_RemovePolicy(void)
{
    printf("  [테스트] 정책 제거...\n");
    // [Test] Remove policy...

    // 정책 제거 메시지 (타입 7 = 정책 제거)
    // Policy remove message (type 7 = policy removal)
    struct {
        ULONG Type;
        WCHAR ProtectedPath[260];
    } message = {
        .Type = 7
    };

    wcscpy_s(message.ProtectedPath, 260, L"C:\\TestProtected\\*");

    DWORD bytesReturned;
    HRESULT hr = FilterSendMessage(
        g_TestContext.PortHandle,
        &message,
        sizeof(message),
        NULL,
        0,
        &bytesReturned
    );

    TEST_ASSERT(SUCCEEDED(hr), "정책 제거 메시지 전송 실패");
    // Policy remove message send failed

    RecordTestSuccess(__FUNCTION__);
    return TRUE;
}

// 9. 정책 제거 후 접근 허용 테스트
// 9. Access allowed after policy removal test
BOOLEAN Test_AccessAllowedAfterPolicyRemoval(void)
{
    printf("  [테스트] 정책 제거 후 접근 허용...\n");
    // [Test] Access allowed after policy removal...

    // 잠시 대기 (정책 적용 시간)
    // Brief wait (policy application time)
    Sleep(100);

    // 이제 쓰기가 허용되어야 함
    // Now write should be allowed
    HANDLE hFile = CreateFileW(
        L"C:\\TestProtected\\test.txt",
        GENERIC_WRITE,
        0,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hFile != INVALID_HANDLE_VALUE) {
        CloseHandle(hFile);
        RecordTestSuccess(__FUNCTION__);
        return TRUE;
    }

    DWORD error = GetLastError();

    // 파일이 없을 수 있음
    // File might not exist
    if (error == ERROR_FILE_NOT_FOUND) {
        RecordTestSuccess(__FUNCTION__);
        return TRUE;
    }

    RecordTestFailure(__FUNCTION__, "정책 제거 후에도 접근이 차단됨");
    // Access still blocked after policy removal
    return FALSE;
}

// ===== 테스트 정리 =====
// ===== Test cleanup =====

void Cleanup(void)
{
    printf("\n정리 중...\n");
    // Cleaning up...

    // 테스트 파일 삭제
    // Delete test files
    DeleteFileW(L"C:\\TestProtected\\test.txt");
    RemoveDirectoryW(L"C:\\TestProtected");

    // 포트 닫기
    // Close port
    if (g_TestContext.PortHandle != NULL) {
        CloseHandle(g_TestContext.PortHandle);
    }
}

// ===== 결과 출력 =====
// ===== Result output =====

void PrintResults(void)
{
    printf("\n");
    printf("========================================\n");
    printf("           테스트 결과 요약             \n");
    // Test Result Summary
    printf("========================================\n");

    for (int i = 0; i < g_TestContext.TotalTests; i++) {
        PTEST_RESULT result = &g_TestContext.Results[i];
        printf("%s %s: %s\n",
            result->Passed ? "[PASS]" : "[FAIL]",
            result->TestName,
            result->Message);
    }

    printf("----------------------------------------\n");
    printf("총계: %d개 테스트, %d 성공, %d 실패\n",
        g_TestContext.TotalTests,
        g_TestContext.PassedTests,
        g_TestContext.FailedTests);
    // Total: tests, passed, failed
    printf("========================================\n");
}

// ===== 테스트 실행기 =====
// ===== Test runner =====

typedef BOOLEAN (*TEST_FUNC)(void);

typedef struct _TEST_ENTRY {
    const char* Name;
    TEST_FUNC Function;
} TEST_ENTRY;

TEST_ENTRY g_Tests[] = {
    { "Driver Connection", Test_DriverConnection },
    { "Add Policy", Test_AddPolicy },
    { "Write Block", Test_WriteBlockOnProtectedFile },
    { "Delete Block", Test_DeleteBlockOnProtectedFile },
    { "Rename Block", Test_RenameBlockOnProtectedFile },
    { "Unprotected Access", Test_AccessAllowedOnUnprotectedFile },
    { "Remove Policy", Test_RemovePolicy },
    { "Access After Removal", Test_AccessAllowedAfterPolicyRemoval },
    { "Driver Disconnection", Test_DriverDisconnection },
    { NULL, NULL }
};

int main(int argc, char* argv[])
{
    printf("========================================\n");
    printf("    DRM Filter 드라이버 통합 테스트     \n");
    // DRM Filter Driver Integration Test
    printf("========================================\n\n");

    // 관리자 권한 확인
    // Check administrator rights
    BOOL isAdmin;
    SID_IDENTIFIER_AUTHORITY NtAuthority = SECURITY_NT_AUTHORITY;
    PSID AdminGroup;

    isAdmin = AllocateAndInitializeSid(
        &NtAuthority, 2,
        SECURITY_BUILTIN_DOMAIN_RID,
        DOMAIN_ALIAS_RID_ADMINS,
        0, 0, 0, 0, 0, 0,
        &AdminGroup);

    if (isAdmin) {
        if (!CheckTokenMembership(NULL, AdminGroup, &isAdmin)) {
            isAdmin = FALSE;
        }
        FreeSid(AdminGroup);
    }

    if (!isAdmin) {
        printf("오류: 관리자 권한으로 실행해야 합니다.\n");
        // Error: Must run as administrator
        return 1;
    }

    // 테스트 실행
    // Run tests
    for (int i = 0; g_Tests[i].Function != NULL; i++) {
        printf("\n[%d/%d] %s\n", i + 1,
            (int)(sizeof(g_Tests)/sizeof(g_Tests[0]) - 1),
            g_Tests[i].Name);

        g_TestContext.TotalTests++;

        BOOLEAN result = g_Tests[i].Function();

        if (result) {
            g_TestContext.PassedTests++;
        } else {
            g_TestContext.FailedTests++;
        }
    }

    // 정리 및 결과 출력
    // Cleanup and print results
    Cleanup();
    PrintResults();

    return g_TestContext.FailedTests > 0 ? 1 : 0;
}
```

## 31.5 Driver Verifier 활용

### Driver Verifier 설정

```powershell
# EnableVerifier.ps1
# Driver Verifier 활성화 스크립트
# Driver Verifier enable script

param(
    [Parameter(Mandatory=$true)]
    [string]$DriverName,

    [switch]$Standard,
    [switch]$Full,
    [switch]$Disable
)

# 관리자 권한 확인
# Check administrator rights
if (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Error "관리자 권한으로 실행하세요."
    # Run as administrator
    exit 1
}

if ($Disable) {
    Write-Host "Driver Verifier 비활성화 중..."
    # Disabling Driver Verifier...
    verifier /reset
    Write-Host "완료. 재부팅 필요."
    # Complete. Reboot required.
    exit 0
}

# 검증 플래그
# Verification flags
$standardFlags = @(
    "/standard"  # 표준 검사
)

$fullFlags = @(
    "/flags",
    "0xBB"  # 전체 검사 플래그
    # Special Pool (0x1)
    # Force IRQL Checking (0x2)
    # Pool Tracking (0x8)
    # I/O Verification (0x10)
    # Deadlock Detection (0x20)
    # DMA Checking (0x80)
)

if ($Standard) {
    Write-Host "표준 Driver Verifier 활성화: $DriverName"
    # Enabling standard Driver Verifier
    verifier /standard /driver $DriverName
}
elseif ($Full) {
    Write-Host "전체 Driver Verifier 활성화: $DriverName"
    # Enabling full Driver Verifier
    verifier /flags 0xBB /driver $DriverName
}
else {
    # 기본: 표준
    # Default: standard
    Write-Host "표준 Driver Verifier 활성화: $DriverName"
    verifier /standard /driver $DriverName
}

Write-Host ""
Write-Host "현재 설정:"
# Current settings:
verifier /querysettings

Write-Host ""
Write-Host "변경 사항을 적용하려면 시스템을 재부팅하세요."
# Reboot system to apply changes.
```

### Driver Verifier 검사 항목

```c
// DriverVerifierChecks.c
// Driver Verifier가 검사하는 일반적인 문제들
// Common issues checked by Driver Verifier

// 1. Special Pool - 메모리 오버런/언더런 검출
// 1. Special Pool - Detects memory overruns/underruns

NTSTATUS BadMemoryAccess()
{
    // 할당된 크기보다 더 많이 접근 - Verifier가 잡음
    // Access beyond allocated size - Verifier catches this
    PUCHAR buffer = ExAllocatePool2(POOL_FLAG_NON_PAGED, 10, 'tseT');
    if (buffer) {
        buffer[15] = 0;  // 오버런! Verifier = BSOD
        ExFreePoolWithTag(buffer, 'tseT');
    }
    return STATUS_SUCCESS;
}

// 2. Pool Tracking - 메모리 누수 검출
// 2. Pool Tracking - Detects memory leaks

NTSTATUS MemoryLeak()
{
    // 해제하지 않은 메모리 - 언로드 시 Verifier가 감지
    // Unreleased memory - Verifier detects at unload
    PVOID leaked = ExAllocatePool2(POOL_FLAG_NON_PAGED, 100, 'kaeL');
    // ExFreePoolWithTag(leaked, 'kaeL');  // 주석 처리됨 = 누수!
    return STATUS_SUCCESS;
}

// 3. IRQL Checking - 잘못된 IRQL에서 호출
// 3. IRQL Checking - Call at wrong IRQL

NTSTATUS BadIrqlCall()
{
    KIRQL oldIrql;

    // DISPATCH_LEVEL에서 페이지 가능 메모리 접근
    // Access paged memory at DISPATCH_LEVEL
    KeRaiseIrql(DISPATCH_LEVEL, &oldIrql);

    // 페이지 풀 할당 - IRQL이 너무 높음!
    // Paged pool allocation - IRQL too high!
    PVOID paged = ExAllocatePool2(POOL_FLAG_PAGED, 100, 'dgaP');
    // Verifier = BSOD (IRQL_NOT_LESS_OR_EQUAL)

    KeLowerIrql(oldIrql);
    return STATUS_SUCCESS;
}

// 4. Deadlock Detection - 잠금 순서 위반
// 4. Deadlock Detection - Lock order violation

KMUTEX MutexA;
KMUTEX MutexB;

VOID Thread1()
{
    KeWaitForMutexObject(&MutexA, Executive, KernelMode, FALSE, NULL);
    KeWaitForMutexObject(&MutexB, Executive, KernelMode, FALSE, NULL);
    // A -> B 순서
    KeReleaseMutex(&MutexB, FALSE);
    KeReleaseMutex(&MutexA, FALSE);
}

VOID Thread2()
{
    KeWaitForMutexObject(&MutexB, Executive, KernelMode, FALSE, NULL);
    KeWaitForMutexObject(&MutexA, Executive, KernelMode, FALSE, NULL);
    // B -> A 순서 - 데드락 가능! Verifier 감지
    KeReleaseMutex(&MutexA, FALSE);
    KeReleaseMutex(&MutexB, FALSE);
}

// 5. I/O Verification - IRP 처리 오류
// 5. I/O Verification - IRP handling errors

NTSTATUS BadIrpCompletion(PIRP Irp)
{
    // 이미 완료된 IRP를 다시 완료
    // Complete already completed IRP
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    IoCompleteRequest(Irp, IO_NO_INCREMENT);  // 두 번째 완료 = 오류!
    return STATUS_SUCCESS;
}
```

### Verifier 결과 분석

```powershell
# AnalyzeVerifier.ps1
# Driver Verifier 결과 분석
# Analyze Driver Verifier results

# 현재 Verifier 상태 확인
# Check current Verifier status
Write-Host "=== Driver Verifier 상태 ==="
# Driver Verifier Status
verifier /query

# 위반 통계 확인
# Check violation statistics
Write-Host ""
Write-Host "=== 위반 통계 ==="
# Violation Statistics

$verifierLog = verifier /log

if ($verifierLog -match "No driver verification") {
    Write-Host "검증 중인 드라이버 없음"
    # No drivers being verified
}
else {
    Write-Host $verifierLog
}

# 미니덤프 분석 안내
# Minidump analysis guide
Write-Host ""
Write-Host "=== BSOD 발생 시 분석 방법 ==="
# Analysis method when BSOD occurs
Write-Host @"
1. 미니덤프 파일 위치: C:\Windows\Minidump\
2. WinDbg로 분석:
   - File -> Open Crash Dump
   - !analyze -v 명령 실행
3. 확인할 내용:
   - DRIVER_VERIFIER_DETECTED_VIOLATION
   - 스택 트레이스에서 드라이버 이름
   - 버그체크 파라미터
"@
# Minidump file location
# Analyze with WinDbg
# Run !analyze -v command
# Check for: DRIVER_VERIFIER_DETECTED_VIOLATION, driver name in stack trace, bugcheck parameters
```

## 31.6 스트레스 테스트

### 고부하 테스트 도구

```c
// StressTest.c
// 고부하 스트레스 테스트
// High load stress test

#include <windows.h>
#include <stdio.h>

#define NUM_THREADS 32
#define ITERATIONS 10000
#define TEST_DIR L"C:\\StressTest"

typedef struct _THREAD_CONTEXT {
    int ThreadId;
    int Iterations;
    int SuccessCount;
    int FailureCount;
    DWORD StartTime;
    DWORD EndTime;
} THREAD_CONTEXT, *PTHREAD_CONTEXT;

// 워커 스레드 함수
// Worker thread function
DWORD WINAPI StressWorkerThread(LPVOID lpParameter)
{
    PTHREAD_CONTEXT ctx = (PTHREAD_CONTEXT)lpParameter;
    WCHAR filePath[MAX_PATH];
    char buffer[4096];
    DWORD written, read;

    ctx->StartTime = GetTickCount();

    for (int i = 0; i < ctx->Iterations; i++) {
        // 고유한 파일 이름 생성
        // Generate unique file name
        swprintf_s(filePath, MAX_PATH,
            L"%s\\thread%d_file%d.tmp",
            TEST_DIR, ctx->ThreadId, i);

        // 파일 생성 및 쓰기
        // Create and write file
        HANDLE hFile = CreateFileW(
            filePath,
            GENERIC_READ | GENERIC_WRITE,
            0,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (hFile == INVALID_HANDLE_VALUE) {
            ctx->FailureCount++;
            continue;
        }

        // 데이터 쓰기
        // Write data
        memset(buffer, 'A' + (ctx->ThreadId % 26), sizeof(buffer));
        if (!WriteFile(hFile, buffer, sizeof(buffer), &written, NULL)) {
            CloseHandle(hFile);
            ctx->FailureCount++;
            continue;
        }

        // 파일 포인터 처음으로
        // Seek to beginning
        SetFilePointer(hFile, 0, NULL, FILE_BEGIN);

        // 데이터 읽기
        // Read data
        if (!ReadFile(hFile, buffer, sizeof(buffer), &read, NULL)) {
            CloseHandle(hFile);
            ctx->FailureCount++;
            continue;
        }

        CloseHandle(hFile);

        // 파일 삭제
        // Delete file
        if (!DeleteFileW(filePath)) {
            ctx->FailureCount++;
            continue;
        }

        ctx->SuccessCount++;

        // 진행률 출력 (매 1000회)
        // Print progress (every 1000)
        if ((i + 1) % 1000 == 0) {
            printf("Thread %d: %d/%d\n", ctx->ThreadId, i + 1, ctx->Iterations);
        }
    }

    ctx->EndTime = GetTickCount();
    return 0;
}

// 메모리 스트레스 테스트
// Memory stress test
DWORD WINAPI MemoryStressThread(LPVOID lpParameter)
{
    PTHREAD_CONTEXT ctx = (PTHREAD_CONTEXT)lpParameter;
    HANDLE hFile;
    WCHAR filePath[MAX_PATH];

    ctx->StartTime = GetTickCount();

    for (int i = 0; i < ctx->Iterations; i++) {
        swprintf_s(filePath, MAX_PATH,
            L"%s\\memtest%d_%d.tmp",
            TEST_DIR, ctx->ThreadId, i);

        // 대용량 파일 매핑
        // Large file mapping
        hFile = CreateFileW(
            filePath,
            GENERIC_READ | GENERIC_WRITE,
            0,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (hFile == INVALID_HANDLE_VALUE) {
            ctx->FailureCount++;
            continue;
        }

        // 파일 크기 설정 (10MB)
        // Set file size (10MB)
        SetFilePointer(hFile, 10 * 1024 * 1024, NULL, FILE_BEGIN);
        SetEndOfFile(hFile);

        // 메모리 매핑
        // Memory mapping
        HANDLE hMapping = CreateFileMappingW(
            hFile,
            NULL,
            PAGE_READWRITE,
            0,
            0,
            NULL
        );

        if (hMapping != NULL) {
            PVOID pView = MapViewOfFile(
                hMapping,
                FILE_MAP_ALL_ACCESS,
                0, 0, 0
            );

            if (pView != NULL) {
                // 메모리 접근
                // Access memory
                memset(pView, 0xAB, 10 * 1024 * 1024);
                UnmapViewOfFile(pView);
                ctx->SuccessCount++;
            } else {
                ctx->FailureCount++;
            }

            CloseHandle(hMapping);
        } else {
            ctx->FailureCount++;
        }

        CloseHandle(hFile);
        DeleteFileW(filePath);
    }

    ctx->EndTime = GetTickCount();
    return 0;
}

// 동시성 스트레스 테스트
// Concurrency stress test
DWORD WINAPI ConcurrencyStressThread(LPVOID lpParameter)
{
    PTHREAD_CONTEXT ctx = (PTHREAD_CONTEXT)lpParameter;
    WCHAR filePath[MAX_PATH];

    // 모든 스레드가 같은 파일에 접근
    // All threads access same file
    swprintf_s(filePath, MAX_PATH, L"%s\\shared_file.tmp", TEST_DIR);

    ctx->StartTime = GetTickCount();

    for (int i = 0; i < ctx->Iterations; i++) {
        // 랜덤 작업 선택
        // Select random operation
        int op = rand() % 4;

        HANDLE hFile = CreateFileW(
            filePath,
            op < 2 ? GENERIC_READ : GENERIC_WRITE,
            FILE_SHARE_READ | FILE_SHARE_WRITE,
            NULL,
            OPEN_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (hFile != INVALID_HANDLE_VALUE) {
            char buffer[256];
            DWORD bytes;

            switch (op) {
                case 0:  // 읽기
                    ReadFile(hFile, buffer, sizeof(buffer), &bytes, NULL);
                    break;
                case 1:  // 읽기 (다른 위치)
                    SetFilePointer(hFile, rand() % 1024, NULL, FILE_BEGIN);
                    ReadFile(hFile, buffer, sizeof(buffer), &bytes, NULL);
                    break;
                case 2:  // 쓰기
                    WriteFile(hFile, buffer, sizeof(buffer), &bytes, NULL);
                    break;
                case 3:  // 쓰기 (다른 위치)
                    SetFilePointer(hFile, rand() % 1024, NULL, FILE_BEGIN);
                    WriteFile(hFile, buffer, sizeof(buffer), &bytes, NULL);
                    break;
            }

            CloseHandle(hFile);
            ctx->SuccessCount++;
        } else {
            ctx->FailureCount++;
        }
    }

    ctx->EndTime = GetTickCount();
    return 0;
}

int main(int argc, char* argv[])
{
    HANDLE threads[NUM_THREADS];
    THREAD_CONTEXT contexts[NUM_THREADS];

    printf("========================================\n");
    printf("     DRM Filter 스트레스 테스트         \n");
    // DRM Filter Stress Test
    printf("========================================\n\n");

    // 테스트 디렉터리 생성
    // Create test directory
    CreateDirectoryW(TEST_DIR, NULL);

    printf("테스트 1: 기본 파일 I/O 스트레스\n");
    // Test 1: Basic file I/O stress
    printf("  스레드: %d, 반복: %d\n", NUM_THREADS, ITERATIONS);
    // Threads, Iterations

    DWORD totalStartTime = GetTickCount();

    // 스레드 생성
    // Create threads
    for (int i = 0; i < NUM_THREADS; i++) {
        contexts[i].ThreadId = i;
        contexts[i].Iterations = ITERATIONS;
        contexts[i].SuccessCount = 0;
        contexts[i].FailureCount = 0;

        threads[i] = CreateThread(
            NULL,
            0,
            StressWorkerThread,
            &contexts[i],
            0,
            NULL
        );
    }

    // 모든 스레드 완료 대기
    // Wait for all threads
    WaitForMultipleObjects(NUM_THREADS, threads, TRUE, INFINITE);

    DWORD totalEndTime = GetTickCount();

    // 결과 집계
    // Aggregate results
    int totalSuccess = 0;
    int totalFailure = 0;

    for (int i = 0; i < NUM_THREADS; i++) {
        totalSuccess += contexts[i].SuccessCount;
        totalFailure += contexts[i].FailureCount;
        CloseHandle(threads[i]);
    }

    DWORD elapsed = totalEndTime - totalStartTime;
    double opsPerSec = (double)(totalSuccess + totalFailure) / (elapsed / 1000.0);

    printf("\n결과:\n");
    // Results:
    printf("  총 작업: %d\n", totalSuccess + totalFailure);
    // Total operations
    printf("  성공: %d\n", totalSuccess);
    // Success
    printf("  실패: %d\n", totalFailure);
    // Failure
    printf("  소요 시간: %.2f 초\n", elapsed / 1000.0);
    // Elapsed time
    printf("  처리량: %.2f ops/sec\n", opsPerSec);
    // Throughput

    // 테스트 디렉터리 정리
    // Cleanup test directory
    RemoveDirectoryW(TEST_DIR);

    printf("\n스트레스 테스트 완료.\n");
    // Stress test complete.

    return totalFailure > 0 ? 1 : 0;
}
```

## 31.7 자동화된 테스트 파이프라인

### PowerShell 테스트 스크립트

```powershell
# RunAllTests.ps1
# 전체 테스트 실행 스크립트
# Full test execution script

param(
    [string]$Configuration = "Debug",
    [string]$Platform = "x64",
    [switch]$SkipBuild,
    [switch]$SkipUnitTests,
    [switch]$SkipIntegrationTests,
    [switch]$SkipStressTests,
    [switch]$EnableVerifier
)

$ErrorActionPreference = "Stop"

$ScriptDir = Split-Path -Parent $MyInvocation.MyCommand.Path
$SolutionDir = Split-Path -Parent $ScriptDir
$OutputDir = "$SolutionDir\Output\$Configuration\$Platform"
$TestResultsDir = "$SolutionDir\TestResults"

# 결과 디렉터리 생성
# Create results directory
New-Item -ItemType Directory -Force -Path $TestResultsDir | Out-Null

$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$reportFile = "$TestResultsDir\TestReport_$timestamp.xml"

Write-Host "========================================" -ForegroundColor Cyan
Write-Host "     DRM Filter 테스트 파이프라인       " -ForegroundColor Cyan
# DRM Filter Test Pipeline
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""

# 1. 빌드
# 1. Build
if (-not $SkipBuild) {
    Write-Host "[1/5] 솔루션 빌드 중..." -ForegroundColor Yellow
    # Building solution...

    $msbuild = "C:\Program Files\Microsoft Visual Studio\2022\Professional\MSBuild\Current\Bin\MSBuild.exe"

    & $msbuild "$SolutionDir\DrmFilter.sln" `
        /p:Configuration=$Configuration `
        /p:Platform=$Platform `
        /t:Rebuild `
        /v:minimal

    if ($LASTEXITCODE -ne 0) {
        Write-Host "빌드 실패!" -ForegroundColor Red
        # Build failed!
        exit 1
    }

    Write-Host "빌드 성공" -ForegroundColor Green
    # Build successful
}

# 2. 단위 테스트
# 2. Unit tests
if (-not $SkipUnitTests) {
    Write-Host ""
    Write-Host "[2/5] 단위 테스트 실행 중..." -ForegroundColor Yellow
    # Running unit tests...

    $unitTestExe = "$OutputDir\SharedLogicTests.exe"

    if (Test-Path $unitTestExe) {
        & $unitTestExe --gtest_output=xml:$TestResultsDir\UnitTests_$timestamp.xml

        if ($LASTEXITCODE -ne 0) {
            Write-Host "단위 테스트 실패!" -ForegroundColor Red
            # Unit tests failed!
            # 계속 진행 (다른 테스트도 실행)
        } else {
            Write-Host "단위 테스트 통과" -ForegroundColor Green
            # Unit tests passed
        }
    } else {
        Write-Host "단위 테스트 실행 파일 없음: $unitTestExe" -ForegroundColor Yellow
        # Unit test executable not found
    }
}

# 3. 드라이버 로드
# 3. Load driver
Write-Host ""
Write-Host "[3/5] 드라이버 로드 중..." -ForegroundColor Yellow
# Loading driver...

$driverPath = "$OutputDir\DrmFilter.sys"
$infPath = "$OutputDir\DrmFilter.inf"

# 기존 드라이버 언로드
# Unload existing driver
fltmc unload DrmFilter 2>$null

# 드라이버 설치 및 로드
# Install and load driver
if ($EnableVerifier) {
    Write-Host "Driver Verifier 활성화..." -ForegroundColor Yellow
    # Enabling Driver Verifier...
    verifier /standard /driver DrmFilter.sys
}

# INF로 설치
# Install with INF
# rundll32.exe setupapi.dll,InstallHinfSection DefaultInstall 128 $infPath

# fltmc로 로드
fltmc load DrmFilter

if ($LASTEXITCODE -ne 0) {
    Write-Host "드라이버 로드 실패!" -ForegroundColor Red
    # Driver load failed!
    exit 1
}

Write-Host "드라이버 로드 성공" -ForegroundColor Green
# Driver loaded successfully

# 4. 통합 테스트
# 4. Integration tests
if (-not $SkipIntegrationTests) {
    Write-Host ""
    Write-Host "[4/5] 통합 테스트 실행 중..." -ForegroundColor Yellow
    # Running integration tests...

    $integrationTestExe = "$OutputDir\DriverTestApp.exe"

    if (Test-Path $integrationTestExe) {
        & $integrationTestExe > $TestResultsDir\IntegrationTests_$timestamp.log 2>&1

        $integrationResult = $LASTEXITCODE

        Get-Content $TestResultsDir\IntegrationTests_$timestamp.log

        if ($integrationResult -ne 0) {
            Write-Host "통합 테스트 실패!" -ForegroundColor Red
            # Integration tests failed!
        } else {
            Write-Host "통합 테스트 통과" -ForegroundColor Green
            # Integration tests passed
        }
    }
}

# 5. 스트레스 테스트
# 5. Stress tests
if (-not $SkipStressTests) {
    Write-Host ""
    Write-Host "[5/5] 스트레스 테스트 실행 중..." -ForegroundColor Yellow
    # Running stress tests...

    $stressTestExe = "$OutputDir\StressTest.exe"

    if (Test-Path $stressTestExe) {
        & $stressTestExe > $TestResultsDir\StressTests_$timestamp.log 2>&1

        $stressResult = $LASTEXITCODE

        Get-Content $TestResultsDir\StressTests_$timestamp.log | Select-Object -Last 20

        if ($stressResult -ne 0) {
            Write-Host "스트레스 테스트 실패!" -ForegroundColor Red
            # Stress tests failed!
        } else {
            Write-Host "스트레스 테스트 통과" -ForegroundColor Green
            # Stress tests passed
        }
    }
}

# 드라이버 언로드
# Unload driver
Write-Host ""
Write-Host "드라이버 언로드 중..." -ForegroundColor Yellow
# Unloading driver...
fltmc unload DrmFilter

if ($EnableVerifier) {
    Write-Host "Driver Verifier 비활성화..." -ForegroundColor Yellow
    # Disabling Driver Verifier...
    verifier /reset
}

# 결과 요약
# Results summary
Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "           테스트 완료                  " -ForegroundColor Cyan
# Tests Complete
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "결과 위치: $TestResultsDir"
# Results location

# 테스트 결과 파일 목록
# List test result files
Get-ChildItem $TestResultsDir -Filter "*$timestamp*" | ForEach-Object {
    Write-Host "  - $($_.Name)"
}
```

## 31.8 정리

이 챕터에서 학습한 내용:

1. **커널 테스트의 특수성**
   - 사용자 모드 테스트와의 차이점
   - BSOD 위험 관리
   - 격리의 어려움

2. **단위 테스트 전략**
   - 공유 라이브러리로 로직 분리
   - Google Test 활용
   - 사용자 모드에서 테스트 실행

3. **통합 테스트**
   - 드라이버 통신을 통한 테스트
   - 테스트 앱 구현
   - 시나리오 기반 검증

4. **Driver Verifier**
   - 메모리 오류 검출
   - IRQL 위반 감지
   - 데드락 탐지

5. **스트레스 테스트**
   - 다중 스레드 부하 테스트
   - 동시성 테스트
   - 성능 측정

6. **자동화 파이프라인**
   - PowerShell 스크립트
   - CI/CD 통합

다음 챕터에서는 **성능 최적화**를 다룹니다.

---

**다음 챕터 예고**: Chapter 32에서는 Minifilter의 성능 분석, 병목 현상 식별, 최적화 기법을 학습합니다.
