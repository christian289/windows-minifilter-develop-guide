# Chapter 14: 고급 WinDbg 기법

## 난이도: ⭐⭐⭐⭐⭐ (고급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- 조건부 브레이크포인트로 정밀한 디버깅을 수행합니다
- WinDbg 스크립트로 반복 작업을 자동화합니다
- LINQ 기반 dx 명령어로 복잡한 데이터를 쿼리합니다
- Time Travel Debugging(TTD)으로 실행을 역추적합니다
- 사용자 정의 확장을 활용합니다

## 도입: 디버깅 생산성의 비약적 향상

지금까지 학습한 기본 명령어로도 대부분의 문제를 해결할 수 있습니다. 하지만 고급 기법을 마스터하면 디버깅 시간을 **획기적으로 단축**할 수 있습니다. 특히 재현이 어려운 버그나 복잡한 조건에서만 발생하는 문제를 분석할 때 진가를 발휘합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    고급 기법의 효과                                  │
├─────────────────────────────────────────────────────────────────────┤
│  기본 접근                    고급 접근                              │
├─────────────────────────────────────────────────────────────────────┤
│  bp로 매번 수동 확인          조건부 BP로 자동 필터링               │
│  명령어 수동 반복 입력        스크립트로 자동화                     │
│  dt로 구조체 일일이 탐색      dx로 LINQ 쿼리                        │
│  버그 재현 후 분석            TTD로 역추적 분석                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 14.1 조건부 브레이크포인트 고급 기법

### 14.1.1 기본 조건부 브레이크포인트

```
// 기본 문법
bp <주소> ".if (<조건>) {} .else {gc}"

// 예: rax가 0이 아닐 때만 멈춤
bp MyDriver!PreWrite ".if (@rax != 0) {} .else {gc}"

// 예: 특정 프로세스에서만 멈춤
bp MyDriver!PreCreate ".if (poi(nt!PsGetCurrentProcess+0x5a8) == 'note') {} .else {gc}"
```

### 14.1.2 문자열 조건

```
// 파일 이름에 특정 문자열 포함 시 멈춤
// (복잡하므로 스크립트 파일 사용 권장)

// 방법 1: UNICODE_STRING 비교
bp MyDriver!PreCreate "
    r $t0 = poi(poi(poi(@rcx+0x10)+8)+0x58);
    r $t1 = poi(@$t0+8);
    .if ($spat(@$t1, \"*sensitive*\")) {}
    .else {gc}
"

// 방법 2: 간단한 확장자 체크 (4문자)
bp MyDriver!PreCreate "
    r $t0 = poi(poi(poi(@rcx+0x10)+8)+0x58);
    r $t1 = wo(@$t0);
    .if (@$t1 >= 8) {
        r $t2 = poi(poi(@$t0+8)+@$t1-8);
        .if (@$t2 == 0x0078006c002e) {.echo XLSX found!} .else {gc}
    }
    .else {gc}
"
```

### 14.1.3 히트 카운트 조건

```
// N번째 히트에서만 멈춤
bp MyDriver!PreWrite 5    // 5번째 히트에서 멈춤

// 카운터 사용
r $t9 = 0
bp MyDriver!PreWrite "r $t9 = @$t9 + 1; .if (@$t9 == 100) {} .else {gc}"
// 100번째 호출에서 멈춤

// 범위 조건
bp MyDriver!PreWrite "r $t9 = @$t9 + 1; .if (@$t9 >= 50 && @$t9 <= 60) {} .else {gc}"
// 50~60번째 호출에서 멈춤
```

### 14.1.4 로깅 브레이크포인트

```
// 멈추지 않고 정보만 기록

// 기본 로깅
bp MyDriver!PreCreate ".printf \"PreCreate called: %p\\n\", @rcx; g"

// 파일 이름 로깅
bp MyDriver!PreCreate "
    .printf \"[%d] \", @$t0;
    r $t0 = @$t0 + 1;
    dS poi(poi(poi(@rcx+0x10)+8)+0x58)+8;
    g
"

// 파일로 로깅
bp MyDriver!PreWrite ".logopen c:\\debug\\writes.log; .printf \"%p\\n\", @rcx; .logclose; g"
```

