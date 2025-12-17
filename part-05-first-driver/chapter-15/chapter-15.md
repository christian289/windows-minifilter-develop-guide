# Chapter 15: Hello World 커널 드라이버

## 난이도: ⭐⭐ (초급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- WDK 프로젝트를 생성하고 구성합니다
- DriverEntry와 DriverUnload 함수를 구현합니다
- 드라이버를 빌드하고 테스트 서명합니다
- 드라이버를 로드/언로드하고 WinDbg로 디버깅합니다
- 커널에서 디버그 메시지를 출력합니다

## 도입: 첫 발걸음

모든 프로그래밍 학습은 "Hello World"에서 시작합니다. 커널 드라이버도 마찬가지입니다. 이 챕터에서는 가장 간단한 커널 드라이버를 만들고, 전체 개발-빌드-테스트 과정을 경험합니다.

```csharp
// C# 콘솔 애플리케이션
class Program
{
    static void Main(string[] args)   // 진입점
    {
        Console.WriteLine("Hello World!");
    }
}

// 커널 드라이버
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, ...)  // 진입점
{
    DbgPrint("Hello World!\n");
    return STATUS_SUCCESS;
}
```

---

## 15.1 WDK 프로젝트 생성

### 15.1.1 Visual Studio에서 프로젝트 생성

1. Visual Studio 2022 실행
2. **파일 > 새로 만들기 > 프로젝트**
3. 템플릿 검색: **"Empty WDM Driver"** 선택
4. 프로젝트 이름: **HelloDriver**
5. 위치: 원하는 경로 지정
6. **만들기** 클릭

```
┌─────────────────────────────────────────────────────────────────────┐
│ Visual Studio 2022                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  새 프로젝트 만들기                                                 │
│                                                                      │
│  [검색: Empty WDM]                                                  │
│                                                                      │
│  ┌──────────────────────────────────────┐                          │
│  │ 📁 Empty WDM Driver                  │ ← 선택                   │
│  │    Windows Driver                    │                           │
│  │    C++                               │                           │
│  └──────────────────────────────────────┘                          │
│                                                                      │
│  프로젝트 이름: HelloDriver                                         │
│  위치: C:\Projects\                                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 15.1.2 프로젝트 구조

생성된 프로젝트 구조:

```
HelloDriver/
├── HelloDriver.sln           // 솔루션 파일
├── HelloDriver/
│   ├── HelloDriver.vcxproj   // 프로젝트 파일
│   ├── HelloDriver.inf       // 드라이버 설치 정보 (나중에 사용)
│   └── Driver.c              // 우리가 작성할 소스
└── x64/
    └── Debug/
        ├── HelloDriver.sys   // 빌드 결과물
        └── HelloDriver.pdb   // 디버그 심볼
```

### 15.1.3 프로젝트 속성 확인

프로젝트 우클릭 > **속성**:

```
┌─────────────────────────────────────────────────────────────────────┐
│ 구성 속성 > Driver Settings                                          │
├─────────────────────────────────────────────────────────────────────┤
│ Target OS Version:     Windows 10                                    │
│ Target Platform:       Desktop                                       │
│ Configuration Type:    Driver                                        │
│ Driver Type:          WDM                                            │
│ Kernel Mode:          Yes                                            │
├─────────────────────────────────────────────────────────────────────┤
│ 구성 속성 > C/C++ > General                                          │
├─────────────────────────────────────────────────────────────────────┤
│ Warning Level:        Level4 (/W4)                                   │
│ Treat Warnings As Errors: Yes (/WX)                                  │
├─────────────────────────────────────────────────────────────────────┤
│ 구성 속성 > Driver Signing                                           │
├─────────────────────────────────────────────────────────────────────┤
│ Sign Mode:            Test Sign                                      │
│ Test Certificate:     (자동 생성됨)                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 15.2 Driver.c 구현

### 15.2.1 기본 구조

