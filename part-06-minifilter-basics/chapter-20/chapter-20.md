# Chapter 20: 컨텍스트 관리

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- 컨텍스트의 개념과 필요성을 이해합니다
- 다양한 컨텍스트 유형을 구분하고 적절히 선택합니다
- 컨텍스트를 등록, 할당, 설정, 해제합니다
- 스트림 컨텍스트로 파일별 정보를 관리합니다
- 컨텍스트 생명주기와 메모리 관리를 올바르게 처리합니다

## 도입: 왜 컨텍스트가 필요한가?

Minifilter 콜백은 **상태를 유지하지 않습니다**. 각 콜백 호출은 독립적이므로, "이 파일이 보호 대상인지", "이전에 분석했는지" 같은 정보를 유지하려면 **컨텍스트**가 필요합니다.

```csharp
// C# 비유
public class FileMonitor
{
    // 상태 없는 콜백
    public void OnFileWrite(FileWriteEventArgs e)
    {
        // 매번 파일이 보호 대상인지 확인해야 함?
        // → 비효율적!
    }
}

// 컨텍스트 사용
public class FileMonitorWithContext
{
    private Dictionary<string, FileContext> _contexts = new();

    public void OnFileOpen(FileOpenEventArgs e)
    {
        var ctx = new FileContext {
            FileName = e.FileName,
            IsProtected = CheckProtection(e.FileName)
        };
        _contexts[e.FileId] = ctx;  // 컨텍스트 저장
    }

    public void OnFileWrite(FileWriteEventArgs e)
    {
        var ctx = _contexts[e.FileId];  // 컨텍스트 조회
        if (ctx.IsProtected) {
            // 보호 로직
        }
    }
}
```

---

## 20.1 컨텍스트 개념

### 20.1.1 컨텍스트란?

컨텍스트는 특정 객체(파일, 볼륨, 인스턴스 등)에 **연결되는 사용자 정의 데이터**입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      컨텍스트 개념도                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐       ┌─────────────────────────────┐         │
│  │   FILE_OBJECT   │ ───── │  Stream Context             │         │
│  │   (test.docx)   │       │  - FileName: "test.docx"    │         │
│  └─────────────────┘       │  - IsProtected: TRUE        │         │
│                            │  - HasSSN: FALSE            │         │
│                            └─────────────────────────────┘         │
│                                                                      │
│  ┌─────────────────┐       ┌─────────────────────────────┐         │
│  │   FLT_INSTANCE  │ ───── │  Instance Context           │         │
│  │   (C: 드라이브) │       │  - VolumeName: "C:"         │         │
│  └─────────────────┘       │  - FileCount: 1234          │         │
│                            └─────────────────────────────┘         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 20.1.2 컨텍스트 유형

```
┌─────────────────────────────────────────────────────────────────────┐
│                      컨텍스트 유형                                   │
├─────────────────────────────────────────────────────────────────────┤
│  유형                    연결 대상              용도                 │
├─────────────────────────────────────────────────────────────────────┤
│  FLT_VOLUME_CONTEXT      볼륨                  볼륨별 설정          │
│  FLT_INSTANCE_CONTEXT    인스턴스              인스턴스별 상태      │
│  FLT_FILE_CONTEXT        파일 (FCB)            파일별 정보 (전역)   │
│  FLT_STREAM_CONTEXT      스트림 (SCB)          스트림별 정보 ★      │
│  FLT_STREAMHANDLE_CONTEXT 스트림 핸들 (CCB)    핸들별 정보          │
│  FLT_TRANSACTION_CONTEXT 트랜잭션              TxF 트랜잭션         │
│  FLT_SECTION_CONTEXT     섹션                  메모리 매핑 섹션     │
└─────────────────────────────────────────────────────────────────────┘

★ Stream Context가 가장 많이 사용됨
   - 파일별 정보 저장에 적합
   - 동일 파일의 여러 핸들이 같은 컨텍스트 공유
```

### 20.1.3 Stream vs StreamHandle 컨텍스트

