# Chapter 17: Filter Manager와 Minifilter 아키텍처

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- Filter Manager의 역할과 동작 원리를 이해합니다
- Minifilter의 구조와 생명주기를 파악합니다
- 고도(Altitude)와 필터 순서를 이해합니다
- Pre/Post Operation 콜백 모델을 이해합니다
- FLT_REGISTRATION 구조체를 구성합니다

## 도입: Minifilter 아키텍처의 핵심

이전 챕터에서 레거시 필터의 복잡성을 살펴봤습니다. 이제 Minifilter가 어떻게 그 복잡성을 **추상화**하는지 상세히 알아봅니다. Filter Manager는 Minifilter와 파일 시스템 사이에서 **프록시** 역할을 합니다.

```csharp
// C# 비유: Filter Manager는 Dependency Injection Container와 유사

public interface IFileOperationFilter
{
    FilterResult PreOperation(FileOperation op);
    void PostOperation(FileOperation op, OperationResult result);
}

// Filter Manager가 모든 IFileOperationFilter 구현체를 관리
// 순서대로 호출, 예외 처리, 리소스 관리 담당
```

---

## 17.1 Filter Manager 아키텍처

### 17.1.1 Filter Manager의 역할

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Filter Manager (fltmgr.sys)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  역할:                                                               │
│  ├─ 레거시 필터로서 파일 시스템에 연결                              │
│  ├─ Minifilter 등록/해제 관리                                       │
│  ├─ I/O 요청을 Minifilter 콜백으로 라우팅                           │
│  ├─ 고도(Altitude) 기반 호출 순서 보장                              │
│  ├─ 컨텍스트 관리 인프라 제공                                       │
│  ├─ 파일 이름 캐싱 및 해석                                          │
│  ├─ FastIO → IRP 변환 자동 처리                                     │
│  └─ 커널-사용자 모드 통신 지원                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 17.1.2 계층 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                        계층 구조                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                  사용자 모드 애플리케이션                  │       │
│  │                    (notepad.exe 등)                       │       │
│  └──────────────────────────────────────────────────────────┘       │
│                              │                                       │
│                              ▼ NtCreateFile                          │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                      I/O Manager                          │       │
│  └──────────────────────────────────────────────────────────┘       │
│                              │                                       │
│                              ▼ IRP                                   │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │              Filter Manager (fltmgr.sys)                  │       │
│  │  ┌─────────────────────────────────────────────────────┐ │       │
│  │  │            Minifilter Frame                         │ │       │
│  │  │  ┌───────────────────────────────────────────────┐  │ │       │
│  │  │  │ Minifilter A (320000) - 안티바이러스         │  │ │       │
│  │  │  ├───────────────────────────────────────────────┤  │ │       │
│  │  │  │ Minifilter B (125000) - DRM 필터             │  │ │       │
│  │  │  ├───────────────────────────────────────────────┤  │ │       │
│  │  │  │ Minifilter C (45000)  - 파일 정보            │  │ │       │
│  │  │  └───────────────────────────────────────────────┘  │ │       │
│  │  └─────────────────────────────────────────────────────┘ │       │
│  └──────────────────────────────────────────────────────────┘       │
│                              │                                       │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │               파일 시스템 (NTFS.sys 등)                  │       │
│  └──────────────────────────────────────────────────────────┘       │
│                              │                                       │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                    스토리지 스택                          │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 17.1.3 Frame 개념

```c
// Frame: Minifilter들이 속한 논리적 그룹
// 기본적으로 Frame 0 하나만 사용

// Frame 구조
typedef struct _FLTP_FRAME {
    FLT_OBJECT_TYPE Type;
    LIST_ENTRY Links;              // 프레임 리스트
    ULONG FrameID;                 // 프레임 ID (보통 0)
    UNICODE_STRING AltitudeIntervalLow;   // 최소 고도
    UNICODE_STRING AltitudeIntervalHigh;  // 최대 고도
    FLT_RESOURCE_LIST_HEAD RegisteredFilters;  // 필터 목록
    FLT_RESOURCE_LIST_HEAD AttachedVolumes;    // 연결된 볼륨
    // ...
} FLTP_FRAME;

// 거의 모든 Minifilter는 Frame 0에 존재
// 특수한 경우(예: 가상화)에만 별도 프레임 사용
```

