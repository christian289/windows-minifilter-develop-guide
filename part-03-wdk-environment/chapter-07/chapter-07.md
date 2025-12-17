# Chapter 7: WDK 개발 환경 설정

## 난이도: ⭐⭐ (초급)

## 학습 목표

- Visual Studio 2022와 WDK 설치
- 드라이버 프로젝트 생성 및 구조 이해
- 테스트 서명 설정
- 테스트용 가상머신 구성

---

## 들어가며

이론은 충분히 배웠습니다. 이제 실제로 드라이버를 개발할 환경을 구축할 차례입니다. .NET 개발자에게 Visual Studio는 익숙하겠지만, WDK(Windows Driver Kit)는 처음일 것입니다.

이 장에서는 개발 환경을 완전히 구축하여 첫 번째 드라이버를 빌드할 준비를 합니다.

---

## 7.1 Visual Studio 2022 설치

### 7.1.1 Visual Studio 다운로드

Visual Studio 2022 Community Edition은 무료로 사용할 수 있습니다:
- 다운로드: https://visualstudio.microsoft.com/vs/

### 7.1.2 필수 워크로드 선택

Visual Studio Installer에서 다음 워크로드와 구성 요소를 선택합니다:

```
☑ Desktop development with C++
   필수 구성 요소:
   ├── ☑ MSVC v143 - VS 2022 C++ x64/x86 build tools (Latest)
   ├── ☑ Windows 11 SDK (10.0.22621.0) 또는 최신
   ├── ☑ C++ ATL for latest build tools (x86 & x64)
   ├── ☑ C++ MFC for latest build tools (x86 & x64)
   └── ☑ MSVC v143 - VS 2022 C++ Spectre-mitigated libs (Latest)

개별 구성 요소 (Individual components) 탭에서:
   ☑ MSVC v143 - VS 2022 C++ x64/x86 Spectre-mitigated libs
```

> ⚠️ **중요**: Spectre-mitigated 라이브러리가 없으면 WDK 빌드 시 오류가 발생합니다.

### 7.1.3 설치 확인

설치 완료 후 확인 사항:

```powershell
# Developer PowerShell for VS 2022 실행
# 컴파일러 버전 확인
cl

# 예상 출력:
# Microsoft (R) C/C++ Optimizing Compiler Version 19.xx.xxxxx for x64
# Copyright (C) Microsoft Corporation.  All rights reserved.
```

---

## 7.2 Windows Driver Kit (WDK) 설치

### 7.2.1 버전 호환성

WDK는 Visual Studio 버전과 맞아야 합니다:

| Visual Studio | WDK 버전 | Windows SDK |
|--------------|----------|-------------|
| VS 2022 17.x | WDK 10.0.22621.x | SDK 10.0.22621.x |
| VS 2022 17.x | WDK 10.0.26100.x | SDK 10.0.26100.x |

### 7.2.2 설치 순서

```
설치 순서 (중요!):
┌─────────────────────────────────────┐
│ 1. Visual Studio 2022 설치          │
│    (Desktop development with C++)   │
├─────────────────────────────────────┤
│ 2. Windows SDK 설치                  │
│    (VS 설치 시 함께 설치됨)          │
├─────────────────────────────────────┤
│ 3. WDK 다운로드 및 설치              │
│    https://go.microsoft.com/        │
│    fwlink/?linkid=2196230           │
├─────────────────────────────────────┤
│ 4. WDK VS Extension 설치             │
│    (WDK 설치 마지막에 자동 제안)     │
└─────────────────────────────────────┘
```

### 7.2.3 WDK 다운로드

1. Microsoft 공식 다운로드 페이지 방문
   - https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk

2. "Download the WDK" 클릭

3. wdksetup.exe 실행

### 7.2.4 WDK 설치 과정

```
WDK 설치 마법사:
├── Install Path: C:\Program Files (x86)\Windows Kits\10\
├── Features: 모두 선택 (기본값)
├── Install
└── 완료 시 "Install Windows Driver Kit Visual Studio Extension" 체크
    → Launch 클릭
```

### 7.2.5 설치 확인

Visual Studio를 재시작한 후:

```
File → New → Project

검색: "Driver" 또는 "Kernel"

다음 템플릿이 보여야 함:
├── Kernel Mode Driver, Empty (KMDF)
├── Kernel Mode Driver (KMDF)
├── User Mode Driver, Empty (UMDF V2)
├── Filter Driver: Filesystem Mini-Filter  ← 우리가 사용할 것
└── ...
```

---

## 7.3 드라이버 프로젝트 생성

### 7.3.1 Minifilter 프로젝트 생성

```
1. Visual Studio 2022 실행
2. File → New → Project
3. 검색창에 "Mini-Filter" 입력
4. "Filter Driver: Filesystem Mini-Filter" 선택
5. Next
6. 프로젝트 설정:
   ├── Project name: SSNProtectFilter
   ├── Location: C:\DriverDev\
   └── Solution name: SSNProtectFilter
7. Create
```

### 7.3.2 프로젝트 구조

생성된 프로젝트 구조:

```
C:\DriverDev\SSNProtectFilter\
├── SSNProtectFilter.sln           # 솔루션 파일
└── SSNProtectFilter\
    ├── SSNProtectFilter.vcxproj   # 프로젝트 파일
    ├── SSNProtectFilter.vcxproj.filters
    ├── SSNProtectFilter.inf       # 설치 정보 파일
    ├── FltMiniFil.c               # 메인 소스 코드
    ├── FltMiniF~1.h               # 헤더 파일
    ├── Resource.rc                # 리소스 파일
    └── sources                    # 빌드 설정 (레거시)
```

### 7.3.3 생성된 코드 살펴보기

```c
// FltMiniFil.c - 자동 생성된 Minifilter 템플릿
// FltMiniFil.c - Auto-generated Minifilter template

#include <fltKernel.h>
#include <dontuse.h>

PFLT_FILTER gFilterHandle;

// Pre-operation 콜백
FLT_PREOP_CALLBACK_STATUS
SamplePreOperation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(Data);
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 콜백 등록
CONST FLT_OPERATION_REGISTRATION Callbacks[] = {
    { IRP_MJ_CREATE,
      0,
      SamplePreOperation,
      NULL },
    { IRP_MJ_OPERATION_END }
};

// 필터 등록
CONST FLT_REGISTRATION FilterRegistration = {
    sizeof(FLT_REGISTRATION),
    FLT_REGISTRATION_VERSION,
    0,
    NULL,
    Callbacks,
    SampleUnload,
    SampleInstanceSetup,
    SampleInstanceQueryTeardown,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL
};

// DriverEntry - 드라이버 진입점
NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath
)
{
    NTSTATUS status;

    status = FltRegisterFilter(DriverObject, &FilterRegistration, &gFilterHandle);
    if (NT_SUCCESS(status)) {
        status = FltStartFiltering(gFilterHandle);
        if (!NT_SUCCESS(status)) {
            FltUnregisterFilter(gFilterHandle);
        }
    }

    return status;
}
```

### 7.3.4 프로젝트 속성 확인

프로젝트를 우클릭 → Properties:

```
Configuration Properties:
├── General
│   ├── Configuration Type: Driver
│   ├── Target Platform: Desktop
│   └── Target OS Version: Windows 10
├── Driver Settings
│   ├── Target OS Version: Windows 10
│   ├── Target Platform: Desktop
│   └── Driver Model: Minifilter
├── C/C++
│   ├── Optimization: Disabled (/Od) [Debug]
│   └── Debug Information Format: Program Database (/Zi)
├── Linker
│   └── Generate Debug Info: Generate Debug Information (/DEBUG)
└── Driver Signing
    ├── Sign Mode: Test Sign
    └── Test Certificate: (설정 필요)
```

---

## 7.4 테스트 서명 설정

Windows 64-bit에서는 서명되지 않은 드라이버를 로드할 수 없습니다. 개발 중에는 **테스트 서명**을 사용합니다.

### 7.4.1 테스트 인증서 생성

Developer PowerShell for VS 2022 또는 관리자 PowerShell에서:

```powershell
# 자체 서명 코드 서명 인증서 생성
# Create self-signed code signing certificate
$cert = New-SelfSignedCertificate `
    -Type CodeSigningCert `
    -Subject "CN=DriverTestCert" `
    -KeyUsage DigitalSignature `
    -FriendlyName "Driver Development Test Certificate" `
    -CertStoreLocation "Cert:\CurrentUser\My" `
    -NotAfter (Get-Date).AddYears(5)

# 인증서 확인
# Verify certificate
Get-ChildItem Cert:\CurrentUser\My -CodeSigningCert

# 인증서 Thumbprint 확인 (나중에 필요)
# Check certificate thumbprint
$cert.Thumbprint
```

### 7.4.2 Visual Studio에서 서명 설정

```
프로젝트 Properties → Configuration Properties → Driver Signing:

├── Sign Mode: Test Sign
├── Test Certificate: <Select From Store...>
│   └── 생성한 "DriverTestCert" 선택
└── File Digest Algorithm: SHA256
```

또는 명령줄로 직접 지정:

```
Test Certificate: $(USERPROFILE)\DriverTestCert.pfx
```

### 7.4.3 인증서 내보내기 (선택)

테스트 VM에 설치할 인증서를 내보냅니다:

```powershell
# PFX 파일로 내보내기 (비밀번호 설정)
# Export to PFX file (with password)
$password = ConvertTo-SecureString -String "TestPassword123" -AsPlainText -Force
Export-PfxCertificate `
    -Cert "Cert:\CurrentUser\My\$($cert.Thumbprint)" `
    -FilePath "C:\DriverDev\DriverTestCert.pfx" `
    -Password $password

# CER 파일로도 내보내기 (공개 키만)
# Also export as CER file (public key only)
Export-Certificate `
    -Cert "Cert:\CurrentUser\My\$($cert.Thumbprint)" `
    -FilePath "C:\DriverDev\DriverTestCert.cer"
```

---

## 7.5 테스트 VM 구성

### 7.5.1 가상화 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| Hyper-V | Windows 내장, 무료 | Pro/Enterprise 필요 |
| VMware Workstation | 안정적, 기능 풍부 | 유료 (Player는 무료) |
| VirtualBox | 무료, 크로스플랫폼 | 드라이버 개발에 제한 |

### 7.5.2 Hyper-V 설정

```powershell
# 관리자 PowerShell에서 Hyper-V 활성화
# Enable Hyper-V in admin PowerShell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All

# 재부팅 후...

# VM 생성
# Create VM
$VMName = "DriverTestVM"
$VHDPath = "C:\VMs\$VMName.vhdx"

# VM 생성
New-VM -Name $VMName `
    -MemoryStartupBytes 4GB `
    -Generation 2 `
    -NewVHDPath $VHDPath `
    -NewVHDSizeBytes 60GB

# 프로세서 수 설정
Set-VMProcessor -VMName $VMName -Count 4

# 동적 메모리 비활성화 (디버깅 안정성)
Set-VMMemory -VMName $VMName -DynamicMemoryEnabled $false

# 보안 부팅 비활성화 (테스트 서명 드라이버용)
Set-VMFirmware -VMName $VMName -EnableSecureBoot Off

# 체크포인트(스냅샷) 위치 설정
Set-VM -VMName $VMName -CheckpointType Standard

# Windows ISO 연결
Set-VMDvdDrive -VMName $VMName -Path "C:\ISO\Windows11.iso"

# VM 시작
Start-VM -Name $VMName
```

### 7.5.3 VMware Workstation 설정

```
1. Create a New Virtual Machine
2. Installer disc image file (iso): Windows 10/11 ISO
3. Guest operating system: Windows 10 x64 또는 Windows 11 x64
4. Virtual machine name: DriverTestVM
5. Maximum disk size: 60 GB (Split into multiple files)
6. Customize Hardware:
   ├── Memory: 4096 MB (4 GB)
   ├── Processors: 4
   ├── Network Adapter: NAT
   └── Display: Accelerate 3D graphics ☐ (비활성화 권장)
7. Finish
```

### 7.5.4 VM에 Windows 설치

1. VM 시작
2. Windows 설치 진행
3. 로컬 계정으로 설정 (Microsoft 계정 불필요)
4. 설치 완료 후 VM Tools/Guest Additions 설치

### 7.5.5 테스트 모드 활성화

VM 내에서 관리자 명령 프롬프트 실행:

```cmd
:: 테스트 서명 모드 활성화
:: Enable test signing mode
bcdedit /set testsigning on