```
┌─────────────────────────────────────────────────────────────────────┐
│              Stream vs StreamHandle 컨텍스트                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  test.docx 파일                                                      │
│       │                                                              │
│       ├─ Handle 1 (Word 프로세스)                                   │
│       │    └─ StreamHandle Context A                                │
│       │                                                              │
│       ├─ Handle 2 (백업 프로세스)                                   │
│       │    └─ StreamHandle Context B                                │
│       │                                                              │
│       └─ Stream Context (공유)  ← 모든 핸들이 공유                  │
│            - FileName                                                │
│            - IsProtected                                             │
│            - 등등                                                    │
│                                                                      │
│  Stream Context: 파일당 하나 (핸들과 무관)                          │
│  StreamHandle Context: 핸들당 하나                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 20.2 컨텍스트 등록

### 20.2.1 컨텍스트 구조체 정의

```c
// FileMonitor.h

// 풀 태그
#define FM_CONTEXT_TAG 'xtCF'

// 스트림 컨텍스트 구조체
typedef struct _FM_STREAM_CONTEXT {
    // 파일 이름 (캐싱)
    UNICODE_STRING FileName;

    // 보호 여부
    BOOLEAN IsProtected;

    // SSN 포함 여부 (분석 결과)
    BOOLEAN ContainsSSN;

    // 분석 완료 여부
    BOOLEAN IsAnalyzed;

    // 참조 카운트 관리용 락
    ERESOURCE Resource;

    // 생성 시간
    LARGE_INTEGER CreateTime;

} FM_STREAM_CONTEXT, *PFM_STREAM_CONTEXT;

// 인스턴스 컨텍스트 구조체
typedef struct _FM_INSTANCE_CONTEXT {
    // 볼륨 이름
    UNICODE_STRING VolumeName;

    // 볼륨 유형
    DEVICE_TYPE VolumeDeviceType;

    // 파일 시스템 유형
    FLT_FILESYSTEM_TYPE FilesystemType;

    // 통계
    volatile LONG FileCount;

} FM_INSTANCE_CONTEXT, *PFM_INSTANCE_CONTEXT;
```

### 20.2.2 컨텍스트 등록 배열

```c
// Driver.c

// 컨텍스트 정리 콜백 (해제 시 호출)
VOID FLTAPI StreamContextCleanup(
    _In_ PFM_STREAM_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType
);

VOID FLTAPI InstanceContextCleanup(
    _In_ PFM_INSTANCE_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType
);

// 컨텍스트 등록 배열
CONST FLT_CONTEXT_REGISTRATION ContextRegistration[] = {

    // Stream Context 등록
    { FLT_STREAM_CONTEXT,                    // 컨텍스트 유형
      0,                                      // 플래그
      (PFLT_CONTEXT_CLEANUP_CALLBACK)StreamContextCleanup,  // 정리 콜백
      sizeof(FM_STREAM_CONTEXT),              // 크기
      FM_CONTEXT_TAG                          // 풀 태그
    },

    // Instance Context 등록
    { FLT_INSTANCE_CONTEXT,
      0,
      (PFLT_CONTEXT_CLEANUP_CALLBACK)InstanceContextCleanup,
      sizeof(FM_INSTANCE_CONTEXT),
      FM_CONTEXT_TAG
    },

    // 배열 종료 마커
    { FLT_CONTEXT_END }
};

// FLT_REGISTRATION에 등록
CONST FLT_REGISTRATION FilterRegistration = {
    sizeof(FLT_REGISTRATION),
    FLT_REGISTRATION_VERSION,
    0,
    ContextRegistration,    // ← 여기!
    Callbacks,
    FilterUnload,
    InstanceSetup,
    // ...
};
```

---

## 20.3 스트림 컨텍스트 관리

### 20.3.1 스트림 컨텍스트 생성 및 설정

```c
// PostCreate에서 스트림 컨텍스트 설정

