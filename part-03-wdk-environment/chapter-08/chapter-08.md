# Chapter 8: 커널 디버깅 연결 설정

## 난이도: ⭐⭐ (초급)

## 학습 목표

- WinDbg 설치 및 구성
- 네트워크 커널 디버깅 설정
- 시리얼 디버깅 설정 (대안)
- 디버거 연결 테스트

---

## 들어가며

.NET 개발에서 Visual Studio 디버거는 필수 도구입니다. 커널 드라이버 개발에서는 **WinDbg**가 그 역할을 합니다. 하지만 커널 디버깅은 사용자 모드 디버깅과 다릅니다. **두 대의 시스템**(또는 VM)이 필요합니다.

이 장에서는 호스트(개발 머신)와 대상(테스트 VM) 간의 디버깅 연결을 설정합니다.

---

## 8.1 WinDbg 설치

### 8.1.1 WinDbg Preview (권장)

Microsoft Store에서 무료로 설치할 수 있습니다:

```
1. Microsoft Store 열기
2. "WinDbg Preview" 검색
3. 설치 (무료)
```

또는 winget 사용:

```powershell
winget install "WinDbg Preview"
```

### 8.1.2 클래식 WinDbg (대안)

Windows SDK에 포함되어 있습니다:

```
설치 경로:
C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe
C:\Program Files (x86)\Windows Kits\10\Debuggers\x86\windbg.exe (32-bit)
```

### 8.1.3 WinDbg Preview vs 클래식 비교

| 기능 | WinDbg Preview | 클래식 WinDbg |
|------|---------------|--------------|
| UI | 현대적 (Fluent) | 레거시 |
| 다크 모드 | ✅ 지원 | ❌ 없음 |
| Time Travel Debugging | ✅ 완전 지원 | ⚠️ 제한적 |
| JavaScript 확장 | ✅ 지원 | ❌ 없음 |
| 자동 업데이트 | ✅ Store 통해 | ❌ 수동 |
| 스크립팅 | ✅ 향상됨 | ✅ 기본 |
| 명령어 호환성 | ✅ 동일 | ✅ 동일 |

> 💡 **권장**: WinDbg Preview를 사용하세요. 명령어는 동일하며 UI가 훨씬 편리합니다.

---

## 8.2 심볼 서버 구성

디버깅 시 **심볼(Symbol)**은 함수 이름, 변수 이름 등을 제공합니다. Microsoft 심볼 서버에서 Windows 커널 심볼을 다운로드할 수 있습니다.

### 8.2.1 환경 변수 설정

```cmd
:: 시스템 환경 변수 설정 (관리자 권한)
:: Set system environment variable (admin)
setx /M _NT_SYMBOL_PATH "srv*C:\Symbols*https://msdl.microsoft.com/download/symbols"
```

또는 PowerShell:

```powershell
# 시스템 환경 변수
[Environment]::SetEnvironmentVariable(
    "_NT_SYMBOL_PATH",
    "srv*C:\Symbols*https://msdl.microsoft.com/download/symbols",
    "Machine"
)
```

### 8.2.2 WinDbg에서 직접 설정

```
WinDbg Preview:
├── File → Settings → Debugging settings
└── Default symbol path:
    srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
```

### 8.2.3 심볼 경로 형식

```
srv*<로컬 캐시>*<심볼 서버 URL>

예:
srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
│    │          │
│    │          └─ Microsoft 공식 심볼 서버
│    └─ 로컬 캐시 폴더 (다운로드된 심볼 저장)
└─ 심볼 서버 프로토콜

여러 경로 연결:
srv*C:\Symbols*https://msdl.microsoft.com/download/symbols;C:\MyDriverSymbols
```

---

## 8.3 네트워크 디버깅 설정 (권장)

네트워크 디버깅은 가장 빠르고 편리한 방법입니다.

### 8.3.1 네트워크 구성 확인

