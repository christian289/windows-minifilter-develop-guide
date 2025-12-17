# Chapter 4: Windows 커널 구조

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표

- 사용자 모드와 커널 모드의 차이 이해
- Windows 커널의 핵심 구성 요소 파악
- IRQL 개념과 중요성 습득
- 시스템 호출 흐름 이해

---

## 들어가며

.NET 개발자로서 여러분은 주로 **사용자 모드(User Mode)**에서 작업해왔습니다. 하지만 커널 드라이버는 **커널 모드(Kernel Mode)**에서 실행됩니다. 이 두 세계는 완전히 다른 규칙으로 동작합니다.

이 장에서는 Windows 커널의 기본 구조와 핵심 개념을 이해합니다. 특히 **IRQL**(Interrupt Request Level)은 커널 개발자가 반드시 이해해야 하는 개념입니다.

---

## 4.1 사용자 모드 vs 커널 모드

### 4.1.1 두 모드의 분리

Windows는 CPU의 권한 수준(Ring)을 활용하여 시스템을 보호합니다:

```
┌─────────────────────────────────────────────────────────────┐
│                    사용자 모드 (Ring 3)                      │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                │
│  │ notepad   │  │ explorer  │  │ MyApp.exe │                │
│  │   .exe    │  │   .exe    │  │ (.NET)    │                │
│  └───────────┘  └───────────┘  └───────────┘                │
│                                                             │
│  특징:                                                      │
│    - 제한된 CPU 명령어만 사용 가능                           │
│    - 자신만의 가상 주소 공간 (격리됨)                        │
│    - 하드웨어 직접 접근 불가                                 │
│    - 크래시해도 해당 프로세스만 종료                         │
├─────────────────────────────────────────────────────────────┤
│                   ═══ 커널 경계 ═══                          │
├─────────────────────────────────────────────────────────────┤
│                    커널 모드 (Ring 0)                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              ntoskrnl.exe (Windows 커널)              │   │
│  │   ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │   │ hal.dll │  │ win32k  │  │ drivers │            │   │
│  │   │         │  │  .sys   │  │  .sys   │            │   │
│  │   └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  특징:                                                      │
│    - 모든 CPU 명령어 사용 가능                               │
│    - 물리 메모리 직접 접근                                   │
│    - 하드웨어 직접 제어                                      │
│    - 크래시 = BSOD (시스템 전체 중단)                        │
└─────────────────────────────────────────────────────────────┘
```

### 4.1.2 .NET 개발자를 위한 비유

```csharp
// 사용자 모드 = 일반 .NET 애플리케이션
// User mode = Normal .NET application
public class MyApplication
{
    public void DoWork()
    {
        // 안전한 환경에서 실행
        // GC가 메모리 관리
        // 예외가 발생해도 프로세스만 종료
        var list = new List<int>();
    }
}

// 커널 모드 = CLR 내부 + unsafe + 무제한 권한
// Kernel mode = Inside CLR + unsafe + unlimited privilege
unsafe class KernelDriver
{
    // 메모리 직접 조작
    // 실수 시 시스템 전체 크래시
    void* RawMemory;
}
```

### 4.1.3 가상 주소 공간

각 프로세스는 독립적인 가상 주소 공간을 가집니다:

```
64-bit Windows 가상 주소 공간:
┌──────────────────────────────────────┐ 0xFFFFFFFF'FFFFFFFF
│                                      │
│           커널 공간                   │ (모든 프로세스가 공유)
│    - 커널, 드라이버                   │
│    - 커널 모드에서만 접근 가능         │
│                                      │
├──────────────────────────────────────┤ 0x00007FFF'FFFFFFFF
│                                      │
│           사용자 공간                 │ (프로세스별로 격리)
│    - 애플리케이션 코드                │
│    - 힙, 스택                        │
│    - DLL                             │
│                                      │
└──────────────────────────────────────┘ 0x00000000'00000000
```

### 4.1.4 사용자 모드 → 커널 모드 전환

사용자 모드 코드가 커널 기능을 사용하려면 **시스템 호출(System Call)**이 필요합니다:

```
사용자 앱: CreateFile("C:\\test.txt")
          │
          ▼
kernel32.dll: CreateFileW()
          │
          ▼
ntdll.dll: NtCreateFile()
          │
          ├── syscall 명령어 실행
          │
═══════════════════════════════════════ ← 커널 경계
          │
          ▼
ntoskrnl.exe: NtCreateFile()  (커널 모드)
```