```c
// Driver.c
// Hello World 커널 드라이버

#include <ntddk.h>

// 함수 선언 (forward declaration)
DRIVER_UNLOAD DriverUnload;

// 드라이버 진입점
NTSTATUS DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath
)
{
    UNREFERENCED_PARAMETER(RegistryPath);

    // 디버그 메시지 출력
    DbgPrint("HelloDriver: Hello World from Kernel!\n");
    // HelloDriver: 커널에서 Hello World!

    // 언로드 루틴 등록
    DriverObject->DriverUnload = DriverUnload;

    DbgPrint("HelloDriver: DriverEntry completed successfully.\n");
    // HelloDriver: DriverEntry 성공적으로 완료.

    return STATUS_SUCCESS;
}

// 드라이버 언로드 루틴
VOID DriverUnload(
    _In_ PDRIVER_OBJECT DriverObject
)
{
    UNREFERENCED_PARAMETER(DriverObject);

    DbgPrint("HelloDriver: Goodbye from Kernel!\n");
    // HelloDriver: 커널에서 안녕!
}
```

### 15.2.2 코드 상세 설명

```c
// 1. 헤더 파일
#include <ntddk.h>
// ntddk.h: WDM 드라이버를 위한 핵심 헤더
// - NTSTATUS, DRIVER_OBJECT 등 기본 타입 정의
// - DbgPrint, ExAllocatePoolWithTag 등 커널 API
// C#의 using System; 과 유사

// 2. SAL 주석 (Source Annotation Language)
NTSTATUS DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,    // _In_: 입력 매개변수
    _In_ PUNICODE_STRING RegistryPath    // 읽기만 함
)
// SAL 주석은 정적 분석 도구가 버그를 찾는 데 도움
// C#의 [NotNull] 특성과 유사한 개념

// 3. UNREFERENCED_PARAMETER 매크로
UNREFERENCED_PARAMETER(RegistryPath);
// "이 매개변수를 의도적으로 사용하지 않음"을 명시
// 컴파일러 경고 방지 (C4100: unreferenced formal parameter)

// 4. DRIVER_OBJECT 구조체
DriverObject->DriverUnload = DriverUnload;
// DriverObject: 드라이버 자체를 표현하는 구조체
// DriverUnload: 드라이버 언로드 시 호출될 함수 등록
// NULL이면 드라이버를 언로드할 수 없음!
```

### 15.2.3 DriverEntry 매개변수

```c
// DriverEntry의 두 매개변수

// 1. PDRIVER_OBJECT DriverObject
// - 드라이버를 대표하는 객체
// - 시스템이 생성하여 전달
// - 드라이버의 함수 포인터, 디바이스 목록 등 저장

typedef struct _DRIVER_OBJECT {
    PDEVICE_OBJECT DeviceObject;           // 디바이스 체인
    PUNICODE_STRING DriverName;            // 드라이버 이름
    PDRIVER_UNLOAD DriverUnload;           // 언로드 함수
    PDRIVER_DISPATCH MajorFunction[28];    // IRP 핸들러 배열
    // ... 기타 필드
} DRIVER_OBJECT;

// 2. PUNICODE_STRING RegistryPath
// - 드라이버의 레지스트리 키 경로
// - 예: \Registry\Machine\System\CurrentControlSet\Services\HelloDriver
// - 드라이버 설정을 저장/읽기하는 데 사용
```

### 15.2.4 반환 값: NTSTATUS

```c
// NTSTATUS: 커널 API의 표준 반환 타입

// 성공
return STATUS_SUCCESS;                    // 0x00000000

// 실패 예시
return STATUS_UNSUCCESSFUL;               // 0xC0000001
return STATUS_INSUFFICIENT_RESOURCES;     // 0xC000009A
return STATUS_INVALID_PARAMETER;          // 0xC000000D

// 상태 확인 매크로
if (NT_SUCCESS(status)) {
    // 성공
}
if (!NT_SUCCESS(status)) {
    // 실패
}

// C# 비유
// NTSTATUS ≈ bool 또는 Exception
// STATUS_SUCCESS ≈ return true; 또는 정상 반환
// STATUS_INVALID_PARAMETER ≈ throw new ArgumentException();
```

---

## 15.3 DbgPrint를 이용한 디버그 출력

### 15.3.1 DbgPrint 함수

```c
// DbgPrint: 커널 디버그 메시지 출력
// 형식 문자열 지원 (printf와 유사)

DbgPrint("Simple message\n");
DbgPrint("Integer: %d\n", 42);
DbgPrint("Pointer: %p\n", DriverObject);
DbgPrint("String: %s\n", "Hello");
DbgPrint("Unicode: %wZ\n", RegistryPath);  // UNICODE_STRING

// 권장: 드라이버 이름 접두사 추가
DbgPrint("HelloDriver: Message here\n");
// 여러 드라이버가 동시에 메시지를 출력할 때 구분 가능
```

