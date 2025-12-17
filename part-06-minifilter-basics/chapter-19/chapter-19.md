# Chapter 19: Minifilter 콜백 심화

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- 다양한 I/O 작업 콜백을 구현합니다
- 콜백에서 작업을 차단하거나 수정합니다
- 파일 이름 정보를 효율적으로 처리합니다
- 안전한 버퍼 접근 방법을 적용합니다
- 비동기 완료 처리를 구현합니다

## 도입: 콜백의 예술

이전 챕터에서 기본적인 모니터링 콜백을 구현했습니다. 이제 실제 DRM 기능에 필요한 **차단, 수정, 검사** 기능을 구현합니다. 콜백에서의 작업은 시스템 성능에 직접 영향을 미치므로 **효율적이고 안전한** 코드를 작성해야 합니다.

---

## 19.1 주요 IRP 유형과 콜백

### 19.1.1 파일 작업 IRP 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│                    주요 IRP 유형                                     │
├─────────────────────────────────────────────────────────────────────┤
│  IRP_MJ_CREATE             파일/디렉터리 열기, 생성                 │
│  IRP_MJ_CLOSE              핸들 닫기                                │
│  IRP_MJ_READ               파일 읽기                                │
│  IRP_MJ_WRITE              파일 쓰기                                │
│  IRP_MJ_QUERY_INFORMATION  파일 정보 조회 (크기, 시간 등)           │
│  IRP_MJ_SET_INFORMATION    파일 정보 설정, 이름 변경, 삭제          │
│  IRP_MJ_CLEANUP            마지막 핸들 닫힘 (파일 정리)             │
│  IRP_MJ_DIRECTORY_CONTROL  디렉터리 열거                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 19.1.2 콜백 등록 전체 예시

```c
CONST FLT_OPERATION_REGISTRATION Callbacks[] = {

    // 파일 열기/생성
    { IRP_MJ_CREATE,
      0,
      PreCreate,
      PostCreate
    },

    // 파일 읽기
    { IRP_MJ_READ,
      FLTFL_OPERATION_REGISTRATION_SKIP_PAGING_IO,  // 페이징 I/O 건너뛰기
      PreRead,
      PostRead
    },

    // 파일 쓰기
    { IRP_MJ_WRITE,
      FLTFL_OPERATION_REGISTRATION_SKIP_PAGING_IO,
      PreWrite,
      NULL
    },

    // 파일 정보 설정 (이름 변경, 삭제 포함)
    { IRP_MJ_SET_INFORMATION,
      0,
      PreSetInformation,
      NULL
    },

    // 파일 정리 (마지막 핸들 닫힘)
    { IRP_MJ_CLEANUP,
      0,
      PreCleanup,
      NULL
    },

    { IRP_MJ_OPERATION_END }
};
```

---

## 19.2 IRP_MJ_CREATE 심화

### 19.2.1 Create 매개변수 분석

