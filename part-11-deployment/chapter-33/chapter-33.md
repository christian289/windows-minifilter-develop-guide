# Chapter 33: 드라이버 서명과 배포

## 난이도: ★★★★☆ (중급-고급)

## 학습 목표
- Windows 드라이버 서명 요구사항 이해
- 테스트 서명과 프로덕션 서명 구분
- EV 코드 서명 인증서 획득 및 사용
- WHQL 인증 프로세스 이해

## 33.1 드라이버 서명 개요

### Windows 드라이버 서명 정책

```
+------------------------------------------------------------------+
|                    Windows 드라이버 서명 정책                      |
+------------------------------------------------------------------+
|                                                                   |
|  Windows Vista/7/8:                                               |
|  +------------------+                                             |
|  | 서명 필수        | ←── 64비트 커널 모드 드라이버               |
|  | (테스트 서명 가능)|                                            |
|  +------------------+                                             |
|                                                                   |
|  Windows 10 (버전 1607+):                                         |
|  +------------------+                                             |
|  | EV 서명 또는      | ←── 더 엄격한 요구사항                      |
|  | WHQL 인증 필수   |     (테스트 서명은 테스트 모드에서만)        |
|  +------------------+                                             |
|                                                                   |
|  Windows 11:                                                      |
|  +------------------+                                             |
|  | WHQL 권장        | ←── Microsoft 서명 선호                     |
|  | (EV 허용)        |     Secure Boot와 통합                       |
|  +------------------+                                             |
|                                                                   |
+------------------------------------------------------------------+
```

### 서명 유형 비교

| 서명 유형 | 용도 | 요구사항 | 비용 |
|-----------|------|----------|------|
| 테스트 서명 | 개발/테스트 | 테스트 모드 활성화 필요 | 무료 |
| EV 코드 서명 | 프로덕션 | EV 인증서 구매 필요 | ~$400/년 |
| WHQL 인증 | 프로덕션 (권장) | HLK 테스트 통과 | ~$250/제출 |
| Azure 증명 | 프로덕션 대안 | Partner Center 계정 | 무료 |

### .NET 개발자를 위한 비교

```csharp
// C# - 어셈블리 서명 (유사 개념)
// C# - Assembly signing (similar concept)

// 1. 강력한 이름 서명 (Strong Name Signing)
// - 어셈블리 무결성 확인
// - .snk 키 파일 사용
// [assembly: AssemblyKeyFile("MyKey.snk")]

// 2. Authenticode 서명
// - 게시자 확인
// - 코드 서명 인증서 필요
// signtool sign /f cert.pfx /p password MyApp.exe

// 커널 드라이버는 더 엄격한 요구사항:
// Kernel drivers have stricter requirements:
// - 반드시 Microsoft가 인정하는 인증 기관(CA)의 인증서 필요
// - Must use certificate from Microsoft-trusted CA
// - 64비트 Windows에서 서명 없이 로드 불가
// - Cannot load without signature on 64-bit Windows
// - 테스트 서명도 별도 모드 활성화 필요
// - Even test signing requires special mode activation
```

## 33.2 테스트 서명

### 테스트 인증서 생성

```powershell
# CreateTestCertificate.ps1
# 테스트 서명용 인증서 생성
# Create test signing certificate

# 인증서 저장 경로
# Certificate store path
$certStore = "Cert:\CurrentUser\My"
$rootStore = "Cert:\LocalMachine\Root"
$trustedPublisherStore = "Cert:\LocalMachine\TrustedPublisher"

# 인증서 주체
# Certificate subject
$subject = "CN=DrmFilter Test Certificate, O=MyCompany, C=KR"

# 테스트 인증서 생성
# Create test certificate
$cert = New-SelfSignedCertificate `
    -Type CodeSigningCert `
    -Subject $subject `
    -KeyUsage DigitalSignature `
    -KeyAlgorithm RSA `
    -KeyLength 2048 `
    -HashAlgorithm SHA256 `
    -CertStoreLocation $certStore `
    -NotAfter (Get-Date).AddYears(5)

Write-Host "인증서 생성됨: $($cert.Thumbprint)" -ForegroundColor Green
# Certificate created

# 인증서를 신뢰할 수 있는 루트로 복사 (관리자 권한 필요)
# Copy certificate to trusted root (requires admin)
$certPath = "$certStore\$($cert.Thumbprint)"
$certFile = ".\TestCert.cer"

Export-Certificate -Cert $certPath -FilePath $certFile -Type CERT

# 루트 저장소에 추가
# Add to root store
Import-Certificate -FilePath $certFile -CertStoreLocation $rootStore

# 신뢰할 수 있는 게시자에 추가
# Add to trusted publisher
Import-Certificate -FilePath $certFile -CertStoreLocation $trustedPublisherStore

Write-Host "인증서가 신뢰할 수 있는 저장소에 추가됨" -ForegroundColor Green
# Certificate added to trusted stores

# PFX로 내보내기 (백업용)
# Export as PFX (for backup)
$password = ConvertTo-SecureString -String "TestPassword123!" -Force -AsPlainText
Export-PfxCertificate -Cert $certPath -FilePath ".\TestCert.pfx" -Password $password

Write-Host "PFX 파일 생성됨: TestCert.pfx" -ForegroundColor Green
# PFX file created
```