### 15.3.2 DbgPrintEx - 필터링 가능한 출력

```c
// DbgPrintEx: 컴포넌트 ID와 레벨 지정 가능

DbgPrintEx(
    DPFLTR_IHVDRIVER_ID,      // 컴포넌트 ID (IHV Driver)
    DPFLTR_INFO_LEVEL,        // 레벨 (Info)
    "HelloDriver: Info message\n"
);

// 레벨:
// DPFLTR_ERROR_LEVEL   (0) - 항상 출력
// DPFLTR_WARNING_LEVEL (1) - 경고
// DPFLTR_TRACE_LEVEL   (2) - 추적
// DPFLTR_INFO_LEVEL    (3) - 정보

// 필터링은 WinDbg 또는 레지스트리에서 설정
```

### 15.3.3 KdPrint vs DbgPrint

```c
// KdPrint: Debug 빌드에서만 출력
// Release 빌드에서는 자동으로 제거됨

#if DBG
#define KdPrint(x) DbgPrint x
#else
#define KdPrint(x)
#endif

// 사용법 (괄호 두 개!)
KdPrint(("HelloDriver: Debug message\n"));

// DbgPrint: 항상 출력 (Debug/Release 모두)
DbgPrint("HelloDriver: Always printed\n");

// 권장: 개발 중에는 DbgPrint, 제품에는 KdPrint
```

### 15.3.4 디버그 메시지 보기

**방법 1: WinDbg (커널 디버깅 연결 상태)**
```
// 자동으로 출력됨
HelloDriver: Hello World from Kernel!
HelloDriver: DriverEntry completed successfully.
```