### 14.1.5 조건부 명령 실행

```
// 조건에 따라 다른 명령 실행
bp MyDriver!PreWrite "
    .if (by(poi(@rcx+0x10)+4) == 0x04) {
        .echo Write operation detected!;
        !fltkd.cbd @rcx;
        kb
    }
    .else {
        gc
    }
"

// 에러 조건에서 상세 분석
bp MyDriver!PreOperation+0x150 "
    .if (@rax == 0) {
        gc
    }
    .else {
        .printf \"Error: 0x%x\\n\", @rax;
        dv;
        kb;
    }
"
```

---

## 14.2 WinDbg 스크립트

### 14.2.1 스크립트 파일 실행

```
// 스크립트 파일 실행
$$>< c:\scripts\myscript.txt       // 공백/줄바꿈 유지
$>< c:\scripts\myscript.txt        // 공백을 세미콜론으로 변환
$$>a< c:\scripts\myscript.txt      // 인수 전달 가능

// 예: analyze.txt
.echo === Starting Analysis ===
!analyze -v
.echo === Process List ===
!process 0 0
.echo === Analysis Complete ===
```

### 14.2.2 변수와 의사 레지스터

```
// 사용자 의사 레지스터 ($t0 - $t19)
r $t0 = 0                          // 초기화
r $t0 = @$t0 + 1                   // 증가
.printf "Count: %d\n", @$t0        // 출력

// 시스템 의사 레지스터
@$ip          // 현재 명령어 포인터 (RIP)
@$csp         // 현재 스택 포인터 (RSP)
@$proc        // 현재 EPROCESS
@$thread      // 현재 ETHREAD
@$peb         // Process Environment Block
@$teb         // Thread Environment Block
@$retreg      // 반환값 레지스터 (RAX)
@$ra          // 리턴 주소

// 예: 현재 프로세스 이름 출력
da @$proc+0x5a8
```

### 14.2.3 제어 흐름

```
// .if / .elsif / .else
.if (@rax == 0) {
    .echo Success
}
.elsif (@rax == 0xc0000001) {
    .echo Unsuccessful
}
.else {
    .printf "Error: 0x%x\n", @rax
}

// .while 루프
r $t0 = 0
.while (@$t0 < 10) {
    .printf "Iteration: %d\n", @$t0
    r $t0 = @$t0 + 1
}

// .for 루프
.for (r $t0 = 0; @$t0 < 10; r $t0 = @$t0 + 1) {
    .printf "i = %d\n", @$t0
}

// .foreach - 명령 출력 반복
.foreach (addr { lm1m }) {
    .printf "Module: %p\n", ${addr}
}
```

### 14.2.4 실용적인 스크립트 예제

**예제 1: 모든 프로세스의 이미지 이름 출력**

```
// process_names.txt
.foreach (proc { !process 0 0 }) {
    .if ($spat("${proc}", "PROCESS*")) {
        r $t0 = ${proc}
        .printf "%p: ", @$t0
        da @$t0+0x5a8
    }
}
```

**예제 2: 메모리 풀 태그 검색**

```
// find_pool_tag.txt
// 사용: $$>a< c:\scripts\find_pool_tag.txt MyDr

r $t0 = 0xfffff80000000000
r $t1 = 0xfffff80100000000

.while (@$t0 < @$t1) {
    .if (poi(@$t0) == 0x7244794d) {
        .printf "Found at: %p\n", @$t0
    }
    r $t0 = @$t0 + 0x1000
}
```

**예제 3: 콜백 실행 시간 측정**