---

## 4.2 커널 핵심 구성 요소

Windows 커널은 여러 하위 시스템으로 구성됩니다:

### 4.2.1 Executive 서브시스템

```
┌─────────────────────────────────────────────────────────────┐
│                      Executive                               │
├──────────┬──────────┬──────────┬──────────┬────────────────┤
│ Object   │   I/O    │ Memory   │ Process  │   Security     │
│ Manager  │ Manager  │ Manager  │ Manager  │ Ref. Monitor   │
├──────────┴──────────┴──────────┴──────────┴────────────────┤
│                        Kernel                                │
│              (스케줄링, 인터럽트, 동기화)                     │
├─────────────────────────────────────────────────────────────┤
│                HAL (Hardware Abstraction Layer)              │
│                    (하드웨어 추상화)                          │
└─────────────────────────────────────────────────────────────┘
```

### Object Manager (객체 관리자)

모든 커널 리소스를 **객체**로 관리합니다:

```c
// 커널 객체 예시
// Kernel object examples
HANDLE FileHandle;     // 파일 객체 핸들
HANDLE ProcessHandle;  // 프로세스 객체 핸들
HANDLE ThreadHandle;   // 스레드 객체 핸들
HANDLE EventHandle;    // 이벤트 객체 핸들

// 객체 이름 예시
// Object name examples
// \Device\HarddiskVolume1\Windows\System32\notepad.exe
// \Registry\Machine\Software
// \Sessions\1\BaseNamedObjects\MyEvent
```

```csharp
// C# 비유: SafeHandle이 커널 객체를 래핑
// C# analogy: SafeHandle wraps kernel objects
using SafeFileHandle handle = File.OpenHandle("test.txt");
// handle.DangerousGetHandle()는 실제 커널 핸들 값
```

### I/O Manager (입출력 관리자)

- 모든 I/O 요청을 **IRP (I/O Request Packet)**로 관리
- 드라이버 스택을 통해 요청 전달
- 완료 시 결과 반환

### Memory Manager (메모리 관리자)

- 가상 메모리와 물리 메모리 매핑
- 페이징 (디스크 스왑)
- 메모리 풀 관리

### Process Manager (프로세스 관리자)

- 프로세스/스레드 생성 및 종료
- 스케줄링 지원

### Security Reference Monitor (보안 참조 모니터)

- 접근 권한 검사
- 토큰 관리
- Chapter 6에서 자세히 다룸

### 4.2.2 핵심 데이터 구조

커널에서 자주 만나게 될 구조체들:

```c
// EPROCESS: 프로세스를 나타내는 구조체 (Executive Process)
// EPROCESS: Structure representing a process
typedef struct _EPROCESS {
    // ... 많은 필드들 ...
    // ImageFileName: 프로세스 이름
    // Token: 보안 토큰
    // ThreadListHead: 스레드 목록
} EPROCESS, *PEPROCESS;

// ETHREAD: 스레드를 나타내는 구조체 (Executive Thread)
// ETHREAD: Structure representing a thread
typedef struct _ETHREAD {
    // ... 많은 필드들 ...
} ETHREAD, *PETHREAD;

// DRIVER_OBJECT: 드라이버를 나타내는 구조체
// DRIVER_OBJECT: Structure representing a driver
typedef struct _DRIVER_OBJECT {
    PDEVICE_OBJECT DeviceObject;        // 장치 객체 목록
    PUNICODE_STRING DriverName;          // 드라이버 이름
    PDRIVER_DISPATCH MajorFunction[28]; // IRP 핸들러 테이블
    PDRIVER_UNLOAD DriverUnload;         // 언로드 함수
} DRIVER_OBJECT, *PDRIVER_OBJECT;

// DEVICE_OBJECT: 장치를 나타내는 구조체
// DEVICE_OBJECT: Structure representing a device
typedef struct _DEVICE_OBJECT {
    struct _DRIVER_OBJECT* DriverObject;
    struct _DEVICE_OBJECT* NextDevice;
    struct _DEVICE_OBJECT* AttachedDevice;
    PVOID DeviceExtension;  // 드라이버 전용 데이터
} DEVICE_OBJECT, *PDEVICE_OBJECT;
```

---

