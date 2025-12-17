# Chapter 12: BSOD 크래시 덤프 분석

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- BSOD(Blue Screen of Death)의 원리를 이해합니다
- 크래시 덤프 파일을 WinDbg로 분석합니다
- !analyze -v 명령어의 출력을 해석합니다
- 주요 Bug Check 코드의 의미를 파악합니다
- 드라이버 개발 중 발생하는 BSOD를 디버깅합니다

## 도입: BSOD는 친구입니다

.NET 개발자에게 예외(Exception)는 친숙합니다. try-catch로 처리하고, 스택 트레이스를 확인합니다. 커널 개발에서 **BSOD**는 처리되지 않은 커널 예외입니다. 처음에는 무섭게 보이지만, BSOD는 문제의 원인을 정확히 알려주는 **진단 도구**입니다.

```csharp
// C# 예외 처리
try
{
    // 위험한 작업
}
catch (Exception ex)
{
    Console.WriteLine(ex.StackTrace);  // 스택 트레이스 출력
}

// 커널에서는 처리되지 않은 예외 = BSOD
// BSOD = 자동으로 저장되는 크래시 덤프 + 스택 트레이스
```

---

## 12.1 BSOD의 원리

### 12.1.1 Bug Check란?

Windows 커널이 복구 불가능한 오류를 감지하면 **Bug Check**를 호출합니다. 이것이 BSOD입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       BSOD 발생 과정                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 커널 오류 발생 (NULL 포인터, 잘못된 메모리 접근 등)             │
│                          ↓                                           │
│  2. KeBugCheckEx() 호출                                              │
│                          ↓                                           │
│  3. 메모리 덤프 생성 (설정에 따라)                                  │
│                          ↓                                           │
│  4. 블루스크린 표시 (Stop 코드 + QR 코드)                           │
│                          ↓                                           │
│  5. 시스템 재시작 (또는 대기)                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.1.2 Bug Check 코드 구조

```c
// KeBugCheckEx 함수 시그니처
VOID KeBugCheckEx(
    ULONG BugCheckCode,      // 4자리 16진수 코드
    ULONG_PTR BugCheckParameter1,
    ULONG_PTR BugCheckParameter2,
    ULONG_PTR BugCheckParameter3,
    ULONG_PTR BugCheckParameter4
);

// 예: IRQL_NOT_LESS_OR_EQUAL
KeBugCheckEx(
    0x0000000A,              // IRQL_NOT_LESS_OR_EQUAL
    0xfffff80012345678,      // 접근하려던 주소
    0x0000000000000002,      // IRQL
    0x0000000000000001,      // 접근 유형 (0=읽기, 1=쓰기)
    0xfffff80011112222       // 오류 발생 주소
);
```

### 12.1.3 크래시 덤프 유형

```
┌─────────────────────────────────────────────────────────────────────┐
│                      크래시 덤프 유형                                │
├─────────────────────────────────────────────────────────────────────┤
│  유형              크기           내용                               │
├─────────────────────────────────────────────────────────────────────┤
│  Small (미니)      256KB~         버그 체크 정보, 스택               │
│  Kernel            수백 MB        커널 메모리만                      │
│  Complete (전체)   RAM 크기       전체 물리 메모리                   │
│  Automatic         가변           Kernel + 필요 시 추가 정보         │
│  Active            RAM 크기       사용 중인 메모리만                 │
└─────────────────────────────────────────────────────────────────────┘

// 개발 중 권장: Kernel Memory Dump 또는 Complete Memory Dump
```

### 12.1.4 덤프 설정 확인 및 변경

```
// 제어판 > 시스템 > 고급 시스템 설정 > 시작 및 복구

// 또는 레지스트리:
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\CrashControl

// 레지스트리 값:
// CrashDumpEnabled:
//   0 = None
//   1 = Complete
//   2 = Kernel
//   3 = Small (Mini)
//   7 = Automatic

// DumpFile: 덤프 파일 경로 (기본: %SystemRoot%\MEMORY.DMP)
// MinidumpDir: 미니 덤프 경로 (기본: %SystemRoot%\Minidump)
```

---

## 12.2 크래시 덤프 열기와 기본 분석

### 12.2.1 덤프 파일 열기