```
// callback_timing.txt
r $t8 = 0
r $t9 = 0

// 시작 시간 기록
bp MyDriver!PreWrite "
    r $t8 = @$prcb->CurrentThread->CycleTime;
    g
"

// 종료 시간 기록 및 계산
bp MyDriver!PreWrite+0x200 "
    r $t9 = @$prcb->CurrentThread->CycleTime - @$t8;
    .printf \"Cycles: %I64d\\n\", @$t9;
    g
"
```

---

## 14.3 dx 명령어 - LINQ 스타일 쿼리

### 14.3.1 dx 기본 사용법

`dx`는 데이터 모델을 LINQ 스타일로 쿼리하는 강력한 명령어입니다.

```
// 기본 문법
dx <expression>

// 예: 현재 프로세스
dx @$curprocess

// 예: 모든 프로세스 목록
dx @$cursession.Processes

// 출력 형식 지정
dx -g @$cursession.Processes     // 그리드 형식
dx -r2 @$curprocess              // 2단계 재귀 표시
```

### 14.3.2 프로세스/스레드 쿼리

```
// 모든 프로세스 이름 출력
dx @$cursession.Processes.Select(p => p.Name)

// 특정 이름의 프로세스 찾기
dx @$cursession.Processes.Where(p => p.Name == "notepad.exe")

// PID로 프로세스 찾기
dx @$cursession.Processes[1234]

// 프로세스의 스레드 목록
dx @$cursession.Processes[1234].Threads

// 스레드 상태 필터링
dx @$cursession.Processes[1234].Threads.Where(t => t.State == "Running")

// 핸들 수가 많은 프로세스 상위 10개
dx @$cursession.Processes
    .OrderByDescending(p => p.HandleCount)
    .Take(10)
    .Select(p => new { Name = p.Name, Handles = p.HandleCount })
```

### 14.3.3 모듈 쿼리

```
// 로드된 모든 커널 모듈
dx @$cursession.Modules

// 특정 모듈 찾기
dx @$cursession.Modules.Where(m => m.Name.Contains("flt"))

// 모듈 기본 주소 확인
dx @$cursession.Modules["MyDriver"].BaseAddress

// 모듈 크기순 정렬
dx @$cursession.Modules.OrderByDescending(m => m.Size).Take(10)
```

### 14.3.4 구조체 탐색

```
// EPROCESS 구조체 탐색
dx -r1 ((nt!_EPROCESS*)@$proc)

// 특정 필드 접근
dx ((nt!_EPROCESS*)@$proc)->UniqueProcessId
dx ((nt!_EPROCESS*)@$proc)->ImageFileName

// FLT_CALLBACK_DATA 탐색
dx -r2 ((fltmgr!_FLT_CALLBACK_DATA*)@rcx)

// Minifilter 관련 구조체
dx -r1 ((fltmgr!_FLT_FILTER*)0xffffb80123464000)
```

### 14.3.5 사용자 정의 함수

```
// 람다 함수 사용
dx @$cursession.Processes.Select(
    p => new {
        Name = p.Name,
        PID = p.Id,
        ThreadCount = p.Threads.Count(),
        HandleCount = p.HandleCount
    }
)

// 복잡한 필터링
dx @$cursession.Processes
    .Where(p => p.Threads.Count() > 10)
    .Where(p => p.HandleCount > 100)
    .OrderBy(p => p.Name)
```

### 14.3.6 디버거 객체 모델

```
// 디버거 상태 정보
dx Debugger
dx Debugger.State
dx Debugger.State.DebuggerVariables

// 브레이크포인트 정보
dx Debugger.State.Breakpoints

// 세션 정보
dx @$cursession

// 현재 컨텍스트
dx @$curprocess
dx @$curthread
dx @$curstack
```

---

## 14.4 Time Travel Debugging (TTD)

### 14.4.1 TTD 개요

