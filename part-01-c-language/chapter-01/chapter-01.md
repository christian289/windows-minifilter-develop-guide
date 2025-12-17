# Chapter 1: C 언어와 C#의 근본적 차이

## 난이도: ⭐⭐ (초급)

## 학습 목표

- C와 C#의 메모리 관리 방식 차이 이해
- 두 언어의 타입 시스템 비교
- 에러 처리 패러다임의 차이 인식
- 빌드/컴파일 과정의 차이 파악

---

## 들어가며

10년 이상 .NET 개발을 해온 여러분에게 C 언어는 어떤 의미일까요? 대학 시절 배웠던 낡은 언어? 포인터 때문에 어려운 언어? 혹은 "요즘 누가 C를 써?"라는 생각이 들 수도 있습니다.

하지만 Windows 커널 드라이버 개발에서 C는 여전히 **유일한 선택지**입니다. C++도 제한적으로 사용할 수 있지만, 대부분의 WDK(Windows Driver Kit) 예제와 문서는 C로 작성되어 있습니다.

이 장에서는 C#과 C의 **근본적인 차이점**을 살펴봅니다. 차이점을 이해하면, C 코드를 읽을 때 "이건 C#에서 이것과 비슷하구나"라고 연결 지을 수 있습니다.

---

## 1.1 메모리 관리: GC vs 수동 관리

### C#의 세계: Garbage Collector

.NET 개발자로서 여러분은 메모리 관리에 대해 크게 걱정하지 않았을 것입니다. `new`로 객체를 생성하고, 더 이상 사용하지 않으면 GC(Garbage Collector)가 알아서 정리해줍니다.

```csharp
// C# - 메모리 관리가 자동
// C# - Memory management is automatic
public void ProcessData()
{
    var list = new List<string>();  // 힙에 할당
    list.Add("Hello");
    list.Add("World");

    // 함수 종료 후, list는 더 이상 참조되지 않음
    // GC가 적절한 시점에 메모리를 회수함
}
```

GC의 동작 원리를 아는 .NET 개발자라면:
- 세대별 가비지 컬렉션 (Generation 0, 1, 2)
- 메모리 압축 (Compaction)
- 대형 객체 힙 (Large Object Heap)

이런 개념들이 익숙할 것입니다. 중요한 점은, **여러분이 직접 메모리를 해제하지 않는다**는 것입니다.

### C의 세계: 수동 메모리 관리

C에서는 완전히 다릅니다. 메모리를 할당하면 **반드시 직접 해제**해야 합니다.

```c
// C - 메모리 관리가 수동
// C - Memory management is manual
void ProcessData()
{
    // 힙에 메모리 할당
    // Allocate memory on heap
    char* buffer = (char*)malloc(256);
    if (buffer == NULL) {
        return;  // 할당 실패 처리
    }

    strcpy(buffer, "Hello World");

    // 사용 완료 후 반드시 해제
    // Must free after use
    free(buffer);
    buffer = NULL;  // 해제 후 NULL 설정 권장
}
```

만약 `free(buffer)`를 호출하지 않으면? **메모리 누수(Memory Leak)**가 발생합니다. 프로그램이 계속 실행되면서 메모리 사용량이 점점 증가하고, 결국 시스템 리소스가 고갈됩니다.

### 커널 모드에서의 메모리 관리

커널 드라이버에서는 `malloc`/`free` 대신 다른 API를 사용합니다:

```c
// 커널 모드 메모리 할당
// Kernel mode memory allocation
#define MY_POOL_TAG 'tliF'  // 'Filt' 역순

PVOID buffer = ExAllocatePoolWithTag(
    NonPagedPool,    // 풀 타입
    256,             // 크기
    MY_POOL_TAG      // 디버깅용 태그
);

if (buffer == NULL) {
    return STATUS_INSUFFICIENT_RESOURCES;
}

// 사용 후 해제
// Free after use
ExFreePoolWithTag(buffer, MY_POOL_TAG);
```

커널에서 메모리 누수가 발생하면 사용자 모드보다 훨씬 심각합니다. 시스템 전체의 안정성에 영향을 미칠 수 있습니다.

### 비교 요약

