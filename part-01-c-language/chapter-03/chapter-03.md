# Chapter 3: 메모리 관리의 모든 것

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표

- 스택과 힙 메모리의 차이 이해
- 사용자 모드 동적 메모리 할당 (malloc/free)
- 커널 모드 메모리 할당 (ExAllocatePool)
- 메모리 풀과 태깅 개념 습득
- 일반적인 메모리 버그 패턴 인식

---

## 들어가며

.NET 개발자에게 메모리 관리는 GC가 알아서 해주는 영역이었습니다. 하지만 C와 커널 드라이버의 세계에서는 **모든 메모리를 직접 관리**해야 합니다.

이 장에서는 메모리가 어디에 어떻게 할당되는지, 그리고 올바르게 해제하는 방법을 배웁니다. 메모리 관리 실수는 커널에서 BSOD로 직결되므로, 이 장의 내용을 확실히 이해해야 합니다.

---

## 3.1 스택과 힙

프로그램의 메모리는 크게 **스택(Stack)**과 **힙(Heap)**으로 나뉩니다.

### 3.1.1 스택 메모리

스택은 함수 호출과 지역 변수를 위한 메모리 영역입니다:

```
스택 구조 (높은 주소 → 낮은 주소로 성장)
┌─────────────────────────┐ ← 높은 주소 (스택 바닥)
│   main() 스택 프레임     │
│   - 지역 변수            │
│   - 반환 주소            │
├─────────────────────────┤
│   FunctionA() 스택 프레임│
│   - 매개변수             │
│   - 지역 변수            │
│   - 반환 주소            │
├─────────────────────────┤
│   FunctionB() 스택 프레임│
│   - 매개변수             │
│   - 지역 변수            │
│   - 반환 주소            │
├─────────────────────────┤
│         ...              │
│    (스택 성장 방향 ↓)     │
└─────────────────────────┘ ← 낮은 주소 (스택 탑)
```

```c
// 스택 메모리 사용 예시
// Stack memory usage example
void ProcessData()
{
    int counter = 0;           // 4 bytes - 스택
    char buffer[256];          // 256 bytes - 스택
    MY_STRUCT data;            // 구조체 크기만큼 - 스택

    // 함수가 반환되면 자동으로 모든 지역 변수 해제
    // All local variables are automatically freed when function returns
}
```

### 스택의 특징

| 특성 | 설명 |
|------|------|
| 할당 속도 | 매우 빠름 (스택 포인터 이동만) |
| 크기 제한 | 작음 (기본 1MB, 커널은 더 작음) |
| 관리 | 자동 (함수 종료 시 해제) |
| 수명 | 함수 범위 내 |
| 접근 패턴 | LIFO (Last In, First Out) |

### 스택 오버플로우

스택 크기를 초과하면 스택 오버플로우가 발생합니다:

```c
// 스택 오버플로우 예시 - 하지 마세요!
// Stack overflow example - Don't do this!
void DangerousFunction()
{
    char hugeBuffer[1024 * 1024];  // 1MB - 스택을 초과할 수 있음!
    // ...
}

// 재귀 호출도 스택 오버플로우의 원인
// Recursive calls can also cause stack overflow
void InfiniteRecursion()
{
    InfiniteRecursion();  // 결국 스택 오버플로우
}
```

> ⚠️ **커널 주의**: 커널 스택은 사용자 모드보다 훨씬 작습니다 (x64: 약 24KB). 큰 지역 변수는 힙에 할당하세요.

### 3.1.2 힙 메모리

힙은 동적으로 할당되는 메모리 영역입니다:

```c
// 힙 메모리 사용 예시
// Heap memory usage example
void ProcessData()
{
    // 힙에 메모리 할당
    // Allocate memory on heap
    int* numbers = (int*)malloc(sizeof(int) * 1000);
    if (numbers == NULL) {
        return;  // 할당 실패
    }

    // 사용
    for (int i = 0; i < 1000; i++) {
        numbers[i] = i * 2;
    }

    // 명시적으로 해제해야 함!
    // Must explicitly free!
    free(numbers);
    numbers = NULL;  // 해제 후 NULL 설정 권장
}
```