TTD는 프로그램 실행을 **녹화**하고 나중에 **재생**할 수 있는 혁신적인 기능입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Time Travel Debugging                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  일반 디버깅:     시작 ───────────> 버그 발생 ───> (????)           │
│                   "왜 이렇게 됐지?"                                  │
│                                                                      │
│  TTD:            시작 ───────────> 버그 발생 <─── 역방향 추적       │
│                  "어디서 잘못됐는지 역추적!"                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 14.4.2 TTD 트레이스 녹화

```
// WinDbg Preview에서:
// File > Start debugging > Launch executable (advanced)
// "Record with Time Travel Debugging" 체크

// 또는 명령줄에서:
ttd.exe -out c:\traces MyApp.exe

// 커널 모드 TTD (제한적 지원)
// Windows 11+에서 일부 시나리오 지원
```

### 14.4.3 TTD 기본 명령어

```
// 트레이스 파일 열기
.opendump c:\traces\MyApp01.run

// 트레이스 정보
dx @$curprocess.TTD

// 시간 위치 확인
!tt                         // 현재 위치
dx @$curposition            // 현재 위치 (dx)

// 시간 이동
!tt 0                       // 시작으로
!tt 100                     // 끝으로 (100%)
!tt 50                      // 중간으로 (50%)
!tt 12:34                   // 특정 위치로

// 역방향 실행
g-                          // 역방향 실행
p-                          // 역방향 Step Over
t-                          // 역방향 Step Into
```

### 14.4.4 TTD 쿼리

```
// 모든 예외 조회
dx @$curprocess.TTD.Events.Where(e => e.Type == "Exception")

// 특정 함수 호출 기록
dx @$curprocess.TTD.Calls("kernel32!CreateFileW")

// 메모리 접근 추적
dx @$curprocess.TTD.Memory(0x12345678, 0x12345680, "w")
// 해당 주소에 쓰기한 모든 위치

// 예외 발생 시점으로 이동
dx @$curprocess.TTD.Events
    .Where(e => e.Type == "Exception")
    .First()
    .SeekTo()

// 함수 호출 횟수
dx @$curprocess.TTD.Calls("MyDriver!PreWrite").Count()

// 특정 반환값을 가진 호출 찾기
dx @$curprocess.TTD.Calls("ntdll!NtCreateFile")
    .Where(c => c.ReturnValue != 0)
```

### 14.4.5 실전 TTD 시나리오

**시나리오: 간헐적으로 발생하는 메모리 손상 추적**

```
// 1. 트레이스 녹화 후 크래시 발생

// 2. 크래시 위치에서 시작
!analyze -v

// 3. 손상된 메모리 주소 확인
// 가정: 0xffffb80123500000 메모리가 손상됨

// 4. 해당 주소에 마지막으로 쓴 위치 찾기
dx @$curprocess.TTD.Memory(0xffffb80123500000, 0xffffb80123500008, "w")

// 결과:
// [0] Position: 45:123, Address: ..., Value: 0x0 -> 0xDEADBEEF

// 5. 해당 시점으로 이동
!tt 45:123

// 6. 누가 이 값을 썼는지 확인
kb

// 7. 그 이전으로 역추적하여 원인 분석
p-
p-
...
```

---

## 14.5 사용자 정의 확장 활용

### 14.5.1 JavaScript 확장 (NatVis)

```javascript
// MyExtension.js
"use strict";

class ProcessAnalyzer {
    toString() {
        return "Process Analyzer Extension";
    }

    listProtectedFiles(filterName) {
        let session = host.currentSession;
        let processes = session.Processes;

        host.diagnostics.debugLog("Searching for protected files...\n");

        for (let proc of processes) {
            if (proc.Name.indexOf(filterName) >= 0) {
                host.diagnostics.debugLog(`Found: ${proc.Name} (PID: ${proc.Id})\n`);
            }
        }
    }
}

function initializeScript() {
    return [
        new host.apiVersionSupport(1, 7),
        new host.namespacePropertyParent(ProcessAnalyzer, "Debugger.Models", "Debugger.Models.ProcessAnalyzer", "ProcessAnalyzer")
    ];
}
```

