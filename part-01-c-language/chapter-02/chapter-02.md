# Chapter 2: C 언어 핵심 문법 속성 코스

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표

- C의 기본 데이터 타입과 Windows 타입 별칭 이해
- 포인터의 개념과 활용법 완전 습득
- 구조체와 공용체 사용법 숙지
- 전처리기 지시문 활용
- 함수 포인터와 콜백 패턴 이해

---

## 들어가며

.NET 개발자라면 C 문법의 많은 부분이 익숙할 것입니다. C#은 C 문법을 상당 부분 계승했기 때문입니다. 하지만 **포인터**라는 개념이 등장하면 상황이 달라집니다.

이 장에서는 커널 드라이버 개발에 필요한 C 문법을 집중적으로 다룹니다. 특히 포인터는 시간을 들여 확실히 이해해야 합니다.

---

## 2.1 데이터 타입

### 2.1.1 기본 타입

C의 기본 타입은 플랫폼에 따라 크기가 달라질 수 있습니다. 64-bit Windows 기준:

| C 타입 | C# 대응 | 크기 | 설명 |
|--------|---------|------|------|
| `char` | `sbyte` | 1 byte | 문자 또는 작은 정수 |
| `unsigned char` | `byte` | 1 byte | 부호 없는 1바이트 |
| `short` | `short` | 2 bytes | 짧은 정수 |
| `unsigned short` | `ushort` | 2 bytes | 부호 없는 짧은 정수 |
| `int` | `int` | 4 bytes | 정수 |
| `unsigned int` | `uint` | 4 bytes | 부호 없는 정수 |
| `long` | `int` | 4 bytes | **Windows에서 4바이트!** |
| `unsigned long` | `uint` | 4 bytes | 부호 없는 long |
| `long long` | `long` | 8 bytes | 큰 정수 |
| `float` | `float` | 4 bytes | 단정밀도 실수 |
| `double` | `double` | 8 bytes | 배정밀도 실수 |

```c
// 크기 확인 예제
// Size check example
#include <stdio.h>

int main()
{
    printf("char: %zu bytes\n", sizeof(char));           // 1
    printf("short: %zu bytes\n", sizeof(short));         // 2
    printf("int: %zu bytes\n", sizeof(int));             // 4
    printf("long: %zu bytes\n", sizeof(long));           // 4 (Windows)
    printf("long long: %zu bytes\n", sizeof(long long)); // 8
    printf("pointer: %zu bytes\n", sizeof(void*));       // 8 (64-bit)
    return 0;
}
```

### 2.1.2 Windows 타입 별칭

Windows 커널에서는 명확성을 위해 타입 별칭을 사용합니다:

```c
// Windows 타입 별칭 (ntdef.h, wdm.h에 정의)
// Windows type aliases (defined in ntdef.h, wdm.h)

// 부호 없는 정수형
typedef unsigned char       UCHAR;      // 1 byte
typedef unsigned short      USHORT;     // 2 bytes
typedef unsigned long       ULONG;      // 4 bytes
typedef unsigned __int64    ULONG64;    // 8 bytes
typedef unsigned long long  ULONGLONG;  // 8 bytes

// 부호 있는 정수형
typedef signed char         CHAR;       // 1 byte
typedef signed short        SHORT;      // 2 bytes
typedef signed long         LONG;       // 4 bytes
typedef signed __int64      LONGLONG;   // 8 bytes

// 불리언
typedef unsigned char       BOOLEAN;    // 1 byte (TRUE/FALSE)
typedef int                 BOOL;       // 4 bytes (Win32 호환)

// 상태 코드
typedef long                NTSTATUS;   // 4 bytes

// 포인터 타입
typedef void*               PVOID;      // 일반 포인터
typedef UCHAR*              PUCHAR;     // UCHAR 포인터
typedef ULONG*              PULONG;     // ULONG 포인터
typedef WCHAR*              PWCHAR;     // 와이드 문자 포인터
typedef WCHAR*              PWSTR;      // 와이드 문자열 포인터

// 핸들
typedef PVOID               HANDLE;     // 커널 객체 핸들
```

### 2.1.3 크기에 독립적인 타입

포인터 크기에 따라 변하는 타입도 있습니다:

```c
// 포인터 크기 타입
// Pointer-sized types
typedef __int64             INT_PTR;    // 32-bit: 4, 64-bit: 8
typedef unsigned __int64    UINT_PTR;   // 32-bit: 4, 64-bit: 8
typedef __int64             LONG_PTR;   // 32-bit: 4, 64-bit: 8
typedef unsigned __int64    ULONG_PTR;  // 32-bit: 4, 64-bit: 8

// SIZE_T는 메모리 크기를 나타냄
typedef ULONG_PTR           SIZE_T;
```

```csharp
// C# 대응
// C# equivalent
IntPtr    // INT_PTR, LONG_PTR
UIntPtr   // UINT_PTR, ULONG_PTR
nuint     // SIZE_T (C# 9+)
```

---

## 2.2 포인터 완전 정복

포인터는 C 언어의 핵심이자, .NET 개발자가 가장 어려워하는 부분입니다. 차근차근 살펴봅시다.

### 2.2.1 포인터란 무엇인가

**포인터(Pointer)**는 메모리 주소를 저장하는 변수입니다.

```
메모리 레이아웃:
┌─────────────┬─────────────┬─────────────┬─────────────┐
│ 주소: 0x1000│ 주소: 0x1004│ 주소: 0x1008│ 주소: 0x100C│
├─────────────┼─────────────┼─────────────┼─────────────┤
│   value=42  │    ptr      │    ...      │    ...      │
│   (int)     │  =0x1000    │             │             │
└─────────────┴─────────────┴─────────────┴─────────────┘
       ↑            │
       └────────────┘ (ptr이 value의 주소를 가리킴)
```

```c
// 포인터 기본
// Pointer basics
int value = 42;      // 일반 변수
int* ptr = &value;   // ptr은 value의 주소를 저장

// &: 주소 연산자 (변수의 주소를 얻음)
// *: 역참조 연산자 (포인터가 가리키는 값에 접근)

printf("value의 값: %d\n", value);    // 42
printf("value의 주소: %p\n", &value); // 0x1000 (예시)
printf("ptr의 값: %p\n", ptr);        // 0x1000
printf("ptr이 가리키는 값: %d\n", *ptr); // 42

*ptr = 100;  // 포인터를 통해 value 수정
printf("value의 값: %d\n", value);    // 100
```

### C#과의 비교

```csharp
// C# - ref 키워드와 유사
// C# - Similar to ref keyword
void ModifyValue(ref int value)
{
    value = 100;  // 원본이 수정됨
}

int number = 42;
ModifyValue(ref number);
Console.WriteLine(number);  // 100
```

```c
// C - 포인터 사용
// C - Using pointer
void ModifyValue(int* valuePtr)
{
    *valuePtr = 100;  // 원본이 수정됨
}

int number = 42;
ModifyValue(&number);
printf("%d\n", number);  // 100
```

### 2.2.2 포인터 선언과 초기화

```c
// 포인터 선언 방법
// Pointer declaration
int* ptr1;       // 스타일 1: 타입 옆에 *
int *ptr2;       // 스타일 2: 변수 옆에 *
int * ptr3;      // 스타일 3: 공백 양쪽

// 주의: 다중 선언 시
int* a, b;       // a는 포인터, b는 일반 int!
int *a, *b;      // 둘 다 포인터

// 초기화
int value = 10;
int* ptr = &value;   // 변수 주소로 초기화
int* nullPtr = NULL; // NULL로 초기화 (권장)
```

### 2.2.3 포인터 연산

포인터에 대한 산술 연산은 타입 크기를 고려합니다:

```c
// 포인터 연산
// Pointer arithmetic
int arr[5] = {10, 20, 30, 40, 50};
int* ptr = arr;  // 배열 이름은 첫 요소의 주소

printf("ptr: %p, *ptr: %d\n", ptr, *ptr);      // 10
ptr++;  // 4바이트 증가 (int 크기)
printf("ptr: %p, *ptr: %d\n", ptr, *ptr);      // 20
ptr += 2;  // 8바이트 증가 (int 2개)
printf("ptr: %p, *ptr: %d\n", ptr, *ptr);      // 40
```

```
메모리 레이아웃:
┌────────┬────────┬────────┬────────┬────────┐
│ arr[0] │ arr[1] │ arr[2] │ arr[3] │ arr[4] │
│   10   │   20   │   30   │   40   │   50   │
├────────┼────────┼────────┼────────┼────────┤
│ 0x1000 │ 0x1004 │ 0x1008 │ 0x100C │ 0x1010 │
└────────┴────────┴────────┴────────┴────────┘
    ↑
   ptr (처음)
           ↑
          ptr++ 후
                            ↑
                          ptr += 2 후
```

