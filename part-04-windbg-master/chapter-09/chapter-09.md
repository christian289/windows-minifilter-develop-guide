# Chapter 9: .NET 개발자를 위한 WinDbg 입문

## 난이도: ⭐⭐ (초급)

## 학습 목표

- Visual Studio 디버거 명령을 WinDbg로 매핑
- WinDbg 인터페이스 구성 요소 이해
- 기본적인 디버깅 워크플로우 습득

---

## 들어가며

10년 넘게 Visual Studio의 강력한 디버거를 사용해온 .NET 개발자에게 WinDbg는 마치 "원시시대로 돌아간 것 같은" 느낌을 줄 수 있습니다. GUI 대신 명령줄, 마우스 클릭 대신 타이핑.

하지만 WinDbg는 커널 디버깅의 **유일한 선택지**입니다. 그리고 놀랍게도, 익숙해지면 오히려 더 강력하다는 것을 알게 될 것입니다.

이 장에서는 VS 디버거 경험을 WinDbg로 매핑하여 빠르게 적응할 수 있도록 돕습니다.

---

## 9.1 왜 WinDbg인가?

### 9.1.1 Visual Studio 디버거의 한계

VS 디버거는 사용자 모드 .NET/C++ 디버깅에는 최고지만:

```
VS 디버거의 한계:
├── ❌ 커널 모드 디버깅 미지원
│   └── 드라이버 코드를 VS에서 직접 디버그할 수 없음
├── ❌ 덤프 분석 기능 제한
│   └── BSOD 덤프 분석 불가
├── ❌ 저수준 메모리 분석 어려움
│   └── 커널 구조체 탐색 제한적
└── ❌ 원격 커널 디버깅 미지원
    └── 다른 시스템의 커널에 연결 불가
```

### 9.1.2 WinDbg의 강점

```
WinDbg의 강점:
├── ✅ 커널/사용자 모드 모두 지원
│   └── 하나의 도구로 모든 수준 디버깅
├── ✅ 강력한 덤프 분석
│   └── BSOD, 행 덤프, 크래시 덤프 모두 분석
├── ✅ 확장 명령어 생태계
│   └── !analyze, !process, !fltkd 등
├── ✅ 스크립팅 지원
│   └── 반복 작업 자동화
└── ✅ 원격 디버깅
    └── 네트워크/시리얼로 다른 시스템 디버그
```

### 9.1.3 WinDbg는 어렵지 않다

VS 디버거에서 하던 작업을 WinDbg에서도 할 수 있습니다. 다만 **명령어로 입력**할 뿐입니다:

```
VS 디버거에서:                    WinDbg에서:
┌─────────────────────────────┐   ┌─────────────────────────────┐
│ F5 버튼 클릭                 │ → │ g 입력 후 Enter             │
│ Watch 창에 변수 추가         │ → │ ?? 변수명                   │
│ Call Stack 창 확인           │ → │ k                           │
│ Memory 창에서 주소 조회      │ → │ db 주소                     │
└─────────────────────────────┘   └─────────────────────────────┘
```

---

## 9.2 VS 디버거 → WinDbg 명령어 매핑

### 9.2.1 실행 제어

| VS 디버거 | 단축키 | WinDbg 명령 | 설명 |
|-----------|--------|-------------|------|
| Continue | F5 | `g` | 실행 계속 |
| Break All | Ctrl+Alt+Break | Ctrl+Break | 실행 중단 |
| Stop Debugging | Shift+F5 | `q` 또는 `.detach` | 디버깅 종료 |
| Restart | Ctrl+Shift+F5 | `.reboot` | 대상 재시작 |
| Step Over | F10 | `p` | 현재 줄 실행 (함수 건너뜀) |
| Step Into | F11 | `t` | 함수 내부로 진입 |
| Step Out | Shift+F11 | `gu` | 현재 함수 탈출 |
| Run to Cursor | Ctrl+F10 | `g <주소>` | 특정 위치까지 실행 |

```
WinDbg 실행 제어 예시:
0: kd> g                    ; 실행 계속 (F5)
0: kd> p                    ; 한 줄 실행 (F10)
0: kd> t                    ; 함수 진입 (F11)
0: kd> gu                   ; 함수 탈출 (Shift+F11)
0: kd> g MyDriver!MyFunc    ; MyFunc까지 실행
```

