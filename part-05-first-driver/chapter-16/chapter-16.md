# Chapter 16: 레거시 필터 드라이버 이해 (개념)

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- 레거시 파일 시스템 필터의 아키텍처를 이해합니다
- 레거시 필터의 문제점과 복잡성을 파악합니다
- Minifilter가 등장한 이유를 이해합니다
- 레거시 필터와 Minifilter의 차이를 비교합니다
- 왜 새로운 드라이버는 Minifilter로 개발해야 하는지 이해합니다

## 도입: 역사를 알면 현재가 보인다

Minifilter를 제대로 이해하려면 그 이전의 **레거시 필터 드라이버**를 알아야 합니다. 마치 .NET의 async/await를 이해하기 위해 Callback 패턴과 Begin/End 패턴을 알면 좋은 것처럼, Minifilter의 설계 철학을 이해하려면 레거시 필터의 문제점을 알아야 합니다.

> ⚠️ **주의**: 이 챕터는 **개념 학습용**입니다. 새로운 드라이버 개발에는 **반드시 Minifilter**를 사용하세요. 레거시 필터 코드를 직접 작성할 필요는 없습니다.

---

## 16.1 파일 시스템 필터의 필요성

### 16.1.1 파일 시스템 필터란?

파일 시스템 필터는 파일 I/O 요청을 **가로채서** 검사하거나 수정하는 드라이버입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   파일 시스템 필터의 용도                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  • 안티바이러스: 파일 읽기 시 악성코드 검사                         │
│  • 암호화: 파일 쓰기 시 자동 암호화, 읽기 시 복호화                 │
│  • 백업: 파일 변경 감지 및 실시간 백업                              │
│  • 감사: 파일 접근 로깅                                             │
│  • DRM: 민감 문서 유출 방지                                         │
│  • 압축: 투명한 파일 압축                                           │
│  • 가상화: 파일 시스템 가상화/리다이렉션                            │
│  • HSM: 계층적 스토리지 관리                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.1.2 왜 커널에서 필터링해야 하는가?

```csharp
// 사용자 모드에서 파일 모니터링? (C# 예시)

// 방법 1: FileSystemWatcher
var watcher = new FileSystemWatcher(@"C:\");
watcher.Changed += (s, e) => Console.WriteLine($"Changed: {e.FullPath}");
// 문제: 변경 후에야 알림 받음, 차단 불가, 우회 가능

// 방법 2: API 후킹
// 문제: 불안정, 호환성 문제, 안티치트에 의해 탐지됨

// 커널 필터가 필요한 이유:
// 1. 모든 파일 접근을 확실히 가로챔
// 2. 접근 전에 차단 가능
// 3. 우회 불가능 (커널 레벨)
// 4. 성능 최적화 가능
```

---

## 16.2 레거시 필터 아키텍처

### 16.2.1 디바이스 스택 개념

Windows의 I/O는 **디바이스 스택**을 통해 전달됩니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       디바이스 스택 구조                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  사용자 애플리케이션                                                │
│         │                                                           │
│         ▼ CreateFile()                                              │
│  ┌─────────────────────┐                                           │
│  │   I/O Manager       │                                           │
│  └─────────────────────┘                                           │
│         │ IRP (I/O Request Packet)                                  │
│         ▼                                                           │
│  ┌─────────────────────┐                                           │
│  │   필터 드라이버 1   │  ← 최상위 (먼저 받음)                     │
│  │   (예: 안티바이러스) │                                           │
│  └─────────────────────┘                                           │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────┐                                           │
│  │   필터 드라이버 2   │                                           │
│  │   (예: 암호화)      │                                           │
│  └─────────────────────┘                                           │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────┐                                           │
│  │   파일 시스템       │  ← NTFS, FAT32 등                         │
│  │   (NTFS.sys)        │                                           │
│  └─────────────────────┘                                           │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────┐                                           │
│  │   스토리지 드라이버 │  ← 디스크, SSD 등                         │
│  └─────────────────────┘                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.2.2 레거시 필터의 연결 방식