FLT_POSTOP_CALLBACK_STATUS PostCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    NTSTATUS status;
    PFM_STREAM_CONTEXT streamCtx = NULL;
    PFM_STREAM_CONTEXT oldStreamCtx = NULL;
    BOOLEAN contextCreated = FALSE;

    UNREFERENCED_PARAMETER(CompletionContext);

    // 드레이닝 또는 실패 시 빠르게 반환
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING) ||
        !NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 디렉터리는 건너뛰기
    BOOLEAN isDirectory = FALSE;
    FltIsDirectory(FltObjects->FileObject, FltObjects->Instance, &isDirectory);
    if (isDirectory) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 1. 기존 컨텍스트 확인
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        &streamCtx
    );

    if (NT_SUCCESS(status)) {
        // 이미 존재함 - 그대로 사용
        FltReleaseContext(streamCtx);
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 2. 새 컨텍스트 할당
    status = FltAllocateContext(
        FltObjects->Filter,
        FLT_STREAM_CONTEXT,
        sizeof(FM_STREAM_CONTEXT),
        NonPagedPool,
        &streamCtx
    );

    if (!NT_SUCCESS(status)) {
        FmDbgPrint("FltAllocateContext failed: 0x%08X", status);
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 3. 컨텍스트 초기화
    RtlZeroMemory(streamCtx, sizeof(FM_STREAM_CONTEXT));
    ExInitializeResourceLite(&streamCtx->Resource);
    KeQuerySystemTime(&streamCtx->CreateTime);

    // 파일 이름 가져와서 저장
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (NT_SUCCESS(status)) {
        FltParseFileNameInformation(nameInfo);

        // 이름 복사
        streamCtx->FileName.Buffer = ExAllocatePoolWithTag(
            NonPagedPool,
            nameInfo->Name.Length,
            FM_CONTEXT_TAG
        );

        if (streamCtx->FileName.Buffer != NULL) {
            RtlCopyMemory(
                streamCtx->FileName.Buffer,
                nameInfo->Name.Buffer,
                nameInfo->Name.Length
            );
            streamCtx->FileName.Length = nameInfo->Name.Length;
            streamCtx->FileName.MaximumLength = nameInfo->Name.Length;
        }

        // 보호 여부 결정
        streamCtx->IsProtected = IsProtectedFile(&nameInfo->Name);

        FltReleaseFileNameInformation(nameInfo);
    }

    // 4. 컨텍스트 설정
    status = FltSetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        FLT_SET_CONTEXT_KEEP_IF_EXISTS,  // 이미 있으면 기존 유지
        streamCtx,
        &oldStreamCtx
    );

    if (!NT_SUCCESS(status)) {
        if (status == STATUS_FLT_CONTEXT_ALREADY_DEFINED) {
            // 다른 스레드가 먼저 설정함 - 정상
            FmDbgPrint("Context already set by another thread");
        } else {
            FmDbgPrint("FltSetStreamContext failed: 0x%08X", status);
        }
    }

    // 5. 정리
    // 설정 성공 여부와 관계없이 참조 해제 필요
    FltReleaseContext(streamCtx);

    if (oldStreamCtx != NULL) {
        FltReleaseContext(oldStreamCtx);
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

### 20.3.2 스트림 컨텍스트 조회

```c
// PreWrite에서 스트림 컨텍스트 조회

FLT_PREOP_CALLBACK_STATUS PreWrite(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    NTSTATUS status;
    PFM_STREAM_CONTEXT streamCtx = NULL;

    UNREFERENCED_PARAMETER(CompletionContext);

    // 커널 모드 요청은 통과
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 스트림 컨텍스트 가져오기
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        &streamCtx
    );

    if (!NT_SUCCESS(status)) {
        // 컨텍스트 없음 - 통과
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 컨텍스트 사용
    if (streamCtx->IsProtected) {
        FmDbgPrint("PreWrite: Protected file write detected: %wZ",
            &streamCtx->FileName);

        // 리소스 락 획득 (읽기)
        ExAcquireResourceSharedLite(&streamCtx->Resource, TRUE);

        // 보호 로직 수행
        // ...

        ExReleaseResourceLite(&streamCtx->Resource);

        // 필요 시 차단
        // Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        // FltReleaseContext(streamCtx);
        // return FLT_PREOP_COMPLETE;
    }

    // 컨텍스트 참조 해제 (중요!)
    FltReleaseContext(streamCtx);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### 20.3.3 스트림 컨텍스트 수정

```c
// 컨텍스트 내용 수정 예시

NTSTATUS UpdateStreamContext(
    PFLT_INSTANCE Instance,
    PFILE_OBJECT FileObject,
    BOOLEAN ContainsSSN
)
{
    NTSTATUS status;
    PFM_STREAM_CONTEXT streamCtx = NULL;

    status = FltGetStreamContext(Instance, FileObject, &streamCtx);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 배타적 락 획득 (쓰기)
    ExAcquireResourceExclusiveLite(&streamCtx->Resource, TRUE);

    // 값 수정
    streamCtx->ContainsSSN = ContainsSSN;
    streamCtx->IsAnalyzed = TRUE;

    ExReleaseResourceLite(&streamCtx->Resource);

    FltReleaseContext(streamCtx);

    return STATUS_SUCCESS;
}
```

### 20.3.4 스트림 컨텍스트 정리 콜백

```c
// 컨텍스트가 해제될 때 호출됨

VOID FLTAPI StreamContextCleanup(
    _In_ PFM_STREAM_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType
)
{
    UNREFERENCED_PARAMETER(ContextType);

    FmDbgPrint("StreamContextCleanup: %wZ", &Context->FileName);

    // 파일 이름 버퍼 해제
    if (Context->FileName.Buffer != NULL) {
        ExFreePoolWithTag(Context->FileName.Buffer, FM_CONTEXT_TAG);
        Context->FileName.Buffer = NULL;
    }

    // 리소스 삭제
    ExDeleteResourceLite(&Context->Resource);
}
```

---

## 20.4 인스턴스 컨텍스트 관리

### 20.4.1 인스턴스 컨텍스트 설정

```c
// InstanceSetup에서 인스턴스 컨텍스트 설정

NTSTATUS FLTAPI InstanceSetup(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_SETUP_FLAGS Flags,
    DEVICE_TYPE VolumeDeviceType,
    FLT_FILESYSTEM_TYPE VolumeFilesystemType
)
{
    NTSTATUS status;
    PFM_INSTANCE_CONTEXT instanceCtx = NULL;

    UNREFERENCED_PARAMETER(Flags);

    FmDbgPrint("InstanceSetup: DeviceType=%d, FSType=%d",
        VolumeDeviceType, VolumeFilesystemType);

    // 특정 볼륨 제외
    if (VolumeFilesystemType == FLT_FSTYPE_RAW ||
        VolumeDeviceType == FILE_DEVICE_CD_ROM_FILE_SYSTEM) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    // 1. 인스턴스 컨텍스트 할당
    status = FltAllocateContext(
        FltObjects->Filter,
        FLT_INSTANCE_CONTEXT,
        sizeof(FM_INSTANCE_CONTEXT),
        NonPagedPool,
        &instanceCtx
    );

    if (!NT_SUCCESS(status)) {
        FmDbgPrint("FltAllocateContext failed: 0x%08X", status);
        return STATUS_SUCCESS;  // 컨텍스트 없이도 연결은 허용
    }

    // 2. 초기화
    RtlZeroMemory(instanceCtx, sizeof(FM_INSTANCE_CONTEXT));
    instanceCtx->VolumeDeviceType = VolumeDeviceType;
    instanceCtx->FilesystemType = VolumeFilesystemType;

    // 볼륨 이름 가져오기
    UCHAR volumeNameBuffer[256];
    ULONG volumeNameLength = 0;

    status = FltGetVolumeName(
        FltObjects->Volume,
        NULL,
        &volumeNameLength
    );

    if (status == STATUS_BUFFER_TOO_SMALL && volumeNameLength <= 256) {
        UNICODE_STRING volumeName;
        volumeName.Buffer = (PWCH)volumeNameBuffer;
        volumeName.MaximumLength = sizeof(volumeNameBuffer);
        volumeName.Length = 0;

        status = FltGetVolumeName(FltObjects->Volume, &volumeName, NULL);

        if (NT_SUCCESS(status)) {
            // 이름 복사
            instanceCtx->VolumeName.Buffer = ExAllocatePoolWithTag(
                NonPagedPool,
                volumeName.Length,
                FM_CONTEXT_TAG
            );

            if (instanceCtx->VolumeName.Buffer != NULL) {
                RtlCopyMemory(
                    instanceCtx->VolumeName.Buffer,
                    volumeName.Buffer,
                    volumeName.Length
                );
                instanceCtx->VolumeName.Length = volumeName.Length;
                instanceCtx->VolumeName.MaximumLength = volumeName.Length;

                FmDbgPrint("InstanceSetup: Volume = %wZ",
                    &instanceCtx->VolumeName);
            }
        }
    }

    // 3. 컨텍스트 설정
    status = FltSetInstanceContext(
        FltObjects->Instance,
        FLT_SET_CONTEXT_KEEP_IF_EXISTS,
        instanceCtx,
        NULL
    );

    // 4. 참조 해제
    FltReleaseContext(instanceCtx);

    return STATUS_SUCCESS;
}
```

### 20.4.2 인스턴스 컨텍스트 조회

```c
// 콜백에서 인스턴스 컨텍스트 사용

FLT_PREOP_CALLBACK_STATUS PreCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    NTSTATUS status;
    PFM_INSTANCE_CONTEXT instanceCtx = NULL;

    UNREFERENCED_PARAMETER(CompletionContext);

    // 인스턴스 컨텍스트 가져오기
    status = FltGetInstanceContext(
        FltObjects->Instance,
        &instanceCtx
    );

    if (NT_SUCCESS(status)) {
        FmDbgPrint("PreCreate on volume: %wZ", &instanceCtx->VolumeName);

        // 파일 카운트 증가 (원자적)
        InterlockedIncrement(&instanceCtx->FileCount);

        FltReleaseContext(instanceCtx);
    }

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

### 20.4.3 인스턴스 컨텍스트 정리

```c
VOID FLTAPI InstanceContextCleanup(
    _In_ PFM_INSTANCE_CONTEXT Context,
    _In_ FLT_CONTEXT_TYPE ContextType
)
{
    UNREFERENCED_PARAMETER(ContextType);

    FmDbgPrint("InstanceContextCleanup: %wZ, FileCount=%ld",
        &Context->VolumeName, Context->FileCount);

    // 볼륨 이름 버퍼 해제
    if (Context->VolumeName.Buffer != NULL) {
        ExFreePoolWithTag(Context->VolumeName.Buffer, FM_CONTEXT_TAG);
        Context->VolumeName.Buffer = NULL;
    }
}

// InstanceTeardownComplete에서 정리 확인
VOID FLTAPI InstanceTeardownComplete(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_TEARDOWN_FLAGS Flags
)
{
    NTSTATUS status;
    PFM_INSTANCE_CONTEXT instanceCtx = NULL;

    UNREFERENCED_PARAMETER(Flags);

    status = FltGetInstanceContext(FltObjects->Instance, &instanceCtx);

    if (NT_SUCCESS(status)) {
        FmDbgPrint("InstanceTeardownComplete: %wZ",
            &instanceCtx->VolumeName);
        FltReleaseContext(instanceCtx);
    }

    // 컨텍스트 삭제 (정리 콜백 호출됨)
    FltDeleteInstanceContext(FltObjects->Instance, NULL);
}
```

---

## 20.5 컨텍스트 참조 카운트 관리

### 20.5.1 참조 카운트 규칙

```
┌─────────────────────────────────────────────────────────────────────┐
│                    참조 카운트 규칙                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  참조 증가 (Reference):                                             │
│  ├─ FltAllocateContext          → 참조 카운트 1로 시작              │
│  ├─ FltGetXxxContext            → 참조 카운트 +1                    │
│  └─ FltReferenceContext         → 참조 카운트 +1                    │
│                                                                      │
│  참조 감소 (Release):                                               │
│  └─ FltReleaseContext           → 참조 카운트 -1                    │
│                                                                      │
│  규칙:                                                               │
│  • 모든 Get/Allocate 후에는 반드시 Release 필요                     │
│  • 참조 카운트가 0이 되면 Cleanup 콜백 호출 후 메모리 해제          │
│  • FltSetXxxContext는 참조를 "넘김" (내부에서 참조 유지)            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 20.5.2 올바른 참조 관리 패턴

```c
// ✅ 올바른 패턴

// 패턴 1: Get 후 Release
status = FltGetStreamContext(Instance, FileObject, &ctx);
if (NT_SUCCESS(status)) {
    // 사용
    FltReleaseContext(ctx);  // 항상!
}

// 패턴 2: Allocate → Set → Release
status = FltAllocateContext(..., &ctx);
if (NT_SUCCESS(status)) {
    // 초기화
    status = FltSetStreamContext(..., ctx, NULL);
    FltReleaseContext(ctx);  // Set 성공 여부와 관계없이!
}

// 패턴 3: Set with old context
PFM_STREAM_CONTEXT oldCtx = NULL;
status = FltSetStreamContext(..., newCtx, &oldCtx);
FltReleaseContext(newCtx);
if (oldCtx != NULL) {
    FltReleaseContext(oldCtx);  // 이전 컨텍스트도 해제!
}

// ❌ 잘못된 패턴

// 패턴 1: Release 누락
status = FltGetStreamContext(..., &ctx);
if (NT_SUCCESS(status)) {
    if (condition) {
        return;  // ctx 누수!
    }
    FltReleaseContext(ctx);
}

// 패턴 2: 이중 Release
FltReleaseContext(ctx);
// ...
FltReleaseContext(ctx);  // 해제된 메모리 접근!
```

### 20.5.3 OrCreate 패턴

```c
// 있으면 가져오고, 없으면 생성하는 패턴

NTSTATUS GetOrCreateStreamContext(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PFM_STREAM_CONTEXT *StreamContext,
    PBOOLEAN ContextCreated
)
{
    NTSTATUS status;
    PFM_STREAM_CONTEXT streamCtx = NULL;
    PFM_STREAM_CONTEXT oldStreamCtx = NULL;

    *StreamContext = NULL;
    *ContextCreated = FALSE;

    // 1. 기존 컨텍스트 확인
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        &streamCtx
    );

    if (NT_SUCCESS(status)) {
        *StreamContext = streamCtx;
        return STATUS_SUCCESS;
    }

    if (status != STATUS_NOT_FOUND) {
        return status;
    }

    // 2. 새 컨텍스트 생성
    status = FltAllocateContext(
        FltObjects->Filter,
        FLT_STREAM_CONTEXT,
        sizeof(FM_STREAM_CONTEXT),
        NonPagedPool,
        &streamCtx
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 3. 초기화
    RtlZeroMemory(streamCtx, sizeof(FM_STREAM_CONTEXT));
    ExInitializeResourceLite(&streamCtx->Resource);

    // 4. 설정 시도
    status = FltSetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        FLT_SET_CONTEXT_KEEP_IF_EXISTS,
        streamCtx,
        &oldStreamCtx
    );

    if (!NT_SUCCESS(status)) {
        // 설정 실패 - 다른 스레드가 먼저 설정했을 수 있음
        FltReleaseContext(streamCtx);

        if (status == STATUS_FLT_CONTEXT_ALREADY_DEFINED) {
            // 다른 스레드가 설정한 것 사용
            *StreamContext = oldStreamCtx;
            return STATUS_SUCCESS;
        }

        return status;
    }

    // 5. 성공
    *StreamContext = streamCtx;
    *ContextCreated = TRUE;

    if (oldStreamCtx != NULL) {
        FltReleaseContext(oldStreamCtx);
    }

    return STATUS_SUCCESS;
}

// 사용 예
FLT_POSTOP_CALLBACK_STATUS PostCreate(...)
{
    PFM_STREAM_CONTEXT streamCtx = NULL;
    BOOLEAN created = FALSE;

    status = GetOrCreateStreamContext(Data, FltObjects, &streamCtx, &created);

    if (NT_SUCCESS(status)) {
        if (created) {
            // 새로 생성됨 - 추가 초기화
            InitializeStreamContext(Data, streamCtx);
        }

        // 사용
        // ...

        FltReleaseContext(streamCtx);  // 항상 해제!
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## 20.6 컨텍스트 동기화

### 20.6.1 ERESOURCE 사용

```c
// 컨텍스트 데이터 보호를 위한 ERESOURCE

typedef struct _FM_STREAM_CONTEXT {
    ERESOURCE Resource;     // 동기화용 리소스
    UNICODE_STRING FileName;
    BOOLEAN IsProtected;
    LONG AccessCount;       // 접근 횟수 통계
} FM_STREAM_CONTEXT;

// 읽기 접근 (공유 락)
VOID ReadContextData(PFM_STREAM_CONTEXT Context)
{
    ExAcquireResourceSharedLite(&Context->Resource, TRUE);

    // 데이터 읽기
    BOOLEAN isProtected = Context->IsProtected;

    ExReleaseResourceLite(&Context->Resource);
}

// 쓰기 접근 (배타적 락)
VOID WriteContextData(PFM_STREAM_CONTEXT Context, BOOLEAN NewValue)
{
    ExAcquireResourceExclusiveLite(&Context->Resource, TRUE);

    // 데이터 수정
    Context->IsProtected = NewValue;

    ExReleaseResourceLite(&Context->Resource);
}

// 초기화 및 정리
VOID InitContext(PFM_STREAM_CONTEXT Context)
{
    ExInitializeResourceLite(&Context->Resource);
}

VOID CleanupContext(PFM_STREAM_CONTEXT Context)
{
    ExDeleteResourceLite(&Context->Resource);
}
```

### 20.6.2 원자적 연산

```c
// 간단한 카운터는 InterlockedXxx 사용

typedef struct _FM_STREAM_CONTEXT {
    volatile LONG ReadCount;
    volatile LONG WriteCount;
} FM_STREAM_CONTEXT;

// 카운터 증가
VOID IncrementReadCount(PFM_STREAM_CONTEXT Context)
{
    InterlockedIncrement(&Context->ReadCount);
}

// 카운터 감소
VOID DecrementReadCount(PFM_STREAM_CONTEXT Context)
{
    InterlockedDecrement(&Context->ReadCount);
}

// 값 비교 및 교환
VOID SetIfNotSet(PFM_STREAM_CONTEXT Context)
{
    // IsAnalyzed가 FALSE(0)일 때만 TRUE(1)로 설정
    InterlockedCompareExchange(&Context->IsAnalyzed, TRUE, FALSE);
}
```

---

## 20.7 컨텍스트 디버깅

### 20.7.1 WinDbg에서 컨텍스트 확인

```
// 스트림 컨텍스트 찾기
0: kd> !fltkd.instance ffffb80123530000
   ...
   Context : ffffb80123540000

// 컨텍스트 내용 덤프
0: kd> dt FileMonitor!_FM_STREAM_CONTEXT ffffb80123540000
   +0x000 FileName         : _UNICODE_STRING "test.docx"
   +0x010 IsProtected      : 0x1 ''
   +0x011 ContainsSSN      : 0x0 ''
   +0x012 IsAnalyzed       : 0x1 ''
   +0x018 Resource         : _ERESOURCE
   +0x080 CreateTime       : _LARGE_INTEGER 0x01d9abcd`12345678

// 파일 이름 출력
0: kd> du poi(ffffb80123540000+0x8)
ffffb80123541000  "test.docx"
```

### 20.7.2 컨텍스트 누수 진단

```
// Driver Verifier로 풀 태그 추적

// 명령 프롬프트 (관리자)
verifier /standard /driver FileMonitor.sys

// 재부팅 후 풀 태그 확인
0: kd> !poolused 2

// 'FMCx' 태그 찾기
Tag    Allocs   Frees   Diff   Bytes   ...
FMCx   100      50      50     2400    FileMonitor!FM_CONTEXT_TAG

// 50개 누수 - 문제!

// 특정 태그 할당 찾기
0: kd> !poolfind FMCx 2
```

---

## 20.8 완전한 컨텍스트 관리 예제

```c
// 완전한 스트림 컨텍스트 관리 모듈

// ContextManager.c

#include "FileMonitor.h"

// 스트림 컨텍스트 생성 또는 가져오기
NTSTATUS FmGetOrCreateStreamContext(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Outptr_ PFM_STREAM_CONTEXT *StreamContext
)
{
    NTSTATUS status;
    PFM_STREAM_CONTEXT ctx = NULL;
    PFM_STREAM_CONTEXT oldCtx = NULL;
    BOOLEAN needsInit = FALSE;

    // 기존 컨텍스트 확인
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        &ctx
    );

    if (NT_SUCCESS(status)) {
        *StreamContext = ctx;
        return STATUS_SUCCESS;
    }

    if (status != STATUS_NOT_FOUND) {
        return status;
    }

    // 새 컨텍스트 할당
    status = FltAllocateContext(
        FltObjects->Filter,
        FLT_STREAM_CONTEXT,
        sizeof(FM_STREAM_CONTEXT),
        NonPagedPool,
        &ctx
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 초기화
    RtlZeroMemory(ctx, sizeof(FM_STREAM_CONTEXT));
    ExInitializeResourceLite(&ctx->Resource);
    KeQuerySystemTime(&ctx->CreateTime);

    // 설정
    status = FltSetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        FLT_SET_CONTEXT_KEEP_IF_EXISTS,
        ctx,
        &oldCtx
    );

    if (status == STATUS_FLT_CONTEXT_ALREADY_DEFINED) {
        // 다른 스레드가 설정함 - 그것 사용
        FltReleaseContext(ctx);
        *StreamContext = oldCtx;
        return STATUS_SUCCESS;
    }

    if (!NT_SUCCESS(status)) {
        FltReleaseContext(ctx);
        return status;
    }

    // 파일 이름 초기화
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (NT_SUCCESS(status)) {
        FltParseFileNameInformation(nameInfo);

        ExAcquireResourceExclusiveLite(&ctx->Resource, TRUE);

        ctx->FileName.Buffer = ExAllocatePoolWithTag(
            NonPagedPool,
            nameInfo->Name.Length,
            FM_CONTEXT_TAG
        );

        if (ctx->FileName.Buffer != NULL) {
            RtlCopyMemory(
                ctx->FileName.Buffer,
                nameInfo->Name.Buffer,
                nameInfo->Name.Length
            );
            ctx->FileName.Length = nameInfo->Name.Length;
            ctx->FileName.MaximumLength = nameInfo->Name.Length;

            ctx->IsProtected = IsProtectedExtension(&nameInfo->Extension);
        }

        ExReleaseResourceLite(&ctx->Resource);

        FltReleaseFileNameInformation(nameInfo);
    }

    if (oldCtx != NULL) {
        FltReleaseContext(oldCtx);
    }

    *StreamContext = ctx;
    return STATUS_SUCCESS;
}

// 보호된 확장자 확인
BOOLEAN IsProtectedExtension(PCUNICODE_STRING Extension)
{
    static const WCHAR* ProtectedExtensions[] = {
        L"docx", L"doc", L"xlsx", L"xls",
        L"pptx", L"ppt", L"pdf", NULL
    };

    if (Extension == NULL || Extension->Length == 0) {
        return FALSE;
    }

    for (int i = 0; ProtectedExtensions[i] != NULL; i++) {
        UNICODE_STRING ext;
        RtlInitUnicodeString(&ext, ProtectedExtensions[i]);

        if (RtlCompareUnicodeString(Extension, &ext, TRUE) == 0) {
            return TRUE;
        }
    }

    return FALSE;
}
```

---

## 요약

이 챕터에서 학습한 내용:

1. **컨텍스트 개념**: 객체에 연결되는 사용자 정의 데이터
2. **컨텍스트 유형**: Stream, Instance, Volume, StreamHandle 등
3. **등록**: FLT_CONTEXT_REGISTRATION 배열
4. **할당/설정**: FltAllocateContext, FltSetXxxContext
5. **조회/해제**: FltGetXxxContext, FltReleaseContext
6. **참조 관리**: 모든 Get/Allocate 후 Release 필수
7. **동기화**: ERESOURCE, InterlockedXxx
8. **정리**: Cleanup 콜백에서 리소스 해제

Part 6 Minifilter 기초를 완료했습니다. 다음 Part에서는 파일 콘텐츠 분석을 학습합니다.

---

## 참고 자료

- [Managing Contexts](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/managing-contexts)
- [FltAllocateContext](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltallocatecontext)
- [FltSetStreamContext](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltsetstreamcontext)
- [Context Types](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/flt-context-registration)
