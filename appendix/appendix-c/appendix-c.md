# 부록 C: IRP 주요 함수 코드 레퍼런스

## I/O Request Packet 완벽 가이드

이 부록은 Minifilter 드라이버 개발에서 처리해야 하는 IRP(I/O Request Packet) Major Function 코드를 정리한 레퍼런스입니다. 각 IRP에 대한 설명과 콜백 구현 패턴을 제공합니다.

---

## C.1 IRP 구조 이해

### IRP Major Function 개요

Windows I/O 관리자는 모든 I/O 작업을 IRP로 표현합니다. 각 IRP는 Major Function 코드로 작업 유형을 나타냅니다.

```
┌─────────────────────────────────────────────────────────────┐
│                          IRP                                 │
├─────────────────────────────────────────────────────────────┤
│  MajorFunction  │  IRP_MJ_CREATE, IRP_MJ_READ, ...          │
├─────────────────────────────────────────────────────────────┤
│  MinorFunction  │  추가 작업 정보 (일부 IRP에서만 사용)      │
├─────────────────────────────────────────────────────────────┤
│  Parameters     │  IRP별 매개변수 (IO_STACK_LOCATION)        │
├─────────────────────────────────────────────────────────────┤
│  IoStatus       │  결과 상태 및 정보                         │
└─────────────────────────────────────────────────────────────┘
```

### Minifilter에서의 IRP 처리

Minifilter는 FLT_CALLBACK_DATA를 통해 IRP에 접근합니다:

```c
FLT_PREOP_CALLBACK_STATUS PreOperation(
    _Inout_ PFLT_CALLBACK_DATA Data,           // IRP 정보 포함
    _In_ PCFLT_RELATED_OBJECTS FltObjects,     // 관련 객체
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    // Major Function 확인
    // Check Major Function
    UCHAR majorFunction = Data->Iopb->MajorFunction;

    // Minor Function 확인 (해당되는 경우)
    // Check Minor Function (if applicable)
    UCHAR minorFunction = Data->Iopb->MinorFunction;

    // 매개변수 접근
    // Access parameters
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
}
```

### C# 비유

```csharp
// C#에서 이벤트 핸들러와 유사한 개념
// Similar concept to event handlers in C#

// C# 파일 시스템 감시
var watcher = new FileSystemWatcher();
watcher.Created += OnFileCreated;   // IRP_MJ_CREATE와 유사
watcher.Changed += OnFileChanged;   // IRP_MJ_WRITE와 유사
watcher.Deleted += OnFileDeleted;   // IRP_MJ_CLEANUP과 유사

// Minifilter는 더 세밀한 제어 가능
// Minifilter allows more granular control
```

---

## C.2 파일 생성/열기 (IRP_MJ_CREATE)

### 개요

파일이나 디렉터리를 열거나 새로 생성할 때 발생합니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_CREATE (0x00) |
| IRQL | PASSIVE_LEVEL |
| 페이지 가능 | Yes |

### 매개변수

```c
typedef struct _FLT_PARAMETERS_CREATE {
    PIO_SECURITY_CONTEXT SecurityContext;     // 보안 컨텍스트
    ULONG Options;                            // 생성 옵션
    USHORT FileAttributes;                    // 파일 속성
    USHORT ShareAccess;                       // 공유 모드
    ULONG EaLength;                           // 확장 속성 길이
    PVOID EaBuffer;                           // 확장 속성 버퍼
    LARGE_INTEGER AllocationSize;             // 초기 할당 크기
} FLT_PARAMETERS_CREATE;
```

### Options 플래그

| 플래그 | 설명 |
|--------|------|
| FILE_DIRECTORY_FILE | 디렉터리 열기/생성 |
| FILE_NON_DIRECTORY_FILE | 파일만 (디렉터리 제외) |
| FILE_DELETE_ON_CLOSE | 닫을 때 삭제 |
| FILE_OPEN_REPARSE_POINT | 리파스 포인트 직접 열기 |
| FILE_SEQUENTIAL_ONLY | 순차 접근만 |
| FILE_RANDOM_ACCESS | 랜덤 접근 |

### Disposition (Options의 상위 8비트)