| 항목 | C# | C (사용자 모드) | C (커널 모드) |
|------|----|--------------|--------------|
| 메모리 할당 | `new` | `malloc()` | `ExAllocatePoolWithTag()` |
| 메모리 해제 | GC 자동 처리 | `free()` | `ExFreePoolWithTag()` |
| 메모리 누수 | 거의 없음 | 개발자 책임 | 개발자 책임 (더 심각) |
| 성능 오버헤드 | GC 일시 정지 | 없음 | 없음 |
| 안전성 | 높음 | 낮음 | 매우 낮음 |

### .NET 개발자가 주의할 점

1. **using 문이 없다**: C#에서 `IDisposable`을 `using`으로 처리하듯, C에서는 모든 리소스를 명시적으로 정리해야 합니다.

2. **try-finally 대신 goto 패턴**: C에서는 정리 로직을 위해 `goto`를 사용하는 패턴이 일반적입니다 (놀랍게도!).

```c
// C의 일반적인 리소스 정리 패턴
// Common resource cleanup pattern in C
NTSTATUS MyFunction()
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID buffer1 = NULL;
    PVOID buffer2 = NULL;

    buffer1 = ExAllocatePoolWithTag(NonPagedPool, 100, 'fuB1');
    if (buffer1 == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    buffer2 = ExAllocatePoolWithTag(NonPagedPool, 200, 'fuB2');
    if (buffer2 == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    // 작업 수행
    // Do work here

Cleanup:
    if (buffer2 != NULL) {
        ExFreePoolWithTag(buffer2, 'fuB2');
    }
    if (buffer1 != NULL) {
        ExFreePoolWithTag(buffer1, 'fuB1');
    }

    return status;
}
```

이 패턴은 C#의 `try-finally`와 유사한 역할을 합니다:

```csharp
// C# 대응 코드
// C# equivalent
public void MyMethod()
{
    Resource1? res1 = null;
    Resource2? res2 = null;

    try
    {
        res1 = new Resource1();
        res2 = new Resource2();

        // 작업 수행
    }
    finally
    {
        res2?.Dispose();
        res1?.Dispose();
    }
}
```

---

## 1.2 타입 시스템 비교

### 값 타입과 참조 타입

C#에서는 `struct`가 값 타입, `class`가 참조 타입입니다. 이 구분은 성능과 메모리 레이아웃에 큰 영향을 미칩니다.

```csharp
// C# - 값 타입 vs 참조 타입
// C# - Value type vs Reference type
struct Point { public int X, Y; }     // 값 타입 - 스택
class Person { public string Name; }   // 참조 타입 - 힙
```

C에서는 이 구분이 없습니다. 모든 데이터는 기본적으로 **값**입니다. 포인터를 사용해야만 참조 의미론을 얻을 수 있습니다.

```c
// C - 모든 것은 값
// C - Everything is a value
typedef struct {
    int X;
    int Y;
} Point;

void ModifyPoint(Point p)        // 값 복사 (수정해도 원본 불변)
{
    p.X = 100;  // 복사본만 수정됨
}

void ModifyPointPtr(Point* p)    // 포인터로 전달 (원본 수정 가능)
{
    p->X = 100;  // 원본이 수정됨
}
```

### 기본 데이터 타입 비교

| C 타입 | C# 대응 | 크기 (64-bit Windows) | 비고 |
|--------|---------|----------------------|------|
| `char` | `sbyte` | 1 byte | 부호 있음 |
| `unsigned char` | `byte` | 1 byte | |
| `short` | `short` | 2 bytes | |
| `unsigned short` | `ushort` | 2 bytes | |
| `int` | `int` | 4 bytes | |
| `unsigned int` | `uint` | 4 bytes | |
| `long` | `int` | 4 bytes | **주의**: Windows에서 4바이트 |
| `unsigned long` | `uint` | 4 bytes | |
| `long long` | `long` | 8 bytes | |
| `float` | `float` | 4 bytes | |
| `double` | `double` | 8 bytes | |
| `void*` | `IntPtr` | 8 bytes (64-bit) | 포인터 |

> ⚠️ **중요**: C의 `long`은 Windows에서 4바이트입니다. C#의 `long`(8바이트)과 다릅니다!

### Windows 커널의 타입 별칭

WDK에서는 명확성을 위해 다양한 타입 별칭을 정의합니다:

```c
// Windows 커널에서 자주 사용하는 타입
// Commonly used types in Windows kernel
UCHAR       // unsigned char (1 byte)
USHORT      // unsigned short (2 bytes)
ULONG       // unsigned long (4 bytes)
ULONG64     // unsigned __int64 (8 bytes)
ULONGLONG   // unsigned long long (8 bytes)

BOOLEAN     // unsigned char (TRUE/FALSE)
NTSTATUS    // long (함수 반환 상태 코드)

PVOID       // void* (일반 포인터)
HANDLE      // void* (핸들)

UNICODE_STRING  // 유니코드 문자열 구조체
```

```csharp
// C# 대응
// C# equivalent
byte        // UCHAR
ushort      // USHORT
uint        // ULONG
ulong       // ULONG64, ULONGLONG

bool        // BOOLEAN (주의: 크기가 다를 수 있음)
int         // NTSTATUS

IntPtr      // PVOID, HANDLE
```

### 문자열의 차이

이것은 매우 중요한 차이점입니다.

```csharp
// C# - 문자열은 불변 객체
// C# - Strings are immutable objects
string name = "Hello";
name = name + " World";  // 새 객체 생성
```

```c
// C - 문자열은 문자 배열
// C - Strings are character arrays
char name[256] = "Hello";
strcat(name, " World");  // 원본 배열 수정

// 또는 유니코드
WCHAR wideName[256] = L"Hello";
wcscat(wideName, L" World");
```

C에서 문자열은 그저 **문자의 배열**일 뿐입니다. 끝에 NULL 문자(`'\0'` 또는 `L'\0'`)가 있어야 문자열의 끝을 알 수 있습니다.

커널에서는 `UNICODE_STRING` 구조체를 주로 사용합니다:

```c
// 커널의 문자열 구조체
// Kernel string structure
typedef struct _UNICODE_STRING {
    USHORT Length;         // 현재 길이 (바이트 단위)
    USHORT MaximumLength;  // 버퍼 최대 크기
    PWCH   Buffer;         // 실제 문자열 버퍼
} UNICODE_STRING;

// 사용 예
// Usage example
UNICODE_STRING myString;
RtlInitUnicodeString(&myString, L"Hello World");
```

---

## 1.3 에러 처리: 예외 vs 반환 코드

### C#의 예외 기반 에러 처리

.NET에서는 예외(Exception)가 에러 처리의 핵심입니다:

```csharp
// C# - 예외 기반 에러 처리
// C# - Exception-based error handling
try
{
    using var file = File.OpenRead("test.txt");
    var content = new byte[file.Length];
    file.Read(content, 0, content.Length);
}
catch (FileNotFoundException ex)
{
    Console.WriteLine($"파일을 찾을 수 없습니다: {ex.FileName}");
}
catch (UnauthorizedAccessException ex)
{
    Console.WriteLine($"접근 권한이 없습니다: {ex.Message}");
}
finally
{
    // 정리 작업
}
```

예외의 장점:
- 에러 처리 코드와 정상 경로 코드가 분리됨
- 호출 스택을 통해 에러가 전파됨
- 풍부한 에러 정보 (스택 트레이스, 내부 예외 등)

### C의 반환 코드 기반 에러 처리

C에서는 예외가 없습니다. 대신 함수의 **반환 값**으로 성공/실패를 나타냅니다:

```c
// C - 반환 코드 기반 에러 처리
// C - Return code-based error handling
FILE* file = fopen("test.txt", "r");
if (file == NULL) {
    // errno로 구체적인 에러 확인
    if (errno == ENOENT) {
        printf("파일을 찾을 수 없습니다\n");
    } else if (errno == EACCES) {
        printf("접근 권한이 없습니다\n");
    }
    return -1;
}

// 파일 사용
char buffer[256];
if (fgets(buffer, sizeof(buffer), file) == NULL) {
    // 읽기 실패 처리
    fclose(file);
    return -1;
}

fclose(file);
return 0;
```

### Windows 커널의 NTSTATUS

커널 드라이버에서는 `NTSTATUS` 타입으로 결과를 반환합니다:

```c
// 커널의 NTSTATUS 기반 에러 처리
// NTSTATUS-based error handling in kernel
NTSTATUS OpenMyFile(PUNICODE_STRING FileName, PHANDLE FileHandle)
{
    OBJECT_ATTRIBUTES objAttr;
    IO_STATUS_BLOCK ioStatus;
    NTSTATUS status;

    InitializeObjectAttributes(
        &objAttr,
        FileName,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        NULL
    );

    status = ZwCreateFile(
        FileHandle,
        GENERIC_READ,
        &objAttr,
        &ioStatus,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        FILE_SHARE_READ,
        FILE_OPEN,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0
    );

    if (!NT_SUCCESS(status)) {
        // 에러 로깅
        DbgPrint("ZwCreateFile failed: 0x%08X\n", status);
        return status;
    }

    return STATUS_SUCCESS;
}
```

### 주요 NTSTATUS 값

```c
// 성공
STATUS_SUCCESS              // 0x00000000 - 성공

// 정보
STATUS_BUFFER_OVERFLOW      // 0x80000005 - 버퍼가 작아 일부만 반환

// 경고
STATUS_NO_MORE_ENTRIES      // 0x8000001A - 더 이상 항목 없음

// 에러
STATUS_UNSUCCESSFUL         // 0xC0000001 - 일반 실패
STATUS_ACCESS_DENIED        // 0xC0000022 - 접근 거부
STATUS_OBJECT_NAME_NOT_FOUND // 0xC0000034 - 객체를 찾을 수 없음
STATUS_INSUFFICIENT_RESOURCES // 0xC000009A - 리소스 부족
```

### NT_SUCCESS 매크로

```c
// NTSTATUS 값 검사 매크로
// NTSTATUS value check macro
#define NT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)
#define NT_INFORMATION(Status) ((((ULONG)(Status)) >> 30) == 1)
#define NT_WARNING(Status) ((((ULONG)(Status)) >> 30) == 2)
#define NT_ERROR(Status) ((((ULONG)(Status)) >> 30) == 3)
```

```c
// 사용 예
// Usage example
NTSTATUS status = SomeKernelFunction();

if (NT_SUCCESS(status)) {
    // 성공 또는 정보 상태
} else {
    // 경고 또는 에러
    if (status == STATUS_ACCESS_DENIED) {
        // 특정 에러 처리
    }
}
```

### Early Return 패턴

C 코드에서는 에러 발생 즉시 반환하는 패턴이 일반적입니다:

```c
// Early Return 패턴
// Early Return pattern
NTSTATUS ProcessFile(PUNICODE_STRING FileName)
{
    HANDLE fileHandle = NULL;
    PVOID buffer = NULL;
    NTSTATUS status;

    // 파일 열기
    status = OpenMyFile(FileName, &fileHandle);
    if (!NT_SUCCESS(status)) {
        return status;  // 즉시 반환
    }

    // 메모리 할당
    buffer = ExAllocatePoolWithTag(PagedPool, 1024, 'fuB1');
    if (buffer == NULL) {
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;  // 즉시 반환
    }

    // 작업 수행
    // ...

    // 정리
    ExFreePoolWithTag(buffer, 'fuB1');
    ZwClose(fileHandle);

    return STATUS_SUCCESS;
}
```

---

## 1.4 빌드 시스템

### C#의 빌드 과정

.NET 개발자에게 익숙한 빌드 과정:

```
소스 코드 (.cs)
     ↓
  C# 컴파일러 (Roslyn)
     ↓
  IL 코드 (.dll/.exe)
     ↓
  JIT 컴파일러 (실행 시)
     ↓
  네이티브 코드
```

- **MSBuild**: 빌드 엔진
- **NuGet**: 패키지 관리자
- **.csproj**: 프로젝트 파일

### C의 빌드 과정

```
소스 코드 (.c, .h)
     ↓
  전처리기 (Preprocessor)
     ↓
  전처리된 소스
     ↓
  컴파일러 (Compiler)
     ↓
  오브젝트 파일 (.obj)
     ↓
  링커 (Linker)
     ↓
  실행 파일 (.exe, .sys)
```

### 각 단계 설명

#### 1. 전처리기 (Preprocessor)

`#include`, `#define`, `#ifdef` 등의 지시문을 처리합니다:

```c
// 전처리기 지시문 예시
// Preprocessor directive examples
#include <ntddk.h>    // 헤더 파일 포함
#define MAX_SIZE 1024  // 상수 정의

#ifdef DBG
    DbgPrint("Debug build\n");
#endif
```