### 힙의 특징

| 특성 | 설명 |
|------|------|
| 할당 속도 | 상대적으로 느림 (메모리 관리 필요) |
| 크기 제한 | 큼 (가용 메모리만큼) |
| 관리 | 수동 (명시적 할당/해제) |
| 수명 | 해제할 때까지 유지 |
| 접근 패턴 | 임의 접근 가능 |

### 3.1.3 스택 vs 힙 비교

```c
// 스택 사용 - 작은 고정 크기 데이터
// Stack usage - Small fixed-size data
void UseStack()
{
    int value = 42;
    char name[32] = "Hello";
    MY_SMALL_STRUCT data = { 0 };
}

// 힙 사용 - 큰 데이터 또는 동적 크기
// Heap usage - Large data or dynamic size
void UseHeap(SIZE_T size)
{
    void* buffer = malloc(size);  // 런타임에 크기 결정
    if (buffer != NULL) {
        // 사용
        free(buffer);
    }
}
```

```csharp
// C# 비교
// C# comparison
void CSharpExample()
{
    // 값 타입 (struct) - 스택 (작은 경우)
    int value = 42;
    Point point = new Point(10, 20);

    // 참조 타입 (class) - 힙
    var list = new List<int>();  // 힙에 할당, GC가 관리
}
```

---

## 3.2 사용자 모드 동적 메모리 할당

### 3.2.1 malloc과 free

C 표준 라이브러리의 메모리 함수:

```c
// malloc: 메모리 할당
// malloc: Allocate memory
void* malloc(size_t size);

// free: 메모리 해제
// free: Free memory
void free(void* ptr);

// 사용 예
// Usage example
#include <stdlib.h>

int* CreateArray(size_t count)
{
    // 메모리 할당 (초기화되지 않음)
    // Allocate memory (uninitialized)
    int* array = (int*)malloc(sizeof(int) * count);

    if (array == NULL) {
        // 할당 실패 처리
        // Handle allocation failure
        return NULL;
    }

    // 0으로 초기화 (선택)
    // Initialize to 0 (optional)
    memset(array, 0, sizeof(int) * count);

    return array;
}

void DestroyArray(int* array)
{
    if (array != NULL) {
        free(array);
    }
}
```

### 3.2.2 calloc과 realloc

```c
// calloc: 0으로 초기화된 메모리 할당
// calloc: Allocate zero-initialized memory
void* calloc(size_t count, size_t size);

// realloc: 메모리 크기 변경
// realloc: Resize memory
void* realloc(void* ptr, size_t new_size);

// calloc 예
// calloc example
int* array = (int*)calloc(100, sizeof(int));  // 100개 int, 0으로 초기화

// realloc 예
// realloc example
int* newArray = (int*)realloc(array, sizeof(int) * 200);
if (newArray != NULL) {
    array = newArray;  // 성공 시 새 포인터 사용
} else {
    // 실패 시 원래 array는 여전히 유효
    // On failure, original array is still valid
}
```

> ⚠️ **realloc 주의**: 실패 시 원래 메모리가 해제되지 않으므로, 결과를 직접 원래 변수에 대입하지 마세요.

---

## 3.3 커널 모드 메모리 관리

커널에서는 `malloc`/`free`를 사용할 수 없습니다. 대신 전용 API를 사용합니다.

### 3.3.1 메모리 풀 개념

커널은 메모리를 **풀(Pool)**이라는 영역에서 할당합니다:

```
Windows 커널 메모리 풀
┌──────────────────────────────────────────┐
│              Non-Paged Pool               │
│  - 항상 물리 메모리에 존재                 │
│  - 페이지 아웃되지 않음                   │
│  - DISPATCH_LEVEL 이상에서 접근 가능      │
│  - 크기가 제한됨                          │
├──────────────────────────────────────────┤
│              Paged Pool                   │
│  - 페이지 아웃 가능 (디스크로)            │
│  - PASSIVE_LEVEL에서만 접근               │
│  - 더 큰 할당 가능                        │
│  - 대부분의 할당에 적합                   │
└──────────────────────────────────────────┘
```