### 9.2.2 브레이크포인트

| VS 디버거 | WinDbg 명령 | 설명 |
|-----------|-------------|------|
| Toggle Breakpoint (F9) | `bp <주소/함수>` | BP 설정 |
| Delete Breakpoint | `bc <번호>` | BP 삭제 |
| Disable Breakpoint | `bd <번호>` | BP 비활성화 |
| Enable Breakpoint | `be <번호>` | BP 활성화 |
| Breakpoints Window | `bl` | BP 목록 보기 |
| Conditional Breakpoint | `bp <주소> "조건"` | 조건부 BP |

```
WinDbg 브레이크포인트 예시:
0: kd> bp MyDriver!DriverEntry      ; 함수에 BP 설정
0: kd> bp fffff801`12345678         ; 주소에 BP 설정
0: kd> bl                           ; BP 목록 확인
 0 e fffff801`12345678 MyDriver!DriverEntry
 1 d fffff801`87654321 MyDriver!Unload
0: kd> bd 1                         ; BP 1번 비활성화
0: kd> bc *                         ; 모든 BP 삭제
```

### 9.2.3 변수/메모리 검사

| VS 디버거 창 | WinDbg 명령 | 설명 |
|-------------|-------------|------|
| Watch Window | `?? 표현식` | C++ 표현식 평가 |
| Locals Window | `dv` | 지역 변수 표시 |
| Autos Window | `dv /t` | 타입과 함께 표시 |
| Call Stack Window | `k` | 호출 스택 |
| Memory Window | `db/dw/dd/dq` | 메모리 표시 |
| QuickWatch (Shift+F9) | `dt 타입 주소` | 구조체 표시 |

```
WinDbg 변수/메모리 검사 예시:
0: kd> dv                           ; 지역 변수 (Locals)
          status = 0n0
          buffer = 0xffffc001`12345678

0: kd> ?? buffer                    ; 표현식 평가 (Watch)
void * 0xffffc001`12345678

0: kd> k                            ; 호출 스택 (Call Stack)
 # Child-SP          RetAddr           Call Site
00 ffffd001`234567a0 fffff801`12345678 MyDriver!MyFunction
01 ffffd001`234567d0 fffff801`87654321 MyDriver!DispatchCreate+0x45

0: kd> db ffffc001`12345678         ; 메모리 바이트 (Memory)
ffffc001`12345678  48 89 5c 24 08 57 48 83-ec 20 48 8b d9 48 8b fa
```

### 9.2.4 코드 탐색

| VS 디버거 | WinDbg 명령 | 설명 |
|-----------|-------------|------|
| Go to Definition | `x 패턴` | 심볼 검색 |
| Find Symbol | `x 모듈!*패턴*` | 패턴으로 검색 |
| Go to Address | `u 주소` | 디스어셈블리 |
| - | `ln 주소` | 가장 가까운 심볼 |

```
WinDbg 코드 탐색 예시:
0: kd> x MyDriver!*Create*          ; Create 포함 심볼 검색
fffff801`12345000 MyDriver!PreCreate
fffff801`12345100 MyDriver!PostCreate

0: kd> u MyDriver!PreCreate         ; 디스어셈블리
MyDriver!PreCreate:
fffff801`12345000 48895c2408      mov     qword ptr [rsp+8],rbx
fffff801`12345005 57              push    rdi

0: kd> ln fffff801`12345050         ; 가장 가까운 심볼
(fffff801`12345000)   MyDriver!PreCreate+0x50
```

---

## 9.3 WinDbg 인터페이스 (Preview)

### 9.3.1 기본 레이아웃

```
┌─────────────────────────────────────────────────────────────────────┐
│ File  Home  View  Debug  Layout                                      │
├─────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────┐ ┌─────────────────────────────────┐ │
│ │       Source Window          │ │        Disassembly               │ │
│ │  (소스 코드 표시)             │ │  (어셈블리 코드)                 │ │
│ │                              │ │                                  │ │
│ │  1: NTSTATUS DriverEntry... │ │  fffff801`12345000 mov rax,...  │ │
│ │  2: {                        │ │  fffff801`12345005 push rbx     │ │
│ │> 3:     NTSTATUS status;     │ │                                  │ │
│ │                              │ │                                  │ │
│ └─────────────────────────────┘ └─────────────────────────────────┘ │
│ ┌─────────────────────────────┐ ┌─────────────────────────────────┐ │
│ │       Command Window         │ │        Stack / Locals            │ │
│ │                              │ │                                  │ │
│ │ 0: kd> _                     │ │  # Call Site                     │ │
│ │                              │ │  0 MyDriver!DriverEntry          │ │
│ │ Connected to...              │ │  1 nt!IopLoadDriver              │ │
│ │                              │ │                                  │ │
│ └─────────────────────────────┘ └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.3.2 주요 창