```
// WinDbg에서 덤프 파일 열기:
// File > Open Crash Dump (Ctrl+D)

// 경로: C:\Windows\MEMORY.DMP (커널 덤프)
// 또는: C:\Windows\Minidump\*.dmp (미니 덤프)

// 덤프 열기 후 출력 예:
Microsoft (R) Windows Debugger Version 10.0.22621.1
Copyright (c) Microsoft Corporation. All rights reserved.

Loading Dump File [C:\Windows\MEMORY.DMP]
Kernel Bitmap Dump File: Kernel address space is available, User address space may not be available.

Symbol search path is: srv*c:\symbols*https://msdl.microsoft.com/download/symbols
Executable search path is:
Windows 10 Kernel Version 19041 MP (8 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Machine Name:
Kernel base = 0xfffff802`1be00000 PsLoadedModuleList = 0xfffff802`1ca2a230
Debug session time: Mon Dec 16 10:30:45.123 2024
System Uptime: 0 days 2:15:30.456
Loading Kernel Symbols
...
Loading User Symbols
Loading unloaded module list
.....
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

Use !analyze -v to get detailed debugging information.

BugCheck D1, {fffff80012345678, 2, 1, fffff80011112222}

Probably caused by : MyDriver.sys ( MyDriver!PreWrite+0x45 )
```

### 12.2.2 기본 정보 확인

```
// 버그 체크 코드 확인
0: kd> .bugcheck
Bugcheck code 000000D1
Arguments fffff800`12345678 00000000`00000002 00000000`00000001 fffff800`11112222

// 버그 체크 코드 해석
// 0x000000D1 = DRIVER_IRQL_NOT_LESS_OR_EQUAL

// 시스템 정보
0: kd> vertarget
Windows 10 Kernel Version 19041 MP (8 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Machine Name:
Debug session time: Mon Dec 16 10:30:45.123 2024
System Uptime: 0 days 2:15:30.456

// 로드된 모듈 확인
0: kd> lm
start             end                 module name
fffff800`10000000 fffff800`10050000   MyDriver   (pdb symbols)
fffff802`1be00000 fffff802`1ca30000   nt         (pdb symbols)
fffff802`1d000000 fffff802`1d100000   fltmgr     (pdb symbols)
...
```

---

## 12.3 !analyze -v 명령어 마스터

### 12.3.1 자동 분석 실행

`!analyze -v`는 BSOD 분석의 핵심 명령어입니다.

```
0: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: fffff80012345678, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000001, value 0 = read operation, 1 = write operation
Arg4: fffff80011112222, address which referenced memory

Debugging Details:
------------------

KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 1234

    Key  : Analysis.DebugAnalysisManager
    Value: Create

    Key  : Analysis.Elapsed.mSec
    Value: 5678

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 123

...

BUGCHECK_CODE:  d1

BUGCHECK_P1: fffff80012345678

BUGCHECK_P2: 2

BUGCHECK_P3: 1

BUGCHECK_P4: fffff80011112222

WRITE_ADDRESS:  fffff80012345678

CURRENT_IRQL:  2

FAULTING_IP:
MyDriver!PreWrite+45
fffff800`11112222 488b4110        mov     rax,qword ptr [rcx+10h]

TRAP_FRAME:  fffff80123456000 -- (.trap 0xfffff80123456000)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=0000000000000000 rbx=0000000000000000 rcx=fffff80012345668
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=fffff80011112222 rsp=fffff80123456100 rbp=fffff80123456200
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000000
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc

STACK_TEXT:
fffff801`23456100 fffff802`1d012345 : ... : MyDriver!PreWrite+0x45
fffff801`23456200 fffff802`1d023456 : ... : fltmgr!FltpPerformPreCallbacks+0x123
fffff801`23456300 fffff802`1d034567 : ... : fltmgr!FltpPassThrough+0x456
fffff801`23456400 fffff802`1be56789 : ... : nt!IofCallDriver+0x89
...

SYMBOL_NAME:  MyDriver!PreWrite+45

MODULE_NAME: MyDriver

IMAGE_NAME:  MyDriver.sys

STACK_COMMAND:  .trap 0xfffff80123456000 ; kb

FAILURE_BUCKET_ID:  AV_MyDriver!PreWrite+45

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {12345678-abcd-1234-5678-90abcdef1234}

