# Chapter 18: 첫 번째 Minifilter 드라이버

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- Minifilter 프로젝트를 생성하고 구성합니다
- 기본 Minifilter 드라이버를 구현합니다
- INF 파일을 작성하여 드라이버를 설치합니다
- 파일 I/O를 모니터링하는 드라이버를 만듭니다
- Minifilter를 빌드, 설치, 테스트합니다

## 도입: 실전 Minifilter 개발

이론을 충분히 학습했으니 이제 직접 Minifilter를 만들어봅니다. 이 챕터에서 만드는 드라이버는 모든 파일 생성(Create) 작업을 로깅하는 간단한 **모니터링 Minifilter**입니다. 이후 챕터에서 이를 확장하여 DRM 기능을 추가합니다.

```csharp
// C# 비유: 우리가 만들 것
public class FileMonitorFilter : IFileSystemFilter
{
    public void OnFileCreate(FileCreateEventArgs e)
    {
        Console.WriteLine($"File accessed: {e.FileName}");
        // 로깅만 수행, 차단하지 않음
    }
}
```

---

## 18.1 Minifilter 프로젝트 생성

### 18.1.1 Visual Studio에서 프로젝트 생성

1. Visual Studio 2022 실행
2. **파일 > 새로 만들기 > 프로젝트**
3. 템플릿 검색: **"Empty WDM Driver"** 또는 **"Kernel Mode Driver (KMDF)"**
4. 프로젝트 이름: **FileMonitor**
5. **만들기** 클릭

> 💡 **팁**: "File System Mini-Filter" 템플릿이 있으면 사용해도 되지만, 직접 구성하는 것이 학습에 도움됩니다.

### 18.1.2 프로젝트 구조 설정

```
FileMonitor/
├── FileMonitor.sln
├── FileMonitor/
│   ├── FileMonitor.vcxproj
│   ├── FileMonitor.inf          // 설치 정보 (우리가 작성)
│   ├── Driver.c                 // 메인 드라이버 코드
│   ├── Callbacks.c              // 콜백 함수들
│   ├── FileMonitor.h            // 공통 헤더
│   └── Context.c                // 컨텍스트 관리 (나중에)
└── x64/
    └── Debug/
        ├── FileMonitor.sys
        ├── FileMonitor.pdb
        ├── FileMonitor.inf
        └── FileMonitor.cat      // 카탈로그 파일
```

### 18.1.3 프로젝트 속성 설정

프로젝트 우클릭 > **속성**에서:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Configuration Properties > C/C++ > General                          │
├─────────────────────────────────────────────────────────────────────┤
│ Additional Include Directories:                                      │
│   $(DDK_INC_PATH)                                                   │
├─────────────────────────────────────────────────────────────────────┤
│ Configuration Properties > Linker > Input                           │
├─────────────────────────────────────────────────────────────────────┤
│ Additional Dependencies:                                             │
│   $(DDK_LIB_PATH)\fltMgr.lib                                        │
│   (또는 그냥 fltMgr.lib)                                            │
├─────────────────────────────────────────────────────────────────────┤
│ Configuration Properties > Inf2Cat > General                        │
├─────────────────────────────────────────────────────────────────────┤
│ Run Inf2Cat: Yes                                                     │
│ Use Local Time: Yes                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 18.2 FileMonitor.h - 공통 헤더