레거시 필터는 **IoAttachDeviceToDeviceStack**으로 디바이스 스택에 연결됩니다.

```c
// 레거시 필터 드라이버의 연결 코드 (개념 설명용)

// 1. 자신의 디바이스 오브젝트 생성
PDEVICE_OBJECT filterDevice;
IoCreateDevice(
    DriverObject,
    sizeof(DEVICE_EXTENSION),
    NULL,
    FILE_DEVICE_DISK_FILE_SYSTEM,
    0,
    FALSE,
    &filterDevice
);

// 2. 대상 디바이스 스택에 연결
PDEVICE_OBJECT lowerDevice = IoAttachDeviceToDeviceStack(
    filterDevice,       // 우리 디바이스
    targetDevice        // 연결할 대상 (파일 시스템 디바이스)
);

// 이제 targetDevice로 가는 모든 IRP는 filterDevice를 먼저 거침!

// 3. DeviceExtension에 하위 디바이스 저장
PDEVICE_EXTENSION devExt = filterDevice->DeviceExtension;
devExt->LowerDeviceObject = lowerDevice;
```

### 16.2.3 IRP 처리 흐름

```c
// 레거시 필터의 IRP 처리 (개념 설명용)

NTSTATUS FilterDispatchRead(
    PDEVICE_OBJECT DeviceObject,
    PIRP Irp
)
{
    PDEVICE_EXTENSION devExt = DeviceObject->DeviceExtension;

    // 옵션 1: IRP를 그대로 하위로 전달
    IoSkipCurrentIrpStackLocation(Irp);
    return IoCallDriver(devExt->LowerDeviceObject, Irp);

    // 옵션 2: IRP를 수정하고 전달
    // IoCopyCurrentIrpStackLocationToNext(Irp);
    // ... 수정 ...
    // return IoCallDriver(devExt->LowerDeviceObject, Irp);

    // 옵션 3: IRP 완료 (하위로 전달하지 않음)
    // Irp->IoStatus.Status = STATUS_ACCESS_DENIED;
    // Irp->IoStatus.Information = 0;
    // IoCompleteRequest(Irp, IO_NO_INCREMENT);
    // return STATUS_ACCESS_DENIED;
}
```

---

## 16.3 레거시 필터의 문제점

### 16.3.1 연결 순서 문제

```
┌─────────────────────────────────────────────────────────────────────┐
│                   문제 1: 연결 순서 비결정적                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  필터 A (안티바이러스)가 먼저 로드되면:                             │
│     앱 → 필터 A → 필터 B → NTFS                                    │
│                                                                      │
│  필터 B (암호화)가 먼저 로드되면:                                   │
│     앱 → 필터 B → 필터 A → NTFS                                    │
│                                                                      │
│  결과: 암호화된 데이터를 안티바이러스가 검사할 수도 있고,           │
│       복호화된 데이터를 검사할 수도 있음!                            │
│                                                                      │
│  해결책: 없음! 로드 순서에 의존하는 불안정한 동작                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.3.2 볼륨 마운트 타이밍 문제

```c
// 문제 2: 볼륨 마운트 시점을 어떻게 알 것인가?

// 레거시 필터는 다음 방법 중 하나를 사용해야 함:
// 1. IoRegisterFsRegistrationChange - 파일 시스템 등록 감지
// 2. 파일 시스템 제어 디바이스 모니터링
// 3. 수동 폴링

// 문제:
// - 드라이버 로드 전에 마운트된 볼륨은 놓침
// - 마운트/언마운트 시점 동기화 어려움
// - 레이스 컨디션 발생 가능
```

### 16.3.3 복잡한 IRP 처리

```c
// 문제 3: 모든 IRP 유형을 직접 처리해야 함