---

## 17.2 고도(Altitude) 시스템

### 17.2.1 고도의 역할

고도는 Minifilter의 **호출 순서**를 결정하는 숫자입니다. 높은 고도가 먼저 호출됩니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     고도 기반 호출 순서                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  I/O 요청 (CreateFile)                                              │
│         │                                                           │
│         ▼ PreOperation (높은 고도 → 낮은 고도)                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Minifilter A (320000) PreCreate    → 통과                   │   │
│  │         │                                                    │   │
│  │         ▼                                                    │   │
│  │  Minifilter B (125000) PreCreate    → 통과                   │   │
│  │         │                                                    │   │
│  │         ▼                                                    │   │
│  │  Minifilter C (45000) PreCreate     → 통과                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         ▼ 파일 시스템 (실제 작업 수행)                              │
│         │                                                           │
│         ▼ PostOperation (낮은 고도 → 높은 고도)                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Minifilter C (45000) PostCreate    ← 처리                   │   │
│  │         │                                                    │   │
│  │         ▼                                                    │   │
│  │  Minifilter B (125000) PostCreate   ← 처리                   │   │
│  │         │                                                    │   │
│  │         ▼                                                    │   │
│  │  Minifilter A (320000) PostCreate   ← 처리                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         ▼ 애플리케이션에 결과 반환                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 17.2.2 고도 범위 상세

```
┌─────────────────────────────────────────────────────────────────────┐
│                       고도 범위 표                                   │
├─────────────────────────────────────────────────────────────────────┤
│  고도 범위            그룹 이름              용도                    │
├─────────────────────────────────────────────────────────────────────┤
│  420000 - 429999      Filter                일반 필터               │
│  400000 - 409999      Top                   가장 먼저 처리          │
│  360000 - 389999      Activity Monitor      활동 모니터링           │
│  320000 - 329998      Anti-Virus            안티바이러스            │
│  300000 - 309998      Replication           복제                    │
│  280000 - 289998      Continuous Backup     지속적 백업             │
│  260000 - 269998      Content Screener      콘텐츠 검사             │
│  240000 - 249999      Quota Management      쿼터 관리               │
│  220000 - 229999      Block Cloning         블록 복제               │
│  200000 - 219999      Cluster FS Filter     클러스터 FS             │
│  180000 - 189999      System Recovery       시스템 복구             │
│  170000 - 174999      Cluster FS Infra      클러스터 인프라         │
│  140000 - 149999      Encryption            암호화                  │
│  130000 - 139999      Compression           압축                    │
│  120000 - 129999      Virtualization        가상화                  │
│  100000 - 109999      Physical Quota        물리적 쿼터             │
│  80000 - 89999        HSM                   계층 스토리지           │
│  60000 - 69999        Copy Protection       복사 보호               │
│  40000 - 49999        Bottom                가장 나중에 처리        │
└─────────────────────────────────────────────────────────────────────┘
```

### 17.2.3 고도 선택 가이드

```c
// DRM/콘텐츠 보호 필터의 권장 고도

// 옵션 1: Content Screener (260000 - 269998)
// - 콘텐츠 분석, DLP, 패턴 감지
// - 암호화 전에 평문 분석 가능

// 옵션 2: Copy Protection (60000 - 69999)
// - 복사 방지 기능
// - 다른 필터들 처리 후 최종 결정

// 우리 DRM 필터 예시: 265000 (Content Screener 범위)
#define MY_FILTER_ALTITUDE L"265000"

// 테스트 시에는 임의 값 사용 가능
// 제품 배포 시 Microsoft에 공식 등록 필요:
// https://docs.microsoft.com/windows-hardware/drivers/ifs/minifilter-altitude-request
```