```c
FLT_PREOP_CALLBACK_STATUS PreCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;

    // 1. Options 분석
    ULONG options = params->Create.Options;

    // 상위 8비트: CreateDisposition (생성 동작)
    ULONG disposition = (options >> 24) & 0xFF;

    // CreateDisposition 값:
    // FILE_SUPERSEDE (0)     - 파일 대체 (삭제 후 생성)
    // FILE_OPEN (1)          - 열기 (없으면 실패)
    // FILE_CREATE (2)        - 생성 (있으면 실패)
    // FILE_OPEN_IF (3)       - 열기 또는 생성
    // FILE_OVERWRITE (4)     - 덮어쓰기 (없으면 실패)
    // FILE_OVERWRITE_IF (5)  - 열기 또는 덮어쓰기

    // 하위 24비트: CreateOptions
    ULONG createOptions = options & 0x00FFFFFF;

    // CreateOptions 플래그:
    // FILE_DIRECTORY_FILE (0x001)      - 디렉터리 열기
    // FILE_NON_DIRECTORY_FILE (0x040)  - 파일만 열기
    // FILE_DELETE_ON_CLOSE (0x1000)    - 닫을 때 삭제
    // FILE_OPEN_FOR_BACKUP_INTENT      - 백업 용도

    // 2. DesiredAccess 분석 (SecurityContext에서)
    ACCESS_MASK desiredAccess =
        params->Create.SecurityContext->DesiredAccess;

    BOOLEAN wantRead = FlagOn(desiredAccess, FILE_READ_DATA);
    BOOLEAN wantWrite = FlagOn(desiredAccess, FILE_WRITE_DATA);
    BOOLEAN wantDelete = FlagOn(desiredAccess, DELETE);

    // 3. ShareAccess 분석
    USHORT shareAccess = params->Create.ShareAccess;

    BOOLEAN shareRead = FlagOn(shareAccess, FILE_SHARE_READ);
    BOOLEAN shareWrite = FlagOn(shareAccess, FILE_SHARE_WRITE);
    BOOLEAN shareDelete = FlagOn(shareAccess, FILE_SHARE_DELETE);

    // 4. FileAttributes
    USHORT fileAttributes = params->Create.FileAttributes;

    BOOLEAN isHidden = FlagOn(fileAttributes, FILE_ATTRIBUTE_HIDDEN);
    BOOLEAN isReadOnly = FlagOn(fileAttributes, FILE_ATTRIBUTE_READONLY);

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

### 19.2.2 PostCreate에서 파일 정보 확인

```c
FLT_POSTOP_CALLBACK_STATUS PostCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 성공한 경우만 처리
    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // Information 필드: 실제 수행된 동작
    ULONG_PTR information = Data->IoStatus.Information;

    // FILE_CREATED (2)    - 새로 생성됨
    // FILE_OPENED (1)     - 기존 파일 열림
    // FILE_OVERWRITTEN (3) - 덮어씌워짐
    // FILE_SUPERSEDED (0)  - 대체됨

    if (information == FILE_CREATED) {
        FmDbgPrint("PostCreate: New file created");
    } else if (information == FILE_OPENED) {
        FmDbgPrint("PostCreate: Existing file opened");
    }

    // 디렉터리 여부 확인
    BOOLEAN isDirectory = FALSE;
    FltIsDirectory(FltObjects->FileObject, FltObjects->Instance, &isDirectory);

    if (isDirectory) {
        FmDbgPrint("PostCreate: This is a directory");
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## 19.3 IRP_MJ_WRITE 심화

### 19.3.1 Write 매개변수 분석

```c
FLT_PREOP_CALLBACK_STATUS PreWrite(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;

    // 쓰기 길이
    ULONG writeLength = params->Write.Length;

    // 쓰기 오프셋
    LARGE_INTEGER byteOffset = params->Write.ByteOffset;

    // 특수 오프셋 값
    // FILE_WRITE_TO_END_OF_FILE: 파일 끝에 추가
    // FILE_USE_FILE_POINTER_POSITION: 현재 위치 사용

    // MDL (Memory Descriptor List) - 버퍼 매핑
    PMDL mdlAddress = params->Write.MdlAddress;

    // 버퍼 (Non-Cached I/O일 때)
    PVOID writeBuffer = params->Write.WriteBuffer;

    FmDbgPrint("PreWrite: Length=%lu, Offset=%lld",
        writeLength, byteOffset.QuadPart);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### 19.3.2 안전한 버퍼 접근

```c
FLT_PREOP_CALLBACK_STATUS PreWrite(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    PVOID buffer = NULL;
    PMDL mdl = NULL;

    // 1. MDL 우선 확인
    if (params->Write.MdlAddress != NULL) {
        // MDL에서 시스템 주소 획득
        buffer = MmGetSystemAddressForMdlSafe(
            params->Write.MdlAddress,
            NormalPagePriority | MdlMappingNoExecute
        );
    }

    // 2. MDL이 없으면 WriteBuffer 사용
    if (buffer == NULL) {
        buffer = params->Write.WriteBuffer;
    }

    // 3. 버퍼 접근 가능 여부 확인
    if (buffer == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 4. IRQL 확인 (페이징 I/O는 DISPATCH_LEVEL일 수 있음)
    if (KeGetCurrentIrql() > APC_LEVEL) {
        // 높은 IRQL - 복잡한 작업 불가
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 5. 사용자 모드 버퍼 검증 (필요 시)
    if (Data->RequestorMode == UserMode) {
        __try {
            ProbeForRead(buffer, params->Write.Length, sizeof(UCHAR));

            // 버퍼 내용 검사
            // 예: 주민등록번호 패턴 검사
            if (ContainsSensitivePattern(buffer, params->Write.Length)) {
                Data->IoStatus.Status = STATUS_ACCESS_DENIED;
                return FLT_PREOP_COMPLETE;  // 쓰기 차단!
            }
        }
        __except (EXCEPTION_EXECUTE_HANDLER) {
            // 잘못된 버퍼
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
        }
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 19.4 IRP_MJ_SET_INFORMATION - 이름 변경 및 삭제

### 19.4.1 FileInformationClass 분류

```c
FLT_PREOP_CALLBACK_STATUS PreSetInformation(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;

    // 어떤 정보를 설정하는지 확인
    FILE_INFORMATION_CLASS infoClass =
        params->SetFileInformation.FileInformationClass;

    switch (infoClass) {

    case FileDispositionInformation:
    case FileDispositionInformationEx:
        // 파일 삭제 시도
        return PreSetDisposition(Data, FltObjects, CompletionContext);

    case FileRenameInformation:
    case FileRenameInformationEx:
        // 파일 이름 변경
        return PreSetRename(Data, FltObjects, CompletionContext);

    case FileBasicInformation:
        // 기본 정보 (시간, 속성) 변경
        break;

    case FileEndOfFileInformation:
        // 파일 크기 변경
        break;

    default:
        break;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### 19.4.2 파일 삭제 차단

```c
FLT_PREOP_CALLBACK_STATUS PreSetDisposition(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(CompletionContext);

    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;

    // 삭제 플래그 확인
    PFILE_DISPOSITION_INFORMATION dispInfo =
        Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

    if (!dispInfo->DeleteFile) {
        // 삭제 해제 요청 - 통과
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 가져오기
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    FltParseFileNameInformation(nameInfo);

    // 보호된 파일인지 확인
    if (IsProtectedFile(&nameInfo->Name)) {
        FmDbgPrint("PreSetDisposition: Blocking delete of %wZ",
            &nameInfo->Name);

        FltReleaseFileNameInformation(nameInfo);

        // 삭제 차단!
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    FltReleaseFileNameInformation(nameInfo);
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### 19.4.3 파일 이름 변경 모니터링/차단

```c
FLT_PREOP_CALLBACK_STATUS PreSetRename(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    UNREFERENCED_PARAMETER(CompletionContext);

    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION sourceNameInfo = NULL;
    PFILE_RENAME_INFORMATION renameInfo;
    UNICODE_STRING targetName;

    // 1. 원본 파일 이름 가져오기
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &sourceNameInfo
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    FltParseFileNameInformation(sourceNameInfo);

    // 2. 대상 이름 가져오기
    renameInfo = Data->Iopb->Parameters.SetFileInformation.InfoBuffer;

    targetName.Buffer = renameInfo->FileName;
    targetName.Length = (USHORT)renameInfo->FileNameLength;
    targetName.MaximumLength = targetName.Length;

    FmDbgPrint("PreSetRename: %wZ -> %wZ",
        &sourceNameInfo->Name, &targetName);

    // 3. 보호된 파일 이름 변경 차단
    if (IsProtectedFile(&sourceNameInfo->Name)) {
        FmDbgPrint("PreSetRename: Blocking rename of protected file");

        FltReleaseFileNameInformation(sourceNameInfo);
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        return FLT_PREOP_COMPLETE;
    }

    // 4. 확장자 변경 차단 (예: .docx -> .txt 금지)
    UNICODE_STRING sourceExt = sourceNameInfo->Extension;
    // 대상 확장자 파싱 필요...

    FltReleaseFileNameInformation(sourceNameInfo);
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 19.5 파일 이름 처리 심화

### 19.5.1 파일 이름 유형

```c
// FltGetFileNameInformation 플래그

// 이름 형식 (택일)
#define FLT_FILE_NAME_NORMALIZED      0x0001  // 정규화된 전체 경로
#define FLT_FILE_NAME_OPENED          0x0002  // 열 때 사용된 이름
#define FLT_FILE_NAME_SHORT           0x0003  // 8.3 형식 짧은 이름

// 쿼리 방법 (택일)
#define FLT_FILE_NAME_QUERY_DEFAULT             0x0100
#define FLT_FILE_NAME_QUERY_CACHE_ONLY          0x0200
#define FLT_FILE_NAME_QUERY_FILESYSTEM_ONLY     0x0300
#define FLT_FILE_NAME_QUERY_ALWAYS_ALLOW_CACHE  0x0400

// 추가 옵션
#define FLT_FILE_NAME_ALLOW_QUERY_ON_REPARSE    0x04000000

// 권장 조합
FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT
```

### 19.5.2 FLT_FILE_NAME_INFORMATION 구조

```c
typedef struct _FLT_FILE_NAME_INFORMATION {
    USHORT Size;              // 구조체 크기
    FLT_FILE_NAME_PARSED_FLAGS NamesParsed;  // 파싱된 항목
    FLT_FILE_NAME_OPTIONS Format;            // 이름 형식
    UNICODE_STRING Name;           // 전체 이름
    UNICODE_STRING Volume;         // 볼륨 (\Device\HarddiskVolume2)
    UNICODE_STRING Share;          // 네트워크 공유
    UNICODE_STRING Extension;      // 확장자 (점 제외)
    UNICODE_STRING Stream;         // 스트림 이름 (NTFS ADS)
    UNICODE_STRING FinalComponent; // 파일/디렉터리 이름
    UNICODE_STRING ParentDir;      // 부모 디렉터리
} FLT_FILE_NAME_INFORMATION, *PFLT_FILE_NAME_INFORMATION;

// 사용 예
void AnalyzeFileName(PFLT_FILE_NAME_INFORMATION nameInfo)
{
    // 반드시 FltParseFileNameInformation 호출 후 사용!
    FltParseFileNameInformation(nameInfo);

    FmDbgPrint("Full Name: %wZ", &nameInfo->Name);
    FmDbgPrint("Volume: %wZ", &nameInfo->Volume);
    FmDbgPrint("Parent: %wZ", &nameInfo->ParentDir);
    FmDbgPrint("Final: %wZ", &nameInfo->FinalComponent);
    FmDbgPrint("Extension: %wZ", &nameInfo->Extension);

    // 예: \Device\HarddiskVolume2\Users\User\Documents\test.docx
    // Volume: \Device\HarddiskVolume2
    // ParentDir: \Users\User\Documents
    // FinalComponent: test.docx
    // Extension: docx
}
```

### 19.5.3 효율적인 파일 이름 캐싱

```c
// 전역 또는 스트림 컨텍스트에 캐싱
typedef struct _MY_STREAM_CONTEXT {
    UNICODE_STRING FileName;      // 캐싱된 파일 이름
    BOOLEAN IsProtected;          // 보호 여부 캐싱
    BOOLEAN IsParsed;             // 분석 완료 여부
} MY_STREAM_CONTEXT, *PMY_STREAM_CONTEXT;

// PostCreate에서 컨텍스트에 캐싱
FLT_POSTOP_CALLBACK_STATUS PostCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    NTSTATUS status;
    PMY_STREAM_CONTEXT streamCtx = NULL;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;

    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 스트림 컨텍스트 생성/획득
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        &streamCtx
    );

    if (status == STATUS_NOT_FOUND) {
        // 새 컨텍스트 생성 및 설정
        // ...
    }

    if (NT_SUCCESS(status) && !streamCtx->IsParsed) {
        // 파일 이름 가져와서 컨텍스트에 저장
        status = FltGetFileNameInformation(
            Data,
            FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
            &nameInfo
        );

        if (NT_SUCCESS(status)) {
            FltParseFileNameInformation(nameInfo);

            // 이름 복사
            CopyUnicodeString(&streamCtx->FileName, &nameInfo->Name);

            // 보호 여부 분석
            streamCtx->IsProtected = IsProtectedFile(&nameInfo->Name);
            streamCtx->IsParsed = TRUE;

            FltReleaseFileNameInformation(nameInfo);
        }
    }

    if (streamCtx != NULL) {
        FltReleaseContext(streamCtx);
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}

// 이후 콜백에서는 컨텍스트에서 가져옴 (빠름!)
FLT_PREOP_CALLBACK_STATUS PreWrite(...)
{
    PMY_STREAM_CONTEXT streamCtx = NULL;

    status = FltGetStreamContext(..., &streamCtx);
    if (NT_SUCCESS(status)) {
        if (streamCtx->IsProtected) {
            // 보호된 파일 처리
        }
        FltReleaseContext(streamCtx);
    }
}
```

---

## 19.6 비동기 처리 패턴

### 19.6.1 FLT_PREOP_PENDING 사용

```c
// 시간이 오래 걸리는 작업은 Pending 처리

typedef struct _WORK_ITEM_CONTEXT {
    PFLT_CALLBACK_DATA Data;
    PCFLT_RELATED_OBJECTS FltObjects;
    PFLT_DEFERRED_IO_WORKITEM WorkItem;
} WORK_ITEM_CONTEXT, *PWORK_ITEM_CONTEXT;

FLT_PREOP_CALLBACK_STATUS PreWrite(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    NTSTATUS status;
    PWORK_ITEM_CONTEXT workCtx;
    PFLT_DEFERRED_IO_WORKITEM workItem;

    // 긴 작업이 필요한 경우 (예: 클라우드 검증)

    // 1. 작업 아이템 할당
    workItem = FltAllocateDeferredIoWorkItem();
    if (workItem == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 2. 컨텍스트 할당
    workCtx = ExAllocatePoolWithTag(NonPagedPool,
        sizeof(WORK_ITEM_CONTEXT), FM_TAG);
    if (workCtx == NULL) {
        FltFreeDeferredIoWorkItem(workItem);
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    workCtx->Data = Data;
    workCtx->FltObjects = FltObjects;
    workCtx->WorkItem = workItem;

    // 3. 작업 큐에 추가
    status = FltQueueDeferredIoWorkItem(
        workItem,
        Data,
        DeferredWorkRoutine,  // 작업 루틴
        DelayedWorkQueue,     // 우선순위
        workCtx
    );

    if (!NT_SUCCESS(status)) {
        ExFreePoolWithTag(workCtx, FM_TAG);
        FltFreeDeferredIoWorkItem(workItem);
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 4. Pending 반환
    return FLT_PREOP_PENDING;
}

// 지연 작업 루틴
VOID DeferredWorkRoutine(
    PFLT_DEFERRED_IO_WORKITEM WorkItem,
    PFLT_CALLBACK_DATA Data,
    PVOID Context
)
{
    PWORK_ITEM_CONTEXT workCtx = (PWORK_ITEM_CONTEXT)Context;

    // 시간 걸리는 작업 수행
    BOOLEAN allowed = PerformCloudVerification(Data);

    if (allowed) {
        // 작업 허용 - 하위로 전달
        FltCompletePendedPreOperation(
            Data,
            FLT_PREOP_SUCCESS_NO_CALLBACK,
            NULL
        );
    } else {
        // 작업 거부
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        FltCompletePendedPreOperation(
            Data,
            FLT_PREOP_COMPLETE,
            NULL
        );
    }

    // 정리
    FltFreeDeferredIoWorkItem(workCtx->WorkItem);
    ExFreePoolWithTag(workCtx, FM_TAG);
}
```

### 19.6.2 PostOperation에서 Safe Post 처리

```c
// Post 콜백이 DISPATCH_LEVEL에서 호출될 수 있는 경우

FLT_POSTOP_CALLBACK_STATUS PostRead(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    // Draining 체크
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // IRQL 확인
    if (KeGetCurrentIrql() > APC_LEVEL) {
        // 높은 IRQL - 안전한 Post 필요
        FLT_POSTOP_CALLBACK_STATUS status;

        status = FltDoCompletionProcessingWhenSafe(
            Data,
            FltObjects,
            CompletionContext,
            Flags,
            SafePostRead,     // 안전한 컨텍스트에서 호출될 함수
            &status
        );

        if (status == FLT_POSTOP_MORE_PROCESSING_REQUIRED) {
            return FLT_POSTOP_MORE_PROCESSING_REQUIRED;
        }
    }

    // 낮은 IRQL - 직접 처리
    return ProcessPostRead(Data, FltObjects, CompletionContext);
}

// 안전한 IRQL에서 호출되는 함수
FLT_POSTOP_CALLBACK_STATUS SafePostRead(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    UNREFERENCED_PARAMETER(Flags);

    return ProcessPostRead(Data, FltObjects, CompletionContext);
}

// 실제 처리 로직
FLT_POSTOP_CALLBACK_STATUS ProcessPostRead(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext
)
{
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    // 읽기 완료 후 처리
    // 예: 데이터 복호화

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## 19.7 콜백 작성 베스트 프랙티스

### 19.7.1 DO와 DON'T

```c
// ✅ DO - 올바른 패턴

// 1. 항상 FltReleaseFileNameInformation 호출
status = FltGetFileNameInformation(..., &nameInfo);
if (NT_SUCCESS(status)) {
    // 사용
    FltReleaseFileNameInformation(nameInfo);  // 항상!
}

// 2. 커널 모드 요청 빠르게 통과
if (Data->RequestorMode == KernelMode) {
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 3. 사용자 모드 버퍼 접근 시 예외 처리
__try {
    ProbeForRead(buffer, length, 1);
    // 버퍼 접근
}
__except (EXCEPTION_EXECUTE_HANDLER) {
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 4. IRQL 확인 후 적절한 작업
if (KeGetCurrentIrql() > APC_LEVEL) {
    // 제한된 작업만
}

// ❌ DON'T - 피해야 할 패턴

// 1. FltReleaseFileNameInformation 누락
status = FltGetFileNameInformation(..., &nameInfo);
if (NT_SUCCESS(status)) {
    if (SomeCondition()) {
        return FLT_PREOP_COMPLETE;  // nameInfo 누수!
    }
}

// 2. 긴 작업을 콜백에서 직접 수행
status = SendToCloud(buffer);  // 네트워크 대기 - 문제!
// 대신 FLT_PREOP_PENDING 사용

// 3. 무조건 PostOperation 요청
return FLT_PREOP_SUCCESS_WITH_CALLBACK;  // 필요 없으면 NO_CALLBACK

// 4. Paged 메모리를 DISPATCH_LEVEL에서 접근
PagedBuffer = ...;  // BSOD 위험!
```

### 19.7.2 성능 최적화 팁

```c
// 1. 불필요한 파일 이름 조회 피하기
// PreCreate에서 파일 이름 조회는 비용이 큼
// 가능하면 PostCreate에서 조회 (파일 시스템이 이미 계산함)

// 2. 콜백 등록 시 Skip 플래그 활용
{ IRP_MJ_READ,
  FLTFL_OPERATION_REGISTRATION_SKIP_PAGING_IO |  // 페이징 I/O 건너뛰기
  FLTFL_OPERATION_REGISTRATION_SKIP_CACHED_IO,   // 캐시 I/O 건너뛰기
  PreRead,
  NULL
}

// 3. 스트림 컨텍스트에 정보 캐싱
// 파일 이름, 보호 여부 등 반복 조회되는 정보 캐싱

// 4. 조기 반환
if (!NeedToProcess(Data)) {
    return FLT_PREOP_SUCCESS_NO_CALLBACK;  // 빠르게 반환
}

// 5. Post 콜백 필요 없으면 등록 안 함
{ IRP_MJ_WRITE, 0, PreWrite, NULL }  // PostWrite 불필요

// 6. 복잡한 조건 캐싱
static BOOLEAN g_FilterEnabled = TRUE;  // 정책 캐싱
if (!g_FilterEnabled) {
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 19.8 디버깅 및 문제 해결

### 19.8.1 콜백 디버깅

```
// WinDbg에서 콜백 브레이크포인트
bu FileMonitor!PreCreate
bu FileMonitor!PreWrite

// 콜백 데이터 분석
!fltkd.cbd @rcx

// 반환값에 브레이크
bp FileMonitor!PreCreate+0x100 ".if (@rax == 2) {.echo COMPLETE!} .else {g}"
```

### 19.8.2 일반적인 문제

```
┌─────────────────────────────────────────────────────────────────────┐
│ 문제: 무한 루프 / 시스템 멈춤                                       │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: 콜백에서 파일 I/O 수행 → 재진입                               │
│ 해결: FltIs... 함수로 재진입 체크                                   │
│       또는 IoStatus 필터 사용                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 문제: 특정 파일에서만 차단이 안 됨                                  │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: 커널 모드 요청은 통과시킴                                     │
│       또는 네트워크/RAW 볼륨 제외됨                                 │
│ 해결: 로깅 추가하여 흐름 확인                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ 문제: 간헐적 BSOD                                                   │
├─────────────────────────────────────────────────────────────────────┤
│ 원인: 리소스 누수 (FltReleaseFileNameInformation 누락)              │
│       또는 잘못된 IRQL에서 작업                                     │
│ 해결: Driver Verifier 활성화, 코드 리뷰                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 요약

이 챕터에서 학습한 내용:

1. **주요 IRP 콜백**: CREATE, READ, WRITE, SET_INFORMATION
2. **매개변수 분석**: Options, DesiredAccess, ShareAccess 해석
3. **버퍼 접근**: MDL과 버퍼 안전하게 처리
4. **파일 이름**: FltGetFileNameInformation, FltParseFileNameInformation
5. **비동기 처리**: FLT_PREOP_PENDING, FltDoCompletionProcessingWhenSafe
6. **베스트 프랙티스**: 리소스 관리, 성능 최적화

다음 챕터에서는 컨텍스트 관리를 상세히 학습합니다.

---

## 참고 자료

- [Minifilter Pre-Operation Callback](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nc-fltkernel-pflt_pre_operation_callback)
- [FltGetFileNameInformation](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltgetfilenameinformation)
- [IRP Major Function Codes](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/irp-major-function-codes)