### 언제 어떤 풀을 사용할까?

| 상황 | 권장 풀 |
|------|---------|
| DPC, ISR에서 접근하는 데이터 | NonPagedPool |
| 스핀락으로 보호되는 데이터 | NonPagedPool |
| 일반적인 데이터 구조 | PagedPool |
| 파일 경로, 문자열 | PagedPool |
| I/O 버퍼 | NonPagedPool (대부분) |

### 3.3.2 ExAllocatePoolWithTag (레거시)

```c
// ExAllocatePoolWithTag: 메모리 풀에서 할당
// ExAllocatePoolWithTag: Allocate from memory pool
PVOID ExAllocatePoolWithTag(
    POOL_TYPE PoolType,    // 풀 타입
    SIZE_T NumberOfBytes,  // 크기
    ULONG Tag              // 태그 (디버깅용)
);

// 풀 태그: 4바이트 ASCII 문자
// Pool tag: 4-byte ASCII characters
#define MY_POOL_TAG 'tliF'  // 'Filt' 역순으로 보임

// 사용 예
// Usage example
PVOID buffer = ExAllocatePoolWithTag(
    NonPagedPool,   // 또는 PagedPool
    1024,           // 1024 바이트
    MY_POOL_TAG     // 태그
);

if (buffer == NULL) {
    return STATUS_INSUFFICIENT_RESOURCES;
}

// 사용...

// 해제
// Free
ExFreePoolWithTag(buffer, MY_POOL_TAG);
```

### 3.3.3 ExAllocatePool2 (권장, Windows 10 2004+)

최신 Windows에서는 `ExAllocatePool2`를 권장합니다:

```c
// ExAllocatePool2: 현대적인 풀 할당 API
// ExAllocatePool2: Modern pool allocation API
PVOID ExAllocatePool2(
    POOL_FLAGS Flags,      // 플래그
    SIZE_T NumberOfBytes,  // 크기
    ULONG Tag              // 태그
);

// 플래그 예시
// Flag examples
POOL_FLAG_NON_PAGED           // NonPagedPool과 동일
POOL_FLAG_PAGED               // PagedPool과 동일
POOL_FLAG_NON_PAGED_EXECUTE   // 실행 가능 NonPaged
POOL_FLAG_UNINITIALIZED       // 초기화하지 않음 (성능용)

// 사용 예
// Usage example
PVOID buffer = ExAllocatePool2(
    POOL_FLAG_NON_PAGED,  // Non-paged pool
    1024,                  // 1024 바이트
    MY_POOL_TAG           // 태그
);

if (buffer == NULL) {
    return STATUS_INSUFFICIENT_RESOURCES;
}

// 해제 (ExFreePoolWithTag 동일하게 사용)
ExFreePoolWithTag(buffer, MY_POOL_TAG);
```

### 3.3.4 풀 태그의 중요성

풀 태그는 메모리 누수 추적에 필수적입니다:

```c
// 다양한 용도별 태그 정의
// Define tags for different purposes
#define TAG_BUFFER   'fubM'   // 'Mbuf' - 일반 버퍼
#define TAG_FILENAME 'maNF'   // 'FNam' - 파일 이름
#define TAG_CONTEXT  'txoC'   // 'Coxt' - 컨텍스트

// WinDbg에서 태그로 메모리 검색
// Search memory by tag in WinDbg
// !poolfind Mbuf
// !poolused 2 Mbuf
```

```
WinDbg 출력 예시:
0: kd> !poolused 2 Mbuf
   Tag    Paged   NonPaged     Used
  Mbuf     0.00    128.00    128.00
```

---

## 3.4 메모리 버그 패턴

커널에서의 메모리 버그는 BSOD로 이어집니다. 주요 패턴을 알아봅시다.

### 3.4.1 메모리 누수 (Memory Leak)