Followup:     MachineOwner
---------
```

### 12.3.2 분석 결과 해석

```
┌─────────────────────────────────────────────────────────────────────┐
│                   !analyze -v 주요 섹션 해석                         │
├─────────────────────────────────────────────────────────────────────┤
│  섹션                   설명                                         │
├─────────────────────────────────────────────────────────────────────┤
│  BUGCHECK_CODE          Bug Check 코드 (예: 0xD1)                   │
│  Arguments              4개의 매개변수 (코드별 의미 다름)           │
│  FAULTING_IP            오류 발생 명령어 위치                       │
│  TRAP_FRAME             크래시 시점 레지스터 상태                   │
│  STACK_TEXT             콜스택 (가장 중요!)                         │
│  SYMBOL_NAME            오류 함수와 오프셋                          │
│  MODULE_NAME            오류 모듈 이름                              │
│  FAILURE_BUCKET_ID      고유 오류 식별자                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.3.3 추가 분석 명령어

```
// Trap Frame 컨텍스트로 전환
0: kd> .trap 0xfffff80123456000

// 크래시 시점 스택
0: kd> kbn

// 크래시 시점 지역 변수
0: kd> dv

// 크래시 시점 소스 코드
0: kd> .srcpath+ c:\projects\mydriver\src
0: kd> lsa .

// Trap Frame 컨텍스트 해제
0: kd> .trap
```

---

## 12.4 주요 Bug Check 코드와 해결 방법

### 12.4.1 DRIVER_IRQL_NOT_LESS_OR_EQUAL (0x0A, 0xD1)

**가장 흔한 드라이버 버그!**

```
// 원인: 높은 IRQL에서 페이저블 메모리 접근

// 매개변수:
// Arg1: 접근하려던 메모리 주소
// Arg2: 접근 시 IRQL (보통 DISPATCH_LEVEL = 2)
// Arg3: 0 = 읽기, 1 = 쓰기
// Arg4: 오류 발생 코드 주소

// 흔한 원인들:
// 1. PagedPool 메모리를 DISPATCH_LEVEL에서 접근
// 2. 페이저블 함수를 높은 IRQL에서 호출
// 3. NULL 포인터 역참조 (주소가 작은 값)
```

```c
// 문제 코드 예시
NTSTATUS PreWrite(PFLT_CALLBACK_DATA Data, ...)
{
    // ❌ PagedPool 사용 - DISPATCH_LEVEL에서 접근 불가
    PVOID buffer = ExAllocatePoolWithTag(PagedPool, 1024, 'tseT');

    // ❌ 이 함수는 DISPATCH_LEVEL에서 호출될 수 있음!
    // PagedPool 메모리 접근 시 BSOD!

    ExFreePool(buffer);
}

// 수정된 코드
NTSTATUS PreWrite(PFLT_CALLBACK_DATA Data, ...)
{
    // ✅ NonPagedPool 사용 - 모든 IRQL에서 안전
    PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 1024, 'tseT');

    // 또는 IRQL 체크
    if (KeGetCurrentIrql() <= APC_LEVEL) {
        // PagedPool 사용 가능
    }

    ExFreePool(buffer);
}
```

### 12.4.2 PAGE_FAULT_IN_NONPAGED_AREA (0x50)

```
// 원인: 비페이저블 영역에서 페이지 폴트 발생

// 매개변수:
// Arg1: 참조된 가상 주소
// Arg2: 0 = 읽기, 1 = 쓰기
// Arg3: 오류 발생 코드 주소
// Arg4: 페이지 폴트 유형

// 흔한 원인:
// 1. 해제된 메모리 접근 (Use-After-Free)
// 2. 손상된 포인터
// 3. 스택 오버플로우
```

```c
// 문제 코드 예시
PVOID g_Buffer = NULL;

NTSTATUS CreateBuffer()
{
    g_Buffer = ExAllocatePoolWithTag(NonPagedPool, 1024, 'tseT');
    return STATUS_SUCCESS;
}

VOID FreeBuffer()
{
    ExFreePoolWithTag(g_Buffer, 'tseT');
    // ❌ g_Buffer를 NULL로 설정하지 않음!
}

NTSTATUS UseBuffer()
{
    // ❌ 이미 해제된 메모리 접근 - BSOD!
    RtlZeroMemory(g_Buffer, 1024);
}

// 수정된 코드
VOID FreeBuffer()
{
    if (g_Buffer != NULL) {
        ExFreePoolWithTag(g_Buffer, 'tseT');
        g_Buffer = NULL;  // ✅ NULL로 설정
    }
}

NTSTATUS UseBuffer()
{
    if (g_Buffer != NULL) {  // ✅ NULL 체크
        RtlZeroMemory(g_Buffer, 1024);
    }
}
```