// DriverEntry에서 모든 IRP 핸들러 등록
for (i = 0; i <= IRP_MJ_MAXIMUM_FUNCTION; i++) {
    DriverObject->MajorFunction[i] = FilterDispatchPassThrough;
}

// 특수 처리가 필요한 IRP 따로 등록
DriverObject->MajorFunction[IRP_MJ_CREATE] = FilterDispatchCreate;
DriverObject->MajorFunction[IRP_MJ_READ] = FilterDispatchRead;
DriverObject->MajorFunction[IRP_MJ_WRITE] = FilterDispatchWrite;
DriverObject->MajorFunction[IRP_MJ_CLEANUP] = FilterDispatchCleanup;
DriverObject->MajorFunction[IRP_MJ_CLOSE] = FilterDispatchClose;
DriverObject->MajorFunction[IRP_MJ_SET_INFORMATION] = FilterDispatchSetInfo;
DriverObject->MajorFunction[IRP_MJ_QUERY_INFORMATION] = FilterDispatchQueryInfo;
// ... 28개 IRP 유형 모두!

// FastIO도 별도 처리 필요
FAST_IO_DISPATCH fastIoDispatch = { ... };  // 20+ 함수 포인터
```

### 16.3.4 재진입과 재귀 문제

```c
// 문제 4: 재진입 문제

NTSTATUS FilterDispatchCreate(PDEVICE_OBJECT Device, PIRP Irp)
{
    // 파일 열기 요청 처리 중...

    // 다른 파일을 열어야 할 경우 (로그 파일 등)
    HANDLE logFile;
    ZwCreateFile(&logFile, ...);  // ← 이것도 IRP_MJ_CREATE!
    // 자기 자신이 다시 호출됨 (재귀)

    // 문제:
    // - 무한 재귀 가능
    // - 락 데드락 가능
    // - 이 IRP가 자기 것인지 아닌지 구분 필요
}

// 해결을 위한 복잡한 코드 필요:
// - 현재 스레드 추적
// - 재진입 카운터
// - ShadowDevice 기법
// - ...
```

### 16.3.5 IRP 완료 처리의 복잡성

```c
// 문제 5: 비동기 IRP 완료 처리

NTSTATUS FilterDispatchRead(PDEVICE_OBJECT Device, PIRP Irp)
{
    // 완료 루틴 설정
    IoSetCompletionRoutine(
        Irp,
        FilterReadComplete,    // 완료 시 호출됨
        Context,               // 컨텍스트
        TRUE,                  // 성공 시 호출
        TRUE,                  // 실패 시 호출
        TRUE                   // 취소 시 호출
    );

    // 하위로 전달
    return IoCallDriver(lowerDevice, Irp);
}

NTSTATUS FilterReadComplete(
    PDEVICE_OBJECT Device,
    PIRP Irp,
    PVOID Context
)
{
    // 여기서 IRQL은?
    // - DISPATCH_LEVEL일 수 있음!
    // - 페이지드 메모리 접근 불가
    // - 복잡한 처리 불가

    // IRP가 Pending 상태였으면 특별 처리 필요
    if (Irp->PendingReturned) {
        IoMarkIrpPending(Irp);
    }

    return STATUS_SUCCESS;
}
```

### 16.3.6 호환성 문제

```
┌─────────────────────────────────────────────────────────────────────┐
│                   문제 6: 다른 필터와의 충돌                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  • 필터 A가 IRP를 완료해버리면 필터 B는 아예 못 봄                  │
│  • 필터 A가 IRP를 수정하면 필터 B가 잘못된 데이터 처리             │
│  • 양쪽이 같은 락을 사용하면 데드락                                 │
│  • 버그가 발생하면 어느 필터 문제인지 파악 어려움                   │
│                                                                      │
│  실제 사례:                                                          │
│  - 안티바이러스 + 백업 소프트웨어 = BSOD                            │
│  - 암호화 + 압축 필터 = 데이터 손상                                 │
│  - 두 안티바이러스 동시 설치 = 시스템 불안정                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 16.4 Minifilter의 등장