```
디버깅 네트워크 구성:
┌─────────────────┐         ┌─────────────────┐
│   호스트 PC      │         │   테스트 VM      │
│   (WinDbg)      │◄───────►│   (드라이버)     │
│                 │  네트워크 │                 │
│ IP: 192.168.1.10│         │ IP: 192.168.1.20│
│ Port: 50000     │         │                 │
└─────────────────┘         └─────────────────┘
```

### 8.3.2 호스트 IP 확인

```cmd
:: 호스트에서 실행
ipconfig

:: 예시 출력:
이더넷 어댑터 이더넷:
   IPv4 주소 . . . . . . . . : 192.168.1.10
```

### 8.3.3 대상 VM에서 디버깅 설정

VM 내에서 **관리자 명령 프롬프트** 실행:

```cmd
:: 커널 디버깅 활성화
:: Enable kernel debugging
bcdedit /debug on

:: 네트워크 디버깅 설정
:: Configure network debugging
bcdedit /dbgsettings net hostip:192.168.1.10 port:50000

:: 설정 확인 (키 값 확인 중요!)
:: Verify settings (note the key!)
bcdedit /dbgsettings

:: 출력 예시:
:: debugtype               Network
:: hostip                  192.168.1.10
:: port                    50000
:: key                     1abc2def3ghi4jkl   ← 이 키를 호스트에서 사용!

:: 재부팅
shutdown /r /t 0
```

### 8.3.4 호스트에서 WinDbg 연결

**방법 1: GUI 사용**

```
WinDbg Preview 실행:
1. File → Attach to kernel
2. Net 탭 선택
3. Port: 50000
4. Key: 1abc2def3ghi4jkl (VM에서 생성된 키)
5. OK
```

**방법 2: 명령줄 사용**

```cmd
:: WinDbg Preview
WinDbgX -k net:port=50000,key=1abc2def3ghi4jkl

:: 클래식 WinDbg
windbg -k net:port=50000,key=1abc2def3ghi4jkl
```

### 8.3.5 연결 대기 화면

```
WinDbg 출력:
Microsoft (R) Windows Debugger Version 10.0.xxxxx.x AMD64
Copyright (c) Microsoft Corporation. All rights reserved.

Using NET for debugging
Waiting to reconnect...
```

VM이 재부팅되면 자동으로 연결됩니다.

---

## 8.4 시리얼 디버깅 설정 (대안)

네트워크 문제가 있을 때 시리얼(COM) 포트를 사용할 수 있습니다.

### 8.4.1 Hyper-V 시리얼 포트 설정

```powershell
# VM 중지 상태에서
# While VM is stopped
Stop-VM -Name "DriverTestVM"

# Named Pipe로 COM 포트 설정
# Configure COM port as Named Pipe
Set-VMComPort -VMName "DriverTestVM" -Number 1 -Path "\\.\pipe\DriverDebug"

# VM 시작
Start-VM -Name "DriverTestVM"
```

### 8.4.2 VMware 시리얼 포트 설정

```
VM Settings:
1. Add → Serial Port
2. Connection: Use named pipe
   ├── \\.\pipe\DriverDebug
   ├── This end is the server
   └── The other end is an application
3. OK
```

### 8.4.3 대상 VM에서 설정

```cmd
:: 커널 디버깅 활성화
bcdedit /debug on

:: 시리얼 디버깅 설정
bcdedit /dbgsettings serial debugport:1 baudrate:115200

:: 설정 확인
bcdedit /dbgsettings

:: 재부팅
shutdown /r /t 0
```

### 8.4.4 호스트에서 WinDbg 연결

```
WinDbg Preview:
1. File → Attach to kernel
2. COM 탭 선택
3. Port: \\.\pipe\DriverDebug
4. Baud Rate: 115200
5. ☑ Pipe
6. ☑ Reconnect
7. OK
```

명령줄:

```cmd
WinDbgX -k com:port=\\.\pipe\DriverDebug,baud=115200,pipe,reconnect
```

---

## 8.5 첫 번째 디버깅 연결 테스트

### 8.5.1 연결 성공 확인

VM이 부팅되면 WinDbg에 다음과 같이 표시됩니다:

```
Connected to Windows 10 19041 x64 compatible target at (Mon Dec 16 10:30:00.000 2024 (UTC + 9:00)), ptr64 TRUE
Kernel Debugger connection established.

************* Path validation summary **************
Response                         Time (ms)     Location
Deferred                                       srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
Symbol search path is: srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
Executable search path is:
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff800`12000000 PsLoadedModuleList = 0xfffff800`12c2a2d0
Debug session time: Mon Dec 16 10:30:00.000 2024 (UTC + 9:00)
System Uptime: 0 days 0:00:30.000
```

### 8.5.2 기본 명령 테스트

```
:: 대상 시스템 정보
0: kd> vertarget
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Machine Name:
Kernel base = 0xfffff800`12000000 PsLoadedModuleList = 0xfffff800`12c2a2d0
System Uptime: 0 days 0:01:23.456

:: 프로세스 목록 (간략)
0: kd> !process 0 0
**** NT ACTIVE PROCESS DUMP ****
PROCESS ffff8001234567890
    SessionId: none  Cid: 0004    Peb: 00000000  ParentCid: 0000
    DirBase: 001ab000  ObjectTable: ffffc00012345678  HandleCount: 2345
    Image: System

PROCESS ffff800123456abc0
    SessionId: none  Cid: 0054    Peb: 00000000  ParentCid: 0004
    DirBase: 123ab000  ObjectTable: ffffc00012345def  HandleCount: 123
    Image: smss.exe
...

:: 로드된 모듈 목록
0: kd> lm
start             end                 module name
fffff800`12000000 fffff800`12e00000   nt         (pdb symbols)
fffff800`13000000 fffff800`13100000   hal        (deferred)
fffff800`14000000 fffff800`14050000   fltmgr     (deferred)
...
```

### 8.5.3 Break와 Continue

```
Ctrl+Break → 대상 시스템 일시 정지 (Break)

0: kd> g    → 대상 시스템 계속 실행 (Go)
```

> ⚠️ **주의**: Break 상태에서는 대상 VM이 완전히 멈춥니다!

### 8.5.4 심볼 로드 확인

```
:: 심볼 상태 확인
0: kd> !sym noisy
noisy mode - symbol prompts on

:: 커널 심볼 강제 로드
0: kd> .reload /f nt
Loading Kernel Symbols
...

:: 모듈 심볼 상태 확인
0: kd> lm m nt
Browse full module list
start             end                 module name
fffff800`12000000 fffff800`12e00000   nt         (pdb symbols)   c:\symbols\ntkrnlmp.pdb\...
```

---

## 8.6 bcdedit 주요 명령 정리

### 디버깅 관련 명령

```cmd
:: 디버깅 활성화/비활성화
:: Enable/Disable debugging
bcdedit /debug on
bcdedit /debug off

:: 현재 디버그 설정 조회
:: Query current debug settings
bcdedit /dbgsettings

:: 네트워크 디버깅 설정
:: Configure network debugging
bcdedit /dbgsettings net hostip:<IP> port:<PORT> [key:<KEY>]

:: 시리얼 디버깅 설정
:: Configure serial debugging
bcdedit /dbgsettings serial debugport:<N> baudrate:<RATE>

:: USB 디버깅 설정 (물리 머신용)
:: Configure USB debugging (for physical machines)
bcdedit /dbgsettings usb targetname:<NAME>
```

### 부팅 관련 명령

```cmd
:: 테스트 서명 모드 활성화/비활성화
:: Enable/Disable test signing mode
bcdedit /set testsigning on
bcdedit /set testsigning off

:: 드라이버 서명 강제 비활성화
:: Disable driver signature enforcement
bcdedit /set nointegritychecks on

:: 부팅 메뉴 타임아웃 설정
:: Set boot menu timeout
bcdedit /timeout 10

:: 현재 설정 전체 조회
:: View all current settings
bcdedit /enum

:: 특정 설정 삭제
:: Delete specific setting
bcdedit /deletevalue nointegritychecks
```

---

## 8.7 문제 해결

### 8.7.1 연결 문제