```c
// FileMonitor.h
// 파일 모니터 Minifilter - 공통 헤더

#pragma once

#include <fltKernel.h>
#include <dontuse.h>
#include <suppress.h>

// 풀 태그 정의 (역순 4문자)
#define FM_TAG 'noMF'  // FMon

// 드라이버 이름 및 고도
#define FM_FILTER_NAME    L"FileMonitor"
#define FM_FILTER_ALTITUDE L"265000"

// 디버그 출력 매크로
#if DBG
#define FmDbgPrint(fmt, ...) \
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, \
        "[FileMonitor] " fmt "\n", ##__VA_ARGS__)
#else
#define FmDbgPrint(fmt, ...)
#endif

// 전역 변수 (외부 선언)
extern PFLT_FILTER gFilterHandle;

// 드라이버 함수 선언
DRIVER_INITIALIZE DriverEntry;
NTSTATUS FLTAPI FilterUnload(
    _In_ FLT_FILTER_UNLOAD_FLAGS Flags
);

// 인스턴스 콜백
NTSTATUS FLTAPI InstanceSetup(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_SETUP_FLAGS Flags,
    _In_ DEVICE_TYPE VolumeDeviceType,
    _In_ FLT_FILESYSTEM_TYPE VolumeFilesystemType
);

NTSTATUS FLTAPI InstanceQueryTeardown(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_QUERY_TEARDOWN_FLAGS Flags
);

VOID FLTAPI InstanceTeardownStart(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_TEARDOWN_FLAGS Flags
);

VOID FLTAPI InstanceTeardownComplete(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_TEARDOWN_FLAGS Flags
);

// Operation 콜백
FLT_PREOP_CALLBACK_STATUS FLTAPI PreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
);

FLT_POSTOP_CALLBACK_STATUS FLTAPI PostCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags
);
```

---

## 18.3 Driver.c - 메인 드라이버

```c
// Driver.c
// 파일 모니터 Minifilter - 메인 드라이버

#include "FileMonitor.h"

// 전역 변수
PFLT_FILTER gFilterHandle = NULL;

// Operation 콜백 등록 배열
CONST FLT_OPERATION_REGISTRATION Callbacks[] = {

    // IRP_MJ_CREATE - 파일 열기/생성
    { IRP_MJ_CREATE,
      0,
      PreCreate,      // Pre 콜백
      PostCreate      // Post 콜백
    },

    // 배열 종료 마커 (필수!)
    { IRP_MJ_OPERATION_END }
};

// Filter 등록 구조체
CONST FLT_REGISTRATION FilterRegistration = {

    sizeof(FLT_REGISTRATION),           // Size
    FLT_REGISTRATION_VERSION,           // Version
    0,                                  // Flags

    NULL,                               // ContextRegistration (나중에)
    Callbacks,                          // OperationRegistration

    FilterUnload,                       // FilterUnloadCallback
    InstanceSetup,                      // InstanceSetupCallback
    InstanceQueryTeardown,              // InstanceQueryTeardownCallback
    InstanceTeardownStart,              // InstanceTeardownStartCallback
    InstanceTeardownComplete,           // InstanceTeardownCompleteCallback

    NULL,                               // GenerateFileNameCallback
    NULL,                               // NormalizeNameComponentCallback
    NULL,                               // NormalizeContextCleanupCallback

    NULL,                               // TransactionNotificationCallback

    NULL,                               // NormalizeNameComponentExCallback
    NULL                                // SectionNotificationCallback
};

// ============================================================================
// 드라이버 진입점
// ============================================================================
NTSTATUS DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath
)
{
    NTSTATUS status;

    UNREFERENCED_PARAMETER(RegistryPath);

    FmDbgPrint("DriverEntry: Starting...");
    // DriverEntry: 시작 중...

    // 1. Filter Manager에 등록
    status = FltRegisterFilter(
        DriverObject,
        &FilterRegistration,
        &gFilterHandle
    );

    if (!NT_SUCCESS(status)) {
        FmDbgPrint("FltRegisterFilter failed: 0x%08X", status);
        // FltRegisterFilter 실패: 0x%08X
        return status;
    }

    FmDbgPrint("FltRegisterFilter succeeded");
    // FltRegisterFilter 성공

    // 2. 필터링 시작
    status = FltStartFiltering(gFilterHandle);

    if (!NT_SUCCESS(status)) {
        FmDbgPrint("FltStartFiltering failed: 0x%08X", status);
        // FltStartFiltering 실패: 0x%08X
        FltUnregisterFilter(gFilterHandle);
        return status;
    }

    FmDbgPrint("FltStartFiltering succeeded - Filter is now active!");
    // FltStartFiltering 성공 - 필터 활성화됨!

    return STATUS_SUCCESS;
}

// ============================================================================
// 필터 언로드
// ============================================================================
NTSTATUS FLTAPI FilterUnload(
    _In_ FLT_FILTER_UNLOAD_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(Flags);

    FmDbgPrint("FilterUnload: Unloading...");
    // FilterUnload: 언로드 중...

    // Filter Manager에서 등록 해제
    if (gFilterHandle != NULL) {
        FltUnregisterFilter(gFilterHandle);
        gFilterHandle = NULL;
    }

    FmDbgPrint("FilterUnload: Complete");
    // FilterUnload: 완료

    return STATUS_SUCCESS;
}

// ============================================================================
// 인스턴스 설정 (볼륨 연결)
// ============================================================================
NTSTATUS FLTAPI InstanceSetup(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_SETUP_FLAGS Flags,
    _In_ DEVICE_TYPE VolumeDeviceType,
    _In_ FLT_FILESYSTEM_TYPE VolumeFilesystemType
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    FmDbgPrint("InstanceSetup: VolumeDeviceType=%d, FSType=%d",
        VolumeDeviceType, VolumeFilesystemType);
    // InstanceSetup: 볼륨유형=%d, 파일시스템유형=%d

    // 특정 파일 시스템 제외
    switch (VolumeFilesystemType) {
    case FLT_FSTYPE_RAW:
        FmDbgPrint("  -> Skipping RAW filesystem");
        return STATUS_FLT_DO_NOT_ATTACH;

    case FLT_FSTYPE_CDFS:
        FmDbgPrint("  -> Skipping CDFS (CD-ROM)");
        return STATUS_FLT_DO_NOT_ATTACH;

    default:
        break;
    }

    // 네트워크 파일 시스템 제외 (옵션)
    if (VolumeDeviceType == FILE_DEVICE_NETWORK_FILE_SYSTEM) {
        FmDbgPrint("  -> Skipping network filesystem");
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    FmDbgPrint("  -> Attached successfully");
    // -> 연결 성공

    return STATUS_SUCCESS;
}

// ============================================================================
// 인스턴스 분리 쿼리
// ============================================================================
NTSTATUS FLTAPI InstanceQueryTeardown(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_QUERY_TEARDOWN_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    FmDbgPrint("InstanceQueryTeardown");

    return STATUS_SUCCESS;  // 분리 허용
}

// ============================================================================
// 인스턴스 분리 시작
// ============================================================================
VOID FLTAPI InstanceTeardownStart(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_TEARDOWN_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    FmDbgPrint("InstanceTeardownStart");
}

// ============================================================================
// 인스턴스 분리 완료
// ============================================================================
VOID FLTAPI InstanceTeardownComplete(
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ FLT_INSTANCE_TEARDOWN_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    FmDbgPrint("InstanceTeardownComplete");
}
```

