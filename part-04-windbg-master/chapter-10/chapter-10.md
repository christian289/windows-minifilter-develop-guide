# Chapter 10: WinDbg 핵심 명령어 체계

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- WinDbg 명령어 체계(표준, 메타, 확장)를 이해하고 구분합니다
- 실행 제어 명령어로 프로그램 흐름을 추적합니다
- 메모리 덤프 명령어로 데이터를 분석합니다
- 심볼과 소스 코드를 연결하여 디버깅합니다
- 브레이크포인트를 효과적으로 활용합니다

## 도입: Visual Studio 디버거에서 WinDbg로

.NET 개발자에게 Visual Studio 디버거는 친숙한 GUI 도구입니다. 마우스 클릭으로 브레이크포인트를 설정하고, Watch 창에서 변수를 확인합니다. WinDbg는 같은 기능을 **명령어**로 수행합니다. 처음에는 불편해 보이지만, 명령어 기반 디버깅은 커널 개발에서 더 강력하고 유연합니다.

---

## 10.1 WinDbg 명령어 체계 이해

### 10.1.1 세 가지 명령어 유형

WinDbg는 세 가지 종류의 명령어를 제공합니다:

| 유형 | 접두사 | 설명 | 예시 |
|------|--------|------|------|
| **표준 명령어** | 없음 | WinDbg 내장 기본 명령어 | `g`, `p`, `t`, `k` |
| **메타 명령어** | `.` (점) | 디버거 자체 제어 | `.cls`, `.reload`, `.sympath` |
| **확장 명령어** | `!` (느낌표) | 확장 DLL 제공 기능 | `!analyze`, `!process`, `!fltkd.filters` |

```
// C# 비유로 이해하기

// 표준 명령어 = 기본 C# 키워드 (if, for, while)
g                    // go (계속 실행)
p                    // step over (한 줄 실행)

// 메타 명령어 = Visual Studio IDE 설정
.cls                 // 화면 지우기
.sympath            // 심볼 경로 설정

// 확장 명령어 = NuGet 패키지 기능
!analyze -v         // 자동 분석 (WinDbg 확장)
!fltkd.filters      // Minifilter 목록 (fltkd 확장)
```

### 10.1.2 명령어 도움말 확인

```
// 모든 명령어 도움말 규칙

// 표준 명령어: ? 또는 .hh 명령어명
? bp                 // bp 명령어 간단 설명
.hh bp              // bp 명령어 상세 도움말 (CHM 파일 열기)

// 메타 명령어: .help 또는 .hh .명령어명
.help               // 메타 명령어 목록
.hh .reload         // .reload 상세 도움말

// 확장 명령어: !확장.help
!fltkd.help         // fltkd 확장 명령어 목록
!analyze -?         // analyze 사용법
```

---

## 10.2 실행 제어 명령어

### 10.2.1 기본 실행 명령어

실행 제어는 디버깅의 핵심입니다. Visual Studio의 F5, F10, F11에 대응하는 명령어를 익힙니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     실행 제어 명령어 비교                            │
├─────────────────────────────────────────────────────────────────────┤
│  Visual Studio          WinDbg          설명                        │
├─────────────────────────────────────────────────────────────────────┤
│  F5 (Continue)          g               계속 실행                   │
│  F10 (Step Over)        p               한 줄 실행 (함수 진입 X)    │
│  F11 (Step Into)        t               한 줄 실행 (함수 진입 O)    │
│  Shift+F11 (Step Out)   gu              현재 함수 빠져나가기        │
│  Shift+F5 (Stop)        .kill / q       디버깅 중지                 │
│  Ctrl+Break             Ctrl+Break      실행 중단                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.2.2 실행 명령어 상세

```
// g (Go) - 계속 실행
g                    // 다음 브레이크포인트까지 실행
g <address>          // 특정 주소까지 실행
g @@(MyFunction)     // MyFunction까지 실행 (심볼 사용)

// 실제 사용 예:
0: kd> g
Breakpoint 0 hit
MyDriver!DriverEntry:
fffff801`12340000 mov     rbp,rsp