```c
// 메모리 누수 예시
// Memory leak example
NTSTATUS LeakyFunction()
{
    PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 1024, 'kaeL');

    if (SomeCondition()) {
        return STATUS_UNSUCCESSFUL;  // buffer 해제 없이 반환!
    }

    // ... 정상 경로에서만 해제
    ExFreePoolWithTag(buffer, 'kaeL');
    return STATUS_SUCCESS;
}
```

**해결 방법**: goto를 사용한 cleanup 패턴

```c
// 올바른 패턴
// Correct pattern
NTSTATUS SafeFunction()
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID buffer = NULL;

    buffer = ExAllocatePoolWithTag(NonPagedPool, 1024, 'efaS');
    if (buffer == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    if (SomeCondition()) {
        status = STATUS_UNSUCCESSFUL;
        goto Cleanup;  // Cleanup으로 점프
    }

    // 정상 처리
    status = STATUS_SUCCESS;

Cleanup:
    if (buffer != NULL) {
        ExFreePoolWithTag(buffer, 'efaS');
    }
    return status;
}
```

### 3.4.2 해제 후 사용 (Use After Free)

```c
// 해제 후 사용 예시 - 위험!
// Use after free example - Dangerous!
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, 'fAU');
// ... 사용 ...
ExFreePoolWithTag(buffer, 'fAU');

// 해제 후 접근 - 크래시 또는 데이터 손상!
// Access after free - Crash or data corruption!
RtlZeroMemory(buffer, 100);  // 정의되지 않은 동작
```

**해결 방법**: 해제 후 NULL 설정

```c
// 올바른 패턴
// Correct pattern
ExFreePoolWithTag(buffer, 'fAU');
buffer = NULL;  // 해제 후 즉시 NULL 설정

// 이후 실수로 접근해도 NULL 체크로 방어 가능
if (buffer != NULL) {
    RtlZeroMemory(buffer, 100);
}
```

### 3.4.3 이중 해제 (Double Free)

```c
// 이중 해제 예시 - 크래시!
// Double free example - Crash!
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, 'fD');
ExFreePoolWithTag(buffer, 'fD');   // 첫 번째 해제
ExFreePoolWithTag(buffer, 'fD');   // 두 번째 해제 - BSOD!
```

**해결 방법**: NULL 체크와 NULL 설정

```c
// 안전한 해제 함수
// Safe free function
void SafeFree(PVOID* Buffer, ULONG Tag)
{
    if (*Buffer != NULL) {
        ExFreePoolWithTag(*Buffer, Tag);
        *Buffer = NULL;
    }
}

// 사용
// Usage
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, 'fD');
SafeFree(&buffer, 'fD');  // buffer가 NULL로 설정됨
SafeFree(&buffer, 'fD');  // NULL이므로 아무 일도 안 함 - 안전
```

### 3.4.4 버퍼 오버플로우

```c
// 버퍼 오버플로우 예시
// Buffer overflow example
char buffer[10];
strcpy(buffer, "This string is too long for the buffer");
// 메모리 손상 발생!
```

**해결 방법**: 안전한 문자열 함수 사용

```c
// 안전한 함수 사용
// Use safe functions
char buffer[10];
strcpy_s(buffer, sizeof(buffer), "Hello");  // 크기 제한됨

// 커널에서
// In kernel
WCHAR buffer[256];
RtlStringCchCopyW(buffer, RTL_NUMBER_OF(buffer), L"Hello");
```

### 3.4.5 초기화되지 않은 메모리 사용

```c
// 초기화되지 않은 메모리 사용
// Using uninitialized memory
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, 'tinU');
// buffer 내용은 정의되지 않음!

if (((PULONG)buffer)[0] == 0) {  // 예측 불가능한 결과
    // ...
}
```

**해결 방법**: 명시적 초기화

```c
// 명시적 초기화
// Explicit initialization
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, 'tinI');
if (buffer != NULL) {
    RtlZeroMemory(buffer, 100);  // 0으로 초기화
}

// 또는 calloc 스타일 할당
// Or calloc-style allocation
PVOID buffer = ExAllocatePoolZero(NonPagedPool, 100, 'tinI');  // 0으로 초기화됨
```