| 창 | 단축키 | 설명 |
|----|--------|------|
| Command | Alt+1 | 명령어 입력 (핵심!) |
| Source | - | 소스 코드 표시 |
| Disassembly | Alt+5 | 어셈블리 코드 |
| Registers | Alt+4 | CPU 레지스터 |
| Stack | Alt+6 | 호출 스택 |
| Locals | Alt+3 | 지역 변수 |
| Memory | Alt+5 | 메모리 뷰 |

### 9.3.3 VS 디버거와 레이아웃 비교

```
VS 디버거:                         WinDbg Preview:
┌───────────────────────────┐     ┌───────────────────────────┐
│ Solution Explorer         │     │ (없음 - 필요 없음)        │
├───────────────────────────┤     ├───────────────────────────┤
│ Code Editor               │     │ Source / Disassembly      │
├───────────────────────────┤     ├───────────────────────────┤
│ Watch / Locals / Autos    │     │ Command (핵심!) / Locals  │
├───────────────────────────┤     ├───────────────────────────┤
│ Output / Error List       │     │ Command Window에 통합     │
└───────────────────────────┘     └───────────────────────────┘
```

> 💡 **핵심**: WinDbg에서 가장 중요한 것은 **Command Window**입니다. 대부분의 작업은 명령어로 수행합니다.

---

## 9.4 첫 번째 디버깅 세션

### 9.4.1 연결 후 기본 탐색

VM에 연결되면 다음 명령으로 상태를 확인합니다:

```
:: 대상 시스템 정보 확인
0: kd> vertarget
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Machine Name: DRIVERTEST-VM
Kernel base = 0xfffff800`12000000 PsLoadedModuleList = 0xfffff800`12c2a2d0

:: 로드된 모듈 목록
0: kd> lm
start             end                 module name
fffff800`12000000 fffff800`12e00000   nt         (pdb symbols)
fffff800`13000000 fffff800`13100000   hal        (deferred)
fffff800`14000000 fffff800`14050000   fltmgr     (pdb symbols)
fffff801`11000000 fffff801`11010000   MyDriver   (private pdb symbols)
...

:: 특정 모듈 찾기 (패턴)
0: kd> lm m My*
start             end                 module name
fffff801`11000000 fffff801`11010000   MyDriver   (private pdb symbols)

:: 현재 프로세스/스레드
0: kd> !process -1 0
PROCESS ffffc001`12345678
    SessionId: 0  Cid: 0004    Peb: 00000000  ParentCid: 0000
    DirBase: 001ab000  ObjectTable: ffffc000`11111111  HandleCount: 2500
    Image: System
```

### 9.4.2 심볼 로드 확인

심볼은 디버깅의 핵심입니다. 심볼이 없으면 함수 이름 대신 주소만 보입니다:

```
:: 심볼 경로 확인
0: kd> .sympath
Symbol search path is: srv*C:\Symbols*https://msdl.microsoft.com/download/symbols

:: 심볼 상태 확인 (noisy 모드)
0: kd> !sym noisy
noisy mode - symbol prompts on

:: 심볼 강제 로드
0: kd> .reload /f
Loading Kernel Symbols
...
Loading User Symbols
...

:: 특정 모듈 심볼 로드
0: kd> .reload /f MyDriver.sys
Loading symbols for fffff801`11000000     MyDriver.sys

:: 심볼 로드 결과 확인
0: kd> lm m MyDriver
start             end                 module name
fffff801`11000000 fffff801`11010000   MyDriver   (private pdb symbols)
                                                  ↑ 심볼 로드 성공!
```

### 9.4.3 Break와 Go

```
:: 실행 중인 대상 중단
Ctrl+Break