// p (steP) - Step Over
p                    // 한 명령어 실행 (함수 호출 시 함수 전체 실행)
p 5                  // 5개 명령어 실행
pc                   // 다음 call 명령어까지 실행
pt                   // 다음 return까지 실행

// t (Trace) - Step Into
t                    // 한 명령어 실행 (함수 호출 시 함수 내부 진입)
t 5                  // 5개 명령어 실행 (함수 진입 포함)
tc                   // 다음 call 명령어까지 (함수 진입 포함)
tt                   // 다음 return까지 (함수 진입 포함)

// gu (Go Up) - Step Out
gu                   // 현재 함수에서 리턴할 때까지 실행
```

### 10.2.3 p와 t 명령어 동작 비교

```c
// 예제 코드
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;

    status = InitializeDriver();    // <- 현재 위치
    if (!NT_SUCCESS(status)) {
        return status;
    }

    return STATUS_SUCCESS;
}

NTSTATUS InitializeDriver()
{
    // 초기화 로직...
    return STATUS_SUCCESS;
}
```

```
// p 명령어 사용 (Step Over)
0: kd> p
// InitializeDriver() 함수 전체가 실행되고
// if (!NT_SUCCESS(status)) 줄에서 멈춤

// t 명령어 사용 (Step Into)
0: kd> t
// InitializeDriver() 함수 내부 첫 줄에서 멈춤
```

---

## 10.3 브레이크포인트 명령어

### 10.3.1 브레이크포인트 유형

```
┌─────────────────────────────────────────────────────────────────────┐
│                     브레이크포인트 유형                              │
├─────────────────────────────────────────────────────────────────────┤
│  명령어      유형              설명                                  │
├─────────────────────────────────────────────────────────────────────┤
│  bp         소프트웨어        int 3 명령어 삽입                      │
│  bu         Unresolved        심볼 로드 시 자동 설정                 │
│  ba         하드웨어(Access)  하드웨어 디버그 레지스터 사용          │
│  bm         패턴 매칭         와일드카드로 여러 함수에 설정          │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.3.2 bp (Breakpoint) - 기본 브레이크포인트

```
// 기본 문법
bp <address>         // 주소에 브레이크포인트 설정
bp <module>!<symbol> // 심볼에 브레이크포인트 설정

// 실제 사용 예:
bp MyDriver!DriverEntry          // DriverEntry 함수에 설정
bp MyDriver!PreCreate            // PreCreate 콜백에 설정
bp fffff801`12345678             // 특정 주소에 설정

// 조건부 브레이크포인트
bp MyDriver!PreWrite ".if (poi(@rcx+0x10) == 0x1234) {.echo Match!} .else {gc}"
// rcx+0x10 위치 값이 0x1234일 때만 멈춤

// 명령어 실행 브레이크포인트
bp MyDriver!PreCreate "kb; g"    // 콜스택 출력 후 계속 실행
```

### 10.3.3 bu (Breakpoint Unresolved) - 지연 브레이크포인트

```
// 아직 로드되지 않은 모듈에 브레이크포인트 설정
bu MyDriver!DriverEntry

// 드라이버가 로드되면 자동으로 브레이크포인트 활성화
// .NET의 Lazy<T>와 유사한 개념
```

**bp vs bu 차이점:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  구분          bp                          bu                       │
├─────────────────────────────────────────────────────────────────────┤
│  심볼 필요     설정 시점에 필요             나중에 로드되어도 됨     │
│  모듈 언로드   삭제됨                       유지됨                   │
│  사용 시점     이미 로드된 모듈             아직 로드되지 않은 모듈  │
│  추천          일반적인 경우                드라이버 Entry 디버깅    │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.3.4 ba (Break on Access) - 하드웨어 브레이크포인트

```
// 문법: ba <access> <size> <address>
// access: r (읽기), w (쓰기), e (실행), i (I/O)
// size: 1, 2, 4, 8 (바이트)