---

## 18.4 Callbacks.c - 콜백 함수

```c
// Callbacks.c
// 파일 모니터 Minifilter - Operation 콜백

#include "FileMonitor.h"

// ============================================================================
// PreCreate - 파일 열기/생성 전
// ============================================================================
FLT_PREOP_CALLBACK_STATUS FLTAPI PreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;

    // 커널 모드 요청은 무시 (시스템 작업)
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 정보 가져오기
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (!NT_SUCCESS(status)) {
        // 이름을 가져올 수 없으면 통과
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 파싱
    status = FltParseFileNameInformation(nameInfo);

    if (NT_SUCCESS(status)) {
        // 프로세스 ID 가져오기
        HANDLE processId = PsGetCurrentProcessId();

        // 파일 접근 로깅
        FmDbgPrint("PreCreate: PID=%d, File=%wZ",
            HandleToUlong(processId),
            &nameInfo->Name);
    }

    // 파일 이름 정보 해제 (중요!)
    FltReleaseFileNameInformation(nameInfo);

    // PostCreate 호출 필요
    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

// ============================================================================
// PostCreate - 파일 열기/생성 후
// ============================================================================
FLT_POSTOP_CALLBACK_STATUS FLTAPI PostCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    // 드레이닝 중이면 빨리 반환
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 성공한 경우만 처리
    if (NT_SUCCESS(Data->IoStatus.Status)) {
        // 파일이 성공적으로 열림
        // 여기서 스트림 컨텍스트 설정 등 가능

        // CreateDisposition 확인 (새로 생성인지 기존 파일인지)
        ULONG disposition = (Data->Iopb->Parameters.Create.Options >> 24) & 0xFF;

        if (disposition == FILE_CREATE ||
            disposition == FILE_OVERWRITE ||
            disposition == FILE_OVERWRITE_IF) {
            // 새 파일 생성 또는 덮어쓰기
            FmDbgPrint("PostCreate: New file created or overwritten");
        }
    } else {
        // 파일 열기 실패
        FmDbgPrint("PostCreate: Open failed with 0x%08X",
            Data->IoStatus.Status);
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## 18.5 FileMonitor.inf - 설치 정보 파일

```ini
;;;
;;; FileMonitor.inf
;;; 파일 모니터 Minifilter 설치 정보
;;;