---

## 17.3 Minifilter 핵심 구조체

### 17.3.1 FLT_REGISTRATION

모든 Minifilter는 `FLT_REGISTRATION` 구조체로 자신을 정의합니다.

```c
// FLT_REGISTRATION 구조체 정의
typedef struct _FLT_REGISTRATION {
    USHORT Size;                          // 구조체 크기
    USHORT Version;                       // 버전
    FLT_REGISTRATION_FLAGS Flags;         // 플래그

    const FLT_CONTEXT_REGISTRATION *ContextRegistration;  // 컨텍스트 등록
    const FLT_OPERATION_REGISTRATION *OperationRegistration;  // 콜백 등록

    PFLT_FILTER_UNLOAD_CALLBACK FilterUnloadCallback;      // 언로드 콜백
    PFLT_INSTANCE_SETUP_CALLBACK InstanceSetupCallback;    // 인스턴스 설정
    PFLT_INSTANCE_QUERY_TEARDOWN_CALLBACK InstanceQueryTeardownCallback;
    PFLT_INSTANCE_TEARDOWN_CALLBACK InstanceTeardownStartCallback;
    PFLT_INSTANCE_TEARDOWN_CALLBACK InstanceTeardownCompleteCallback;

    PFLT_GENERATE_FILE_NAME GenerateFileNameCallback;      // 파일 이름 생성
    PFLT_NORMALIZE_NAME_COMPONENT NormalizeNameComponentCallback;
    PFLT_NORMALIZE_CONTEXT_CLEANUP NormalizeContextCleanupCallback;

    PFLT_TRANSACTION_NOTIFICATION_CALLBACK TransactionNotificationCallback;
    PFLT_NORMALIZE_NAME_COMPONENT_EX NormalizeNameComponentExCallback;

    PFLT_SECTION_CONFLICT_NOTIFICATION_CALLBACK SectionNotificationCallback;
} FLT_REGISTRATION, *PFLT_REGISTRATION;
```

### 17.3.2 FLT_REGISTRATION 구성 예시

```c
// 기본적인 FLT_REGISTRATION 구성

// 1. 작업 콜백 배열
const FLT_OPERATION_REGISTRATION OperationCallbacks[] = {
    { IRP_MJ_CREATE,
      0,
      PreCreate,    // Pre-operation 콜백
      PostCreate    // Post-operation 콜백
    },

    { IRP_MJ_WRITE,
      0,
      PreWrite,
      NULL          // Post 콜백 없음
    },

    { IRP_MJ_SET_INFORMATION,
      0,
      PreSetInformation,
      NULL
    },

    { IRP_MJ_CLEANUP,
      0,
      PreCleanup,
      NULL
    },

    { IRP_MJ_OPERATION_END }  // 배열 종료 마커
};

// 2. 컨텍스트 등록 배열
const FLT_CONTEXT_REGISTRATION ContextRegistration[] = {
    { FLT_STREAM_CONTEXT,
      0,
      CleanupStreamContext,
      sizeof(MY_STREAM_CONTEXT),
      MY_CONTEXT_TAG
    },

    { FLT_CONTEXT_END }  // 배열 종료 마커
};

// 3. FLT_REGISTRATION 구조체
const FLT_REGISTRATION FilterRegistration = {
    sizeof(FLT_REGISTRATION),           // Size
    FLT_REGISTRATION_VERSION,           // Version
    0,                                  // Flags

    ContextRegistration,                // ContextRegistration
    OperationCallbacks,                 // OperationRegistration

    FilterUnloadCallback,               // FilterUnloadCallback
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
```

### 17.3.3 FLT_OPERATION_REGISTRATION