**방법 2: DebugView (Mark Russinovich's Sysinternals)**
```
1. DebugView를 관리자 권한으로 실행
2. Capture > Capture Kernel 활성화
3. 드라이버 로드 시 메시지 확인

// 주의: 커널 디버거가 연결되어 있으면 DebugView로 안 보임
```

---

## 15.4 빌드 및 서명

### 15.4.1 빌드 실행

```
1. Visual Studio에서:
   - 빌드 > 솔루션 빌드 (Ctrl+Shift+B)
   - 또는 빌드 > HelloDriver 빌드

2. 빌드 결과:
   x64\Debug\HelloDriver.sys    // 드라이버 파일
   x64\Debug\HelloDriver.pdb    // 디버그 심볼

3. 출력 창 확인:
   ========== 빌드: 1 성공, 0 실패, 0 최신, 0 건너뜀 ==========
```

### 15.4.2 테스트 서명 확인

```
// 빌드 시 자동으로 테스트 서명됨

// 서명 확인 (PowerShell)
Get-AuthenticodeSignature .\x64\Debug\HelloDriver.sys

// 출력:
SignerCertificate      : [Subject]
                         CN=WDKTestCert [사용자이름] ...
                       [Issuer]
                         CN=WDKTestCert [사용자이름] ...
Status                 : Valid
StatusMessage          : Signature verified.
```

### 15.4.3 일반적인 빌드 오류

```
┌─────────────────────────────────────────────────────────────────────┐
│ 오류: LNK2019: unresolved external symbol _DriverEntry@8           │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: DriverEntry 함수 이름/시그니처 오류                           │
│ 해결: 함수 이름이 정확히 DriverEntry인지 확인                       │
│       매개변수 타입이 올바른지 확인                                 │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 오류: C2220: warning treated as error                               │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: /WX 옵션으로 경고가 오류로 처리됨                             │
│ 해결: 모든 경고 해결 (UNREFERENCED_PARAMETER 등)                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 오류: Inf2Cat error                                                  │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: INF 파일 문제 (아직 사용 안 함)                               │
│ 해결: 프로젝트 속성 > Inf2Cat > Run Inf2Cat = No                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 15.5 드라이버 로드 및 테스트

### 15.5.1 테스트 서명 모드 활성화

테스트 서명된 드라이버를 로드하려면 테스트 모드가 필요합니다.

```powershell
# 관리자 권한 명령 프롬프트에서:

# 테스트 모드 활성화
bcdedit /set testsigning on

# 재부팅 필요!
shutdown /r /t 0

# 활성화 확인
# 바탕화면 우측 하단에 "테스트 모드" 워터마크 표시됨
```

### 15.5.2 SC 명령어로 드라이버 관리

```cmd
# 관리자 권한 명령 프롬프트에서:

# 드라이버 서비스 생성
sc create HelloDriver type=kernel binPath="C:\full\path\to\HelloDriver.sys"

# 드라이버 시작 (로드)
sc start HelloDriver

# 드라이버 상태 확인
sc query HelloDriver

# 드라이버 중지 (언로드)
sc stop HelloDriver

# 드라이버 서비스 삭제
sc delete HelloDriver
```

### 15.5.3 전체 테스트 과정

```cmd
# 1. 서비스 생성
C:\> sc create HelloDriver type=kernel binPath="C:\Projects\HelloDriver\x64\Debug\HelloDriver.sys"
[SC] CreateService SUCCESS

# 2. 드라이버 시작
C:\> sc start HelloDriver
SERVICE_NAME: HelloDriver
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
        ...

# DebugView 또는 WinDbg에서 메시지 확인:
# HelloDriver: Hello World from Kernel!
# HelloDriver: DriverEntry completed successfully.

# 3. 드라이버 중지
C:\> sc stop HelloDriver
SERVICE_NAME: HelloDriver
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 1  STOPPED
        ...

# DebugView 또는 WinDbg에서 메시지 확인:
# HelloDriver: Goodbye from Kernel!

# 4. 서비스 삭제
C:\> sc delete HelloDriver
[SC] DeleteService SUCCESS
```

### 15.5.4 일반적인 로드 오류

```
┌─────────────────────────────────────────────────────────────────────┐
│ 오류: 서비스를 시작하지 못했습니다. 시스템 오류 577.                │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: 테스트 모드가 활성화되지 않음                                 │
│ 해결: bcdedit /set testsigning on 후 재부팅                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 오류: 서비스를 시작하지 못했습니다. 시스템 오류 2.                  │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: binPath의 파일 경로가 잘못됨                                  │
│ 해결: 절대 경로 확인, 파일 존재 확인                                │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 오류: BSOD (Blue Screen of Death)                                   │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: DriverEntry에서 오류 발생                                     │
│ 해결: WinDbg 커널 디버깅으로 분석                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 15.6 WinDbg로 디버깅

### 15.6.1 커널 디버깅 연결

```
// Chapter 8에서 설정한 네트워크 디버깅 사용

// 호스트 PC에서 WinDbg 실행:
windbg -k net:port=50000,key=1.2.3.4

// 대상 VM에서 재부팅하면 자동 연결

// 연결 확인:
0: kd> .sympath+ C:\Projects\HelloDriver\x64\Debug
0: kd> .reload
```

### 15.6.2 DriverEntry 브레이크포인트

```
// 드라이버 로드 전에 브레이크포인트 설정
0: kd> bu HelloDriver!DriverEntry
0: kd> g

// 대상 VM에서:
sc start HelloDriver

// WinDbg에서 브레이크:
Breakpoint 0 hit
HelloDriver!DriverEntry:
fffff801`12340000 mov     rbp,rsp

// 스텝 실행
0: kd> p
0: kd> p

// 변수 확인
0: kd> dv
         DriverObject = 0xffffb801`23450000
         RegistryPath = 0xffffb801`23451000 "\REGISTRY\MACHINE\SYSTEM\..."

// 계속 실행
0: kd> g
```

### 15.6.3 DbgPrint 출력 보기

```
// WinDbg가 연결되어 있으면 DbgPrint 출력이 자동으로 표시됨

0: kd> g
HelloDriver: Hello World from Kernel!
HelloDriver: DriverEntry completed successfully.

// 드라이버 언로드 시
0: kd> g
HelloDriver: Goodbye from Kernel!
```

### 15.6.4 드라이버 정보 확인

```
// 로드된 모듈 확인
0: kd> lm m HelloDriver
Browse full module list
start             end                 module name
fffff801`12340000 fffff801`12341000   HelloDriver   (pdb symbols)

// 드라이버 오브젝트 찾기
0: kd> !drvobj \Driver\HelloDriver 2
Driver object (ffffb80123460000) is for:
 \Driver\HelloDriver
DriverEntry:   fffff801`12340000  HelloDriver!DriverEntry
DriverUnload:  fffff801`123400a0  HelloDriver!DriverUnload
...

// 드라이버 심볼 확인
0: kd> x HelloDriver!*
fffff801`12340000 HelloDriver!DriverEntry
fffff801`123400a0 HelloDriver!DriverUnload
```

---

## 15.7 확장: 추가 기능 구현

### 15.7.1 드라이버 태그 추가

```c
// Pool 태그 정의 (디버깅 시 메모리 추적용)
#define HELLO_POOL_TAG 'lleH'  // 'Hell' 역순

// 메모리 할당 예시
PVOID buffer = ExAllocatePool2(
    POOL_FLAG_NON_PAGED,    // NonPaged Pool
    1024,                   // 크기
    HELLO_POOL_TAG          // 태그
);

if (buffer != NULL) {
    // 사용
    ExFreePoolWithTag(buffer, HELLO_POOL_TAG);
}
```

### 15.7.2 레지스트리 매개변수 읽기

```c
NTSTATUS ReadRegistryParameter(PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;
    HANDLE keyHandle;
    OBJECT_ATTRIBUTES objAttr;
    ULONG resultLength;
    PKEY_VALUE_PARTIAL_INFORMATION valueInfo;

    // 레지스트리 키 열기
    InitializeObjectAttributes(
        &objAttr,
        RegistryPath,
        OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE,
        NULL,
        NULL
    );

    status = ZwOpenKey(&keyHandle, KEY_READ, &objAttr);
    if (!NT_SUCCESS(status)) {
        DbgPrint("HelloDriver: Failed to open registry key: 0x%X\n", status);
        return status;
    }

    // 값 읽기 (예: "DebugLevel")
    UNICODE_STRING valueName = RTL_CONSTANT_STRING(L"DebugLevel");

    // 크기 먼저 확인
    status = ZwQueryValueKey(
        keyHandle,
        &valueName,
        KeyValuePartialInformation,
        NULL,
        0,
        &resultLength
    );

    if (status == STATUS_BUFFER_TOO_SMALL) {
        valueInfo = ExAllocatePool2(
            POOL_FLAG_PAGED,
            resultLength,
            HELLO_POOL_TAG
        );

        if (valueInfo != NULL) {
            status = ZwQueryValueKey(
                keyHandle,
                &valueName,
                KeyValuePartialInformation,
                valueInfo,
                resultLength,
                &resultLength
            );

            if (NT_SUCCESS(status) && valueInfo->Type == REG_DWORD) {
                ULONG debugLevel = *(PULONG)valueInfo->Data;
                DbgPrint("HelloDriver: DebugLevel = %lu\n", debugLevel);
            }

            ExFreePoolWithTag(valueInfo, HELLO_POOL_TAG);
        }
    }

    ZwClose(keyHandle);
    return STATUS_SUCCESS;
}
```

### 15.7.3 시스템 시간 출력

```c
#include <ntddk.h>

VOID PrintSystemTime(VOID)
{
    LARGE_INTEGER systemTime;
    LARGE_INTEGER localTime;
    TIME_FIELDS timeFields;

    // 시스템 시간 획득 (UTC)
    KeQuerySystemTime(&systemTime);

    // 로컬 시간으로 변환
    ExSystemTimeToLocalTime(&systemTime, &localTime);

    // 사람이 읽을 수 있는 형식으로 변환
    RtlTimeToTimeFields(&localTime, &timeFields);

    DbgPrint("HelloDriver: Current time: %04d-%02d-%02d %02d:%02d:%02d\n",
        timeFields.Year,
        timeFields.Month,
        timeFields.Day,
        timeFields.Hour,
        timeFields.Minute,
        timeFields.Second
    );
}
```

---

## 15.8 완성된 코드

```c
// Driver.c - Hello World 커널 드라이버 완성판

#include <ntddk.h>

#define HELLO_POOL_TAG 'lleH'

// 전역 변수 (옵션)
ULONG g_DebugLevel = 0;

// 함수 선언
DRIVER_UNLOAD DriverUnload;
NTSTATUS ReadRegistryParameter(_In_ PUNICODE_STRING RegistryPath);

// 드라이버 진입점
NTSTATUS DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath
)
{
    NTSTATUS status = STATUS_SUCCESS;

    DbgPrint("===========================================\n");
    DbgPrint("HelloDriver: DriverEntry called\n");
    // HelloDriver: DriverEntry 호출됨
    DbgPrint("HelloDriver: DriverObject = %p\n", DriverObject);
    DbgPrint("HelloDriver: RegistryPath = %wZ\n", RegistryPath);
    DbgPrint("===========================================\n");

    // 언로드 루틴 등록
    DriverObject->DriverUnload = DriverUnload;

    // 레지스트리에서 설정 읽기 (옵션)
    ReadRegistryParameter(RegistryPath);

    // 시스템 정보 출력 (옵션)
    {
        RTL_OSVERSIONINFOW versionInfo;
        versionInfo.dwOSVersionInfoSize = sizeof(RTL_OSVERSIONINFOW);

        if (NT_SUCCESS(RtlGetVersion(&versionInfo))) {
            DbgPrint("HelloDriver: Windows %lu.%lu.%lu\n",
                versionInfo.dwMajorVersion,
                versionInfo.dwMinorVersion,
                versionInfo.dwBuildNumber
            );
        }
    }

    DbgPrint("HelloDriver: DriverEntry completed. Status = 0x%X\n", status);
    // HelloDriver: DriverEntry 완료. 상태 = 0x0

    return status;
}