// 메모리 쓰기 감시
ba w4 MyDriver!g_GlobalCounter    // 4바이트 쓰기 시 중단

// 메모리 읽기 감시
ba r8 fffff801`12345678           // 8바이트 읽기 시 중단

// 실행 감시 (코드 패치 감지)
ba e1 MyDriver!DriverEntry        // 코드 실행 시 중단
```

**하드웨어 브레이크포인트 제한:**
- x64에서 최대 4개까지만 설정 가능 (DR0-DR3 레지스터 사용)
- 매우 빠름 (CPU 레벨에서 감지)

### 10.3.5 bm (Breakpoint Match) - 패턴 매칭 브레이크포인트

```
// 와일드카드로 여러 함수에 한번에 설정
bm MyDriver!Pre*                  // Pre로 시작하는 모든 함수
bm MyDriver!*Create*              // Create가 포함된 모든 함수
bm nt!Nt*File                     // nt 모듈의 Nt...File 함수들

// 설정 결과 확인:
0: kd> bm MyDriver!Pre*
  1: fffff801`12340100 @!"MyDriver!PreCreate"
  2: fffff801`12340200 @!"MyDriver!PreRead"
  3: fffff801`12340300 @!"MyDriver!PreWrite"
breakpoint 1 redefined
```

### 10.3.6 브레이크포인트 관리

```
// 목록 확인
bl                              // 모든 브레이크포인트 목록

// 출력 예:
0: kd> bl
     0 e Disable Clear  fffff801`12340100  0001 (0001) MyDriver!DriverEntry
     1 e Disable Clear  fffff801`12340200  0001 (0001) MyDriver!PreCreate
     2 d Disable Clear  fffff801`12340300  0001 (0001) MyDriver!PreWrite

// 상태: e(enabled), d(disabled)

// 비활성화/활성화
bd 1                            // 1번 브레이크포인트 비활성화
be 1                            // 1번 브레이크포인트 활성화
bd *                            // 모든 브레이크포인트 비활성화

// 삭제
bc 1                            // 1번 브레이크포인트 삭제
bc *                            // 모든 브레이크포인트 삭제
```

---

## 10.4 메모리 덤프 명령어

### 10.4.1 d (Display) 명령어 계열

메모리 내용을 다양한 형식으로 표시합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      메모리 덤프 명령어                              │
├─────────────────────────────────────────────────────────────────────┤
│  명령어      형식              설명                                  │
├─────────────────────────────────────────────────────────────────────┤
│  db         byte              바이트 단위 (16진수 + ASCII)          │
│  dw         word              2바이트 단위                          │
│  dd         dword             4바이트 단위                          │
│  dq         qword             8바이트 단위 (x64 포인터)             │
│  dp         pointer           포인터 크기 (32비트=4, 64비트=8)      │
│  da         ASCII             ASCII 문자열                          │
│  du         Unicode           유니코드 문자열                       │
│  dS         UNICODE_STRING    UNICODE_STRING 구조체                 │
│  ds         ANSI_STRING       ANSI_STRING 구조체                    │
│  dt         type              구조체 타입 덤프                      │
│  dv         variables         지역 변수                             │
│  dyb        binary            바이너리 형식                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.4.2 바이트/워드/더블워드/쿼드워드 덤프

```
// db - 바이트 덤프 (16진수 + ASCII)
0: kd> db fffff801`12345000
fffff801`12345000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
fffff801`12345010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......

// dw - 워드 덤프 (2바이트)
0: kd> dw fffff801`12345000
fffff801`12345000  5a4d 0090 0003 0000 0004 0000 ffff 0000

// dd - 더블워드 덤프 (4바이트)
0: kd> dd fffff801`12345000
fffff801`12345000  00905a4d 00000003 00000004 0000ffff