### 2.2.4 포인터와 배열의 관계

```c
// 포인터와 배열은 밀접한 관계
// Pointers and arrays are closely related
int arr[5] = {10, 20, 30, 40, 50};

// 다음은 모두 같은 의미
arr[2]      // 30
*(arr + 2)  // 30
*(2 + arr)  // 30
2[arr]      // 30 (문법적으로 유효하지만 권장하지 않음!)

// 배열 이름은 포인터처럼 동작
int* ptr = arr;     // &arr[0]과 동일
ptr[2] = 100;       // arr[2] = 100과 동일
```

### 2.2.5 다중 포인터 (포인터의 포인터)

```c
// 이중 포인터
// Double pointer
int value = 100;
int* ptr = &value;    // value를 가리킴
int** pptr = &ptr;    // ptr을 가리킴

printf("value: %d\n", value);      // 100
printf("*ptr: %d\n", *ptr);        // 100
printf("**pptr: %d\n", **pptr);    // 100

**pptr = 200;
printf("value: %d\n", value);      // 200
```

```
메모리 레이아웃:
┌─────────────┬─────────────┬─────────────┐
│ value = 200 │ ptr = 0x1000│ pptr=0x1008 │
│ 주소: 0x1000│ 주소: 0x1008│ 주소: 0x1010│
└─────────────┴─────────────┴─────────────┘
       ↑            ↑              │
       │            └──────────────┘
       └─────────────────────────────────────
```

### 다중 포인터가 필요한 이유

함수에서 포인터 자체를 변경해야 할 때 사용합니다:

```c
// 메모리 할당 함수 예시
// Memory allocation function example
NTSTATUS AllocateBuffer(PVOID* Buffer, SIZE_T Size)
{
    *Buffer = ExAllocatePoolWithTag(NonPagedPool, Size, 'fubA');
    if (*Buffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }
    return STATUS_SUCCESS;
}

// 사용
// Usage
PVOID myBuffer = NULL;
NTSTATUS status = AllocateBuffer(&myBuffer, 1024);
// myBuffer가 할당된 메모리를 가리킴
```

```csharp
// C# 대응: out 매개변수
// C# equivalent: out parameter
bool AllocateBuffer(out byte[] buffer, int size)
{
    buffer = new byte[size];
    return true;
}
```

### 2.2.6 void 포인터

`void*`는 타입이 정해지지 않은 범용 포인터입니다:

```c
// void 포인터
// Void pointer
void* generic = NULL;
int intValue = 42;
float floatValue = 3.14f;

generic = &intValue;
printf("int: %d\n", *(int*)generic);  // 캐스팅 필요

generic = &floatValue;
printf("float: %f\n", *(float*)generic);

// 커널에서 PVOID는 void*
// PVOID in kernel is void*
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, 'fubA');
PUCHAR byteBuffer = (PUCHAR)buffer;  // UCHAR*로 캐스팅
```

---

## 2.3 구조체와 공용체

### 2.3.1 구조체 정의

```c
// 구조체 정의 방법 1: 기본
// Structure definition method 1: Basic
struct Point {
    int X;
    int Y;
};

struct Point pt1;  // 사용 시 struct 키워드 필요
pt1.X = 10;
pt1.Y = 20;

// 구조체 정의 방법 2: typedef 사용 (권장)
// Method 2: Using typedef (recommended)
typedef struct _POINT {
    int X;
    int Y;
} POINT, *PPOINT;  // POINT는 구조체, PPOINT는 포인터 타입

POINT pt2;         // struct 키워드 없이 사용
PPOINT pPt = &pt2; // 포인터
```

```csharp
// C# 대응
// C# equivalent
public struct Point
{
    public int X;
    public int Y;
}
```

### 2.3.2 구조체 멤버 접근

```c
// 구조체 멤버 접근
// Accessing structure members
typedef struct _MY_DATA {
    ULONG Id;
    WCHAR Name[64];
    BOOLEAN IsActive;
} MY_DATA, *PMY_DATA;

// 직접 접근 (. 연산자)
MY_DATA data;
data.Id = 1;
wcscpy_s(data.Name, 64, L"Test");
data.IsActive = TRUE;

// 포인터를 통한 접근 (-> 연산자)
PMY_DATA pData = &data;
pData->Id = 2;                    // (*pData).Id = 2와 동일
wcscpy_s(pData->Name, 64, L"Test2");
pData->IsActive = FALSE;
```