```c
// 각 I/O 작업에 대한 콜백 등록

typedef struct _FLT_OPERATION_REGISTRATION {
    UCHAR MajorFunction;                // IRP_MJ_XXX
    FLT_OPERATION_REGISTRATION_FLAGS Flags;  // 플래그
    PFLT_PRE_OPERATION_CALLBACK PreOperation;   // Pre 콜백
    PFLT_POST_OPERATION_CALLBACK PostOperation; // Post 콜백
    PVOID Reserved1;                    // 예약됨
} FLT_OPERATION_REGISTRATION, *PFLT_OPERATION_REGISTRATION;

// 플래그 옵션
#define FLTFL_OPERATION_REGISTRATION_SKIP_CACHED_IO     0x00000001
#define FLTFL_OPERATION_REGISTRATION_SKIP_PAGING_IO     0x00000002
#define FLTFL_OPERATION_REGISTRATION_SKIP_NON_DASD_IO   0x00000004

// 예: 페이징 I/O 건너뛰기
{ IRP_MJ_READ,
  FLTFL_OPERATION_REGISTRATION_SKIP_PAGING_IO,
  PreRead,
  PostRead
}
```

---

## 17.4 Pre/Post Operation 콜백 모델

### 17.4.1 Pre-Operation 콜백

```c
// Pre-Operation 콜백: I/O 작업 전에 호출

FLT_PREOP_CALLBACK_STATUS FLTAPI
PreOperationCallback(
    _Inout_ PFLT_CALLBACK_DATA Data,           // I/O 데이터
    _In_ PCFLT_RELATED_OBJECTS FltObjects,     // 관련 객체들
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext  // Post로 전달할 컨텍스트
);

// 반환 값
typedef enum _FLT_PREOP_CALLBACK_STATUS {
    FLT_PREOP_SUCCESS_WITH_CALLBACK,    // 계속 진행, Post 콜백 호출
    FLT_PREOP_SUCCESS_NO_CALLBACK,      // 계속 진행, Post 콜백 안 함
    FLT_PREOP_PENDING,                  // 작업을 Pending 처리
    FLT_PREOP_DISALLOW_FASTIO,          // FastIO 거부 (IRP로 재시도)
    FLT_PREOP_COMPLETE,                 // 작업 완료 (하위로 안 감)
    FLT_PREOP_SYNCHRONIZE               // 동기화된 Post 콜백 필요
} FLT_PREOP_CALLBACK_STATUS;
```

```c
// Pre-Operation 예시: 파일 생성 모니터링

FLT_PREOP_CALLBACK_STATUS
PreCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(CompletionContext);
    UNREFERENCED_PARAMETER(FltObjects);

    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    NTSTATUS status;

    // 커널 모드 요청은 무시
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 가져오기
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (NT_SUCCESS(status)) {
        FltParseFileNameInformation(nameInfo);

        DbgPrint("PreCreate: %wZ\n", &nameInfo->Name);

        // 특정 파일 차단
        if (wcsstr(nameInfo->Name.Buffer, L".confidential")) {
            Data->IoStatus.Status = STATUS_ACCESS_DENIED;
            Data->IoStatus.Information = 0;
            FltReleaseFileNameInformation(nameInfo);
            return FLT_PREOP_COMPLETE;  // 하위로 전달하지 않음
        }

        FltReleaseFileNameInformation(nameInfo);
    }

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;  // PostCreate 호출
}
```

### 17.4.2 Post-Operation 콜백