// dq - 쿼드워드 덤프 (8바이트) - x64에서 포인터 크기
0: kd> dq fffff801`12345000
fffff801`12345000  00000003`00905a4d 0000ffff`00000004

// 길이 지정
db fffff801`12345000 L100    // 256바이트(0x100) 덤프
```

### 10.4.3 문자열 덤프

```
// da - ASCII 문자열
0: kd> da fffff801`12345100
fffff801`12345100  "MyDriver.sys"

// du - 유니코드 문자열
0: kd> du fffff801`12345200
fffff801`12345200  "C:\Windows\System32\drivers\MyDriver.sys"

// dS - UNICODE_STRING 구조체 (커널에서 많이 사용)
0: kd> dS fffff801`12345300
String(32,34) at fffff801`12345300: "\Device\MyDevice"
```

```c
// C# 비유
// da = Encoding.ASCII.GetString(bytes)
// du = Encoding.Unicode.GetString(bytes)

// UNICODE_STRING 구조체 (커널 문자열)
typedef struct _UNICODE_STRING {
    USHORT Length;           // 현재 문자열 길이 (바이트)
    USHORT MaximumLength;    // 버퍼 최대 크기 (바이트)
    PWCH   Buffer;           // 실제 문자열 포인터
} UNICODE_STRING;

// C#의 string과 달리 null-terminated가 아닐 수 있음!
```

### 10.4.4 dt (Display Type) - 구조체 덤프

`dt`는 가장 자주 사용하는 명령어 중 하나입니다.

```
// 구조체 정의 보기
dt nt!_EPROCESS                  // EPROCESS 구조체 정의
dt nt!_IRP                       // IRP 구조체 정의
dt fltmgr!_FLT_CALLBACK_DATA     // FLT_CALLBACK_DATA 구조체

// 특정 주소의 구조체 내용 보기
dt nt!_EPROCESS fffff801`12345000

// 출력 예:
0: kd> dt nt!_EPROCESS fffff801`12345000
   +0x000 Pcb              : _KPROCESS
   +0x2e0 ProcessLock      : _EX_PUSH_LOCK
   +0x2e8 UniqueProcessId  : 0x00000000`00001234
   +0x2f0 ActiveProcessLinks : _LIST_ENTRY
   ...

// 특정 필드만 보기
dt nt!_EPROCESS fffff801`12345000 UniqueProcessId
dt nt!_EPROCESS fffff801`12345000 ImageFileName

// 중첩 구조체 펼치기 (-r 옵션)
dt -r1 nt!_EPROCESS fffff801`12345000    // 1단계 중첩까지
dt -r2 nt!_EPROCESS fffff801`12345000    // 2단계 중첩까지
```

### 10.4.5 dv (Display Variables) - 지역 변수

```
// 현재 함수의 지역 변수 표시
dv                              // 모든 지역 변수
dv /t                           // 타입 정보 포함
dv /V                           // 주소 정보 포함
dv /i                           // 변수 분류 (parameter, local)

// 출력 예:
0: kd> dv /t /V
prv param  fffff801`12345000 @rcx   struct _DRIVER_OBJECT * DriverObject
prv param  fffff801`12345008 @rdx   struct _UNICODE_STRING * RegistryPath
prv local  fffff801`12345010 @rsp+20 unsigned long status
```

---

## 10.5 레지스터와 스택 명령어

### 10.5.1 r (Registers) - 레지스터 확인/수정

```
// 모든 레지스터 표시
r

// 출력 예:
0: kd> r
rax=0000000000000000 rbx=fffff801123450a0 rcx=fffff80112345000
rdx=fffff80112345100 rsi=0000000000000000 rdi=fffff80112345200
rip=fffff80112340000 rsp=fffff80112348000 rbp=fffff80112348100
 r8=0000000000000000  r9=0000000000000001 r10=0000000000000000
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246

// 특정 레지스터만 표시
r rax                           // rax 레지스터
r rcx, rdx                      // rcx, rdx 레지스터
r rip                           // 현재 명령어 포인터

// 레지스터 수정
r rax=0x1234                    // rax를 0x1234로 변경
r @rcx=0                        // rcx를 0으로 변경 (@ 접두사 사용 가능)
```