### 2.3.3 구조체 메모리 레이아웃과 패딩

컴파일러는 성능 최적화를 위해 구조체 멤버 사이에 패딩을 추가합니다:

```c
// 패딩 예시
// Padding example
typedef struct _BAD_LAYOUT {
    UCHAR  a;    // 1 byte
    // 3 bytes 패딩
    ULONG  b;    // 4 bytes (4바이트 경계에 정렬)
    UCHAR  c;    // 1 byte
    // 3 bytes 패딩
} BAD_LAYOUT;    // 총 12 bytes!

typedef struct _GOOD_LAYOUT {
    ULONG  b;    // 4 bytes
    UCHAR  a;    // 1 byte
    UCHAR  c;    // 1 byte
    // 2 bytes 패딩
} GOOD_LAYOUT;   // 총 8 bytes
```

```
BAD_LAYOUT 메모리:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ a │pad│pad│pad│   b (4 bytes)   │ c │pad│pad│pad│
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

GOOD_LAYOUT 메모리:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│   b (4 bytes)   │ a │ c │pad│pad│
└───┴───┴───┴───┴───┴───┴───┴───┘
```

### 패딩 제어

```c
// #pragma pack으로 패딩 제어
// Control padding with #pragma pack
#pragma pack(push, 1)  // 1바이트 정렬
typedef struct _PACKED_DATA {
    UCHAR  a;    // 1 byte
    ULONG  b;    // 4 bytes (패딩 없음)
    UCHAR  c;    // 1 byte
} PACKED_DATA;   // 총 6 bytes
#pragma pack(pop)      // 이전 정렬로 복원
```

> ⚠️ **주의**: 패딩을 제거하면 메모리 정렬이 깨져 성능이 저하되거나 일부 아키텍처에서 오류가 발생할 수 있습니다.

### 2.3.4 공용체 (Union)

공용체는 같은 메모리 공간을 여러 타입으로 해석할 때 사용합니다:

```c
// 공용체 정의
// Union definition
typedef union _VALUE_UNION {
    ULONG  AsUlong;      // 4바이트 정수로 해석
    UCHAR  AsBytes[4];   // 4개의 바이트로 해석
    struct {
        USHORT Low;
        USHORT High;
    } Parts;             // 2개의 USHORT로 해석
} VALUE_UNION;

// 사용 예
// Usage example
VALUE_UNION value;
value.AsUlong = 0x12345678;

printf("AsUlong: 0x%08X\n", value.AsUlong);  // 0x12345678
printf("Bytes: %02X %02X %02X %02X\n",
    value.AsBytes[0], value.AsBytes[1],
    value.AsBytes[2], value.AsBytes[3]);     // 78 56 34 12 (리틀 엔디안)
printf("Low: 0x%04X, High: 0x%04X\n",
    value.Parts.Low, value.Parts.High);       // 0x5678, 0x1234
```

```
메모리 레이아웃 (4 bytes 공유):
┌────────┬────────┬────────┬────────┐
│  0x78  │  0x56  │  0x34  │  0x12  │
├────────┴────────┴────────┴────────┤
│        AsUlong = 0x12345678       │
├────────┬────────┬────────┬────────┤
│Bytes[0]│Bytes[1]│Bytes[2]│Bytes[3]│
├────────┴────────┼────────┴────────┤
│   Low = 0x5678  │  High = 0x1234  │
└─────────────────┴─────────────────┘
```

### 커널에서의 공용체 활용

```c
// WDK의 LARGE_INTEGER 예시
// LARGE_INTEGER example from WDK
typedef union _LARGE_INTEGER {
    struct {
        ULONG LowPart;
        LONG  HighPart;
    };
    LONGLONG QuadPart;
} LARGE_INTEGER;

// 64비트 값을 32비트 부분으로 접근 가능
// Access 64-bit value as 32-bit parts
LARGE_INTEGER fileSize;
fileSize.QuadPart = 0x0000000100000000LL;  // 4GB
printf("Low: %u, High: %d\n", fileSize.LowPart, fileSize.HighPart);
```

---

## 2.4 전처리기

### 2.4.1 매크로 상수

```c
// 상수 정의
// Constant definition
#define MAX_PATH 260
#define DRIVER_TAG 'tliF'  // 'Filt' 역순

// 사용
// Usage
WCHAR path[MAX_PATH];
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, DRIVER_TAG);
```