[Version]
Signature   = "$Windows NT$"
Class       = "ActivityMonitor"             ; WDK 인식 클래스
ClassGuid   = {b86dff51-a31e-4bac-b3cf-e8cfe75c9fc2}
Provider    = %ManufacturerName%
DriverVer   =                               ; 빌드 시 자동 설정
CatalogFile = FileMonitor.cat
PnpLockdown = 1

[DestinationDirs]
DefaultDestDir          = 12                ; %windir%\system32\drivers
FileMonitor.DriverFiles = 12

[DefaultInstall.NTAMD64]
OptionDesc  = %ServiceDescription%
CopyFiles   = FileMonitor.DriverFiles

[DefaultInstall.NTAMD64.Services]
AddService  = %ServiceName%,,FileMonitor.Service

[DefaultUninstall.NTAMD64]
DelFiles    = FileMonitor.DriverFiles
LegacyUninstall = 1

[DefaultUninstall.NTAMD64.Services]
DelService  = %ServiceName%,0x200           ; SPSVCINST_STOPSERVICE

;
; 서비스 구성
;
[FileMonitor.Service]
DisplayName      = %ServiceName%
Description      = %ServiceDescription%
ServiceBinary    = %12%\%DriverName%.sys    ; %windir%\system32\drivers\
Dependencies     = FltMgr                   ; Filter Manager 종속
ServiceType      = 2                        ; SERVICE_FILE_SYSTEM_DRIVER
StartType        = 3                        ; SERVICE_DEMAND_START (수동)
ErrorControl     = 1                        ; SERVICE_ERROR_NORMAL
LoadOrderGroup   = "FSFilter Activity Monitor"
AddReg           = FileMonitor.AddRegistry

;
; 레지스트리 설정
;
[FileMonitor.AddRegistry]
HKR,"Instances","DefaultInstance",0x00000000,%DefaultInstance%
HKR,"Instances\"%Instance.Name%,"Altitude",0x00000000,%Instance.Altitude%
HKR,"Instances\"%Instance.Name%,"Flags",0x00010001,0x0

;
; 파일 복사
;
[FileMonitor.DriverFiles]
%DriverName%.sys

;
; 소스 디스크
;
[SourceDisksFiles]
FileMonitor.sys = 1,,

[SourceDisksNames]
1 = %DiskName%,,,

;
; 문자열
;
[Strings]
ManufacturerName        = "My Company"
ServiceName             = "FileMonitor"
ServiceDescription      = "File System Activity Monitor Minifilter"
DriverName              = "FileMonitor"
DiskName                = "FileMonitor Installation Disk"

; 인스턴스 설정
DefaultInstance         = "FileMonitor Instance"
Instance.Name           = "FileMonitor Instance"
Instance.Altitude       = "265000"
```

---

## 18.6 빌드 및 설치

### 18.6.1 빌드

```
1. Visual Studio에서:
   - 빌드 > 솔루션 빌드 (Ctrl+Shift+B)

2. 빌드 결과물 확인:
   x64\Debug\FileMonitor\
   ├── FileMonitor.sys     ← 드라이버 파일
   ├── FileMonitor.pdb     ← 디버그 심볼
   ├── FileMonitor.inf     ← 설치 정보
   └── FileMonitor.cat     ← 카탈로그 (테스트 서명됨)