```c
// Post-Operation 콜백: I/O 작업 후에 호출

FLT_POSTOP_CALLBACK_STATUS FLTAPI
PostOperationCallback(
    _Inout_ PFLT_CALLBACK_DATA Data,           // I/O 데이터
    _In_ PCFLT_RELATED_OBJECTS FltObjects,     // 관련 객체들
    _In_opt_ PVOID CompletionContext,          // Pre에서 전달한 컨텍스트
    _In_ FLT_POST_OPERATION_FLAGS Flags        // 플래그
);

// 반환 값
typedef enum _FLT_POSTOP_CALLBACK_STATUS {
    FLT_POSTOP_FINISHED_PROCESSING,     // 처리 완료
    FLT_POSTOP_MORE_PROCESSING_REQUIRED // 추가 처리 필요 (DPC에서)
} FLT_POSTOP_CALLBACK_STATUS;
```

```c
// Post-Operation 예시: 파일 생성 결과 확인

FLT_POSTOP_CALLBACK_STATUS
PostCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(CompletionContext);
    UNREFERENCED_PARAMETER(FltObjects);

    // 드레이닝 중이면 빨리 반환
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 성공적으로 열린 경우만 처리
    if (NT_SUCCESS(Data->IoStatus.Status)) {
        // FILE_OBJECT에 컨텍스트 연결 등 작업 가능
        DbgPrint("PostCreate: File opened successfully\n");
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

### 17.4.3 콜백 흐름 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│                    콜백 흐름 상세                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CreateFile("C:\test.txt")                                          │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ PreCreate (Minifilter A, 고도 320000)                       │   │
│  │   → FLT_PREOP_SUCCESS_WITH_CALLBACK 반환                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ PreCreate (Minifilter B, 고도 125000)                       │   │
│  │   → FLT_PREOP_SUCCESS_NO_CALLBACK 반환 (Post 안 함)         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 파일 시스템 (NTFS)                                          │   │
│  │   → 실제 파일 열기 수행                                     │   │
│  │   → STATUS_SUCCESS 반환                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ PostCreate (Minifilter A, 고도 320000)                      │   │
│  │   → FLT_POSTOP_FINISHED_PROCESSING 반환                     │   │
│  │   (Minifilter B는 Post 등록 안 함 → 건너뜀)                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                           │
│         ▼                                                           │
│  애플리케이션에 핸들 반환                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 17.5 FLT_CALLBACK_DATA 상세

### 17.5.1 구조체 정의

```c
// FLT_CALLBACK_DATA: 모든 I/O 정보를 담는 컨테이너

typedef struct _FLT_CALLBACK_DATA {
    FLT_CALLBACK_DATA_FLAGS Flags;            // 데이터 플래그
    PETHREAD Thread;                          // 요청 스레드
    PFLT_IO_PARAMETER_BLOCK Iopb;            // I/O 매개변수 (핵심!)
    IO_STATUS_BLOCK IoStatus;                 // I/O 결과 상태
    struct _FLT_TAG_DATA_BUFFER *TagData;     // 재파싱 데이터
    union {
        struct { ... } QueueLinks;            // 큐 연결
        PVOID QueueContext[2];                // 큐 컨텍스트
    };
    PVOID FilterContext[4];                   // 필터 컨텍스트
    KPROCESSOR_MODE RequestorMode;            // UserMode/KernelMode
    // ...
} FLT_CALLBACK_DATA, *PFLT_CALLBACK_DATA;
```

### 17.5.2 핵심 필드 접근

```c
// 콜백에서 자주 사용하는 정보 접근