### 12.4.3 KMODE_EXCEPTION_NOT_HANDLED (0x1E)

```
// 원인: 커널 모드에서 처리되지 않은 예외

// 매개변수:
// Arg1: 예외 코드 (예: 0xC0000005 = ACCESS_VIOLATION)
// Arg2: 예외 발생 주소
// Arg3: 예외 매개변수 1
// Arg4: 예외 매개변수 2

// 예외 코드:
// 0xC0000005 = STATUS_ACCESS_VIOLATION
// 0xC0000094 = STATUS_INTEGER_DIVIDE_BY_ZERO
// 0xC00000FD = STATUS_STACK_OVERFLOW
```

```c
// 0으로 나누기 예시
NTSTATUS CalculateRatio(ULONG total, ULONG count)
{
    // ❌ count가 0이면 BSOD!
    ULONG ratio = total / count;
    return STATUS_SUCCESS;
}

// 수정된 코드
NTSTATUS CalculateRatio(ULONG total, ULONG count)
{
    if (count == 0) {
        return STATUS_INVALID_PARAMETER;  // ✅ 0 체크
    }
    ULONG ratio = total / count;
    return STATUS_SUCCESS;
}
```

### 12.4.4 SYSTEM_SERVICE_EXCEPTION (0x3B)

```
// 원인: 시스템 서비스 루틴에서 예외 발생

// 매개변수:
// Arg1: 예외 코드
// Arg2: 예외 발생 주소
// Arg3: 예외 레코드 주소
// Arg4: 컨텍스트 레코드 주소

// 분석 방법:
0: kd> .exr Arg3    // 예외 레코드 분석
0: kd> .cxr Arg4    // 컨텍스트로 전환
0: kd> kb           // 스택 확인
```

### 12.4.5 KERNEL_DATA_INPAGE_ERROR (0x7A)

```
// 원인: 페이지 파일에서 데이터를 읽지 못함

// 매개변수:
// Arg1: 잠금 유형 (1, 2, 3)
// Arg2: 에러 상태
// Arg3: 가상 주소 또는 PTE
// Arg4: 페이지 파일 오프셋

// 원인:
// 1. 디스크 오류 (하드웨어 문제)
// 2. 디스크 컨트롤러 문제
// 3. 메모리 손상
// 4. 바이러스/악성코드

// 주로 하드웨어 문제 - 드라이버 코드 문제가 아닌 경우 많음
```

### 12.4.6 CRITICAL_STRUCTURE_CORRUPTION (0x109)

```
// 원인: 중요 커널 데이터 구조 손상 감지

// 보안 기능 (PatchGuard/Kernel Patch Protection)이 감지
// 드라이버가 커널 구조를 불법으로 수정한 경우

// 해결: 커널 데이터 구조를 직접 수정하지 말 것!
// 문서화된 API만 사용
```

---

## 12.5 드라이버별 분석 기법

### 12.5.1 Minifilter 크래시 분석

```
// Minifilter 관련 크래시는 !fltkd 확장 활용

// 1. 필터 상태 확인
0: kd> !fltkd.filters
Filter List: fffff8011234a000 "Frame 0"
   FLT_FILTER: fffff80112340000 "MyFilter" "123456"
      FLT_INSTANCE: fffff80112350000 "MyFilter Instance" "123456"
         Volume: fffff80112360000 "\Device\HarddiskVolume3"

// 2. 콜백 정보 확인
0: kd> !fltkd.filter fffff80112340000

// 3. 인스턴스 정보
0: kd> !fltkd.instance fffff80112350000

// 4. 볼륨 정보
0: kd> !fltkd.volume fffff80112360000
```

### 12.5.2 크래시 시점 IRP 분석

```
// 크래시 시점에 처리 중이던 IRP 확인

// 1. 현재 스레드의 IRP 리스트
0: kd> dt nt!_ETHREAD @$thread IrpList
   +0x660 IrpList : _LIST_ENTRY [...]

// 2. 스택에서 IRP 찾기
0: kd> !irp poi(@rsp+0x20)

// 3. FLT_CALLBACK_DATA에서 정보 추출
// (콜백 함수 내에서 크래시한 경우)
0: kd> dt fltmgr!_FLT_CALLBACK_DATA @rcx
```