```
// 사용법
.scriptload c:\scripts\MyExtension.js
dx Debugger.Models.ProcessAnalyzer.listProtectedFiles("MyDriver")
```

### 14.5.2 유용한 커뮤니티 확장

```
// MEX (Microsoft Extension)
.load mex

// 모든 잠긴 뮤텍스/리소스 표시
!mex.locks

// 핸들 테이블 분석
!mex.handles

// 스레드 풀 분석
!mex.tpool

// SwishDbgExt
.load SwishDbgExt

// 보안 분석용
!drivers        // 드라이버 목록
!callbacks      // 콜백 등록 목록
!ssdt           // SSDT 분석
```

---

## 14.6 원격 디버깅 고급 설정

### 14.6.1 디버그 서버 설정

```
// 디버거를 서버로 실행
windbg -server tcp:port=5005

// 다른 PC에서 연결
windbg -remote tcp:server=192.168.1.100,port=5005

// 명명된 파이프 사용
windbg -server npipe:pipe=MyDebugPipe
windbg -remote npipe:server=\\SERVER,pipe=MyDebugPipe
```

### 14.6.2 심볼 서버 최적화

```
// 로컬 캐시 + Microsoft 심볼 서버
.sympath cache*c:\symbols;srv*https://msdl.microsoft.com/download/symbols

// 여러 심볼 서버 사용
.sympath cache*c:\symbols;srv*https://msdl.microsoft.com/download/symbols;srv*https://chromium-browser-symsrv.commondatastorage.googleapis.com

// 심볼 다운로드 상태 확인
!sym noisy              // 상세 심볼 로딩 정보
!sym quiet              // 조용히

// 심볼 강제 로드
.reload /f MyDriver.sys
ld MyDriver             // 심볼만 로드
```

---

## 14.7 성능 분석 기법

### 14.7.1 CPU 프로파일링 (기본)

```
// 모든 스레드 스택 덤프
!process 0 17

// 특정 프로세스의 모든 스레드 스택
!process <EPROCESS주소> 17

// 대기 중인 스레드 분석
!stacks 2              // 커널 스택 요약
!running -it           // 실행 중인 스레드
```

### 14.7.2 메모리 프로파일링

```
// 풀 사용량 분석
!poolused 2            // 태그별 NonPaged Pool 사용량
!poolused 4            // 태그별 Paged Pool 사용량

// 특정 태그 상세
!poolfind MyDr 2       // MyDr 태그 모든 할당

// 메모리 누수 추적
// 1차 스냅샷
!poolused 2 > c:\debug\pool1.txt

// (시간 경과 후) 2차 스냅샷
!poolused 2 > c:\debug\pool2.txt

// 비교 (외부 도구 필요)
```

### 14.7.3 DPC/ISR 분석

```
// DPC 통계
!dpcs

// 인터럽트 분석
!idt                   // IDT 테이블
!isr <vector>          // 특정 ISR 정보
```

---

## 14.8 실전 고급 디버깅 시나리오

### 시나리오 1: 재현 어려운 경쟁 상태 분석

```
// 문제: 가끔씩만 발생하는 Use-After-Free

// 1. 특수 풀 활성화
verifier /flags 1 /driver MyDriver.sys
// (재부팅 필요)

// 2. 조건부 로깅 브레이크포인트 설정
r $t0 = 0
bp MyDriver!AllocateContext "
    r $t0 = @$t0 + 1;
    .printf \"[%d] Alloc: %p\\n\", @$t0, @rax;
    g
"

bp MyDriver!FreeContext "
    .printf \"Free: %p\\n\", @rcx;
    g
"

// 3. 트레이스 수집 후 분석
// 로그에서 Free 후 사용 패턴 찾기
```

### 시나리오 2: 성능 병목 식별