:: 드라이버 서명 강제 비활성화 (추가 보험)
:: Disable driver signature enforcement
bcdedit /set nointegritychecks on

:: 커널 디버깅 활성화 (Chapter 8에서 상세 설명)
:: Enable kernel debugging
bcdedit /debug on

:: 재부팅
:: Reboot
shutdown /r /t 0
```

### 7.5.6 테스트 모드 확인

재부팅 후 바탕화면 우측 하단에 워터마크가 표시됩니다:

```
┌─────────────────────────────────────────────┐
│                                             │
│                 바탕화면                     │
│                                             │
│                                             │
│                     테스트 모드             │
│                     Windows 11 Pro          │
│                     빌드 22621              │
└─────────────────────────────────────────────┘
```

### 7.5.7 스냅샷 생성

> ⚠️ **매우 중요**: 드라이버 테스트 전에 항상 스냅샷을 생성하세요!

```powershell
# Hyper-V
Checkpoint-VM -Name "DriverTestVM" -SnapshotName "Clean State"

# 복원 시
Restore-VMCheckpoint -Name "Clean State" -VMName "DriverTestVM" -Confirm:$false
```

---

## 7.6 첫 번째 빌드

### 7.6.1 빌드 구성 선택

```
Configuration: Debug
Platform: x64

메뉴: Build → Build Solution (Ctrl+Shift+B)
```

### 7.6.2 빌드 출력

성공적인 빌드 후 출력:

```
x64\Debug\SSNProtectFilter\
├── SSNProtectFilter.sys    # 드라이버 바이너리
├── SSNProtectFilter.pdb    # 디버그 심볼
├── SSNProtectFilter.inf    # 설치 정보 파일
├── SSNProtectFilter.cat    # 카탈로그 파일 (서명됨)
└── WdfCoinstaller*.dll     # WDF 코인스톨러 (KMDF의 경우)
```

### 7.6.3 일반적인 빌드 오류

| 오류 메시지 | 원인 | 해결 방법 |
|------------|------|----------|
| `WDK not found` | WDK 미설치 또는 버전 불일치 | WDK 재설치 |
| `Spectre mitigations` | Spectre 라이브러리 미설치 | VS Installer에서 설치 |
| `inf2cat failed` | INF 파일 문법 오류 | INF 파일 검토 |
| `Certificate not found` | 서명 인증서 없음 | 인증서 재생성/재선택 |
| `LNK2019 unresolved external` | 라이브러리 누락 | 프로젝트 설정 확인 |

### 7.6.4 INF 파일 수정

기본 INF 파일에서 Altitude를 설정합니다:

```inf
; SSNProtectFilter.inf
[Version]
Signature   = "$WINDOWS NT$"
Class       = "ContentScreener"              ; 클래스 변경
ClassGuid   = {3e3f0674-c83c-4558-bb26-9820e1eba5c5}
Provider    = %ManufacturerName%
DriverVer   =
CatalogFile = SSNProtectFilter.cat
PnpLockDown = 1

[SourceDisksFiles]
SSNProtectFilter.sys = 1,,

[DestinationDirs]
DefaultDestDir            = 12    ; %windir%\system32\drivers
MiniFilter.DriverFiles    = 12

[DefaultInstall.NTAMD64]
OptionDesc  = %ServiceDescription%
CopyFiles   = MiniFilter.DriverFiles

[DefaultInstall.NTAMD64.Services]
AddService  = %ServiceName%,,MiniFilter.Service

[MiniFilter.Service]
DisplayName     = %ServiceName%
Description     = %ServiceDescription%
ServiceBinary   = %12%\%DriverName%.sys
ServiceType     = 2                         ; SERVICE_FILE_SYSTEM_DRIVER
StartType       = 3                         ; SERVICE_DEMAND_START
ErrorControl    = 1                         ; SERVICE_ERROR_NORMAL
LoadOrderGroup  = "FSFilter Content Screener"
AddReg          = MiniFilter.AddRegistry

[MiniFilter.AddRegistry]
HKR,,"SupportedFeatures",0x00010001,0x3
HKR,"Instances","DefaultInstance",0x00000000,%DefaultInstance%
HKR,"Instances\"%Instance1.Name%,"Altitude",0x00000000,%Instance1.Altitude%
HKR,"Instances\"%Instance1.Name%,"Flags",0x00010001,%Instance1.Flags%