// 드라이버 언로드
VOID DriverUnload(
    _In_ PDRIVER_OBJECT DriverObject
)
{
    UNREFERENCED_PARAMETER(DriverObject);

    DbgPrint("===========================================\n");
    DbgPrint("HelloDriver: DriverUnload called\n");
    // HelloDriver: DriverUnload 호출됨
    DbgPrint("HelloDriver: Cleanup completed. Goodbye!\n");
    // HelloDriver: 정리 완료. 안녕!
    DbgPrint("===========================================\n");
}

// 레지스트리 매개변수 읽기
NTSTATUS ReadRegistryParameter(
    _In_ PUNICODE_STRING RegistryPath
)
{
    UNREFERENCED_PARAMETER(RegistryPath);

    // 실제 구현은 필요에 따라 추가
    DbgPrint("HelloDriver: Registry path: %wZ\n", RegistryPath);

    return STATUS_SUCCESS;
}
```

---

## 15.9 문제 해결 가이드

### Q: 드라이버가 로드되지 않습니다

```
1. 테스트 모드 확인:
   bcdedit | findstr testsigning
   → "testsigning    Yes" 여야 함

2. 드라이버 경로 확인:
   dir C:\full\path\to\HelloDriver.sys
   → 파일이 존재해야 함

3. 서비스 상태 확인:
   sc query HelloDriver
   → STATE가 STOPPED인 경우 sc start로 시작