```

### 18.6.2 설치 (관리자 권한 명령 프롬프트)

```cmd
:: 대상 머신의 관리자 명령 프롬프트에서:

:: 1. 테스트 모드 활성화 (아직 안 했으면)
bcdedit /set testsigning on
:: 재부팅 필요!

:: 2. 드라이버 파일 복사
copy FileMonitor.sys C:\Windows\System32\drivers\
copy FileMonitor.inf C:\Windows\System32\drivers\
copy FileMonitor.cat C:\Windows\System32\drivers\

:: 3. INF로 설치
cd C:\Windows\System32\drivers
rundll32.exe setupapi.dll,InstallHinfSection DefaultInstall 132 .\FileMonitor.inf

:: 또는 PNPUTIL 사용 (Windows 10+)
pnputil /add-driver FileMonitor.inf /install
```

### 18.6.3 수동 설치 (sc 명령)

```cmd
:: INF 없이 수동 설치

:: 1. 서비스 생성
sc create FileMonitor type=filesys binPath="C:\Windows\System32\drivers\FileMonitor.sys"

:: 2. Altitude 레지스트리 설정
reg add "HKLM\SYSTEM\CurrentControlSet\Services\FileMonitor\Instances" /v "DefaultInstance" /t REG_SZ /d "FileMonitor Instance" /f
reg add "HKLM\SYSTEM\CurrentControlSet\Services\FileMonitor\Instances\FileMonitor Instance" /v "Altitude" /t REG_SZ /d "265000" /f
reg add "HKLM\SYSTEM\CurrentControlSet\Services\FileMonitor\Instances\FileMonitor Instance" /v "Flags" /t REG_DWORD /d 0 /f

:: 3. 서비스 시작
sc start FileMonitor
```

### 18.6.4 설치 확인

```cmd
:: 필터 상태 확인
fltmc

:: 출력 예:
Filter Name                     Num Instances    Altitude    Frame
------------------------------  -------------    --------    -----
FileMonitor                             3        265000         0
WdFilter                                5        328010         0

:: 특정 필터 상세 정보
fltmc instances -f FileMonitor
```

---

## 18.7 테스트

### 18.7.1 DebugView로 로그 확인

```
1. DebugView를 관리자 권한으로 실행
2. Capture > Capture Kernel 활성화
3. 파일을 열거나 생성
4. 로그 확인:

[FileMonitor] PreCreate: PID=1234, File=\Device\HarddiskVolume2\Users\User\test.txt
[FileMonitor] PostCreate: New file created or overwritten
```

### 18.7.2 WinDbg로 디버깅

```
// 호스트 PC에서:
windbg -k net:port=50000,key=1.2.3.4

// 심볼 경로 추가
.sympath+ C:\Projects\FileMonitor\x64\Debug

// 필터 확인
!fltkd.filters

// 콜백에 브레이크포인트
bp FileMonitor!PreCreate
g

// 파일 열기 시 브레이크
[FileMonitor] PreCreate...
Breakpoint 0 hit
FileMonitor!PreCreate:
fffff801`12340000 mov     rbp,rsp

// 파일 이름 확인
!fltkd.cbd @rcx
```

### 18.7.3 드라이버 언로드

```cmd
:: 필터 언로드
fltmc unload FileMonitor

:: 또는 서비스 중지
sc stop FileMonitor

:: 서비스 삭제 (필요 시)
sc delete FileMonitor
```

---

## 18.8 확장: Read/Write 모니터링 추가