---

## 3.5 안전한 메모리 관리 패턴

### 3.5.1 표준 할당/해제 패턴

```c
// 안전한 메모리 관리 템플릿
// Safe memory management template
NTSTATUS MyFunction()
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID buffer1 = NULL;
    PVOID buffer2 = NULL;
    HANDLE handle = NULL;

    // 리소스 할당
    // Allocate resources
    buffer1 = ExAllocatePoolWithTag(PagedPool, 100, 'fuB1');
    if (buffer1 == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    buffer2 = ExAllocatePoolWithTag(PagedPool, 200, 'fuB2');
    if (buffer2 == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    status = OpenSomeHandle(&handle);
    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // 작업 수행
    // Do work
    status = DoActualWork(buffer1, buffer2, handle);

Cleanup:
    // 역순으로 정리
    // Cleanup in reverse order
    if (handle != NULL) {
        ZwClose(handle);
    }
    if (buffer2 != NULL) {
        ExFreePoolWithTag(buffer2, 'fuB2');
    }
    if (buffer1 != NULL) {
        ExFreePoolWithTag(buffer1, 'fuB1');
    }

    return status;
}
```

### 3.5.2 컨텍스트 구조체 패턴

관련 리소스를 구조체로 묶으면 관리가 쉬워집니다:

```c
// 리소스 컨텍스트 구조체
// Resource context structure
typedef struct _MY_CONTEXT {
    PVOID Buffer;
    SIZE_T BufferSize;
    HANDLE FileHandle;
    PFILE_OBJECT FileObject;
    BOOLEAN Initialized;
} MY_CONTEXT, *PMY_CONTEXT;

#define CONTEXT_TAG 'txCM'

// 초기화 함수
// Initialization function
NTSTATUS InitializeContext(PMY_CONTEXT Context, SIZE_T BufferSize)
{
    RtlZeroMemory(Context, sizeof(MY_CONTEXT));

    Context->Buffer = ExAllocatePoolWithTag(PagedPool, BufferSize, CONTEXT_TAG);
    if (Context->Buffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    Context->BufferSize = BufferSize;
    Context->Initialized = TRUE;

    return STATUS_SUCCESS;
}

// 정리 함수
// Cleanup function
VOID CleanupContext(PMY_CONTEXT Context)
{
    if (!Context->Initialized) {
        return;
    }

    if (Context->FileObject != NULL) {
        ObDereferenceObject(Context->FileObject);
        Context->FileObject = NULL;
    }

    if (Context->FileHandle != NULL) {
        ZwClose(Context->FileHandle);
        Context->FileHandle = NULL;
    }

    if (Context->Buffer != NULL) {
        ExFreePoolWithTag(Context->Buffer, CONTEXT_TAG);
        Context->Buffer = NULL;
    }

    Context->Initialized = FALSE;
}
```

### 3.5.3 RAII 스타일 (C++ 한정)

커널 드라이버에서 C++을 사용할 수 있다면:

```cpp
// C++ RAII 패턴 (커널 C++)
// C++ RAII pattern (Kernel C++)
class PoolBuffer {
private:
    PVOID m_buffer;
    ULONG m_tag;

public:
    PoolBuffer(SIZE_T size, ULONG tag) : m_buffer(nullptr), m_tag(tag) {
        m_buffer = ExAllocatePoolWithTag(PagedPool, size, tag);
    }

    ~PoolBuffer() {
        if (m_buffer != nullptr) {
            ExFreePoolWithTag(m_buffer, m_tag);
        }
    }

    // 복사 금지
    PoolBuffer(const PoolBuffer&) = delete;
    PoolBuffer& operator=(const PoolBuffer&) = delete;

    PVOID Get() const { return m_buffer; }
    bool IsValid() const { return m_buffer != nullptr; }
};

// 사용
NTSTATUS MyFunction()
{
    PoolBuffer buffer(1024, 'fuB1');
    if (!buffer.IsValid()) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // 사용...

    // 함수 종료 시 자동 해제
    return STATUS_SUCCESS;
}
```