### 16.4.1 Filter Manager 아키텍처

Microsoft는 이러한 문제를 해결하기 위해 **Filter Manager (fltmgr.sys)**를 도입했습니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Filter Manager 아키텍처                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  사용자 애플리케이션                                                │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────┐                       │
│  │         Filter Manager (fltmgr.sys)     │                       │
│  │  ┌─────────────────────────────────┐    │                       │
│  │  │  Minifilter A (고도: 320000)   │    │  ← 안티바이러스       │
│  │  ├─────────────────────────────────┤    │                       │
│  │  │  Minifilter B (고도: 130000)   │    │  ← 우리 DRM 필터      │
│  │  ├─────────────────────────────────┤    │                       │
│  │  │  Minifilter C (고도: 45000)    │    │  ← 파일 정보          │
│  │  └─────────────────────────────────┘    │                       │
│  │                                          │                       │
│  │  • 고도(Altitude)로 순서 보장            │                       │
│  │  • 공통 인프라 제공                      │                       │
│  │  • 콜백 기반 단순 API                    │                       │
│  └─────────────────────────────────────────┘                       │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────┐                                           │
│  │   파일 시스템       │                                           │
│  └─────────────────────┘                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.4.2 Minifilter의 장점

```
┌─────────────────────────────────────────────────────────────────────┐
│           레거시 필터 vs Minifilter 비교                             │
├─────────────────────────────────────────────────────────────────────┤
│  문제                 레거시 필터           Minifilter               │
├─────────────────────────────────────────────────────────────────────┤
│  연결 순서            비결정적              고도(Altitude)로 보장    │
│  볼륨 마운트          수동 감지             자동 알림                │
│  IRP 처리             모든 유형 구현        필요한 것만 콜백 등록   │
│  FastIO               별도 구현             Filter Manager 처리     │
│  재진입               수동 처리             FltIs... 함수 제공      │
│  완료 루틴            복잡한 IRQL 관리      PostOperation 콜백      │
│  컨텍스트             DeviceExtension       FltAllocateContext      │
│  파일 이름            수동 구축             FltGetFileNameInfo      │
│  개발 복잡도          매우 높음             상대적으로 낮음         │
│  코드 크기            수천 줄               수백 줄                  │
│  디버깅               어려움                !fltkd 확장 지원        │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.4.3 고도 (Altitude) 시스템

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Altitude 범위                                 │
├─────────────────────────────────────────────────────────────────────┤
│  420000 - 429999    Filter                                           │
│  400000 - 409999    Top                                              │
│  320000 - 329998    Anti-Virus        ← 안티바이러스                │
│  300000 - 309998    Replication                                      │
│  260000 - 269998    Continuous Backup                                │
│  240000 - 249999    Content Screener  ← DLP/DRM 가능 위치           │
│  220000 - 229999    Quota Management                                 │
│  180000 - 189999    System Recovery                                  │
│  140000 - 149999    Encryption        ← 암호화                      │
│  130000 - 139999    Compression                                      │
│  80000 - 89999      HSM                                              │
│  60000 - 69999      Copy Protection                                  │
│  40000 - 49999      Bottom            ← 파일 정보 수집             │
└─────────────────────────────────────────────────────────────────────┘

// 고도는 Microsoft에 요청하여 공식 할당받음
// 테스트 시에는 임의 값 사용 가능
```

---

## 16.5 레거시 필터 코드 vs Minifilter 코드

### 16.5.1 레거시 필터: IRP_MJ_CREATE 처리 (약 200줄)