```csharp
// C# 대응
// C# equivalent
const int MaxPath = 260;
const uint DriverTag = 0x746C6946;  // 'Filt'
```

### 2.4.2 함수형 매크로

```c
// 함수형 매크로
// Function-like macro
#define MAX(a, b) ((a) > (b) ? (a) : (b))
#define MIN(a, b) ((a) < (b) ? (a) : (b))
#define ALIGN_UP(x, align) (((x) + (align) - 1) & ~((align) - 1))

// 사용
int larger = MAX(10, 20);  // 20
SIZE_T aligned = ALIGN_UP(100, 16);  // 112 (16의 배수로 올림)
```

> ⚠️ **주의**: 매크로는 단순 텍스트 치환이므로 부작용에 주의해야 합니다.

```c
// 매크로 부작용 예시
// Macro side effect example
#define SQUARE(x) ((x) * (x))

int a = 5;
int result = SQUARE(a++);  // ((a++) * (a++)) - 예상과 다른 결과!
// a는 7이 되고, result는 플랫폼에 따라 다름 (정의되지 않은 동작)
```

### 2.4.3 조건부 컴파일

```c
// 조건부 컴파일
// Conditional compilation

// DEBUG 빌드에서만 포함
// Include only in DEBUG build
#ifdef DBG
    #define DEBUG_PRINT(fmt, ...) DbgPrint(fmt, ##__VA_ARGS__)
#else
    #define DEBUG_PRINT(fmt, ...)  // 아무것도 안 함
#endif

// 헤더 가드
// Header guard
#ifndef _MY_HEADER_H_
#define _MY_HEADER_H_

// 헤더 내용
typedef struct _MY_STRUCT {
    int value;
} MY_STRUCT;

#endif // _MY_HEADER_H_

// 또는 pragma once 사용 (현대적 방식)
#pragma once
```

### 2.4.4 WDK에서 자주 사용하는 매크로

```c
// NT_SUCCESS: NTSTATUS 성공 여부 검사
// NT_SUCCESS: Check NTSTATUS success
#define NT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)

NTSTATUS status = SomeFunction();
if (NT_SUCCESS(status)) {
    // 성공
}

// PAGED_CODE: 페이지 가능 코드임을 표시
// PAGED_CODE: Indicate pageable code
VOID MyPagedFunction()
{
    PAGED_CODE();  // IRQL이 APC_LEVEL 이하인지 확인

    // 페이지 아웃 가능한 코드
}

// CONTAINING_RECORD: 멤버로부터 구조체 주소 계산
// CONTAINING_RECORD: Calculate structure address from member
typedef struct _MY_ITEM {
    LIST_ENTRY ListEntry;  // 연결 리스트 엔트리
    ULONG Data;
} MY_ITEM, *PMY_ITEM;

// ListEntry 주소로부터 MY_ITEM 주소 계산
PMY_ITEM item = CONTAINING_RECORD(
    listEntryPtr,    // 멤버 주소
    MY_ITEM,         // 구조체 타입
    ListEntry        // 멤버 이름
);
```

---

## 2.5 함수 포인터와 콜백

### 2.5.1 함수 포인터란

함수도 메모리에 있으므로, 함수의 주소를 포인터로 저장할 수 있습니다:

```c
// 함수 정의
// Function definition
int Add(int a, int b)
{
    return a + b;
}

int Subtract(int a, int b)
{
    return a - b;
}

// 함수 포인터 선언
// Function pointer declaration
int (*operation)(int, int);  // int 2개를 받아 int를 반환하는 함수 포인터

// 함수 포인터 사용
// Using function pointer
operation = Add;
int result1 = operation(10, 5);  // 15

operation = Subtract;
int result2 = operation(10, 5);  // 5
```

### 2.5.2 typedef로 함수 포인터 타입 정의

```c
// 함수 포인터 타입 정의
// Define function pointer type
typedef int (*BINARY_OPERATION)(int a, int b);

// 함수
int Add(int a, int b) { return a + b; }
int Multiply(int a, int b) { return a * b; }

// 사용
BINARY_OPERATION ops[] = { Add, Multiply };
int sum = ops[0](3, 4);      // 7
int product = ops[1](3, 4);  // 12
```