:: 현재 상태 확인
0: kd> r
rax=0000000000000000 rbx=ffffc001aaaabbbb rcx=0000000000000001
rdx=0000000000000000 rsi=ffffc001ccccdddd rdi=0000000000000000
rip=fffff80012345678 rsp=ffffd00123456780 rbp=0000000000000000
...

:: 실행 계속
0: kd> g
```

---

## 9.5 도움말 사용법

WinDbg에는 방대한 도움말이 내장되어 있습니다.

### 9.5.1 명령어 도움말

```
:: 특정 명령어 도움말 (브라우저로 열림)
0: kd> .hh bp

:: 명령어 간단 사용법
0: kd> ?
  ... 기본 명령어 목록 ...

:: 메타 명령어 도움말
0: kd> .help
  ... 메타 명령어 목록 ...
```

### 9.5.2 확장 명령어 도움말

```
:: 모든 확장 도움말
0: kd> !help

:: 특정 확장 도움말
0: kd> !fltkd.help
Filter Manager Debugger Extensions
===================================
!fltkd.filters    - List all filters
!fltkd.filter     - Display filter details
!fltkd.instances  - List all instances
...

:: 개별 확장 명령어 도움말
0: kd> !fltkd.filter -?
```

---

## 9.6 자주 사용하는 명령어 치트시트

### VS 디버거 작업별 WinDbg 명령

```
┌────────────────────────────────────────────────────────────────────┐
│ "F5로 실행하고 싶어요"                                              │
│   → g                                                              │
├────────────────────────────────────────────────────────────────────┤
│ "F9로 여기에 브레이크포인트 걸고 싶어요"                             │
│   → bp MyDriver!MyFunction                                         │
├────────────────────────────────────────────────────────────────────┤
│ "이 변수 값을 보고 싶어요"                                          │
│   → ?? 변수명                                                      │
├────────────────────────────────────────────────────────────────────┤
│ "Call Stack을 보고 싶어요"                                         │
│   → k                                                              │
├────────────────────────────────────────────────────────────────────┤
│ "이 메모리 주소의 내용을 보고 싶어요"                                │
│   → db 주소 (바이트) / dd 주소 (4바이트) / dq 주소 (8바이트)        │
├────────────────────────────────────────────────────────────────────┤
│ "이 구조체의 내용을 보고 싶어요"                                     │
│   → dt 타입명 주소                                                 │
├────────────────────────────────────────────────────────────────────┤
│ "어떤 함수들이 있는지 찾고 싶어요"                                   │
│   → x MyDriver!*                                                   │
├────────────────────────────────────────────────────────────────────┤
│ "현재 프로세스 정보를 보고 싶어요"                                   │
│   → !process -1 0                                                  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 정리

### VS 디버거 → WinDbg 핵심 매핑

| VS 디버거 | WinDbg | 기억법 |
|-----------|--------|--------|
| F5 (Continue) | `g` | **G**o |
| F10 (Step Over) | `p` | ste**P** over |
| F11 (Step Into) | `t` | s**T**ep into |
| Shift+F11 (Step Out) | `gu` | **G**o **U**p |
| F9 (Breakpoint) | `bp` | **B**reak**P**oint |
| Watch | `??` | 평가(**?**) 두 번 |
| Locals | `dv` | **D**isplay **V**ariables |
| Call Stack | `k` | stac**K** |
| Memory | `db/dd/dq` | **D**isplay **B**ytes/DWord/Qword |

### 다음 장 미리보기

Chapter 10에서는 WinDbg 명령어를 체계적으로 학습합니다:
- 실행 제어 명령어 상세
- 브레이크포인트 종류와 활용
- 메모리 검사 명령어
- 스택 분석

---

## 연습 문제

### 1. 명령어 변환

다음 VS 디버거 동작을 WinDbg 명령으로 변환하세요:

1. F11로 함수 안으로 진입
2. 변수 `status`의 값 확인
3. 호출 스택 확인

### 2. 심볼 문제

`lm` 명령으로 모듈을 확인했더니 `(deferred)`라고 표시됩니다. 심볼을 로드하려면?

### 3. 브레이크포인트

`MyDriver!DriverEntry` 함수에 브레이크포인트를 설정하는 명령은?

---

## 참고 자료

- [Getting Started with WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg)
- [Debugger Commands](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-commands)
- [Symbol Path](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/symbol-path)