### 12.5.3 메모리 손상 분석

```
// 풀 손상 확인
0: kd> !pool <주소>

// 풀 태그 확인
0: kd> !pooltag MyDr

// 특수 풀 활성화 (사후 분석용 아님, 재현 시 사용)
// verifier /flags 1 /driver MyDriver.sys

// 메모리 패턴 분석
0: kd> db <주소>
// 0xAB = 해제된 풀 메모리 (Debug)
// 0xBAADF00D = 초기화되지 않은 힙
// 0xDEADBEEF = 일반적인 "죽은" 포인터 마커
// 0xFEEEFEEE = 해제된 힙 메모리
```

---

## 12.6 실전: BSOD 디버깅 워크플로우

### 12.6.1 크래시 재현 및 분석 단계

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BSOD 분석 워크플로우                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 크래시 덤프 수집                                                │
│     └─ C:\Windows\MEMORY.DMP 또는 Minidump 폴더                     │
│                          ↓                                           │
│  2. WinDbg로 덤프 열기                                              │
│     └─ File > Open Crash Dump                                        │
│                          ↓                                           │
│  3. 심볼 경로 설정                                                  │
│     └─ .sympath srv*c:\symbols*https://msdl.microsoft.com/...       │
│     └─ .sympath+ <내 드라이버 PDB 경로>                             │
│                          ↓                                           │
│  4. !analyze -v 실행                                                │
│     └─ 버그 체크 코드, 원인 모듈, 스택 확인                         │
│                          ↓                                           │
│  5. 상세 분석                                                       │
│     └─ .trap / .cxr 로 크래시 시점 컨텍스트                         │
│     └─ kb, dv 로 스택, 변수 확인                                    │
│     └─ 소스 코드 연결 (.srcpath)                                    │
│                          ↓                                           │
│  6. 원인 파악 및 수정                                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.6.2 실전 분석 예시

**시나리오: PreWrite 콜백에서 BSOD 발생**

```
// 1. 덤프 열기 후 기본 분석
0: kd> !analyze -v

DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
...
FAULTING_IP:
MyDriver!PreWrite+45
fffff800`11112222 488b4118        mov     rax,qword ptr [rcx+18h]

STACK_TEXT:
fffff801`23456100 : MyDriver!PreWrite+0x45
fffff801`23456200 : fltmgr!FltpPerformPreCallbacks+0x123
...

SYMBOL_NAME:  MyDriver!PreWrite+45

// 2. 문제 코드 위치 확인
0: kd> u MyDriver!PreWrite+45
MyDriver!PreWrite+0x45:
fffff800`11112222 488b4118        mov     rax,qword ptr [rcx+18h]
fffff800`11112226 488945f0        mov     qword ptr [rbp-10h],rax
...

// 3. 소스 코드 연결
0: kd> .srcpath c:\projects\mydriver\src
0: kd> lsa MyDriver!PreWrite+45

// 소스 코드:
//    42:     PFLT_CALLBACK_DATA Data = ...
//    43:     PFLT_IO_PARAMETER_BLOCK Iopb = Data->Iopb;
// >  44:     PFILE_OBJECT FileObject = Iopb->TargetFileObject;  // 여기서 크래시!
//    45:     ...

// 4. rcx 값 확인 (Iopb 포인터)
0: kd> .trap <trap frame 주소>
0: kd> r rcx
rcx=0000000000000000    // NULL!

// 5. 원인: Iopb가 NULL인데 접근 시도
```

**수정 코드:**

```c
// 문제 코드
FLT_PREOP_CALLBACK_STATUS PreWrite(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext)
{
    PFLT_IO_PARAMETER_BLOCK Iopb = Data->Iopb;
    PFILE_OBJECT FileObject = Iopb->TargetFileObject;  // ❌ Iopb가 NULL일 수 있음!
    ...
}