| 증상 | 가능한 원인 | 해결 방법 |
|------|-----------|----------|
| "Waiting to reconnect..." 계속 표시 | 키 불일치 | `bcdedit /dbgsettings`로 키 재확인 |
| 연결 안 됨 | 방화벽 | 호스트 방화벽에서 포트 허용 |
| 연결 안 됨 | 네트워크 문제 | ping 테스트, IP 확인 |
| 연결 후 끊김 | 네트워크 드라이버 | 다른 네트워크 어댑터 시도 |
| VM이 부팅 안 됨 | 설정 오류 | 안전 모드로 부팅 후 설정 수정 |

### 8.7.2 방화벽 설정

```powershell
# 호스트에서 디버그 포트 허용
# Allow debug port on host
New-NetFirewallRule -DisplayName "Kernel Debug" `
    -Direction Inbound `
    -LocalPort 50000 `
    -Protocol UDP `
    -Action Allow
```

### 8.7.3 심볼 로드 실패

```
:: 심볼 디버그 모드
0: kd> !sym noisy

:: 심볼 강제 리로드
0: kd> .reload /f

:: 특정 모듈 심볼 로드
0: kd> .reload /f nt

:: 심볼 경로 확인
0: kd> .sympath
Symbol search path is: srv*C:\Symbols*https://msdl.microsoft.com/download/symbols

:: 심볼 경로 설정
0: kd> .sympath srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
```

### 8.7.4 연결 복구

VM이 응답하지 않을 때:

```
1. WinDbg에서 Debug → Break (Ctrl+Break)
2. 0: kd> .reboot  (VM 재부팅)
3. 또는 Hyper-V/VMware에서 VM 강제 재시작
```

---

## 8.8 디버깅 워크플로우 정리

```
개발 워크플로우:
┌─────────────────────────────────────────────────────────────┐
│                        호스트 PC                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ Visual Studio   │───▶│   WinDbg        │                │
│  │ (코드 수정)      │    │ (디버깅)         │                │
│  └─────────────────┘    └────────┬────────┘                │
│                                  │                          │
│         ┌───────────────────────┘                          │
│         │ 네트워크/시리얼 디버깅 연결                        │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                      테스트 VM                       │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │  Windows + 드라이버                          │    │   │
│  │  │  - bcdedit /debug on                        │    │   │
│  │  │  - testsigning on                           │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

일반적인 디버깅 세션:
1. WinDbg 시작 → 커널 연결 대기
2. VM 시작/재시작 → 자동 연결
3. Break (Ctrl+Break) → 브레이크포인트 설정
4. Go (g) → 실행 계속
5. 브레이크포인트 히트 → 변수/스택 검사
6. 수정 필요 시 → VS에서 수정 → 재빌드 → VM에 복사 → 리로드
```

---

## 정리

### 체크리스트

- [ ] WinDbg Preview 설치
- [ ] 심볼 경로 설정 (_NT_SYMBOL_PATH)
- [ ] VM에 디버깅 활성화 (bcdedit /debug on)
- [ ] 네트워크 또는 시리얼 디버깅 설정
- [ ] 디버거 연결 성공
- [ ] 기본 명령 테스트 (vertarget, !process)
- [ ] 심볼 로드 확인

### 다음 장 미리보기

Part 4에서는 WinDbg를 마스터합니다:
- VS 디버거 → WinDbg 명령어 매핑
- 핵심 명령어 체계
- 커널 구조체 분석
- BSOD 덤프 분석

---

## 연습 문제

### 1. 디버그 키

네트워크 디버깅에서 키(key)의 역할은 무엇인가요?

### 2. 심볼

심볼 없이 디버깅하면 어떤 문제가 발생하나요?

### 3. Break

WinDbg에서 Break 상태일 때 대상 VM은 어떤 상태인가요?

---

## 참고 자료

- [Setting Up Kernel-Mode Debugging](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-kernel-mode-debugging-in-windbg--cdb--or-ntsd)
- [Symbol Path](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/symbol-path)
- [BCDEdit Command-Line Options](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/bcdedit--dbgsettings)