```c
// Disposition 추출
// Extract Disposition
ULONG disposition = (params->Create.Options >> 24) & 0xFF;
```

| 값 | 이름 | 설명 |
|----|------|------|
| 0 | FILE_SUPERSEDE | 기존 파일 대체 |
| 1 | FILE_OPEN | 기존 파일 열기 (없으면 실패) |
| 2 | FILE_CREATE | 새 파일 생성 (있으면 실패) |
| 3 | FILE_OPEN_IF | 열기 또는 생성 |
| 4 | FILE_OVERWRITE | 열고 덮어쓰기 (없으면 실패) |
| 5 | FILE_OVERWRITE_IF | 열고 덮어쓰기 또는 생성 |

### Desired Access 플래그

```c
ACCESS_MASK desiredAccess = params->Create.SecurityContext->DesiredAccess;
```

| 플래그 | 설명 |
|--------|------|
| FILE_READ_DATA | 데이터 읽기 |
| FILE_WRITE_DATA | 데이터 쓰기 |
| FILE_APPEND_DATA | 데이터 추가 |
| FILE_READ_ATTRIBUTES | 속성 읽기 |
| FILE_WRITE_ATTRIBUTES | 속성 쓰기 |
| FILE_EXECUTE | 실행 |
| DELETE | 삭제 |
| READ_CONTROL | 보안 정보 읽기 |
| WRITE_DAC | DACL 수정 |
| WRITE_OWNER | 소유자 수정 |
| SYNCHRONIZE | 동기화 대기 |

### 콜백 구현