[Strings]
ManufacturerName    = "My Company"
ServiceName         = "SSNProtectFilter"
ServiceDescription  = "SSN Protect Minifilter Driver"
DriverName          = "SSNProtectFilter"
DefaultInstance     = "SSNProtectFilter Instance"
Instance1.Name      = "SSNProtectFilter Instance"
Instance1.Altitude  = "324000"              ; 암호화 범위 내 Altitude
Instance1.Flags     = 0x0
```

---

## 7.7 드라이버 배포 (테스트)

### 7.7.1 파일 복사

호스트에서 VM으로 필요한 파일을 복사합니다:

```
복사할 파일:
├── SSNProtectFilter.sys
├── SSNProtectFilter.inf
├── SSNProtectFilter.cat
└── DriverTestCert.cer (인증서)
```

복사 방법:
- Hyper-V: Enhanced Session Mode 또는 공유 폴더
- VMware: Shared Folders 또는 드래그 앤 드롭
- 둘 다: 네트워크 공유

### 7.7.2 인증서 설치 (VM에서)

```powershell
# VM에서 관리자 PowerShell
# 신뢰할 수 있는 루트 저장소에 인증서 설치
Import-Certificate `
    -FilePath "C:\Drivers\DriverTestCert.cer" `
    -CertStoreLocation "Cert:\LocalMachine\Root"

# 신뢰할 수 있는 게시자 저장소에도 설치
Import-Certificate `
    -FilePath "C:\Drivers\DriverTestCert.cer" `
    -CertStoreLocation "Cert:\LocalMachine\TrustedPublisher"
```

### 7.7.3 드라이버 설치

방법 1: INF 기반 설치

```cmd
:: VM의 관리자 명령 프롬프트
cd C:\Drivers
rundll32.exe setupapi.dll,InstallHinfSection DefaultInstall 132 .\SSNProtectFilter.inf
```

방법 2: sc 명령 (서비스 직접 생성)

```cmd
:: 드라이버 파일 복사
copy SSNProtectFilter.sys C:\Windows\System32\drivers\

:: 서비스 생성
sc create SSNProtectFilter type= filesys binPath= C:\Windows\System32\drivers\SSNProtectFilter.sys

:: Minifilter 등록은 별도 필요...
```

방법 3: fltmc 사용 (Minifilter 권장)

```cmd
:: 드라이버 파일이 drivers 폴더에 있어야 함
fltmc load SSNProtectFilter

:: 로드 확인
fltmc

:: 언로드
fltmc unload SSNProtectFilter
```

### 7.7.4 설치 확인

```cmd
:: 필터 목록 확인
fltmc filters

:: 예상 출력:
Filter Name                     Num Instances    Altitude    Frame
------------------------------  -------------  ------------  -----
SSNProtectFilter                        1       324000           0
luafv                                   1       135000           0
```

---

## 정리

### 개발 환경 체크리스트

- [ ] Visual Studio 2022 설치 (C++ Desktop 워크로드)
- [ ] Spectre-mitigated 라이브러리 설치
- [ ] WDK 설치 및 Extension 설치
- [ ] 테스트 인증서 생성
- [ ] 프로젝트 서명 설정
- [ ] 테스트 VM 생성
- [ ] VM에 테스트 모드 활성화
- [ ] VM 스냅샷 생성
- [ ] 첫 빌드 성공

### 다음 장 미리보기

Chapter 8에서는 커널 디버깅 연결을 설정합니다:
- WinDbg 설치
- 네트워크 디버깅 구성
- 첫 번째 디버거 연결

---

## 연습 문제

### 1. WDK 버전

현재 설치된 WDK 버전을 확인하는 방법은?

### 2. 테스트 모드

bcdedit 명령으로 테스트 모드가 활성화되어 있는지 확인하려면?

### 3. Altitude

Altitude 324000은 어떤 카테고리에 속하나요?

---

## 참고 자료

- [Download the WDK](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk)
- [Building a Driver](https://docs.microsoft.com/en-us/windows-hardware/drivers/develop/building-a-driver)
- [Minifilter Driver Installation](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/installing-a-minifilter-driver)