// 수정된 코드
FLT_PREOP_CALLBACK_STATUS PreWrite(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext)
{
    // ✅ NULL 체크 추가
    if (Data == NULL || Data->Iopb == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    PFLT_IO_PARAMETER_BLOCK Iopb = Data->Iopb;
    PFILE_OBJECT FileObject = Iopb->TargetFileObject;
    ...
}
```

---

## 12.7 BSOD 예방을 위한 코딩 가이드라인

### 12.7.1 필수 체크 사항

```c
// 1. 모든 포인터 NULL 체크
if (ptr == NULL) {
    return STATUS_INVALID_PARAMETER;
}

// 2. IRQL 확인
KIRQL currentIrql = KeGetCurrentIrql();
if (currentIrql > APC_LEVEL) {
    // PagedPool 사용 불가
}

// 3. 버퍼 크기 검증
if (bufferSize < requiredSize) {
    return STATUS_BUFFER_TOO_SMALL;
}

// 4. 정수 오버플로우 방지
if (ULONG_MAX - a < b) {
    return STATUS_INTEGER_OVERFLOW;
}

// 5. 풀 할당 성공 확인
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, size, 'tseT');
if (buffer == NULL) {
    return STATUS_INSUFFICIENT_RESOURCES;
}
```

### 12.7.2 안전한 메모리 사용

```c
// 올바른 풀 유형 선택
//
// NonPagedPool: DISPATCH_LEVEL 이상에서 접근
// NonPagedPoolNx: 실행 불가 메모리 (보안 강화)
// PagedPool: APC_LEVEL 이하에서만 접근

// 해제 후 NULL 설정
if (g_Buffer != NULL) {
    ExFreePoolWithTag(g_Buffer, 'tseT');
    g_Buffer = NULL;
}

// Lookaside List 사용 (빈번한 할당/해제)
NPAGED_LOOKASIDE_LIST LookasideList;
ExInitializeNPagedLookasideList(&LookasideList, ...);
```

### 12.7.3 Driver Verifier 활용

```
// Driver Verifier 활성화 (개발 중 필수!)
// 관리자 권한 명령 프롬프트에서:

verifier /standard /driver MyDriver.sys

// 특정 옵션만:
verifier /flags 0x1 /driver MyDriver.sys    // 특수 풀
verifier /flags 0x2 /driver MyDriver.sys    // IRQL 체크
verifier /flags 0x4 /driver MyDriver.sys    // 풀 추적

// 상태 확인
verifier /query

// 비활성화
verifier /reset
```

---

## 12.8 Bug Check 코드 빠른 참조

| 코드 | 이름 | 주요 원인 |
|------|------|-----------|
| 0x0A | IRQL_NOT_LESS_OR_EQUAL | 높은 IRQL에서 페이저블 메모리 접근 |
| 0x1E | KMODE_EXCEPTION_NOT_HANDLED | 처리되지 않은 커널 예외 |
| 0x3B | SYSTEM_SERVICE_EXCEPTION | 시스템 서비스에서 예외 |
| 0x50 | PAGE_FAULT_IN_NONPAGED_AREA | 비페이저블 영역 페이지 폴트 |
| 0x7E | SYSTEM_THREAD_EXCEPTION_NOT_HANDLED | 시스템 스레드 예외 |
| 0x7F | UNEXPECTED_KERNEL_MODE_TRAP | 예기치 않은 커널 트랩 |
| 0x9F | DRIVER_POWER_STATE_FAILURE | 전원 상태 전환 실패 |
| 0xC4 | DRIVER_VERIFIER_DETECTED_VIOLATION | Verifier가 위반 감지 |
| 0xD1 | DRIVER_IRQL_NOT_LESS_OR_EQUAL | 드라이버 IRQL 오류 |

---

## 요약

이 챕터에서 학습한 내용:

1. **BSOD 원리**: KeBugCheckEx 호출, 덤프 생성, Stop 코드
2. **덤프 분석**: WinDbg로 열기, !analyze -v 실행
3. **주요 Bug Check**: 0xD1, 0x50, 0x1E 등의 원인과 해결책
4. **분석 워크플로우**: 덤프 수집 → 심볼 설정 → 분석 → 수정
5. **예방**: NULL 체크, IRQL 확인, Driver Verifier

다음 챕터에서는 Minifilter 특화 디버깅 기법을 학습합니다.

---

## 참고 자료

- [Bug Check Code Reference](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-code-reference)
- [Using !analyze Extension](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-analyze)
- [Driver Verifier](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/driver-verifier)