### 10.5.2 x64 호출 규약과 레지스터

```
┌─────────────────────────────────────────────────────────────────────┐
│                   x64 Windows 호출 규약 (fastcall)                   │
├─────────────────────────────────────────────────────────────────────┤
│  매개변수      레지스터        참고                                  │
├─────────────────────────────────────────────────────────────────────┤
│  1번째         RCX             정수/포인터                          │
│  2번째         RDX             정수/포인터                          │
│  3번째         R8              정수/포인터                          │
│  4번째         R9              정수/포인터                          │
│  5번째 이상    스택            RSP+0x28, RSP+0x30, ...              │
│  반환값        RAX             NTSTATUS 등                          │
├─────────────────────────────────────────────────────────────────────┤
│  부동소수점    XMM0-XMM3       1-4번째 float/double 매개변수        │
└─────────────────────────────────────────────────────────────────────┘
```

```c
// 예: PreOperation 콜백
FLT_PREOP_CALLBACK_STATUS
PreOperationCallback(
    PFLT_CALLBACK_DATA Data,       // RCX
    PCFLT_RELATED_OBJECTS FltObjects,  // RDX
    PVOID *CompletionContext       // R8
);

// 디버깅 시:
// rcx = Data 포인터
// rdx = FltObjects 포인터
// r8 = CompletionContext 포인터
```

### 10.5.3 k (Stack) - 콜스택 확인

```
// 기본 콜스택
k                               // 기본 콜스택

// 출력 예:
0: kd> k
 # Child-SP          RetAddr           Call Site
00 fffff801`12348000 fffff801`11110000 MyDriver!PreCreate
01 fffff801`12348100 fffff801`11120000 fltmgr!FltpPerformPreCallbacks
02 fffff801`12348200 fffff801`11130000 fltmgr!FltpPassThrough
03 fffff801`12348300 fffff801`11140000 nt!IofCallDriver
04 fffff801`12348400 fffff801`11150000 nt!IoCallDriver
...

// 다양한 콜스택 옵션
kb                              // 처음 3개 매개변수 포함
kp                              // 모든 매개변수 (심볼 있을 때)
kv                              // FPO 정보, 호출 규약 포함
kn                              // 프레임 번호 포함
kf                              // 프레임 크기 포함

// 프레임 수 제한
k 5                             // 처음 5개 프레임만
k 20                            // 처음 20개 프레임

// 가장 유용한 조합
kbn                             // 프레임 번호 + 3개 매개변수
```

### 10.5.4 .frame - 스택 프레임 전환

```
// 특정 스택 프레임으로 컨텍스트 전환
.frame 2                        // 2번 프레임으로 전환

// 전환 후 지역 변수 확인 가능
.frame 2
dv                              // 2번 프레임의 지역 변수

// 프레임 정보 표시
.frame /c 2                     // 2번 프레임 컨텍스트 설정 + 정보 표시
```

---

## 10.6 심볼 명령어

### 10.6.1 심볼 경로 설정

```
// 현재 심볼 경로 확인
.sympath

// 심볼 경로 설정
.sympath srv*c:\symbols*https://msdl.microsoft.com/download/symbols

// 심볼 경로 추가 (+)
.sympath+ c:\mydriver\x64\Debug

// 권장 심볼 경로 구성:
.sympath srv*c:\symbols*https://msdl.microsoft.com/download/symbols
.sympath+ c:\Projects\MyDriver\x64\Debug
```

### 10.6.2 심볼 로드