### 드라이버 테스트 서명

```powershell
# SignTestDriver.ps1
# 테스트 서명 스크립트
# Test signing script

param(
    [Parameter(Mandatory=$true)]
    [string]$DriverPath,

    [Parameter(Mandatory=$true)]
    [string]$CertificateThumbprint,

    [string]$TimestampServer = "http://timestamp.digicert.com"
)

# WDK signtool 경로
# WDK signtool path
$signtool = "C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64\signtool.exe"

# 인증서 확인
# Verify certificate
$cert = Get-ChildItem -Path "Cert:\CurrentUser\My\$CertificateThumbprint" -ErrorAction SilentlyContinue
if ($cert -eq $null) {
    Write-Error "인증서를 찾을 수 없습니다: $CertificateThumbprint"
    # Certificate not found
    exit 1
}

Write-Host "사용 인증서: $($cert.Subject)" -ForegroundColor Cyan
# Using certificate

# 드라이버 서명
# Sign driver
Write-Host "드라이버 서명 중: $DriverPath" -ForegroundColor Yellow
# Signing driver

& $signtool sign `
    /sha1 $CertificateThumbprint `
    /fd SHA256 `
    /t $TimestampServer `
    /v `
    $DriverPath

if ($LASTEXITCODE -ne 0) {
    Write-Error "서명 실패!"
    # Signing failed
    exit 1
}