## 4.3 IRQL (Interrupt Request Level)

**IRQL**은 커널 개발에서 가장 중요한 개념 중 하나입니다. IRQL 규칙을 위반하면 BSOD가 발생합니다.

### 4.3.1 IRQL 수준

```
높은 우선순위 ↑
┌─────────────────────────────────────────────────────────────┐
│ HIGH_LEVEL (31)     │ 시스템 정지 수준. 모든 인터럽트 차단   │
├─────────────────────────────────────────────────────────────┤
│ POWER_LEVEL (30)    │ 전원 장애                              │
├─────────────────────────────────────────────────────────────┤
│ IPI_LEVEL (29)      │ 프로세서 간 인터럽트                    │
├─────────────────────────────────────────────────────────────┤
│ CLOCK_LEVEL (28)    │ 시스템 클록                             │
├─────────────────────────────────────────────────────────────┤
│ DIRQL (3-26)        │ 장치 인터럽트 (Device IRQL)             │
├─────────────────────────────────────────────────────────────┤
│ DISPATCH_LEVEL (2)  │ 스케줄러, DPC, 스핀락                   │
├─────────────────────────────────────────────────────────────┤
│ APC_LEVEL (1)       │ APC (Asynchronous Procedure Call)       │
├─────────────────────────────────────────────────────────────┤
│ PASSIVE_LEVEL (0)   │ 일반 스레드 실행 (가장 낮음)            │
└─────────────────────────────────────────────────────────────┘
낮은 우선순위 ↓
```

### 4.3.2 IRQL 규칙 (매우 중요!)

#### PASSIVE_LEVEL (0) - 안전 지대

```c
// PASSIVE_LEVEL에서 가능한 작업
// Operations allowed at PASSIVE_LEVEL
VOID SafeFunction()
{
    PAGED_CODE();  // PASSIVE_LEVEL 또는 APC_LEVEL인지 확인

    // ✅ 가능한 작업들
    PVOID buffer = ExAllocatePoolWithTag(PagedPool, 1024, 'gaPM');  // Paged Pool OK
    KeWaitForSingleObject(...);  // 대기 가능
    ZwCreateFile(...);           // 파일 생성 가능

    // 페이지 폴트 허용됨
}
```

#### DISPATCH_LEVEL (2) - 제한 지대

```c
// DISPATCH_LEVEL에서의 제한
// Restrictions at DISPATCH_LEVEL
VOID RestrictedFunction()
{
    // ❌ 불가능한 작업들 - BSOD 발생!
    // ExAllocatePoolWithTag(PagedPool, ...);  // Paged Pool 접근 불가
    // KeWaitForSingleObject(...);             // 대기 불가
    // 페이지 가능 코드 실행 불가

    // ✅ 가능한 작업들
    PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 1024, 'PNPM');  // Non-Paged만
    KeAcquireSpinLockAtDpcLevel(...);  // 스핀락 획득
}
```

### IRQL 규칙 요약

| IRQL | 페이지 폴트 | Paged Pool | 대기 (Wait) | 대부분의 API |
|------|-----------|------------|-------------|-------------|
| PASSIVE_LEVEL | ✅ 허용 | ✅ 허용 | ✅ 허용 | ✅ 허용 |
| APC_LEVEL | ✅ 허용 | ✅ 허용 | ⚠️ 제한 | ⚠️ 제한 |
| DISPATCH_LEVEL | ❌ BSOD | ❌ BSOD | ❌ BSOD | ❌ 대부분 불가 |
| DIRQL 이상 | ❌ BSOD | ❌ BSOD | ❌ BSOD | ❌ 거의 불가 |

### 4.3.3 C# 비유

```csharp
// IRQL은 마치 이런 제약과 비슷합니다
// IRQL is similar to these constraints

// PASSIVE_LEVEL = 일반 코드
public async Task NormalCode()
{
    await Task.Delay(1000);  // 대기 가능
    var data = File.ReadAllText("test.txt");  // I/O 가능
}

// DISPATCH_LEVEL = lock 블록 안
private object _lock = new object();
public void InLock()
{
    lock (_lock)
    {
        // 이 안에서는:
        // - await 사용 불가 (컴파일 에러)
        // - 장시간 작업 지양
        // - 다른 lock 획득 시 교착 상태 주의
    }
}
```

### 4.3.4 IRQL 확인 및 변경