FLT_PREOP_CALLBACK_STATUS PreOperation(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    // 1. MajorFunction (IRP 유형)
    UCHAR majorFunction = Data->Iopb->MajorFunction;
    // IRP_MJ_CREATE, IRP_MJ_READ, IRP_MJ_WRITE, ...

    // 2. 대상 파일 객체
    PFILE_OBJECT fileObject = Data->Iopb->TargetFileObject;

    // 3. 요청 모드
    KPROCESSOR_MODE mode = Data->RequestorMode;
    if (mode == KernelMode) {
        // 커널 모드 요청
    }

    // 4. 스레드 정보
    PETHREAD thread = Data->Thread;
    HANDLE processId = PsGetThreadProcessId(thread);

    // 5. I/O 결과 설정 (차단 시)
    Data->IoStatus.Status = STATUS_ACCESS_DENIED;
    Data->IoStatus.Information = 0;

    // 6. 작업별 매개변수 (유니온)
    // Create 작업:
    ULONG options = Data->Iopb->Parameters.Create.Options;

    // Write 작업:
    ULONG writeLength = Data->Iopb->Parameters.Write.Length;
    LARGE_INTEGER offset = Data->Iopb->Parameters.Write.ByteOffset;

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 17.6 FLT_RELATED_OBJECTS

### 17.6.1 구조체 정의

```c
// FLT_RELATED_OBJECTS: 현재 작업과 관련된 객체들

typedef struct _FLT_RELATED_OBJECTS {
    USHORT Size;                         // 구조체 크기
    USHORT TransactionContext;           // 트랜잭션 컨텍스트
    PFLT_FILTER Filter;                  // 현재 필터
    PFLT_VOLUME Volume;                  // 대상 볼륨
    PFLT_INSTANCE Instance;              // 현재 인스턴스
    PFILE_OBJECT FileObject;             // 파일 객체
    PKTRANSACTION Transaction;           // 트랜잭션 (TxF)
} FLT_RELATED_OBJECTS, *PFLT_RELATED_OBJECTS;
```

### 17.6.2 사용 예시

```c
FLT_PREOP_CALLBACK_STATUS PreOperation(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    // 현재 필터 확인
    PFLT_FILTER filter = FltObjects->Filter;

    // 볼륨 정보
    PFLT_VOLUME volume = FltObjects->Volume;

    // 인스턴스 (볼륨에 연결된 필터 인스턴스)
    PFLT_INSTANCE instance = FltObjects->Instance;

    // 파일 객체 (Pre-Create에서는 불완전할 수 있음)
    PFILE_OBJECT fileObject = FltObjects->FileObject;

    // 인스턴스 컨텍스트 가져오기
    PMY_INSTANCE_CONTEXT instCtx = NULL;
    NTSTATUS status = FltGetInstanceContext(
        FltObjects->Instance,
        &instCtx
    );

    if (NT_SUCCESS(status)) {
        // 컨텍스트 사용
        DbgPrint("Volume: %wZ\n", &instCtx->VolumeName);
        FltReleaseContext(instCtx);
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 17.7 인스턴스 생명주기

### 17.7.1 인스턴스란?

```
┌─────────────────────────────────────────────────────────────────────┐
│                        인스턴스 개념                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  하나의 Minifilter가 여러 볼륨에 연결 → 각각 별도 인스턴스          │
│                                                                      │
│  ┌──────────────────┐                                               │
│  │  MyDRMFilter     │ (FLT_FILTER)                                  │
│  └────────┬─────────┘                                               │
│           │                                                          │
│   ┌───────┼───────────────────────┐                                 │
│   │       │                       │                                 │
│   ▼       ▼                       ▼                                 │
│  ┌───┐  ┌───┐                   ┌───┐                              │
│  │C: │  │D: │                   │E: │                              │
│  └─┬─┘  └─┬─┘                   └─┬─┘                              │
│    │      │                       │                                 │
│    ▼      ▼                       ▼                                 │
│ Instance Instance               Instance                            │
│  #1       #2                      #3                                │
│                                                                      │
│  각 인스턴스는:                                                      │
│  - 자신의 볼륨에만 영향                                             │
│  - 별도의 인스턴스 컨텍스트 보유 가능                               │
│  - 독립적으로 연결/해제                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 17.7.2 인스턴스 콜백

```c
// InstanceSetupCallback: 볼륨에 연결될 때 호출

NTSTATUS InstanceSetup(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_SETUP_FLAGS Flags,
    DEVICE_TYPE VolumeDeviceType,
    FLT_FILESYSTEM_TYPE VolumeFilesystemType
)
{
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("InstanceSetup: DeviceType=%d, FSType=%d\n",
        VolumeDeviceType, VolumeFilesystemType);

    // 특정 볼륨 유형 제외
    if (VolumeDeviceType == FILE_DEVICE_CD_ROM_FILE_SYSTEM) {
        DbgPrint("  → Skipping CD-ROM\n");
        return STATUS_FLT_DO_NOT_ATTACH;  // 연결하지 않음
    }

    if (VolumeFilesystemType == FLT_FSTYPE_RAW) {
        DbgPrint("  → Skipping RAW filesystem\n");
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    // 인스턴스 컨텍스트 할당 가능
    // ...

    return STATUS_SUCCESS;  // 연결 허용
}

// InstanceQueryTeardownCallback: 분리 요청 시 호출

NTSTATUS InstanceQueryTeardown(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_QUERY_TEARDOWN_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(Flags);

    // 분리 허용
    return STATUS_SUCCESS;
}

// InstanceTeardownStartCallback: 분리 시작 시 호출

VOID InstanceTeardownStart(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_TEARDOWN_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("InstanceTeardownStart\n");

    // 진행 중인 I/O 완료 대기 시작
    // 새 작업 거부 등
}

// InstanceTeardownCompleteCallback: 분리 완료 시 호출

VOID InstanceTeardownComplete(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_TEARDOWN_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(Flags);

    DbgPrint("InstanceTeardownComplete\n");

    // 인스턴스 컨텍스트 정리 등
}
```

---

## 17.8 Minifilter 생명주기 요약

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Minifilter 생명주기                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 드라이버 로드                                                   │
│     └─ DriverEntry() 호출                                           │
│           └─ FltRegisterFilter() - 필터 등록                        │
│           └─ FltStartFiltering() - 필터링 시작                      │
│                                                                      │
│  2. 볼륨 연결 (각 볼륨마다)                                         │
│     └─ InstanceSetupCallback() - 연결 여부 결정                     │
│           └─ STATUS_SUCCESS: 인스턴스 생성                          │
│           └─ STATUS_FLT_DO_NOT_ATTACH: 연결 거부                    │
│                                                                      │
│  3. I/O 처리 (파일 작업마다)                                        │
│     └─ PreOperation() - 작업 전 처리                                │
│           └─ 파일 시스템 작업 수행                                  │
│     └─ PostOperation() - 작업 후 처리                               │
│                                                                      │
│  4. 볼륨 분리                                                       │
│     └─ InstanceQueryTeardownCallback() - 분리 허용 여부             │
│     └─ InstanceTeardownStartCallback() - 분리 시작                  │
│     └─ InstanceTeardownCompleteCallback() - 분리 완료               │
│                                                                      │
│  5. 드라이버 언로드                                                 │
│     └─ FilterUnloadCallback() - 정리 작업                           │
│     └─ FltUnregisterFilter() - 필터 등록 해제                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 요약

이 챕터에서 학습한 핵심 개념:

1. **Filter Manager**: Minifilter와 파일 시스템 사이의 중개자
2. **고도(Altitude)**: 콜백 호출 순서를 결정하는 숫자
3. **FLT_REGISTRATION**: Minifilter 정의 구조체
4. **Pre/Post 콜백**: I/O 작업 전후 처리 모델
5. **FLT_CALLBACK_DATA**: 모든 I/O 정보를 담는 컨테이너
6. **인스턴스**: 볼륨별 필터 연결 단위

다음 챕터에서는 첫 번째 Minifilter 드라이버를 직접 작성합니다.

---

## 참고 자료

- [Filter Manager Concepts](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/filter-manager-concepts)
- [FLT_REGISTRATION](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/ns-fltkernel-_flt_registration)
- [Pre-Operation Callback](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nc-fltkernel-pflt_pre_operation_callback)
- [Allocated Altitudes](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/allocated-altitudes)