4. 이벤트 로그 확인:
   이벤트 뷰어 > Windows 로그 > 시스템
   → "드라이버" 관련 오류 찾기
```

### Q: DbgPrint 출력이 안 보입니다

```
1. WinDbg 연결 확인:
   - 커널 디버거가 연결되어 있으면 DebugView로 안 보임
   - WinDbg에서 g 명령 후 확인

2. DebugView 설정:
   - 관리자 권한으로 실행
   - Capture > Capture Kernel 체크
   - Capture > Pass-Through 해제

3. 레지스트리 필터 확인:
   HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter
   → DEFAULT 값을 0xF로 설정 (모든 레벨 출력)
```

### Q: BSOD가 발생합니다

```
1. 미니 덤프 분석:
   C:\Windows\Minidump\*.dmp
   WinDbg로 열어서 !analyze -v

2. 일반적인 원인:
   - NULL 포인터 역참조
   - DriverUnload 미등록 후 언로드 시도
   - 잘못된 IRQL에서 API 호출

3. 디버깅:
   bu HelloDriver!DriverEntry
   g
   (각 라인 스텝 실행하며 문제 위치 파악)
```

---

## 요약

이 챕터에서 학습한 내용:

1. **프로젝트 생성**: Visual Studio에서 Empty WDM Driver 템플릿
2. **DriverEntry**: 드라이버 진입점, NTSTATUS 반환
3. **DriverUnload**: 드라이버 언로드 시 정리 작업
4. **DbgPrint**: 커널 디버그 메시지 출력
5. **빌드/서명**: WDK 빌드, 테스트 서명
6. **로드/테스트**: sc 명령어로 드라이버 관리
7. **디버깅**: WinDbg로 DriverEntry 브레이크포인트

다음 챕터에서는 레거시 필터 드라이버의 개념을 학습하고, Minifilter와의 차이를 이해합니다.

---

## 참고 자료

- [Write a Hello World Windows Driver](https://docs.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/writing-a-very-small-kmdf-driver)
- [DriverEntry routine](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nc-wdm-driver_initialize)
- [DbgPrint function](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-dbgprint)