# CAT 파일도 서명 (있는 경우)
# Sign CAT file too (if exists)
$catPath = [System.IO.Path]::ChangeExtension($DriverPath, ".cat")
if (Test-Path $catPath) {
    Write-Host "CAT 파일 서명 중: $catPath" -ForegroundColor Yellow
    # Signing CAT file

    & $signtool sign `
        /sha1 $CertificateThumbprint `
        /fd SHA256 `
        /t $TimestampServer `
        /v `
        $catPath
}

# 서명 확인
# Verify signature
Write-Host ""
Write-Host "서명 확인 중..." -ForegroundColor Yellow
# Verifying signature

& $signtool verify /pa /v $DriverPath

Write-Host ""
Write-Host "테스트 서명 완료!" -ForegroundColor Green
# Test signing complete
```

### 테스트 모드 활성화

```powershell
# EnableTestSigning.ps1
# 테스트 서명 모드 활성화
# Enable test signing mode

# 관리자 권한 확인
# Check admin rights
if (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Error "관리자 권한으로 실행하세요."
    # Run as administrator
    exit 1
}

# 현재 상태 확인
# Check current state
$bootConfig = bcdedit /enum | Select-String "testsigning"

if ($bootConfig -match "Yes") {
    Write-Host "테스트 서명이 이미 활성화되어 있습니다." -ForegroundColor Yellow
    # Test signing already enabled
}
else {
    Write-Host "테스트 서명 모드 활성화 중..." -ForegroundColor Cyan
    # Enabling test signing mode

    # 테스트 서명 활성화
    # Enable test signing
    bcdedit /set testsigning on

    if ($LASTEXITCODE -eq 0) {
        Write-Host "테스트 서명 모드가 활성화되었습니다." -ForegroundColor Green
        # Test signing mode enabled
        Write-Host "변경 사항을 적용하려면 시스템을 재부팅하세요." -ForegroundColor Yellow
        # Reboot to apply changes
    }
    else {
        Write-Error "테스트 서명 활성화 실패"
        # Failed to enable test signing

        # Secure Boot가 활성화된 경우
        # If Secure Boot is enabled
        Write-Host ""
        Write-Host "참고: Secure Boot가 활성화된 경우 비활성화해야 합니다." -ForegroundColor Yellow
        # Note: Disable Secure Boot if enabled
        Write-Host "BIOS/UEFI 설정에서 Secure Boot를 비활성화하세요." -ForegroundColor Yellow
        # Disable Secure Boot in BIOS/UEFI settings
    }
}

# 비활성화 방법 안내
# How to disable
Write-Host ""
Write-Host "테스트 서명 비활성화: bcdedit /set testsigning off" -ForegroundColor Gray
# To disable test signing
```

## 33.3 EV 코드 서명

### EV 인증서 획득 과정

```
+------------------------------------------------------------------+
|                    EV 인증서 획득 프로세스                         |
+------------------------------------------------------------------+
|                                                                   |
|  1. 인증 기관(CA) 선택                                            |
|     - DigiCert, Sectigo, GlobalSign 등                           |
|     - 가격: $400-500/년                                           |
|                                                                   |
|  2. 신청 서류 제출                                                 |
|     - 사업자등록증                                                 |
|     - 신분증 (대표자)                                              |
|     - 전화 인증                                                    |
|                                                                   |
|  3. 검증 프로세스 (1-2주)                                         |
|     - 회사 실체 확인                                               |
|     - 전화 콜백 인증                                               |
|     - 법적 문서 검토                                               |
|                                                                   |
|  4. HSM(하드웨어 보안 모듈) 수령                                   |
|     - USB 토큰 또는 HSM 장치                                       |
|     - 개인 키는 장치에서만 사용 가능                               |
|                                                                   |
|  5. 드라이버 서명                                                  |
|     - signtool + HSM 사용                                         |
|     - Microsoft 하드웨어 대시보드 등록                             |
|                                                                   |
+------------------------------------------------------------------+
```

### EV 인증서로 서명

```powershell
# SignWithEV.ps1
# EV 인증서로 드라이버 서명
# Sign driver with EV certificate

param(
    [Parameter(Mandatory=$true)]
    [string]$DriverPath,

    [string]$TokenPassword,
    [string]$TimestampServer = "http://timestamp.digicert.com"
)

$signtool = "C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x64\signtool.exe"

# EV 인증서는 보통 토큰에 저장됨
# EV certificate is usually stored on token
# SafeNet, YubiKey 등의 드라이버 필요
# Requires SafeNet, YubiKey, etc. drivers

Write-Host "EV 인증서로 서명 중..." -ForegroundColor Cyan
# Signing with EV certificate

# 방법 1: 자동으로 인증서 선택 (토큰이 연결된 경우)
# Method 1: Auto-select certificate (when token is connected)
& $signtool sign `
    /a `
    /fd SHA256 `
    /tr $TimestampServer `
    /td SHA256 `
    /v `
    $DriverPath

# 방법 2: 특정 인증서 지정
# Method 2: Specify certificate
# & $signtool sign `
#     /n "MyCompany Inc." `
#     /fd SHA256 `
#     /tr $TimestampServer `
#     /td SHA256 `
#     /v `
#     $DriverPath

if ($LASTEXITCODE -ne 0) {
    Write-Error "EV 서명 실패!"
    # EV signing failed
    exit 1
}

# 서명 검증
# Verify signature
Write-Host ""
Write-Host "서명 검증 중..." -ForegroundColor Yellow
# Verifying signature

& $signtool verify /pa /v $DriverPath

Write-Host ""
Write-Host "EV 서명 완료!" -ForegroundColor Green
# EV signing complete
```

## 33.4 WHQL 인증

### HLK (Hardware Lab Kit) 테스트

```
+------------------------------------------------------------------+
|                    WHQL 인증 프로세스                              |
+------------------------------------------------------------------+
|                                                                   |
|  1. 환경 준비                                                      |
|     +------------------+     +------------------+                 |
|     | HLK Controller   |────→| HLK Client       |                 |
|     | (테스트 서버)     |     | (테스트 대상 PC)  |                 |
|     +------------------+     +------------------+                 |
|                                                                   |
|  2. 테스트 실행                                                    |
|     - Device Fundamentals 테스트                                  |
|     - 파일 시스템 Minifilter 테스트                                |
|     - 드라이버 검증기 테스트                                       |
|                                                                   |
|  3. 결과 패키징                                                    |
|     - HLKX 파일 생성                                              |
|     - 테스트 로그 포함                                             |
|                                                                   |
|  4. Partner Center 제출                                           |
|     - HLKX 업로드                                                 |
|     - 드라이버 패키지 제출                                         |
|                                                                   |
|  5. Microsoft 서명                                                 |
|     - 검토 및 승인 (1-2주)                                        |
|     - 서명된 드라이버 다운로드                                     |
|                                                                   |
+------------------------------------------------------------------+
```

### HLK 테스트 자동화 스크립트

```powershell
# RunHLKTests.ps1
# HLK 테스트 자동화
# HLK test automation

param(
    [Parameter(Mandatory=$true)]
    [string]$HLKControllerName,

    [Parameter(Mandatory=$true)]
    [string]$HLKClientName,

    [Parameter(Mandatory=$true)]
    [string]$DriverInfPath,

    [string]$ProjectName = "DrmFilter_WHQL"
)

# HLK Object Model 로드
# Load HLK Object Model
Add-Type -Path "C:\Program Files (x86)\Windows Kits\10\Hardware Lab Kit\Studio\Microsoft.Windows.Kits.Hardware.ObjectModel.dll"
Add-Type -Path "C:\Program Files (x86)\Windows Kits\10\Hardware Lab Kit\Studio\Microsoft.Windows.Kits.Hardware.ObjectModel.DBConnection.dll"
Add-Type -Path "C:\Program Files (x86)\Windows Kits\10\Hardware Lab Kit\Studio\Microsoft.Windows.Kits.Hardware.ObjectModel.Submission.dll"

# 컨트롤러 연결
# Connect to controller
Write-Host "HLK 컨트롤러 연결 중: $HLKControllerName" -ForegroundColor Cyan
# Connecting to HLK Controller

$manager = New-Object Microsoft.Windows.Kits.Hardware.ObjectModel.DBConnection.DatabaseProjectManager($HLKControllerName)

# 새 프로젝트 생성
# Create new project
Write-Host "프로젝트 생성: $ProjectName" -ForegroundColor Yellow
# Creating project

$project = $manager.CreateProject($ProjectName)

# 클라이언트 머신 풀 가져오기
# Get client machine pool
$rootPool = $manager.GetRootMachinePool()
$defaultPool = $rootPool.GetChildPools() | Where-Object { $_.Name -eq "Default Pool" }

# 대상 장치 찾기
# Find target device
Write-Host "대상 장치 검색 중..." -ForegroundColor Yellow
# Searching for target device

$machine = $defaultPool.GetMachines() | Where-Object { $_.Name -eq $HLKClientName }
if ($machine -eq $null) {
    Write-Error "클라이언트 머신을 찾을 수 없습니다: $HLKClientName"
    # Cannot find client machine
    exit 1
}

# 대상 추가
# Add target
$targetFamily = $project.CreateProductInstance("DrmFilter", $defaultPool, $machine.OSPlatform)
$target = $targetFamily.GetTargets() | Where-Object { $_.Name -like "*DrmFilter*" }

if ($target -eq $null) {
    Write-Host "드라이버가 설치되어 있는지 확인하세요." -ForegroundColor Red
    # Ensure driver is installed
    exit 1
}

# 테스트 목록 가져오기
# Get test list
Write-Host "테스트 목록 조회 중..." -ForegroundColor Yellow
# Getting test list

$tests = $target.GetTests()

Write-Host "총 $($tests.Count)개 테스트 발견" -ForegroundColor Green
# Total tests found

# 필수 테스트 필터링
# Filter required tests
$requiredTests = $tests | Where-Object {
    $_.ScheduleOptions -ne [Microsoft.Windows.Kits.Hardware.ObjectModel.TestScheduleOptions]::Manual -and
    $_.GetRequirements().Count -gt 0
}

Write-Host "필수 테스트: $($requiredTests.Count)개" -ForegroundColor Cyan
# Required tests

# 테스트 실행
# Run tests
Write-Host "테스트 실행 중..." -ForegroundColor Yellow
# Running tests

foreach ($test in $requiredTests) {
    Write-Host "  - $($test.Name)" -ForegroundColor Gray

    try {
        $test.QueueTest()
    }
    catch {
        Write-Host "    스케줄 실패: $_" -ForegroundColor Red
        # Schedule failed
    }
}

# 완료 대기
# Wait for completion
Write-Host ""
Write-Host "테스트 완료를 기다리는 중... (오래 걸릴 수 있습니다)" -ForegroundColor Yellow
# Waiting for completion (may take a long time)

do {
    Start-Sleep -Seconds 60

    $runningTests = $project.GetTests() | Where-Object {
        $_.Status -eq [Microsoft.Windows.Kits.Hardware.ObjectModel.TestResultStatus]::InQueue -or
        $_.Status -eq [Microsoft.Windows.Kits.Hardware.ObjectModel.TestResultStatus]::Running
    }

    Write-Host "  실행 중인 테스트: $($runningTests.Count)" -ForegroundColor Gray
    # Running tests

} while ($runningTests.Count -gt 0)

# 결과 요약
# Results summary
Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "           테스트 결과 요약             " -ForegroundColor Cyan
# Test Results Summary
Write-Host "========================================" -ForegroundColor Cyan

$allResults = $project.GetTests()
$passed = ($allResults | Where-Object { $_.Status -eq [Microsoft.Windows.Kits.Hardware.ObjectModel.TestResultStatus]::Passed }).Count
$failed = ($allResults | Where-Object { $_.Status -eq [Microsoft.Windows.Kits.Hardware.ObjectModel.TestResultStatus]::Failed }).Count

Write-Host "통과: $passed" -ForegroundColor Green
# Passed
Write-Host "실패: $failed" -ForegroundColor Red
# Failed

# HLKX 패키지 생성
# Create HLKX package
if ($failed -eq 0) {
    Write-Host ""
    Write-Host "HLKX 패키지 생성 중..." -ForegroundColor Yellow
    # Creating HLKX package

    $packagePath = ".\$ProjectName.hlkx"
    $package = New-Object Microsoft.Windows.Kits.Hardware.ObjectModel.Submission.PackageWriter($project)
    $package.Save($packagePath)

    Write-Host "패키지 생성 완료: $packagePath" -ForegroundColor Green
    # Package created
}
else {
    Write-Host ""
    Write-Host "실패한 테스트가 있어 패키지를 생성할 수 없습니다." -ForegroundColor Red
    # Cannot create package due to failed tests
}
```

## 33.5 Azure 증명 서명

### Partner Center를 통한 서명

```powershell
# AzureAttestation.ps1
# Azure 증명 서명 프로세스
# Azure attestation signing process

# 1. Partner Center 계정 설정
# 1. Partner Center account setup
Write-Host @"
=== Azure 증명 서명 사전 준비 ===
Azure Attestation Signing Prerequisites

1. Microsoft Partner Center 계정 등록
   Register for Microsoft Partner Center account
   https://partner.microsoft.com/dashboard/hardware

2. EV 인증서로 Partner Center 등록
   Register Partner Center with EV certificate
   - 회사 계정 연결
   - Link company account
   - EV 인증서로 조직 확인
   - Verify organization with EV certificate

3. 드라이버 패키지 준비
   Prepare driver package
   - 드라이버 파일 (.sys, .inf, .cat)
   - Driver files (.sys, .inf, .cat)
   - EV 인증서로 서명
   - Sign with EV certificate
"@

# 2. 드라이버 패키지 생성
# 2. Create driver package
function New-DriverSubmissionPackage {
    param(
        [string]$DriverPath,
        [string]$InfPath,
        [string]$OutputPath
    )

    $tempDir = New-Item -ItemType Directory -Path "$env:TEMP\DriverPackage_$(Get-Random)" -Force

    # 드라이버 파일 복사
    # Copy driver files
    Copy-Item $DriverPath -Destination $tempDir
    Copy-Item $InfPath -Destination $tempDir

    # CAT 파일 생성
    # Create CAT file
    $infName = [System.IO.Path]::GetFileNameWithoutExtension($InfPath)
    $catPath = "$tempDir\$infName.cat"

    $inf2cat = "C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x86\inf2cat.exe"

    & $inf2cat /driver:$tempDir /os:10_X64 /verbose

    # CAB으로 패키징
    # Package as CAB
    $cabPath = "$OutputPath\DriverPackage.cab"

    # makecab 사용
    # Use makecab
    $ddfContent = @"
.OPTION EXPLICIT
.Set CabinetNameTemplate=$cabPath
.Set DiskDirectoryTemplate=
.Set CompressionType=MSZIP
.Set Cabinet=on
.Set Compress=on
"@

    Get-ChildItem $tempDir | ForEach-Object {
        $ddfContent += "`n`"$($_.FullName)`""
    }

    $ddfPath = "$tempDir\package.ddf"
    $ddfContent | Out-File -FilePath $ddfPath -Encoding ASCII

    & makecab /F $ddfPath

    # 정리
    # Cleanup
    Remove-Item $tempDir -Recurse -Force

    return $cabPath
}

# 3. Partner Center API로 제출
# 3. Submit via Partner Center API
function Submit-DriverToPartnerCenter {
    param(
        [string]$CabPath,
        [string]$AccessToken
    )

    $apiUrl = "https://manage.devcenter.microsoft.com/v2.0/my/hardware/products"

    # 제품 생성
    # Create product
    $productBody = @{
        productName = "DRM Filter Driver"
        testHarness = "attestation"
        deviceType = "filter"
        announcementDate = (Get-Date).AddDays(7).ToString("yyyy-MM-dd")
    } | ConvertTo-Json

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type" = "application/json"
    }

    $product = Invoke-RestMethod -Uri $apiUrl -Method Post -Headers $headers -Body $productBody

    Write-Host "제품 ID: $($product.id)" -ForegroundColor Green
    # Product ID

    # 제출 생성
    # Create submission
    $submissionUrl = "$apiUrl/$($product.id)/submissions"
    $submission = Invoke-RestMethod -Uri $submissionUrl -Method Post -Headers $headers

    # CAB 업로드
    # Upload CAB
    $uploadUrl = $submission.fileUploadUrl

    Write-Host "드라이버 패키지 업로드 중..." -ForegroundColor Yellow
    # Uploading driver package

    $uploadHeaders = @{
        "x-ms-blob-type" = "BlockBlob"
    }

    Invoke-RestMethod -Uri $uploadUrl -Method Put -Headers $uploadHeaders -InFile $CabPath

    # 제출 커밋
    # Commit submission
    $commitUrl = "$submissionUrl/$($submission.id)/commit"
    Invoke-RestMethod -Uri $commitUrl -Method Post -Headers $headers

    Write-Host "제출 완료! 검토를 기다리세요." -ForegroundColor Green
    # Submission complete! Wait for review.

    return $submission.id
}
```

## 33.6 INF 파일 작성

### 프로덕션 INF 파일

```ini
; DrmFilter.inf
; DRM Filter Driver INF 파일
; DRM Filter Driver INF file

[Version]
Signature   = "$Windows NT$"
Class       = "ActivityMonitor"
ClassGuid   = {b86dff51-a31e-4bac-b3cf-e8cfe75c9fc2}
Provider    = %ManufacturerName%
DriverVer   = 12/15/2025,1.0.0.0
CatalogFile = DrmFilter.cat
PnpLockdown = 1

[DestinationDirs]
DefaultDestDir          = 12          ; %windir%\system32\drivers
DrmFilter.DriverFiles   = 12

[SourceDisksFiles]
DrmFilter.sys = 1,,

[SourceDisksNames]
1 = %DiskName%,,,""

; ===== 설치 섹션 =====
; ===== Install sections =====

[DefaultInstall.NTamd64]
OptionDesc = %ServiceDescription%
CopyFiles  = DrmFilter.DriverFiles

[DefaultInstall.NTamd64.Services]
AddService = %ServiceName%,,DrmFilter.Service

[DefaultUninstall.NTamd64]
LegacyUninstall = 1
DelFiles        = DrmFilter.DriverFiles

[DefaultUninstall.NTamd64.Services]
DelService = %ServiceName%,0x200

; ===== 서비스 정의 =====
; ===== Service definition =====

[DrmFilter.Service]
DisplayName      = %ServiceDisplayName%
Description      = %ServiceDescription%
ServiceBinary    = %12%\DrmFilter.sys
Dependencies     = FltMgr
ServiceType      = 2                    ; SERVICE_FILE_SYSTEM_DRIVER
StartType        = 3                    ; SERVICE_DEMAND_START
ErrorControl     = 1                    ; SERVICE_ERROR_NORMAL
LoadOrderGroup   = FSFilter Activity Monitor
AddReg           = DrmFilter.AddRegistry

; ===== 레지스트리 설정 =====
; ===== Registry settings =====

[DrmFilter.AddRegistry]
HKR,"Instances","DefaultInstance",0x00000000,%DefaultInstance%
HKR,"Instances\"%DefaultInstance%,"Altitude",0x00000000,"365000"
HKR,"Instances\"%DefaultInstance%,"Flags",0x00010001,0x0

; 추가 설정
; Additional settings
HKR,"Parameters","ProtectedExtensions",0x00010000,".docx",".xlsx",".pdf"
HKR,"Parameters","LogLevel",0x00010001,0x1
HKR,"Parameters","EnableBlocking",0x00010001,0x1

; ===== 파일 복사 =====
; ===== File copy =====

[DrmFilter.DriverFiles]
DrmFilter.sys

; ===== 문자열 =====
; ===== Strings =====

[Strings]
ManufacturerName     = "MyCompany Inc."
ServiceName          = "DrmFilter"
ServiceDisplayName   = "DRM Filter Driver"
ServiceDescription   = "파일 시스템 DRM 보호 드라이버"
                     ; File system DRM protection driver
DiskName             = "DRM Filter Installation Disk"
DefaultInstance      = "DrmFilter Instance"

; 지역화된 문자열 (한국어)
; Localized strings (Korean)
[Strings.0412]
ServiceDescription   = "파일 시스템 DRM 보호 드라이버"
DiskName             = "DRM Filter 설치 디스크"
```

### CAT 파일 생성

```powershell
# CreateCatalog.ps1
# 카탈로그 파일 생성
# Create catalog file

param(
    [Parameter(Mandatory=$true)]
    [string]$DriverDirectory,

    [Parameter(Mandatory=$true)]
    [string]$InfName,

    [string]$OSVersion = "10_X64"
)

$inf2cat = "C:\Program Files (x86)\Windows Kits\10\bin\10.0.22621.0\x86\inf2cat.exe"

# INF 파일 검증
# Validate INF file
$infPath = Join-Path $DriverDirectory "$InfName.inf"
if (-not (Test-Path $infPath)) {
    Write-Error "INF 파일을 찾을 수 없습니다: $infPath"
    # INF file not found
    exit 1
}

Write-Host "카탈로그 파일 생성 중..." -ForegroundColor Cyan
# Creating catalog file

# inf2cat 실행
# Run inf2cat
& $inf2cat /driver:$DriverDirectory /os:$OSVersion /verbose

if ($LASTEXITCODE -ne 0) {
    Write-Error "카탈로그 생성 실패!"
    # Catalog creation failed
    exit 1
}

$catPath = Join-Path $DriverDirectory "$InfName.cat"

if (Test-Path $catPath) {
    Write-Host "카탈로그 파일 생성됨: $catPath" -ForegroundColor Green
    # Catalog file created

    # 카탈로그 내용 확인
    # View catalog contents
    Write-Host ""
    Write-Host "카탈로그 내용:" -ForegroundColor Yellow
    # Catalog contents

    & signtool verify /pa /c $catPath $DriverDirectory\*.sys
}
else {
    Write-Error "카탈로그 파일이 생성되지 않았습니다."
    # Catalog file was not created
}
```

## 33.7 설치 프로그램 작성

### NSIS 기반 설치 프로그램

```nsis
; DrmFilterSetup.nsi
; NSIS 설치 스크립트
; NSIS installation script

!include "MUI2.nsh"
!include "x64.nsh"
!include "WinVer.nsh"

; 기본 설정
; Basic settings
Name "DRM Filter Driver"
OutFile "DrmFilterSetup.exe"
InstallDir "$PROGRAMFILES64\DrmFilter"
RequestExecutionLevel admin

; 버전 정보
; Version info
VIProductVersion "1.0.0.0"
VIAddVersionKey "ProductName" "DRM Filter Driver"
VIAddVersionKey "CompanyName" "MyCompany Inc."
VIAddVersionKey "FileVersion" "1.0.0.0"

; UI 설정
; UI settings
!define MUI_ICON "installer.ico"
!define MUI_UNICON "uninstaller.ico"

; 페이지
; Pages
!insertmacro MUI_PAGE_WELCOME
!insertmacro MUI_PAGE_LICENSE "LICENSE.txt"
!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_PAGE_FINISH

!insertmacro MUI_UNPAGE_CONFIRM
!insertmacro MUI_UNPAGE_INSTFILES

!insertmacro MUI_LANGUAGE "Korean"

; 설치 섹션
; Install section
Section "Install"
    SetOutPath $INSTDIR

    ; 64비트 확인
    ; Check 64-bit
    ${IfNot} ${RunningX64}
        MessageBox MB_OK|MB_ICONSTOP "이 프로그램은 64비트 Windows에서만 실행됩니다."
        ; This program requires 64-bit Windows
        Abort
    ${EndIf}

    ; Windows 10 이상 확인
    ; Check Windows 10 or later
    ${IfNot} ${AtLeastWin10}
        MessageBox MB_OK|MB_ICONSTOP "Windows 10 이상이 필요합니다."
        ; Windows 10 or later required
        Abort
    ${EndIf}

    ; 파일 복사
    ; Copy files
    File "DrmFilter.sys"
    File "DrmFilter.inf"
    File "DrmFilter.cat"
    File "DrmFilterService.exe"
    File "DrmFilterManager.exe"

    ; 드라이버 설치
    ; Install driver
    DetailPrint "드라이버 설치 중..."
    ; Installing driver...

    ; DIFX (Driver Install Frameworks) 또는 직접 설치
    ; Using DIFX or direct installation
    nsExec::ExecToLog 'rundll32.exe setupapi.dll,InstallHinfSection DefaultInstall 128 "$INSTDIR\DrmFilter.inf"'
    Pop $0

    ${If} $0 != 0
        MessageBox MB_OK|MB_ICONEXCLAMATION "드라이버 설치에 실패했습니다. 오류 코드: $0"
        ; Driver installation failed. Error code:
    ${EndIf}

    ; 서비스 등록
    ; Register service
    DetailPrint "서비스 등록 중..."
    ; Registering service...

    nsExec::ExecToLog '"$INSTDIR\DrmFilterService.exe" install'

    ; 시작 메뉴 바로가기
    ; Start menu shortcuts
    CreateDirectory "$SMPROGRAMS\DRM Filter"
    CreateShortCut "$SMPROGRAMS\DRM Filter\DRM Filter Manager.lnk" "$INSTDIR\DrmFilterManager.exe"
    CreateShortCut "$SMPROGRAMS\DRM Filter\Uninstall.lnk" "$INSTDIR\Uninstall.exe"

    ; 언인스톨러 생성
    ; Create uninstaller
    WriteUninstaller "$INSTDIR\Uninstall.exe"

    ; 레지스트리에 프로그램 추가/제거 정보
    ; Add to registry for Add/Remove programs
    WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\DrmFilter" \
        "DisplayName" "DRM Filter Driver"
    WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\DrmFilter" \
        "UninstallString" "$\"$INSTDIR\Uninstall.exe$\""
    WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\DrmFilter" \
        "Publisher" "MyCompany Inc."
    WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\DrmFilter" \
        "DisplayVersion" "1.0.0.0"

    ; 드라이버 시작
    ; Start driver
    DetailPrint "드라이버 시작 중..."
    ; Starting driver...

    nsExec::ExecToLog 'fltmc load DrmFilter'

SectionEnd

; 제거 섹션
; Uninstall section
Section "Uninstall"
    ; 드라이버 중지
    ; Stop driver
    DetailPrint "드라이버 중지 중..."
    ; Stopping driver...

    nsExec::ExecToLog 'fltmc unload DrmFilter'

    ; 서비스 제거
    ; Remove service
    DetailPrint "서비스 제거 중..."
    ; Removing service...

    nsExec::ExecToLog '"$INSTDIR\DrmFilterService.exe" uninstall'

    ; 드라이버 제거
    ; Uninstall driver
    DetailPrint "드라이버 제거 중..."
    ; Uninstalling driver...

    nsExec::ExecToLog 'rundll32.exe setupapi.dll,InstallHinfSection DefaultUninstall 128 "$INSTDIR\DrmFilter.inf"'

    ; 파일 삭제
    ; Delete files
    Delete "$INSTDIR\DrmFilter.sys"
    Delete "$INSTDIR\DrmFilter.inf"
    Delete "$INSTDIR\DrmFilter.cat"
    Delete "$INSTDIR\DrmFilterService.exe"
    Delete "$INSTDIR\DrmFilterManager.exe"
    Delete "$INSTDIR\Uninstall.exe"

    ; 바로가기 삭제
    ; Delete shortcuts
    RMDir /r "$SMPROGRAMS\DRM Filter"

    ; 디렉터리 삭제
    ; Remove directory
    RMDir "$INSTDIR"

    ; 레지스트리 정리
    ; Clean registry
    DeleteRegKey HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\DrmFilter"

SectionEnd
```

## 33.8 정리

이 챕터에서 학습한 내용:

1. **서명 요구사항**
   - Windows 버전별 정책
   - 테스트 vs 프로덕션 서명

2. **테스트 서명**
   - 자체 서명 인증서 생성
   - 테스트 모드 활성화
   - 개발 환경 설정

3. **EV 코드 서명**
   - 인증서 획득 과정
   - HSM 토큰 사용
   - signtool 활용

4. **WHQL 인증**
   - HLK 테스트 환경
   - Partner Center 제출
   - Microsoft 서명 획득

5. **Azure 증명**
   - 대안적 서명 방법
   - API 기반 제출

6. **설치 프로그램**
   - INF 파일 작성
   - NSIS 설치 스크립트
   - 자동화된 배포

다음 챕터에서는 **업데이트와 유지보수**를 다룹니다.

---

**다음 챕터 예고**: Chapter 34에서는 드라이버 업데이트 전략, 버전 관리, 롤백 메커니즘을 학습합니다.