```
// 심볼 리로드
.reload                         // 변경된 심볼 리로드
.reload /f                      // 강제 리로드
.reload /f MyDriver.sys         // 특정 모듈만 리로드
.reload /user                   // 사용자 모드 모듈 리로드

// 심볼 로드 상태 확인
lm                              // 로드된 모든 모듈
lm m MyDriver                   // MyDriver 모듈 정보
lm v m MyDriver                 // 상세 정보 (심볼 경로 포함)

// 출력 예:
0: kd> lm v m MyDriver
Browse full module list
start             end                 module name
fffff801`12340000 fffff801`12350000   MyDriver   (private pdb symbols)
    Loaded symbol image file: MyDriver.sys
    Mapped memory image file: c:\Projects\MyDriver\x64\Debug\MyDriver.sys
    Symbol file: c:\Projects\MyDriver\x64\Debug\MyDriver.pdb
```

### 10.6.3 심볼 검색

```
// x (Examine) - 심볼 검색
x MyDriver!*                    // MyDriver의 모든 심볼
x MyDriver!Pre*                 // Pre로 시작하는 심볼
x MyDriver!*Create*             // Create 포함 심볼
x nt!Nt*File                    // nt 모듈의 Nt...File 심볼

// 출력 예:
0: kd> x MyDriver!Pre*
fffff801`12340100 MyDriver!PreCreate
fffff801`12340200 MyDriver!PreRead
fffff801`12340300 MyDriver!PreWrite
fffff801`12340400 MyDriver!PreCleanup

// 주소로 심볼 찾기
ln fffff801`12340100            // 해당 주소의 심볼 이름
```

### 10.6.4 소스 코드 연결

```
// 소스 경로 설정
.srcpath c:\Projects\MyDriver\src
.srcpath+ c:\Projects\MyDriver\include

// 소스 라인 보기
.lines                          // 소스 라인 정보 활성화
lsa .                           // 현재 위치 소스 코드
lsa MyDriver!PreCreate          // 특정 함수 소스 코드

// 소스 모드 디버깅
l+t                             // 소스 모드 활성화
l-t                             // 어셈블리 모드로 전환
```

---

## 10.7 메모리 검색과 수정

### 10.7.1 s (Search) - 메모리 검색

```
// 바이트 패턴 검색
s -b fffff801`12340000 L10000 4d 5a    // "MZ" 시그니처 검색
s -b fffff801`12340000 L10000 48 89 5c // 명령어 패턴 검색

// 문자열 검색
s -a fffff801`12340000 L10000 "MyDriver"  // ASCII 문자열
s -u fffff801`12340000 L10000 "MyDriver"  // 유니코드 문자열

// DWORD 검색
s -d fffff801`12340000 L10000 0x12345678

// 포인터 검색
s -q fffff801`12340000 L10000 fffff801`00000000
```

### 10.7.2 e (Enter) - 메모리 수정

```
// 바이트 수정
eb fffff801`12345000 90 90 90   // 3바이트를 NOP(0x90)으로

// 워드/더블워드/쿼드워드 수정
ew fffff801`12345000 0x1234     // 2바이트
ed fffff801`12345000 0x12345678 // 4바이트
eq fffff801`12345000 0x1234567890abcdef  // 8바이트

// 문자열 입력
ea fffff801`12345000 "Hello"    // ASCII 문자열
eu fffff801`12345000 "Hello"    // 유니코드 문자열
```

> ⚠️ **주의**: 커널 메모리 수정은 시스템 크래시를 유발할 수 있습니다. 신중하게 사용하세요.

### 10.7.3 u (Unassemble) - 디스어셈블

```
// 현재 위치 디스어셈블
u                               // 현재 RIP부터
u .                             // 현재 위치
u fffff801`12340000             // 특정 주소

// 길이 지정
u fffff801`12340000 L10         // 16개 명령어

// 함수 전체 디스어셈블
uf MyDriver!PreCreate           // PreCreate 함수 전체

// 출력 예:
0: kd> uf MyDriver!PreCreate
MyDriver!PreCreate:
fffff801`12340100 mov     rbp,rsp
fffff801`12340103 sub     rsp,20h
fffff801`12340107 mov     qword ptr [rbp+10h],rcx
fffff801`1234010b mov     qword ptr [rbp+18h],rdx
...
```