```
// 문제: 특정 작업 시 시스템 느려짐

// 1. CPU 소비 스레드 찾기
!running -it

// 2. 해당 스레드 스택 분석
!thread <ETHREAD>
.thread <ETHREAD>
kbn

// 3. 핫 함수에 로깅 브레이크포인트
r $t0 = 0
bp MyDriver!ExpensiveFunction "r $t0 = @$t0 + 1; g"

// (잠시 후)
.printf "Call count: %d\n", @$t0

// 4. 실행 시간 측정
bp MyDriver!ExpensiveFunction "r $t8 = @$prcb->CurrentThread->CycleTime; g"
bp MyDriver!ExpensiveFunction+0x150 "
    r $t9 = @$prcb->CurrentThread->CycleTime - @$t8;
    .if (@$t9 > 1000000) {
        .printf \"Slow call: %I64d cycles\\n\", @$t9;
        kb
    }
    g
"
```

### 시나리오 3: 복잡한 데이터 구조 분석

```
// 문제: 연결 리스트 손상 의심

// 1. 리스트 헤드 확인
dt nt!_LIST_ENTRY MyDriver!g_ContextList

// 2. dx로 리스트 순회
dx -r0 Debugger.Utility.Collections.FromListEntry(
    (nt!_LIST_ENTRY*)&MyDriver!g_ContextList,
    "MyDriver!_MY_CONTEXT",
    "ListEntry"
)

// 3. 각 엔트리 검증
dx -g Debugger.Utility.Collections.FromListEntry(
    (nt!_LIST_ENTRY*)&MyDriver!g_ContextList,
    "MyDriver!_MY_CONTEXT",
    "ListEntry"
).Select(c => new {
    Address = &c,
    RefCount = c.RefCount,
    IsValid = c.Magic == 0x12345678
})
```

---

## 14.9 디버깅 체크리스트

```
┌─────────────────────────────────────────────────────────────────────┐
│                    고급 디버깅 체크리스트                            │
├─────────────────────────────────────────────────────────────────────┤
│  준비 단계                                                           │
│  □ 심볼 경로 설정 (.sympath)                                        │
│  □ 소스 경로 설정 (.srcpath)                                        │
│  □ Driver Verifier 활성화                                           │
│  □ 필요 시 특수 풀 활성화                                           │
├─────────────────────────────────────────────────────────────────────┤
│  분석 단계                                                           │
│  □ !analyze -v 실행                                                  │
│  □ 콜스택 분석 (kb, kv)                                             │
│  □ 관련 구조체 덤프 (dt, !fltkd)                                    │
│  □ 조건부 브레이크포인트로 정밀 분석                                │
├─────────────────────────────────────────────────────────────────────┤
│  재현 어려운 문제                                                    │
│  □ TTD 트레이스 녹화                                                │
│  □ 로깅 브레이크포인트 설정                                         │
│  □ dx LINQ 쿼리로 패턴 분석                                         │
├─────────────────────────────────────────────────────────────────────┤
│  성능 문제                                                           │
│  □ !running으로 CPU 소비 스레드 확인                                │
│  □ !poolused로 메모리 사용량 확인                                   │
│  □ 핫 경로 실행 시간 측정                                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 요약

이 챕터에서 학습한 고급 기법:

1. **조건부 브레이크포인트**: .if/.else로 정밀한 조건 필터링
2. **스크립트**: 반복 작업 자동화, 변수 및 제어 흐름
3. **dx 명령어**: LINQ 스타일 데이터 쿼리, 구조체 탐색
4. **TTD**: 실행 녹화 및 역방향 디버깅
5. **성능 분석**: CPU, 메모리 프로파일링

이것으로 Part 4 WinDbg 마스터를 완료했습니다. 다음 Part에서는 첫 번째 커널 드라이버를 직접 개발합니다.

---

## 참고 자료

- [Time Travel Debugging](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview)
- [Debugger Data Model](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/data-model)
- [JavaScript Debugger Scripting](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/javascript-debugger-scripting)
- [WinDbg Preview](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugging-using-windbg-preview)