```c
// Callbacks 배열에 추가
CONST FLT_OPERATION_REGISTRATION Callbacks[] = {

    { IRP_MJ_CREATE, 0, PreCreate, PostCreate },

    // Read 모니터링 추가
    { IRP_MJ_READ, 0, PreRead, NULL },

    // Write 모니터링 추가
    { IRP_MJ_WRITE, 0, PreWrite, NULL },

    { IRP_MJ_OPERATION_END }
};

// PreRead 콜백
FLT_PREOP_CALLBACK_STATUS FLTAPI PreRead(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 읽기 크기와 오프셋
    ULONG readLength = Data->Iopb->Parameters.Read.Length;
    LARGE_INTEGER offset = Data->Iopb->Parameters.Read.ByteOffset;

    FmDbgPrint("PreRead: Length=%lu, Offset=%lld",
        readLength, offset.QuadPart);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// PreWrite 콜백
FLT_PREOP_CALLBACK_STATUS FLTAPI PreWrite(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 쓰기 크기와 오프셋
    ULONG writeLength = Data->Iopb->Parameters.Write.Length;
    LARGE_INTEGER offset = Data->Iopb->Parameters.Write.ByteOffset;

    FmDbgPrint("PreWrite: Length=%lu, Offset=%lld",
        writeLength, offset.QuadPart);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 18.9 일반적인 문제와 해결

### 18.9.1 설치 오류

```
┌─────────────────────────────────────────────────────────────────────┐
│ 문제: 서비스 생성 시 "ERROR_SERVICE_EXISTS"                         │
├─────────────────────────────────────────────────────────────────────┤
│ 해결: sc delete FileMonitor 후 다시 생성                            │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 문제: 드라이버 로드 시 "ERROR_DRIVER_BLOCKED"                       │
├─────────────────────────────────────────────────────────────────────┤
│ 해결: bcdedit /set testsigning on 후 재부팅                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 문제: fltmc에서 필터가 보이지 않음                                  │
├─────────────────────────────────────────────────────────────────────┤
│ 해결: Altitude 레지스트리 설정 확인, sc start 실행 확인             │
└─────────────────────────────────────────────────────────────────────┘
```

### 18.9.2 런타임 오류

```
┌─────────────────────────────────────────────────────────────────────┐
│ 문제: BSOD - IRQL_NOT_LESS_OR_EQUAL                                 │
├─────────────────────────────────────────────────────────────────────┤
│ 해결: FltReleaseFileNameInformation 누락 확인                       │
│       Paged 메모리 사용 여부 확인                                   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 문제: 로그가 안 보임                                                │
├─────────────────────────────────────────────────────────────────────┤
│ 해결: WinDbg 연결 시 DebugView 안 됨                                │
│       커널 디버거 연결 해제 후 DebugView 사용                       │
│       또는 WinDbg에서 직접 확인                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 18.10 완성된 코드 요약

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FileMonitor 프로젝트 구조                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  FileMonitor.h      - 공통 헤더, 함수 선언                          │
│  Driver.c           - DriverEntry, FilterUnload, Instance 콜백      │
│  Callbacks.c        - PreCreate, PostCreate, PreRead, PreWrite      │
│  FileMonitor.inf    - 설치 정보, Altitude 설정                      │
│                                                                      │
│  주요 기능:                                                          │
│  ├─ FltRegisterFilter로 Filter Manager에 등록                       │
│  ├─ FltStartFiltering으로 필터링 시작                               │
│  ├─ PreCreate에서 파일 접근 로깅                                    │
│  ├─ FltGetFileNameInformation으로 파일 이름 획득                    │
│  └─ InstanceSetup에서 특정 볼륨 제외                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 요약

이 챕터에서 구현한 내용:

1. **프로젝트 설정**: Minifilter 프로젝트 구성, fltMgr.lib 링크
2. **FLT_REGISTRATION**: 필터 등록 구조체 작성
3. **DriverEntry**: FltRegisterFilter, FltStartFiltering
4. **콜백**: PreCreate, PostCreate로 파일 접근 모니터링
5. **파일 이름**: FltGetFileNameInformation 사용법
6. **INF 파일**: Altitude 설정, 설치 정보
7. **설치/테스트**: fltmc, DebugView, WinDbg

다음 챕터에서는 Minifilter 콜백을 심화 학습합니다.

---

## 참고 자료

- [FltRegisterFilter](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltregisterfilter)
- [FltStartFiltering](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltstartfiltering)
- [FltGetFileNameInformation](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltgetfilenameinformation)
- [INF File Sections](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/creating-an-inf-file-for-a-minifilter-driver)