---

## 10.8 프로세스와 스레드 명령어

### 10.8.1 !process - 프로세스 정보

```
// 현재 프로세스
!process -1 0                   // 현재 프로세스 기본 정보

// 모든 프로세스
!process 0 0                    // 모든 프로세스 목록 (간략)
!process 0 1                    // 모든 프로세스 (스레드 정보 포함)

// 특정 프로세스
!process <EPROCESS주소> 7       // 상세 정보

// 이름으로 검색
!process 0 0 notepad.exe        // notepad.exe 프로세스 찾기

// 출력 예:
0: kd> !process 0 0 notepad.exe
PROCESS fffff801`12345000
    SessionId: 1  Cid: 1234  Peb: 00000000`7ffde000  ParentCid: 0abc
    DirBase: 12345678  ObjectTable: fffffa80`12345678  HandleCount: 123
    Image: notepad.exe
```

### 10.8.2 !thread - 스레드 정보

```
// 현재 스레드
!thread -1 0                    // 현재 스레드 기본 정보

// 특정 스레드
!thread <ETHREAD주소>           // 스레드 상세 정보

// 출력에서 중요한 정보:
// - State: 스레드 상태 (Running, Waiting, Ready 등)
// - IRQL: 현재 IRQL
// - Teb: Thread Environment Block
// - Win32Thread: GUI 스레드 정보
```

### 10.8.3 컨텍스트 전환

```
// 프로세스 컨텍스트 전환 (메모리 공간 전환)
.process /i <EPROCESS주소>      // 인터럽트 + 컨텍스트 전환
g                               // 실행하여 전환 완료

// 스레드 컨텍스트 전환
.thread <ETHREAD주소>           // 스레드 컨텍스트로 전환

// 원래 컨텍스트로 복귀
.process /P                     // 기본 프로세스로
.thread                         // 기본 스레드로
```

---

## 10.9 자주 사용하는 확장 명령어

### 10.9.1 !analyze - 자동 분석

```
// 크래시 분석 (BSOD 발생 후)
!analyze -v                     // 상세 분석

// 출력 내용:
// - BUGCHECK_CODE: 버그 체크 코드
// - FAULTING_MODULE: 문제 모듈
// - STACK_TEXT: 크래시 콜스택
// - 원인 분석 및 권장 조치
```

### 10.9.2 !irp - IRP 분석

```
// IRP 구조 분석
!irp <IRP주소>

// 출력 예:
0: kd> !irp fffff801`12345000
Irp is active with 5 stacks 3 is current (= 0xfffff80112345100)
 cmd  flg cl Device   File     Completion-Loss-Info  Args
>[IRP_MJ_CREATE(0),N/A(0)]
            0 0 fffff801`12340000 fffff801`12350000 00000000-00000000
                       \FileSystem\Ntfs
...
```

### 10.9.3 !pool - 메모리 풀 분석

```
// 풀 할당 정보
!pool <주소>                    // 해당 주소의 풀 정보
!poolused                       // 풀 사용량 통계

// 출력 예:
0: kd> !pool fffff801`12345000
Pool page fffff801`12345000 region is Nonpaged pool
*fffff80112345000 size:   80 previous size:    0  (Allocated) *MyDr
```

### 10.9.4 !locks - 락 분석

```
// 커널 락 분석
!locks                          // 모든 잠긴 리소스
!locks -v                       // 상세 정보

// 데드락 분석에 유용
```

---

## 10.10 명령어 조합과 스크립팅

### 10.10.1 명령어 구분자

```
// 세미콜론으로 여러 명령어 연결
bp MyDriver!PreCreate "kb; r; g"  // 콜스택, 레지스터 출력 후 계속

// 실제 활용:
bp MyDriver!PreCreate ".echo === PreCreate Hit ===; kb 5; g"
```