```c
// 레거시 필터 - IRP_MJ_CREATE 처리 (간략화된 버전도 복잡함)

// 1. 디스패치 루틴 등록
DriverObject->MajorFunction[IRP_MJ_CREATE] = FilterDispatchCreate;

// 2. 디스패치 함수
NTSTATUS FilterDispatchCreate(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PDEVICE_EXTENSION devExt = DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);
    PFILE_OBJECT fileObject = irpSp->FileObject;
    NTSTATUS status;

    // 재진입 체크
    if (IsReentry()) {
        IoSkipCurrentIrpStackLocation(Irp);
        return IoCallDriver(devExt->LowerDeviceObject, Irp);
    }

    // 파일 이름 구축 (매우 복잡!)
    UNICODE_STRING fullPath;
    status = BuildFullPathName(DeviceObject, fileObject, &fullPath);
    if (!NT_SUCCESS(status)) {
        // 에러 처리...
    }

    // 파일 검사 로직
    if (ShouldBlockFile(&fullPath)) {
        Irp->IoStatus.Status = STATUS_ACCESS_DENIED;
        Irp->IoStatus.Information = 0;
        IoCompleteRequest(Irp, IO_NO_INCREMENT);
        FreePathName(&fullPath);
        return STATUS_ACCESS_DENIED;
    }

    // 완료 루틴 설정
    IoSetCompletionRoutine(Irp, FilterCreateComplete, context, TRUE, TRUE, TRUE);

    // 하위로 전달
    IoCopyCurrentIrpStackLocationToNext(Irp);
    status = IoCallDriver(devExt->LowerDeviceObject, Irp);

    FreePathName(&fullPath);
    return status;
}

// 3. 완료 루틴
NTSTATUS FilterCreateComplete(PDEVICE_OBJECT Device, PIRP Irp, PVOID Context)
{
    if (Irp->PendingReturned) {
        IoMarkIrpPending(Irp);
    }

    // DISPATCH_LEVEL에서 실행될 수 있음 - 제한된 작업만 가능

    return STATUS_SUCCESS;
}

// 4. 파일 경로 구축 함수 (별도 100줄 이상)
// 5. 재진입 체크 함수
// 6. FastIO 처리 함수들
// ... 계속
```

### 16.5.2 Minifilter: 같은 기능 (약 50줄)

```c
// Minifilter - IRP_MJ_CREATE 처리

// 1. 콜백 등록
const FLT_OPERATION_REGISTRATION OperationCallbacks[] = {
    { IRP_MJ_CREATE, 0, PreCreate, PostCreate },
    { IRP_MJ_OPERATION_END }
};

// 2. PreOperation 콜백
FLT_PREOP_CALLBACK_STATUS PreCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;

    // 파일 이름 획득 (한 줄!)
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (NT_SUCCESS(status)) {
        status = FltParseFileNameInformation(nameInfo);

        if (NT_SUCCESS(status)) {
            // 파일 검사
            if (ShouldBlockFile(&nameInfo->Name)) {
                Data->IoStatus.Status = STATUS_ACCESS_DENIED;
                FltReleaseFileNameInformation(nameInfo);
                return FLT_PREOP_COMPLETE;  // 완료, 하위로 안 감
            }
        }

        FltReleaseFileNameInformation(nameInfo);
    }

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;  // PostCreate 호출
}

// 3. PostOperation 콜백
FLT_POSTOP_CALLBACK_STATUS PostCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    // 파일 열기 성공 후 처리
    // Filter Manager가 IRQL 관리
    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## 16.6 왜 Minifilter를 사용해야 하는가?

### 16.6.1 개발 생산성

```
┌─────────────────────────────────────────────────────────────────────┐
│                    개발 시간 비교 (추정)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  기능                    레거시 필터        Minifilter               │
├─────────────────────────────────────────────────────────────────────┤
│  기본 구조 구현          2-3주              2-3일                    │
│  Create/Read/Write       1-2주              1-2일                    │
│  파일 이름 처리          1주                몇 시간                  │
│  컨텍스트 관리           1주                1일                      │
│  FastIO 처리             1주                불필요                   │
│  볼륨 마운트 처리        3-5일              자동                     │
│  디버깅/안정화           수주               수일                     │
├─────────────────────────────────────────────────────────────────────┤
│  총 예상 시간            2-3개월            2-3주                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.6.2 안정성