```c
// 현재 IRQL 확인
// Check current IRQL
KIRQL currentIrql = KeGetCurrentIrql();
DbgPrint("Current IRQL: %d\n", currentIrql);

// IRQL 높이기 (주의해서 사용)
// Raise IRQL (use with caution)
KIRQL oldIrql;
KeRaiseIrql(DISPATCH_LEVEL, &oldIrql);
// ... DISPATCH_LEVEL에서 작업 ...
KeLowerIrql(oldIrql);  // 반드시 원래대로 복원

// 스핀락 사용 시 자동으로 DISPATCH_LEVEL로 상승
// Spinlock automatically raises to DISPATCH_LEVEL
KSPIN_LOCK spinLock;
KeInitializeSpinLock(&spinLock);

KIRQL oldIrql;
KeAcquireSpinLock(&spinLock, &oldIrql);  // IRQL 상승
// ... 임계 영역 ...
KeReleaseSpinLock(&spinLock, oldIrql);   // IRQL 복원
```

### 4.3.5 IRQL 관련 BSOD

IRQL 규칙 위반 시 발생하는 대표적인 BSOD:

```
IRQL_NOT_LESS_OR_EQUAL (0x0000000A)
- 높은 IRQL에서 페이지 가능 메모리 접근
- 가장 흔한 IRQL 관련 BSOD

DRIVER_IRQL_NOT_LESS_OR_EQUAL (0x000000D1)
- 드라이버가 부적절한 IRQL에서 메모리 접근
- 위와 유사하지만 드라이버가 원인임을 명시
```

---

## 4.4 시스템 호출 흐름

### 4.4.1 CreateFile 호출 예시

사용자 모드에서 파일을 열 때의 전체 흐름:

```
┌─────────────────────────────────────────────────────────────┐
│                        사용자 모드                           │
├─────────────────────────────────────────────────────────────┤
│  C# Application                                              │
│    File.Open("C:\\test.txt")                                │
│         │                                                    │
│         ▼                                                    │
│  .NET Runtime                                                │
│    SafeFileHandle → Win32 API 호출                          │
│         │                                                    │
│         ▼                                                    │
│  kernel32.dll                                                │
│    CreateFileW("C:\\test.txt", ...)                         │
│         │                                                    │
│         ▼                                                    │
│  ntdll.dll                                                   │
│    NtCreateFile(...)                                         │
│         │                                                    │
│         ├── syscall 명령어 (x64)                            │
│         │   또는 sysenter (x86)                             │
│         │                                                    │
├─────────┼───────────────────────────────────────────────────┤
│         │           ═══ 커널 경계 ═══                        │
├─────────┼───────────────────────────────────────────────────┤
│         ▼                        커널 모드                   │
│  ntoskrnl.exe                                                │
│    NtCreateFile()                                            │
│         │                                                    │
│         ▼                                                    │
│  I/O Manager                                                 │
│    IRP (I/O Request Packet) 생성                            │
│         │                                                    │
│         ▼                                                    │
│  Filter Manager (fltmgr.sys)                                 │
│    ┌─────────────────────────────┐                          │
│    │ Minifilter 1 (Pre-Create)   │                          │
│    │ Minifilter 2 (Pre-Create)   │  ← 우리 드라이버!        │
│    │ Minifilter 3 (Pre-Create)   │                          │
│    └─────────────────────────────┘                          │
│         │                                                    │
│         ▼                                                    │
│  File System Driver (ntfs.sys)                               │
│    실제 파일 열기                                            │
│         │                                                    │
│         ▼                                                    │
│  Volume Manager → Disk Driver                                │
│    물리적 디스크 접근                                        │
└─────────────────────────────────────────────────────────────┘
```

### 4.4.2 반환 흐름

```
  결과 반환 흐름:

  Disk Driver
       │ (I/O 완료)
       ▼
  File System (ntfs.sys)
       │
       ▼
  Filter Manager
    ┌─────────────────────────────┐
    │ Minifilter 3 (Post-Create)  │
    │ Minifilter 2 (Post-Create)  │  ← 우리 드라이버!
    │ Minifilter 1 (Post-Create)  │
    └─────────────────────────────┘
       │
       ▼
  I/O Manager (IRP 완료)
       │
═══════════════════════════════════════ ← 커널 경계
       │
       ▼
  ntdll.dll: 결과 반환
       │
       ▼
  kernel32.dll: HANDLE 반환
       │
       ▼
  Application: 파일 핸들 사용
```