### 10.10.2 조건문

```
// .if / .else 조건문
bp MyDriver!PreWrite ".if (@rax == 0) {.echo SUCCESS} .else {.echo FAILED}"

// 조건부 브레이크
bp MyDriver!PreCreate ".if (poi(@rcx+0x38) == 2) {} .else {gc}"
// rcx+0x38이 2가 아니면 계속 실행 (gc = go conditional)
```

### 10.10.3 출력 제어

```
// .echo - 문자열 출력
.echo "=== Checkpoint reached ==="

// .printf - 형식화된 출력
.printf "Process ID: %p\n", poi(@rcx+0x2e8)

// .formats - 값 다양한 형식으로 표시
.formats fffff801`12345678
```

---

## 10.11 실전: Minifilter 디버깅 시나리오

### 시나리오: PreCreate 콜백 분석

```
// 1. 드라이버 로드 대기
bu MyDriver!DriverEntry
g

// 2. DriverEntry 히트 - 심볼 확인
lm m MyDriver
x MyDriver!Pre*

// 3. PreCreate에 브레이크포인트 설정
bp MyDriver!PreCreate

// 4. 파일 열기 트리거
g

// 5. PreCreate 히트 - 분석
0: kd> kb
 # RetAddr           : Args to Child                                                           : Call Site
00 fffff801`11110000 : fffff801`12345000 fffff801`12345100 00000000`00000000 00000000`00000000 : MyDriver!PreCreate
01 fffff801`11120000 : ...

// 6. FLT_CALLBACK_DATA 구조 확인
dt fltmgr!_FLT_CALLBACK_DATA @rcx

// 7. 파일 이름 확인
dt fltmgr!_FLT_CALLBACK_DATA @rcx Iopb.TargetFileObject
dt nt!_FILE_OBJECT poi(@rcx+0x30) FileName
du poi(poi(@rcx+0x30)+0x58)

// 8. 계속 실행
g
```

---

## 10.12 명령어 빠른 참조표

```
┌─────────────────────────────────────────────────────────────────────┐
│                      WinDbg 필수 명령어                              │
├─────────────────────────────────────────────────────────────────────┤
│  실행 제어        g, p, t, gu, Ctrl+Break                           │
│  브레이크포인트   bp, bu, ba, bm, bl, bc, bd, be                    │
│  메모리 덤프      db, dw, dd, dq, da, du, dt, dv                    │
│  레지스터         r, r <reg>=<value>                                 │
│  콜스택           k, kb, kp, kn, .frame                              │
│  심볼             .sympath, .reload, lm, x, ln                       │
│  메모리 검색      s -b/-a/-u/-d                                      │
│  메모리 수정      eb, ew, ed, eq, ea, eu                             │
│  디스어셈블       u, uf                                               │
│  프로세스         !process, .process                                  │
│  스레드           !thread, .thread                                    │
│  분석             !analyze -v, !irp, !pool                            │
│  화면 제어        .cls, .echo, .printf                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 요약

이 챕터에서 학습한 내용:

1. **명령어 체계**: 표준(없음), 메타(.), 확장(!) 세 가지 유형
2. **실행 제어**: g(계속), p(Step Over), t(Step Into), gu(Step Out)
3. **브레이크포인트**: bp(기본), bu(지연), ba(하드웨어), bm(패턴)
4. **메모리 덤프**: d계열 명령어로 다양한 형식 표시
5. **구조체 분석**: dt 명령어로 커널 구조체 탐색
6. **심볼 관리**: .sympath, .reload로 심볼 설정
7. **스크립팅**: 세미콜론, 조건문으로 명령어 자동화

다음 챕터에서는 이 명령어들을 활용하여 실제 커널 구조체를 분석하는 실습을 진행합니다.

---

## 참고 자료

- [WinDbg Commands](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/commands)
- [Debugging Symbols](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/symbols)
- [Using Breakpoints](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/using-breakpoints)