```
• Filter Manager가 테스트된 인프라 제공
• 많은 엣지 케이스를 자동 처리
• Microsoft 지원 및 문서화
• 다른 Minifilter와 충돌 최소화
• !fltkd 디버깅 도구 지원
```

### 16.6.3 Microsoft의 권장 사항

> "New file system filter drivers should be developed using the filter manager model instead of the legacy file system filter model."
>
> "새로운 파일 시스템 필터 드라이버는 레거시 모델 대신 Filter Manager 모델(Minifilter)을 사용하여 개발해야 합니다."
>
> — Microsoft Documentation

### 16.6.4 레거시 필터를 사용해야 하는 경우

```
거의 없음!

예외:
• 기존 레거시 필터 유지보수 (신규 개발 아님)
• Windows XP 지원 필요 (현실적으로 불필요)
• Filter Manager 이전에 연결해야 하는 특수 경우 (매우 드묾)
```

---

## 16.7 마이그레이션 고려 사항

기존 레거시 필터를 Minifilter로 마이그레이션하는 경우:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    마이그레이션 체크리스트                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  □ DriverEntry → FltRegisterFilter로 변환                          │
│  □ IoAttachDevice → FltStartFiltering으로 변환                      │
│  □ IRP 핸들러 → Pre/PostOperation 콜백으로 변환                     │
│  □ DeviceExtension → FLT_CONTEXT로 변환                            │
│  □ 파일 이름 구축 → FltGetFileNameInformation 사용                 │
│  □ FastIO 제거 (Filter Manager가 처리)                              │
│  □ 완료 루틴 → PostOperation 콜백으로 변환                          │
│  □ 재진입 처리 → FltIsOperationSynchronous 등 사용                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 16.8 요약: Minifilter 선택의 당위성

```
┌─────────────────────────────────────────────────────────────────────┐
│                        결론                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  레거시 필터:                                                        │
│  ├─ 복잡한 디바이스 스택 관리                                       │
│  ├─ 모든 IRP, FastIO 직접 구현                                      │
│  ├─ 파일 이름 직접 구축                                             │
│  ├─ 연결 순서 비결정적                                              │
│  ├─ 다른 필터와 충돌 위험                                           │
│  └─ 개발/디버깅 매우 어려움                                         │
│                                                                      │
│  Minifilter:                                                         │
│  ├─ Filter Manager가 복잡성 추상화                                  │
│  ├─ 필요한 콜백만 등록                                              │
│  ├─ FltGetFileNameInformation 제공                                  │
│  ├─ 고도(Altitude)로 순서 보장                                      │
│  ├─ 표준화된 인터페이스                                             │
│  └─ Microsoft 지원 및 도구                                          │
│                                                                      │
│  ───────────────────────────────────────────                         │
│  결론: 새로운 드라이버는 반드시 Minifilter로 개발!                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 요약

이 챕터에서 학습한 내용:

1. **레거시 필터 아키텍처**: 디바이스 스택, IRP 처리 방식
2. **레거시 필터의 문제점**: 순서, 복잡성, 호환성
3. **Filter Manager 도입**: 추상화 계층으로 복잡성 해결
4. **Minifilter의 장점**: 간단한 API, 고도 시스템, 자동화된 처리
5. **코드 비교**: 같은 기능을 훨씬 적은 코드로 구현

다음 Part에서는 본격적으로 Minifilter 드라이버를 개발합니다.

---

## 참고 자료

- [File System Filter Drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/file-system-filter-drivers)
- [Filter Manager Concepts](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/filter-manager-concepts)
- [Advantages of the Filter Manager Model](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/advantages-of-the-filter-manager-model)
- [Load Order Groups and Altitudes](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/allocated-altitudes)