전처리 후에는 모든 `#include`가 실제 코드로 대체되고, `#define`이 적용됩니다.

#### 2. 컴파일러 (Compiler)

전처리된 C 코드를 기계어(오브젝트 파일)로 변환합니다:
- 문법 검사
- 타입 검사
- 최적화
- 기계어 생성

#### 3. 링커 (Linker)

여러 오브젝트 파일과 라이브러리를 하나의 실행 파일로 결합합니다:
- 심볼 해결 (함수와 변수 주소 연결)
- 라이브러리 결합
- 최종 바이너리 생성

### WDK 빌드 시스템

드라이버 빌드에는 특별한 설정이 필요합니다:

```
Visual Studio 2022
       +
Windows Driver Kit (WDK)
       +
Windows SDK
       ↓
드라이버 프로젝트 (.vcxproj)
       ↓
빌드 (MSBuild + WDK 도구)
       ↓
드라이버 파일 (.sys) + 카탈로그 파일 (.cat)
```

### C# 프로젝트 vs C 드라이버 프로젝트 비교

| 항목 | C# 프로젝트 | C 드라이버 프로젝트 |
|------|-----------|-------------------|
| 프로젝트 파일 | .csproj | .vcxproj |
| 의존성 관리 | NuGet | WDK 라이브러리 |
| 빌드 결과물 | .dll, .exe | .sys, .cat, .inf |
| 디버그 정보 | .pdb | .pdb |
| 배포 | xcopy, 패키지 | 드라이버 서명 필요 |

### 드라이버 프로젝트 필수 파일

```
MyDriver/
├── MyDriver.vcxproj       # 프로젝트 파일
├── MyDriver.c             # 소스 코드
├── MyDriver.h             # 헤더 파일
├── MyDriver.inf           # 설치 정보 파일
└── MyDriver.rc            # 리소스 파일 (버전 정보)
```

INF 파일은 드라이버 설치에 필요한 정보를 담고 있습니다:

```inf
; MyDriver.inf
[Version]
Signature   = "$WINDOWS NT$"
Class       = "ActivityMonitor"
ClassGuid   = {b86dff51-a31e-4bac-b3cf-e8cfe75c9fc2}
Provider    = %ManufacturerName%
DriverVer   =
CatalogFile = MyDriver.cat

[Strings]
ManufacturerName = "My Company"
```

---

## 정리

이 장에서 배운 C#과 C의 주요 차이점:

| 영역 | C# | C |
|------|----|----|
| **메모리 관리** | GC 자동 관리 | 수동 할당/해제 |
| **타입 시스템** | 값/참조 타입 구분 | 모든 것이 값, 포인터로 참조 |
| **에러 처리** | 예외 (Exception) | 반환 코드 (NTSTATUS) |
| **빌드** | Roslyn + JIT | 전처리 → 컴파일 → 링크 |
| **문자열** | 불변 객체 | 문자 배열 |

### 다음 장 미리보기

Chapter 2에서는 C 언어의 핵심 문법을 배웁니다:
- 포인터 완전 정복
- 구조체와 공용체
- 전처리기 매크로
- 함수 포인터와 콜백

.NET 개발자로서의 경험을 바탕으로, 각 개념을 C#과 비교하며 학습합니다.

---

## 연습 문제

### 1. 메모리 관리

다음 C# 코드를 C 스타일로 변환할 때, 어떤 점을 주의해야 할까요?

```csharp
public List<int> GetNumbers()
{
    var numbers = new List<int> { 1, 2, 3, 4, 5 };
    return numbers;
}
```

**힌트**: C에서는 함수 내에서 할당한 메모리를 반환할 때 주의가 필요합니다.

### 2. 타입 크기

다음 중 64-bit Windows에서 크기가 8바이트인 타입은? (복수 선택)
- [ ] `long`
- [ ] `long long`
- [ ] `int*`
- [ ] `double`

### 3. 에러 처리

`STATUS_ACCESS_DENIED`가 반환되었을 때 `NT_SUCCESS(status)`의 결과는?

---

## 참고 자료

- [Microsoft Docs: Windows Driver Development](https://docs.microsoft.com/en-us/windows-hardware/drivers/)
- [NTSTATUS Values](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55)
- [WDK Samples on GitHub](https://github.com/microsoft/Windows-driver-samples)