```c
FLT_PREOP_CALLBACK_STATUS PreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    ACCESS_MASK desiredAccess;
    ULONG disposition;
    ULONG createOptions;
    BOOLEAN isDirectory;

    UNREFERENCED_PARAMETER(CompletionContext);

    // 파일 이름 가져오기
    // Get file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo);

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_WITH_CALLBACK;
    }

    status = FltParseFileNameInformation(nameInfo);
    if (!NT_SUCCESS(status)) {
        FltReleaseFileNameInformation(nameInfo);
        return FLT_PREOP_SUCCESS_WITH_CALLBACK;
    }

    // 생성 매개변수 추출
    // Extract create parameters
    desiredAccess = params->Create.SecurityContext->DesiredAccess;
    disposition = (params->Create.Options >> 24) & 0xFF;
    createOptions = params->Create.Options & 0x00FFFFFF;
    isDirectory = BooleanFlagOn(createOptions, FILE_DIRECTORY_FILE);

    // DRM 정책 확인
    // Check DRM policy
    if (IsProtectedFile(&nameInfo->Name)) {
        // 쓰기 접근 차단
        // Block write access
        if (desiredAccess & (FILE_WRITE_DATA | FILE_APPEND_DATA | DELETE)) {
            if (!IsAuthorizedProcess(PsGetCurrentProcessId())) {
                FltReleaseFileNameInformation(nameInfo);

                Data->IoStatus.Status = STATUS_ACCESS_DENIED;
                Data->IoStatus.Information = 0;
                return FLT_PREOP_COMPLETE;
            }
        }
    }

    // 컨텍스트에 정보 저장 (PostCreate에서 사용)
    // Save info to context (for use in PostCreate)
    PCREATE_CONTEXT createCtx = ExAllocatePool2(
        POOL_FLAG_NON_PAGED,
        sizeof(CREATE_CONTEXT),
        TAG_POOL);

    if (createCtx != NULL) {
        createCtx->IsProtected = IsProtectedFile(&nameInfo->Name);
        createCtx->DesiredAccess = desiredAccess;
        *CompletionContext = createCtx;
    }

    FltReleaseFileNameInformation(nameInfo);
    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

FLT_POSTOP_CALLBACK_STATUS PostCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    NTSTATUS status = Data->IoStatus.Status;
    PCREATE_CONTEXT createCtx = (PCREATE_CONTEXT)CompletionContext;

    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        goto Cleanup;
    }

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // 성공적으로 열린 경우 스트림 컨텍스트 설정
    // If successfully opened, set stream context
    if (createCtx != NULL && createCtx->IsProtected) {
        SetupStreamContext(FltObjects, createCtx->DesiredAccess);
    }

Cleanup:
    if (createCtx != NULL) {
        ExFreePoolWithTag(createCtx, TAG_POOL);
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## C.3 파일 읽기 (IRP_MJ_READ)

### 개요

파일에서 데이터를 읽을 때 발생합니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_READ (0x03) |
| IRQL | PASSIVE_LEVEL (대부분), DISPATCH_LEVEL (페이징 I/O) |
| 페이지 가능 | 조건부 |

### 매개변수

```c
typedef struct _FLT_PARAMETERS_READ {
    ULONG Length;                    // 읽을 바이트 수
    ULONG Key;                       // 바이트 범위 잠금 키
    LARGE_INTEGER ByteOffset;        // 시작 오프셋
    PVOID ReadBuffer;                // 버퍼 (MdlAddress 또는 직접)
    PMDL MdlAddress;                 // MDL (있는 경우)
} FLT_PARAMETERS_READ;
```

### I/O 유형 플래그

```c
// I/O 플래그 확인
// Check I/O flags
BOOLEAN isPaging = FlagOn(Data->Iopb->IrpFlags, IRP_PAGING_IO);
BOOLEAN isNonCached = FlagOn(Data->Iopb->IrpFlags, IRP_NOCACHE);
BOOLEAN isSynchronous = FlagOn(Data->Iopb->IrpFlags, IRP_SYNCHRONOUS_API);
```

| 플래그 | 설명 |
|--------|------|
| IRP_PAGING_IO | 메모리 관리자의 페이징 I/O |
| IRP_NOCACHE | 캐시 우회 |
| IRP_SYNCHRONOUS_API | 동기 호출 |
| IRP_SYNCHRONOUS_PAGING_IO | 동기 페이징 I/O |

### 콜백 구현

```c
FLT_PREOP_CALLBACK_STATUS PreRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    PSTREAM_CONTEXT streamCtx = NULL;
    NTSTATUS status;

    UNREFERENCED_PARAMETER(CompletionContext);

    // 페이징 I/O는 빠르게 통과
    // Pass paging I/O quickly
    if (FlagOn(Data->Iopb->IrpFlags, IRP_PAGING_IO)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 스트림 컨텍스트 확인
    // Check stream context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&streamCtx);

    if (!NT_SUCCESS(status) || streamCtx == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 보호된 파일인 경우 접근 로깅
    // Log access for protected files
    if (streamCtx->IsProtected) {
        LogFileAccess(
            FltObjects,
            FILE_ACCESS_READ,
            params->Read.ByteOffset.QuadPart,
            params->Read.Length);
    }

    FltReleaseContext(streamCtx);
    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

FLT_POSTOP_CALLBACK_STATUS PostRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    PVOID buffer = NULL;
    ULONG bytesRead;

    UNREFERENCED_PARAMETER(CompletionContext);

    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    bytesRead = (ULONG)Data->IoStatus.Information;
    if (bytesRead == 0) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 버퍼 접근
    // Access buffer
    if (params->Read.MdlAddress != NULL) {
        buffer = MmGetSystemAddressForMdlSafe(
            params->Read.MdlAddress,
            NormalPagePriority | MdlMappingNoExecute);
    } else {
        buffer = params->Read.ReadBuffer;
    }

    if (buffer != NULL) {
        // 복호화 또는 콘텐츠 검사
        // Decryption or content inspection
        ProcessReadData(buffer, bytesRead);
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## C.4 파일 쓰기 (IRP_MJ_WRITE)

### 개요

파일에 데이터를 쓸 때 발생합니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_WRITE (0x04) |
| IRQL | PASSIVE_LEVEL (대부분), DISPATCH_LEVEL (페이징 I/O) |
| 페이지 가능 | 조건부 |

### 매개변수

```c
typedef struct _FLT_PARAMETERS_WRITE {
    ULONG Length;                    // 쓸 바이트 수
    ULONG Key;                       // 바이트 범위 잠금 키
    LARGE_INTEGER ByteOffset;        // 시작 오프셋
    PVOID WriteBuffer;               // 버퍼
    PMDL MdlAddress;                 // MDL (있는 경우)
} FLT_PARAMETERS_WRITE;
```

### 특수 오프셋 값

| 값 | 의미 |
|----|------|
| FILE_WRITE_TO_END_OF_FILE | 파일 끝에 추가 (0xFFFFFFFF FFFFFFFF) |
| FILE_USE_FILE_POINTER_POSITION | 현재 파일 포인터 위치 사용 |

### 콜백 구현

```c
FLT_PREOP_CALLBACK_STATUS PreWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    PSTREAM_CONTEXT streamCtx = NULL;
    NTSTATUS status;
    PVOID buffer = NULL;

    UNREFERENCED_PARAMETER(CompletionContext);

    // 페이징 I/O는 빠르게 통과
    // Pass paging I/O quickly
    if (FlagOn(Data->Iopb->IrpFlags, IRP_PAGING_IO)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 쓸 내용이 없으면 통과
    // Pass if nothing to write
    if (params->Write.Length == 0) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 스트림 컨텍스트 확인
    // Check stream context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&streamCtx);

    if (!NT_SUCCESS(status) || streamCtx == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    if (streamCtx->IsProtected) {
        // 버퍼 접근
        // Access buffer
        if (params->Write.MdlAddress != NULL) {
            buffer = MmGetSystemAddressForMdlSafe(
                params->Write.MdlAddress,
                NormalPagePriority | MdlMappingNoExecute);
        } else {
            buffer = params->Write.WriteBuffer;
        }

        if (buffer != NULL) {
            // SSN 패턴 검사
            // Check for SSN pattern
            if (ContainsSsnPattern(buffer, params->Write.Length)) {
                FltReleaseContext(streamCtx);

                // 차단
                // Block
                Data->IoStatus.Status = STATUS_ACCESS_DENIED;
                Data->IoStatus.Information = 0;
                return FLT_PREOP_COMPLETE;
            }
        }
    }

    FltReleaseContext(streamCtx);
    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## C.5 파일 정리 (IRP_MJ_CLEANUP)

### 개요

파일 핸들이 닫힐 때 발생합니다. 파일에 대한 마지막 사용자 핸들이 닫힐 때 호출됩니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_CLEANUP (0x12) |
| IRQL | PASSIVE_LEVEL |
| 페이지 가능 | Yes |

### CLEANUP vs CLOSE 차이

```
┌─────────────────────────────────────────────────────────────┐
│                      파일 핸들 참조                          │
├─────────────────────────────────────────────────────────────┤
│  CreateFile()       → FileObject 생성 (참조 수 = 1)        │
│  DuplicateHandle()  → 참조 수 증가                          │
│  CloseHandle()      → 참조 수 감소                          │
├─────────────────────────────────────────────────────────────┤
│  마지막 핸들 닫힘   → IRP_MJ_CLEANUP 발생                   │
│  마지막 참조 해제   → IRP_MJ_CLOSE 발생                     │
└─────────────────────────────────────────────────────────────┘
```

### 콜백 구현

```c
FLT_PREOP_CALLBACK_STATUS PreCleanup(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    PSTREAM_CONTEXT streamCtx = NULL;
    NTSTATUS status;

    UNREFERENCED_PARAMETER(Data);
    UNREFERENCED_PARAMETER(CompletionContext);

    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&streamCtx);

    if (NT_SUCCESS(status) && streamCtx != NULL) {
        // 파일 수정 여부 확인 및 로깅
        // Check if file was modified and log
        if (streamCtx->WasModified) {
            LogFileModification(FltObjects, streamCtx);
        }

        // 캐시된 데이터 플러시
        // Flush cached data
        if (streamCtx->HasCachedData) {
            FlushCachedData(streamCtx);
        }

        FltReleaseContext(streamCtx);
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## C.6 파일 닫기 (IRP_MJ_CLOSE)

### 개요

파일 객체에 대한 마지막 참조가 해제될 때 발생합니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_CLOSE (0x02) |
| IRQL | PASSIVE_LEVEL (또는 임의 스레드 컨텍스트) |
| 페이지 가능 | No (임의 스레드에서 호출될 수 있음) |

### 주의사항

- IRP_MJ_CLOSE는 **실패할 수 없습니다**
- 임의의 스레드 컨텍스트에서 호출될 수 있습니다
- 페이지 가능 메모리 접근에 주의해야 합니다

### 콜백 구현

```c
FLT_PREOP_CALLBACK_STATUS PreClose(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    UNREFERENCED_PARAMETER(Data);
    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    // Close는 차단하거나 실패시킬 수 없음
    // Cannot block or fail Close
    // 정리 작업만 수행
    // Only perform cleanup

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## C.7 정보 쿼리 (IRP_MJ_QUERY_INFORMATION)

### 개요

파일 정보를 쿼리할 때 발생합니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_QUERY_INFORMATION (0x05) |
| IRQL | PASSIVE_LEVEL |
| 페이지 가능 | Yes |

### 정보 클래스

```c
FILE_INFORMATION_CLASS infoClass =
    Data->Iopb->Parameters.QueryFileInformation.FileInformationClass;
```

| 클래스 | 설명 | 구조체 |
|--------|------|--------|
| FileBasicInformation | 기본 정보 (시간, 속성) | FILE_BASIC_INFORMATION |
| FileStandardInformation | 표준 정보 (크기, 링크 수) | FILE_STANDARD_INFORMATION |
| FileNameInformation | 파일 이름 | FILE_NAME_INFORMATION |
| FilePositionInformation | 파일 포인터 위치 | FILE_POSITION_INFORMATION |
| FileAllInformation | 모든 정보 | FILE_ALL_INFORMATION |
| FileNetworkOpenInformation | 네트워크 열기 정보 | FILE_NETWORK_OPEN_INFORMATION |

### 콜백 구현

```c
FLT_POSTOP_CALLBACK_STATUS PostQueryInformation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    FILE_INFORMATION_CLASS infoClass;
    PVOID buffer;

    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    infoClass = params->QueryFileInformation.FileInformationClass;
    buffer = params->QueryFileInformation.InfoBuffer;

    // 보호된 파일의 크기 숨기기 (예시)
    // Hide size of protected files (example)
    if (infoClass == FileStandardInformation) {
        PFILE_STANDARD_INFORMATION stdInfo =
            (PFILE_STANDARD_INFORMATION)buffer;
        // 필요시 정보 수정
        // Modify info if needed
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## C.8 정보 설정 (IRP_MJ_SET_INFORMATION)

### 개요

파일 정보를 설정(변경)할 때 발생합니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_SET_INFORMATION (0x06) |
| IRQL | PASSIVE_LEVEL |
| 페이지 가능 | Yes |

### 주요 정보 클래스

| 클래스 | 설명 | 중요 필드 |
|--------|------|----------|
| FileBasicInformation | 시간/속성 변경 | FileAttributes, CreationTime, ... |
| FileDispositionInformation | 삭제 마킹 | DeleteFile |
| FileEndOfFileInformation | 파일 크기 변경 | EndOfFile |
| FileRenameInformation | 이름 변경/이동 | FileName, RootDirectory |
| FileLinkInformation | 하드 링크 생성 | FileName, RootDirectory |
| FileValidDataLengthInformation | 유효 데이터 길이 | ValidDataLength |

### 삭제 작업 감지

```c
FLT_PREOP_CALLBACK_STATUS PreSetInformation(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    FILE_INFORMATION_CLASS infoClass;

    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    infoClass = params->SetFileInformation.FileInformationClass;

    switch (infoClass) {
    case FileDispositionInformation:
    case FileDispositionInformationEx:
        {
            PFILE_DISPOSITION_INFORMATION dispInfo =
                (PFILE_DISPOSITION_INFORMATION)
                params->SetFileInformation.InfoBuffer;

            if (dispInfo->DeleteFile) {
                // 삭제 요청 감지
                // Deletion request detected
                if (IsProtectedFile(FltObjects)) {
                    Data->IoStatus.Status = STATUS_ACCESS_DENIED;
                    return FLT_PREOP_COMPLETE;
                }
            }
        }
        break;

    case FileRenameInformation:
    case FileRenameInformationEx:
        {
            PFILE_RENAME_INFORMATION renameInfo =
                (PFILE_RENAME_INFORMATION)
                params->SetFileInformation.InfoBuffer;

            // 이름 변경/이동 감지
            // Rename/move detected
            LogRenameOperation(FltObjects, renameInfo);
        }
        break;

    case FileEndOfFileInformation:
        {
            PFILE_END_OF_FILE_INFORMATION eofInfo =
                (PFILE_END_OF_FILE_INFORMATION)
                params->SetFileInformation.InfoBuffer;

            // 파일 크기 변경 (잘라내기 포함)
            // File size change (including truncation)
            if (eofInfo->EndOfFile.QuadPart == 0) {
                // 파일 내용 삭제 시도
                // Attempt to delete file content
            }
        }
        break;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## C.9 디렉터리 제어 (IRP_MJ_DIRECTORY_CONTROL)

### 개요

디렉터리 열거 및 변경 알림에 사용됩니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_DIRECTORY_CONTROL (0x0C) |
| Minor Functions | IRP_MN_QUERY_DIRECTORY, IRP_MN_NOTIFY_CHANGE_DIRECTORY |
| IRQL | PASSIVE_LEVEL |
| 페이지 가능 | Yes |

### Minor Function

| Minor Function | 설명 |
|----------------|------|
| IRP_MN_QUERY_DIRECTORY | 디렉터리 내용 열거 |
| IRP_MN_NOTIFY_CHANGE_DIRECTORY | 변경 알림 요청 |

### 디렉터리 열거 정보 클래스

| 클래스 | 설명 |
|--------|------|
| FileBothDirectoryInformation | 표준 + 8.3 이름 |
| FileDirectoryInformation | 기본 정보 |
| FileFullDirectoryInformation | 확장 속성 포함 |
| FileIdBothDirectoryInformation | 파일 ID 포함 |
| FileNamesInformation | 이름만 |

### 콜백 구현 (파일 숨기기)

```c
FLT_POSTOP_CALLBACK_STATUS PostDirectoryControl(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    PVOID buffer;
    ULONG bufferLength;

    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 디렉터리 열거만 처리
    // Only handle directory query
    if (Data->Iopb->MinorFunction != IRP_MN_QUERY_DIRECTORY) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    buffer = params->DirectoryControl.QueryDirectory.DirectoryBuffer;
    bufferLength = (ULONG)Data->IoStatus.Information;

    if (buffer == NULL || bufferLength == 0) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 숨겨야 할 파일 필터링
    // Filter files that should be hidden
    switch (params->DirectoryControl.QueryDirectory.FileInformationClass) {
    case FileBothDirectoryInformation:
        FilterBothDirectoryInfo(buffer, &bufferLength);
        break;
    case FileIdBothDirectoryInformation:
        FilterIdBothDirectoryInfo(buffer, &bufferLength);
        break;
    }

    Data->IoStatus.Information = bufferLength;
    return FLT_POSTOP_FINISHED_PROCESSING;
}

// 파일 숨기기 구현 예시
// File hiding implementation example
VOID FilterBothDirectoryInfo(
    _Inout_ PVOID Buffer,
    _Inout_ PULONG BufferLength)
{
    PFILE_BOTH_DIR_INFORMATION current =
        (PFILE_BOTH_DIR_INFORMATION)Buffer;
    PFILE_BOTH_DIR_INFORMATION previous = NULL;
    ULONG nextOffset;

    while (TRUE) {
        UNICODE_STRING fileName;
        fileName.Length = (USHORT)current->FileNameLength;
        fileName.MaximumLength = fileName.Length;
        fileName.Buffer = current->FileName;

        // 숨길 파일인지 확인
        // Check if file should be hidden
        if (ShouldHideFile(&fileName)) {
            if (previous == NULL) {
                // 첫 번째 항목 숨기기
                // Hide first entry
                if (current->NextEntryOffset == 0) {
                    *BufferLength = 0;
                    return;
                }
                RtlMoveMemory(
                    Buffer,
                    (PUCHAR)current + current->NextEntryOffset,
                    *BufferLength - current->NextEntryOffset);
                *BufferLength -= current->NextEntryOffset;
                continue;
            } else {
                // 중간/마지막 항목 숨기기
                // Hide middle/last entry
                if (current->NextEntryOffset == 0) {
                    previous->NextEntryOffset = 0;
                } else {
                    previous->NextEntryOffset += current->NextEntryOffset;
                }
            }
        } else {
            previous = current;
        }

        nextOffset = current->NextEntryOffset;
        if (nextOffset == 0) {
            break;
        }
        current = (PFILE_BOTH_DIR_INFORMATION)
            ((PUCHAR)current + nextOffset);
    }
}
```

---

## C.10 파일 시스템 제어 (IRP_MJ_FILE_SYSTEM_CONTROL)

### 개요

FSCTL(File System Control) 작업에 사용됩니다.

| 속성 | 값 |
|------|---|
| Major Function | IRP_MJ_FILE_SYSTEM_CONTROL (0x0D) |
| Minor Functions | IRP_MN_USER_FS_REQUEST, IRP_MN_MOUNT_VOLUME, ... |
| IRQL | PASSIVE_LEVEL (대부분) |
| 페이지 가능 | 대부분 Yes |

### 주요 FSCTL 코드

| FSCTL | 설명 |
|-------|------|
| FSCTL_GET_REPARSE_POINT | 리파스 포인트 가져오기 |
| FSCTL_SET_REPARSE_POINT | 리파스 포인트 설정 |
| FSCTL_DELETE_REPARSE_POINT | 리파스 포인트 삭제 |
| FSCTL_SET_SPARSE | 희소 파일 설정 |
| FSCTL_SET_ZERO_DATA | 데이터 영역 0으로 설정 |
| FSCTL_OPLOCK_BREAK_NOTIFY | Oplock 해제 알림 |

### 콜백 구현

```c
FLT_PREOP_CALLBACK_STATUS PreFileSystemControl(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    ULONG fsctlCode;

    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    if (Data->Iopb->MinorFunction != IRP_MN_USER_FS_REQUEST) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    fsctlCode = params->FileSystemControl.Common.FsControlCode;

    switch (fsctlCode) {
    case FSCTL_SET_ZERO_DATA:
        // 데이터 삭제 시도 감지
        // Detect data deletion attempt
        LogZeroDataOperation(FltObjects, params);
        break;

    case FSCTL_SET_REPARSE_POINT:
        // 리파스 포인트 설정 감시
        // Monitor reparse point setting
        break;
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## C.11 IRP Major Function 참조표

### 전체 IRP Major Function 목록

| 코드 | 값 | 설명 | Minifilter 지원 |
|------|---|------|----------------|
| IRP_MJ_CREATE | 0x00 | 파일 열기/생성 | Yes |
| IRP_MJ_CREATE_NAMED_PIPE | 0x01 | 명명된 파이프 생성 | Yes |
| IRP_MJ_CLOSE | 0x02 | 파일 닫기 | Yes |
| IRP_MJ_READ | 0x03 | 읽기 | Yes |
| IRP_MJ_WRITE | 0x04 | 쓰기 | Yes |
| IRP_MJ_QUERY_INFORMATION | 0x05 | 정보 쿼리 | Yes |
| IRP_MJ_SET_INFORMATION | 0x06 | 정보 설정 | Yes |
| IRP_MJ_QUERY_EA | 0x07 | 확장 속성 쿼리 | Yes |
| IRP_MJ_SET_EA | 0x08 | 확장 속성 설정 | Yes |
| IRP_MJ_FLUSH_BUFFERS | 0x09 | 버퍼 플러시 | Yes |
| IRP_MJ_QUERY_VOLUME_INFORMATION | 0x0A | 볼륨 정보 쿼리 | Yes |
| IRP_MJ_SET_VOLUME_INFORMATION | 0x0B | 볼륨 정보 설정 | Yes |
| IRP_MJ_DIRECTORY_CONTROL | 0x0C | 디렉터리 제어 | Yes |
| IRP_MJ_FILE_SYSTEM_CONTROL | 0x0D | 파일 시스템 제어 | Yes |
| IRP_MJ_DEVICE_CONTROL | 0x0E | 장치 제어 | Yes |
| IRP_MJ_INTERNAL_DEVICE_CONTROL | 0x0F | 내부 장치 제어 | Yes |
| IRP_MJ_SHUTDOWN | 0x10 | 시스템 종료 | Yes |
| IRP_MJ_LOCK_CONTROL | 0x11 | 파일 잠금 | Yes |
| IRP_MJ_CLEANUP | 0x12 | 정리 | Yes |
| IRP_MJ_CREATE_MAILSLOT | 0x13 | 메일슬롯 생성 | Yes |
| IRP_MJ_QUERY_SECURITY | 0x14 | 보안 정보 쿼리 | Yes |
| IRP_MJ_SET_SECURITY | 0x15 | 보안 정보 설정 | Yes |
| IRP_MJ_POWER | 0x16 | 전원 관리 | No |
| IRP_MJ_SYSTEM_CONTROL | 0x17 | WMI | No |
| IRP_MJ_DEVICE_CHANGE | 0x18 | 장치 변경 | No |
| IRP_MJ_QUERY_QUOTA | 0x19 | 쿼터 쿼리 | Yes |
| IRP_MJ_SET_QUOTA | 0x1A | 쿼터 설정 | Yes |
| IRP_MJ_PNP | 0x1B | PnP | No |
| IRP_MJ_ACQUIRE_FOR_SECTION_SYNC | - | 섹션 동기화 획득 | Yes (FastIO) |
| IRP_MJ_RELEASE_FOR_SECTION_SYNC | - | 섹션 동기화 해제 | Yes (FastIO) |
| IRP_MJ_ACQUIRE_FOR_MOD_WRITE | - | 수정 쓰기 획득 | Yes (FastIO) |
| IRP_MJ_RELEASE_FOR_MOD_WRITE | - | 수정 쓰기 해제 | Yes (FastIO) |
| IRP_MJ_NETWORK_QUERY_OPEN | - | 네트워크 쿼리 열기 | Yes |

---

## C.12 콜백 등록 구조

### FLT_REGISTRATION에서 콜백 배열 정의

```c
const FLT_OPERATION_REGISTRATION Callbacks[] = {
    {
        IRP_MJ_CREATE,
        0,                           // 플래그
        PreCreate,                   // PreOperation
        PostCreate                   // PostOperation
    },
    {
        IRP_MJ_READ,
        0,
        PreRead,
        PostRead
    },
    {
        IRP_MJ_WRITE,
        0,
        PreWrite,
        NULL                         // PostOperation 없음
    },
    {
        IRP_MJ_SET_INFORMATION,
        0,
        PreSetInformation,
        NULL
    },
    {
        IRP_MJ_CLEANUP,
        0,
        PreCleanup,
        NULL
    },
    {
        IRP_MJ_DIRECTORY_CONTROL,
        0,
        NULL,                        // PreOperation 없음
        PostDirectoryControl
    },
    { IRP_MJ_OPERATION_END }         // 배열 종료 표시
};
```

### 플래그 옵션

| 플래그 | 설명 |
|--------|------|
| FLTFL_OPERATION_REGISTRATION_SKIP_PAGING_IO | 페이징 I/O 건너뛰기 |
| FLTFL_OPERATION_REGISTRATION_SKIP_CACHED_IO | 캐시된 I/O 건너뛰기 |
| FLTFL_OPERATION_REGISTRATION_SKIP_NON_DASD_IO | DASD 외 건너뛰기 |

---

## 요약

IRP Major Function을 이해하는 것은 Minifilter 개발의 핵심입니다:

1. **IRP_MJ_CREATE**: 파일 접근 제어의 핵심
2. **IRP_MJ_READ/WRITE**: 데이터 흐름 모니터링
3. **IRP_MJ_SET_INFORMATION**: 삭제, 이름 변경 감지
4. **IRP_MJ_CLEANUP/CLOSE**: 리소스 정리
5. **IRP_MJ_DIRECTORY_CONTROL**: 파일 숨기기, 열거 수정

각 IRP에 대한 적절한 처리와 IRQL 제한을 준수하면 안정적인 Minifilter를 개발할 수 있습니다.