```csharp
// C# 대응: delegate
// C# equivalent: delegate
delegate int BinaryOperation(int a, int b);

int Add(int a, int b) => a + b;
int Multiply(int a, int b) => a * b;

BinaryOperation[] ops = { Add, Multiply };
int sum = ops[0](3, 4);      // 7
int product = ops[1](3, 4);  // 12
```

### 2.5.3 콜백 패턴

커널 드라이버에서 콜백은 핵심 개념입니다:

```c
// 콜백 함수 타입 정의
// Callback function type definition
typedef NTSTATUS (*PFN_PROCESS_FILE)(
    PUNICODE_STRING FileName,
    PVOID Context
);

// 콜백을 받는 함수
// Function that accepts callback
NTSTATUS EnumerateFiles(
    PUNICODE_STRING Directory,
    PFN_PROCESS_FILE Callback,
    PVOID Context
)
{
    UNICODE_STRING fileName;
    // ... 파일 열거 ...

    // 각 파일에 대해 콜백 호출
    // Call callback for each file
    NTSTATUS status = Callback(&fileName, Context);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    return STATUS_SUCCESS;
}

// 콜백 함수 구현
// Callback function implementation
NTSTATUS MyFileProcessor(
    PUNICODE_STRING FileName,
    PVOID Context
)
{
    DbgPrint("Processing: %wZ\n", FileName);
    return STATUS_SUCCESS;
}

// 사용
// Usage
EnumerateFiles(&directory, MyFileProcessor, NULL);
```

### 2.5.4 Minifilter 콜백 예시

실제 Minifilter에서의 콜백 구조:

```c
// Minifilter Pre-Operation 콜백 타입
// Minifilter Pre-Operation callback type
typedef FLT_PREOP_CALLBACK_STATUS
(*PFLT_PRE_OPERATION_CALLBACK)(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
);

// 구현
// Implementation
FLT_PREOP_CALLBACK_STATUS
MyPreCreateCallback(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    // 파일 생성/열기 요청 처리
    // Handle file create/open request
    UNREFERENCED_PARAMETER(CompletionContext);

    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    NTSTATUS status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED,
        &nameInfo
    );

    if (NT_SUCCESS(status)) {
        DbgPrint("PreCreate: %wZ\n", &nameInfo->Name);
        FltReleaseFileNameInformation(nameInfo);
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 등록
// Registration
const FLT_OPERATION_REGISTRATION Callbacks[] = {
    { IRP_MJ_CREATE, 0, MyPreCreateCallback, NULL },
    { IRP_MJ_OPERATION_END }
};
```

---

## 정리

이 장에서 배운 핵심 개념:

| 개념 | C | C# 대응 |
|------|---|---------|
| **기본 타입** | `int`, `long`, `char` | `int`, `int`(!), `sbyte` |
| **Windows 타입** | `ULONG`, `NTSTATUS`, `PVOID` | `uint`, `int`, `IntPtr` |
| **포인터** | `int*`, `void*` | `ref`/`out`, `IntPtr` |
| **구조체** | `typedef struct` | `struct` |
| **공용체** | `union` | `[StructLayout]` + `FieldOffset` |
| **매크로** | `#define` | `const`, 메서드 |
| **함수 포인터** | `typedef ... (*)(...)` | `delegate`, `Func<>` |

### 다음 장 미리보기

Chapter 3에서는 메모리 관리를 심도 있게 다룹니다:
- 스택과 힙의 차이
- 커널 모드 메모리 풀
- 메모리 버그와 방지 패턴

---

## 연습 문제

### 1. 포인터

다음 코드의 출력은?

```c
int arr[] = {10, 20, 30, 40, 50};
int* p = arr + 2;
printf("%d, %d, %d\n", *p, *(p-1), *(p+2));
```

### 2. 구조체 크기

다음 구조체의 크기는? (64-bit Windows, 기본 정렬)

```c
typedef struct _DATA {
    UCHAR  a;
    ULONG  b;
    USHORT c;
    UCHAR  d;
} DATA;
```

### 3. 함수 포인터

다음 콜백 타입을 사용하는 `ForEach` 함수를 완성하세요:

```c
typedef void (*ITEM_CALLBACK)(int item, void* context);

void ForEach(int* array, int count, ITEM_CALLBACK callback, void* context)
{
    // 구현
}
```

---

## 참고 자료

- [C Reference](https://en.cppreference.com/w/c)
- [Windows Data Types](https://docs.microsoft.com/en-us/windows/win32/winprog/windows-data-types)
- [Kernel-Mode Driver Architecture](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/)