---

## 3.6 Lookaside List (고급)

빈번한 할당/해제가 발생하는 경우, Lookaside List를 사용하면 성능이 향상됩니다:

```c
// Lookaside List 초기화
// Lookaside List initialization
NPAGED_LOOKASIDE_LIST gLookasideList;

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    // Lookaside List 초기화
    ExInitializeNPagedLookasideList(
        &gLookasideList,
        NULL,                  // Allocate 함수 (NULL = 기본)
        NULL,                  // Free 함수 (NULL = 기본)
        0,                     // 플래그
        sizeof(MY_STRUCTURE),  // 각 항목 크기
        'aLM',                 // 태그
        0                      // 깊이 (0 = 시스템 결정)
    );

    return STATUS_SUCCESS;
}

VOID DriverUnload(PDRIVER_OBJECT DriverObject)
{
    // Lookaside List 삭제
    ExDeleteNPagedLookasideList(&gLookasideList);
}

// 할당 및 해제
PMY_STRUCTURE AllocateItem()
{
    return (PMY_STRUCTURE)ExAllocateFromNPagedLookasideList(&gLookasideList);
}

VOID FreeItem(PMY_STRUCTURE Item)
{
    ExFreeToNPagedLookasideList(&gLookasideList, Item);
}
```

### Lookaside List의 장점

1. **빠른 할당/해제**: 풀에서 직접 할당하는 것보다 빠름
2. **메모리 단편화 감소**: 같은 크기의 블록만 관리
3. **캐시 효율성**: 자주 사용되는 메모리가 캐시에 유지

---

## 정리

### 메모리 영역 비교

| 영역 | 할당 | 해제 | 속도 | 크기 |
|------|------|------|------|------|
| 스택 | 자동 | 자동 | 매우 빠름 | 제한적 |
| 힙 (사용자) | malloc | free | 보통 | 큼 |
| NonPagedPool | ExAllocatePool* | ExFreePool* | 보통 | 제한적 |
| PagedPool | ExAllocatePool* | ExFreePool* | 보통 | 큼 |

### 메모리 버그 체크리스트

- [ ] 모든 할당에 NULL 체크가 있는가?
- [ ] 모든 경로에서 메모리가 해제되는가?
- [ ] 해제 후 포인터를 NULL로 설정하는가?
- [ ] 버퍼 크기를 초과하는 접근이 없는가?
- [ ] 초기화되지 않은 메모리를 사용하지 않는가?

### 다음 장 미리보기

Part 2에서는 Windows 커널 아키텍처를 다룹니다:
- 사용자 모드와 커널 모드
- Windows 커널 구성 요소
- 파일 시스템 아키텍처

---

## 연습 문제

### 1. 버그 찾기

다음 코드의 문제점은?

```c
NTSTATUS BuggyFunction()
{
    PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 100, 'guB');

    if (!Condition1()) {
        return STATUS_UNSUCCESSFUL;
    }

    ProcessBuffer(buffer);

    if (!Condition2()) {
        ExFreePoolWithTag(buffer, 'guB');
        return STATUS_UNSUCCESSFUL;
    }

    ExFreePoolWithTag(buffer, 'guB');
    return STATUS_SUCCESS;
}
```

### 2. 풀 선택

다음 상황에서 어떤 풀을 사용해야 할까요?

1. DPC에서 접근하는 연결 리스트 노드
2. 파일 전체 경로를 저장하는 버퍼
3. I/O 요청에 첨부할 완료 컨텍스트

### 3. 코드 개선

위 "버그 찾기" 문제의 코드를 올바르게 수정하세요.

---

## 참고 자료

- [Pool Memory Management](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/managing-memory-sections)
- [Lookaside Lists](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/using-lookaside-lists)
- [Memory Pool Monitor (poolmon.exe)](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/poolmon)