---

## 4.5 드라이버 모델 개요

Windows에서 드라이버를 작성하는 여러 방법이 있습니다:

### 4.5.1 WDM (Windows Driver Model)

```
WDM (레거시):
├── 직접 IRP 처리
├── 복잡한 Plug and Play 지원
├── 직접 전원 관리
└── 많은 상용구 코드 필요
```

### 4.5.2 WDF (Windows Driver Framework)

```
WDF:
├── KMDF (Kernel-Mode Driver Framework)
│   ├── WDM 추상화
│   ├── 객체 지향 모델
│   └── 일반 하드웨어 드라이버에 적합
│
└── UMDF (User-Mode Driver Framework)
    ├── 사용자 모드에서 실행
    ├── 안정성 높음 (크래시해도 BSOD 없음)
    └── 성능 제한 있음
```

### 4.5.3 Minifilter (이 책의 주제)

```
Minifilter:
├── Filter Manager 기반
├── 파일 시스템 필터링 전용
├── 간단한 콜백 모델
├── 안전한 로드/언로드
└── 컨텍스트 관리 지원

특징:
  ✓ IRP 대신 콜백 데이터 사용
  ✓ Filter Manager가 복잡한 부분 처리
  ✓ Altitude로 스택 순서 관리
  ✓ 레거시 필터보다 개발 용이
```

### 드라이버 모델 비교

| 특성 | WDM | KMDF | Minifilter |
|------|-----|------|------------|
| 복잡도 | 높음 | 중간 | 낮음 |
| 파일 시스템 필터링 | 가능 | 가능 | 최적화 |
| IRP 직접 처리 | 필수 | 추상화 | 콜백 데이터 |
| 권장 용도 | 레거시 | 일반 드라이버 | 파일 필터 |

---

## 정리

### 핵심 개념

| 개념 | 설명 |
|------|------|
| **사용자 모드** | Ring 3, 제한된 권한, 크래시 시 프로세스만 종료 |
| **커널 모드** | Ring 0, 무제한 권한, 크래시 시 BSOD |
| **IRQL** | 인터럽트 요청 수준, 높을수록 제한 많음 |
| **PASSIVE_LEVEL** | 가장 낮은 IRQL, 대부분의 API 사용 가능 |
| **DISPATCH_LEVEL** | 페이지 폴트/대기 불가, Non-Paged Pool만 |

### IRQL 체크리스트

- [ ] 현재 IRQL 수준을 알고 있는가?
- [ ] DISPATCH_LEVEL에서 Paged Pool을 사용하지 않는가?
- [ ] 스핀락 보유 중 대기(Wait)하지 않는가?
- [ ] PAGED_CODE() 매크로를 적절히 사용하는가?

### 다음 장 미리보기

Chapter 5에서는 파일 시스템 아키텍처를 다룹니다:
- IRP의 구조와 흐름
- 파일 시스템 드라이버 스택
- Filter Manager와 Minifilter 관계

---

## 연습 문제

### 1. IRQL 이해

다음 코드의 문제점은?

```c
VOID SpinLockFunction(PKSPIN_LOCK SpinLock)
{
    KIRQL oldIrql;
    KeAcquireSpinLock(SpinLock, &oldIrql);

    PVOID buffer = ExAllocatePoolWithTag(PagedPool, 100, 'tsT');

    KeReleaseSpinLock(SpinLock, oldIrql);
}
```

### 2. 메모리 접근

DISPATCH_LEVEL에서 안전하게 접근할 수 있는 것은? (복수 선택)

- [ ] 스택 지역 변수
- [ ] Paged Pool 할당 버퍼
- [ ] Non-Paged Pool 할당 버퍼
- [ ] 전역 변수 (paged 섹션)

### 3. 커널 객체

다음 중 커널 객체가 아닌 것은?

- [ ] 파일 핸들
- [ ] 프로세스 핸들
- [ ] .NET 객체 참조
- [ ] 이벤트 핸들

---

## 참고 자료

- [Windows Internals, 7th Edition](https://docs.microsoft.com/en-us/sysinternals/resources/windows-internals)
- [IRQL Documentation](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/managing-hardware-priorities)
- [Introduction to Windows Drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/)
