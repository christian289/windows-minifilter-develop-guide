# Chapter 21: 파일 콘텐츠 읽기

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- Minifilter에서 파일 콘텐츠를 안전하게 읽습니다
- FltReadFile과 FltReadFileEx를 사용합니다
- 버퍼 관리와 메모리 할당을 올바르게 처리합니다
- 대용량 파일을 청크 단위로 읽습니다
- 읽기 작업의 성능을 최적화합니다

## 도입: DRM을 위한 콘텐츠 분석

우리 DRM 필터의 핵심 기능은 파일 내용에서 **주민등록번호(SSN) 패턴**을 감지하는 것입니다. 이를 위해 파일 콘텐츠를 읽어야 합니다. 커널에서 파일을 읽는 것은 사용자 모드와 달리 여러 제약과 주의사항이 있습니다.

```csharp
// C# 비유: 사용자 모드에서 파일 읽기
string content = File.ReadAllText("test.txt");  // 간단!

// 커널에서는?
// - 직접 파일 API 호출 불가
// - IRQL 제약
// - 재진입 방지
// - 메모리 관리 직접 수행
```

---

## 21.1 커널에서 파일 읽기 방법

### 21.1.1 세 가지 접근 방법

```
┌─────────────────────────────────────────────────────────────────────┐
│                 커널에서 파일 읽기 방법                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. ZwReadFile (Native API)                                         │
│     ├─ 장점: 유연함, 모든 옵션 사용 가능                            │
│     ├─ 단점: 핸들 관리 필요, 복잡함                                 │
│     └─ 용도: 새 파일 열어서 읽기                                    │
│                                                                      │
│  2. FltReadFile (Filter Manager API) ★                              │
│     ├─ 장점: FILE_OBJECT 사용, 핸들 불필요                          │
│     ├─ 단점: 동기 I/O만                                             │
│     └─ 용도: 콜백에서 현재 파일 읽기                                │
│                                                                      │
│  3. 직접 버퍼 접근 (IRP_MJ_READ)                                    │
│     ├─ 장점: 이미 읽은 데이터 활용                                  │
│     ├─ 단점: PostRead에서만 가능                                    │
│     └─ 용도: 읽기 완료 후 데이터 검사                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 21.1.2 선택 가이드

```c
// 상황별 권장 방법

// 상황 1: PostCreate에서 파일 전체 검사
// → FltReadFile 사용
FLT_POSTOP_CALLBACK_STATUS PostCreate(...)
{
    // FltReadFile로 파일 읽기
}

// 상황 2: PreWrite에서 현재 쓰려는 데이터 검사
// → IRP 버퍼 직접 접근
FLT_PREOP_CALLBACK_STATUS PreWrite(...)
{
    PVOID buffer = Data->Iopb->Parameters.Write.WriteBuffer;
    // buffer 검사
}

// 상황 3: PostRead에서 읽은 데이터 검사/수정
// → IRP 버퍼 직접 접근
FLT_POSTOP_CALLBACK_STATUS PostRead(...)
{
    PVOID buffer = Data->Iopb->Parameters.Read.ReadBuffer;
    // buffer 검사 또는 수정 (예: 복호화)
}

// 상황 4: 다른 파일 열어서 읽기 (예: 설정 파일)
// → ZwReadFile 사용
NTSTATUS ReadConfigFile(...)
{
    ZwCreateFile(...);
    ZwReadFile(...);
    ZwClose(...);
}
```

---

## 21.2 FltReadFile 사용법

### 21.2.1 FltReadFile 함수 시그니처

```c
NTSTATUS FLTAPI FltReadFile(
    _In_ PFLT_INSTANCE Instance,
    _In_ PFILE_OBJECT FileObject,
    _In_opt_ PLARGE_INTEGER ByteOffset,
    _In_ ULONG Length,
    _Out_writes_bytes_(Length) PVOID Buffer,
    _In_ FLT_IO_OPERATION_FLAGS Flags,
    _Out_opt_ PULONG BytesRead,
    _In_opt_ PFLT_COMPLETED_ASYNC_IO_CALLBACK CallbackRoutine,
    _In_opt_ PVOID CallbackContext
);

// 주요 매개변수:
// Instance: 필터 인스턴스
// FileObject: 파일 객체
// ByteOffset: 읽기 시작 위치 (NULL이면 현재 위치)
// Length: 읽을 바이트 수
// Buffer: 데이터를 받을 버퍼
// Flags: FLTFL_IO_OPERATION_xxx
// BytesRead: 실제 읽은 바이트 수
```

### 21.2.2 기본 읽기 예제

```c
// PostCreate에서 파일 내용 읽기

FLT_POSTOP_CALLBACK_STATUS PostCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    NTSTATUS status;
    PVOID buffer = NULL;
    ULONG bytesRead = 0;
    LARGE_INTEGER byteOffset = { 0 };

    UNREFERENCED_PARAMETER(CompletionContext);

    // 드레이닝 또는 실패 시 건너뛰기
    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING) ||
        !NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 디렉터리 건너뛰기
    BOOLEAN isDir = FALSE;
    FltIsDirectory(FltObjects->FileObject, FltObjects->Instance, &isDir);
    if (isDir) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 읽기 접근 권한 확인
    if (!FlagOn(Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess,
                FILE_READ_DATA)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 버퍼 할당 (4KB)
    #define READ_BUFFER_SIZE 4096

    buffer = ExAllocatePoolWithTag(
        NonPagedPool,
        READ_BUFFER_SIZE,
        FM_TAG
    );

    if (buffer == NULL) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 파일 읽기
    status = FltReadFile(
        FltObjects->Instance,
        FltObjects->FileObject,
        &byteOffset,              // 파일 시작
        READ_BUFFER_SIZE,
        buffer,
        FLTFL_IO_OPERATION_NON_CACHED |   // 비캐시 읽기
        FLTFL_IO_OPERATION_DO_NOT_UPDATE_BYTE_OFFSET,  // 오프셋 유지
        &bytesRead,
        NULL,                     // 동기 호출
        NULL
    );

    if (NT_SUCCESS(status) && bytesRead > 0) {
        FmDbgPrint("Read %lu bytes from file", bytesRead);

        // 패턴 검사 수행
        if (ContainsSensitivePattern(buffer, bytesRead)) {
            FmDbgPrint("Sensitive pattern detected!");
            // 스트림 컨텍스트에 표시
        }
    } else {
        FmDbgPrint("FltReadFile failed: 0x%08X", status);
    }

    // 버퍼 해제
    ExFreePoolWithTag(buffer, FM_TAG);

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

### 21.2.3 FltReadFile 플래그

```c
// FLT_IO_OPERATION_FLAGS

// 비캐시 읽기 (캐시 우회, 직접 디스크 읽기)
FLTFL_IO_OPERATION_NON_CACHED

// 페이징 I/O
FLTFL_IO_OPERATION_PAGING

// 파일 오프셋 업데이트 안 함
FLTFL_IO_OPERATION_DO_NOT_UPDATE_BYTE_OFFSET

// 동기 페이징 I/O
FLTFL_IO_OPERATION_SYNCHRONOUS_PAGING

// 권장 조합
#define SAFE_READ_FLAGS \
    (FLTFL_IO_OPERATION_NON_CACHED | \
     FLTFL_IO_OPERATION_DO_NOT_UPDATE_BYTE_OFFSET)
```

---

## 21.3 대용량 파일 청크 읽기

### 21.3.1 청크 기반 읽기

```c
// 대용량 파일을 청크 단위로 읽기

#define CHUNK_SIZE (64 * 1024)  // 64KB 청크

NTSTATUS ReadFileInChunks(
    PFLT_INSTANCE Instance,
    PFILE_OBJECT FileObject,
    PFLT_CALLBACK_DATA Data,
    BOOLEAN *ContainsSensitiveData
)
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID buffer = NULL;
    LARGE_INTEGER fileSize = { 0 };
    LARGE_INTEGER offset = { 0 };
    ULONG bytesRead = 0;
    ULONG totalRead = 0;

    *ContainsSensitiveData = FALSE;

    // 1. 파일 크기 확인
    FILE_STANDARD_INFORMATION fileInfo;
    status = FltQueryInformationFile(
        Instance,
        FileObject,
        &fileInfo,
        sizeof(fileInfo),
        FileStandardInformation,
        NULL
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    fileSize = fileInfo.EndOfFile;

    // 너무 큰 파일은 건너뛰기 (예: 100MB 이상)
    if (fileSize.QuadPart > 100 * 1024 * 1024) {
        FmDbgPrint("File too large, skipping: %lld bytes",
            fileSize.QuadPart);
        return STATUS_FILE_TOO_LARGE;
    }

    // 2. 버퍼 할당
    buffer = ExAllocatePoolWithTag(NonPagedPool, CHUNK_SIZE, FM_TAG);
    if (buffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // 3. 청크 단위로 읽기
    while (offset.QuadPart < fileSize.QuadPart) {

        ULONG toRead = CHUNK_SIZE;

        // 마지막 청크 크기 조정
        if (fileSize.QuadPart - offset.QuadPart < CHUNK_SIZE) {
            toRead = (ULONG)(fileSize.QuadPart - offset.QuadPart);
        }

        status = FltReadFile(
            Instance,
            FileObject,
            &offset,
            toRead,
            buffer,
            FLTFL_IO_OPERATION_NON_CACHED |
            FLTFL_IO_OPERATION_DO_NOT_UPDATE_BYTE_OFFSET,
            &bytesRead,
            NULL,
            NULL
        );

        if (!NT_SUCCESS(status)) {
            FmDbgPrint("FltReadFile failed at offset %lld: 0x%08X",
                offset.QuadPart, status);
            break;
        }

        if (bytesRead == 0) {
            break;  // EOF
        }

        totalRead += bytesRead;

        // 4. 청크 분석
        if (ScanChunkForPattern(buffer, bytesRead)) {
            *ContainsSensitiveData = TRUE;
            // 조기 종료 가능
            break;
        }

        offset.QuadPart += bytesRead;
    }

    FmDbgPrint("Total read: %lu bytes", totalRead);

    // 5. 정리
    ExFreePoolWithTag(buffer, FM_TAG);

    return status;
}
```

### 21.3.2 경계 처리

패턴이 청크 경계에 걸쳐 있는 경우를 처리해야 합니다.

```c
// 청크 경계 패턴 감지

#define PATTERN_MAX_LENGTH 14  // 주민번호 최대 길이 (예: 123456-1234567)

typedef struct _CHUNK_READER {
    PVOID Buffer;
    ULONG BufferSize;
    UCHAR OverlapBuffer[PATTERN_MAX_LENGTH];
    ULONG OverlapSize;
    LARGE_INTEGER CurrentOffset;
} CHUNK_READER, *PCHUNK_READER;

NTSTATUS ReadNextChunk(
    PFLT_INSTANCE Instance,
    PFILE_OBJECT FileObject,
    PCHUNK_READER Reader,
    PULONG BytesRead
)
{
    NTSTATUS status;

    // 이전 청크 끝부분을 버퍼 앞에 복사 (오버랩)
    if (Reader->OverlapSize > 0) {
        RtlCopyMemory(
            Reader->Buffer,
            Reader->OverlapBuffer,
            Reader->OverlapSize
        );
    }

    // 새 데이터 읽기 (오버랩 뒤에)
    ULONG toRead = Reader->BufferSize - Reader->OverlapSize;
    ULONG actualRead = 0;

    status = FltReadFile(
        Instance,
        FileObject,
        &Reader->CurrentOffset,
        toRead,
        (PUCHAR)Reader->Buffer + Reader->OverlapSize,
        FLTFL_IO_OPERATION_NON_CACHED |
        FLTFL_IO_OPERATION_DO_NOT_UPDATE_BYTE_OFFSET,
        &actualRead,
        NULL,
        NULL
    );

    if (!NT_SUCCESS(status) || actualRead == 0) {
        *BytesRead = Reader->OverlapSize;  // 남은 오버랩만
        Reader->OverlapSize = 0;
        return status;
    }

    *BytesRead = Reader->OverlapSize + actualRead;

    // 다음을 위해 끝부분 저장
    if (*BytesRead >= PATTERN_MAX_LENGTH) {
        RtlCopyMemory(
            Reader->OverlapBuffer,
            (PUCHAR)Reader->Buffer + *BytesRead - PATTERN_MAX_LENGTH,
            PATTERN_MAX_LENGTH
        );
        Reader->OverlapSize = PATTERN_MAX_LENGTH;
    }

    Reader->CurrentOffset.QuadPart += actualRead;

    return status;
}
```

---

## 21.4 버퍼 접근 (IRP Read/Write)

### 21.4.1 IRP 버퍼 유형

```c
// I/O 버퍼 접근 방법 (IRP 유형에 따라 다름)

// 1. Buffered I/O
// - SystemBuffer 사용
// - 커널이 버퍼 복사

// 2. Direct I/O
// - MdlAddress 사용
// - 사용자 버퍼를 커널에 매핑

// 3. Neither I/O
// - UserBuffer 사용
// - 직접 사용자 버퍼 접근 (위험!)
```

### 21.4.2 안전한 버퍼 접근

```c
// PostRead에서 읽은 데이터 검사

FLT_POSTOP_CALLBACK_STATUS PostRead(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID CompletionContext,
    FLT_POST_OPERATION_FLAGS Flags
)
{
    PVOID buffer = NULL;
    ULONG length = 0;

    UNREFERENCED_PARAMETER(FltObjects);
    UNREFERENCED_PARAMETER(CompletionContext);

    if (FlagOn(Flags, FLTFL_POST_OPERATION_DRAINING)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 읽기 실패 시 건너뛰기
    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    length = (ULONG)Data->IoStatus.Information;
    if (length == 0) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 버퍼 획득
    buffer = GetReadBuffer(Data);

    if (buffer == NULL) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 데이터 검사
    if (ScanBufferForPattern(buffer, length)) {
        FmDbgPrint("Pattern found in read data!");
        // 필요 시 데이터 수정 (복호화 등)
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}

// 헬퍼 함수: 읽기 버퍼 획득
PVOID GetReadBuffer(PFLT_CALLBACK_DATA Data)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    PVOID buffer = NULL;

    // MDL 우선 확인
    if (params->Read.MdlAddress != NULL) {
        buffer = MmGetSystemAddressForMdlSafe(
            params->Read.MdlAddress,
            NormalPagePriority | MdlMappingNoExecute
        );
    }

    // MDL 없으면 직접 버퍼
    if (buffer == NULL) {
        buffer = params->Read.ReadBuffer;
    }

    return buffer;
}

// 헬퍼 함수: 쓰기 버퍼 획득
PVOID GetWriteBuffer(PFLT_CALLBACK_DATA Data)
{
    PFLT_PARAMETERS params = &Data->Iopb->Parameters;
    PVOID buffer = NULL;

    if (params->Write.MdlAddress != NULL) {
        buffer = MmGetSystemAddressForMdlSafe(
            params->Write.MdlAddress,
            NormalPagePriority | MdlMappingNoExecute
        );
    }

    if (buffer == NULL) {
        buffer = params->Write.WriteBuffer;
    }

    return buffer;
}
```

### 21.4.3 사용자 모드 버퍼 검증

```c
// 사용자 모드 버퍼 안전 접근

NTSTATUS SafeAccessUserBuffer(
    PFLT_CALLBACK_DATA Data,
    PVOID Buffer,
    ULONG Length,
    BOOLEAN IsWrite
)
{
    // 커널 모드 요청은 바로 접근 가능
    if (Data->RequestorMode == KernelMode) {
        return STATUS_SUCCESS;
    }

    // 사용자 모드 버퍼 검증
    __try {
        if (IsWrite) {
            ProbeForRead(Buffer, Length, sizeof(UCHAR));
        } else {
            ProbeForWrite(Buffer, Length, sizeof(UCHAR));
        }
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        return GetExceptionCode();
    }

    return STATUS_SUCCESS;
}

// 사용 예
FLT_PREOP_CALLBACK_STATUS PreWrite(...)
{
    PVOID buffer = GetWriteBuffer(Data);
    ULONG length = Data->Iopb->Parameters.Write.Length;

    if (buffer == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    NTSTATUS status = SafeAccessUserBuffer(Data, buffer, length, TRUE);
    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 이제 buffer 접근 안전
    __try {
        // 버퍼 검사
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        // 예외 처리
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

---

## 21.5 FltReadFileEx 사용

### 21.5.1 FltReadFileEx 장점

```c
// FltReadFileEx: FltReadFile의 확장 버전
// - 추가 매개변수 제공
// - 키 값 지정 가능 (암호화 등)

NTSTATUS FLTAPI FltReadFileEx(
    _In_ PFLT_INSTANCE Instance,
    _In_ PFILE_OBJECT FileObject,
    _In_opt_ PLARGE_INTEGER ByteOffset,
    _In_ ULONG Length,
    _Out_writes_bytes_(Length) PVOID Buffer,
    _In_ FLT_IO_OPERATION_FLAGS Flags,
    _Out_opt_ PULONG BytesRead,
    _In_opt_ PFLT_COMPLETED_ASYNC_IO_CALLBACK CallbackRoutine,
    _In_opt_ PVOID CallbackContext,
    _In_opt_ PULONG Key,        // 추가: 키 값
    _In_opt_ PMDL Mdl           // 추가: MDL 직접 제공
);
```

### 21.5.2 비동기 읽기

```c
// 비동기 읽기 컨텍스트
typedef struct _ASYNC_READ_CONTEXT {
    PFLT_CALLBACK_DATA OriginalData;
    PFLT_INSTANCE Instance;
    PFILE_OBJECT FileObject;
    PVOID Buffer;
    ULONG BufferSize;
    KEVENT CompletionEvent;
    NTSTATUS Status;
    ULONG BytesRead;
} ASYNC_READ_CONTEXT, *PASYNC_READ_CONTEXT;

// 비동기 완료 콜백
VOID AsyncReadComplete(
    PFLT_CALLBACK_DATA CallbackData,
    PFLT_CONTEXT Context
)
{
    PASYNC_READ_CONTEXT asyncCtx = (PASYNC_READ_CONTEXT)Context;

    UNREFERENCED_PARAMETER(CallbackData);

    // 결과 저장
    asyncCtx->Status = CallbackData->IoStatus.Status;
    asyncCtx->BytesRead = (ULONG)CallbackData->IoStatus.Information;

    // 완료 시그널
    KeSetEvent(&asyncCtx->CompletionEvent, IO_NO_INCREMENT, FALSE);
}

// 비동기 읽기 요청
NTSTATUS StartAsyncRead(
    PFLT_INSTANCE Instance,
    PFILE_OBJECT FileObject,
    PASYNC_READ_CONTEXT AsyncContext
)
{
    NTSTATUS status;
    LARGE_INTEGER offset = { 0 };

    // 이벤트 초기화
    KeInitializeEvent(&AsyncContext->CompletionEvent, NotificationEvent, FALSE);

    AsyncContext->Instance = Instance;
    AsyncContext->FileObject = FileObject;

    // 비동기 읽기 시작
    status = FltReadFile(
        Instance,
        FileObject,
        &offset,
        AsyncContext->BufferSize,
        AsyncContext->Buffer,
        FLTFL_IO_OPERATION_NON_CACHED,
        NULL,                       // BytesRead (콜백에서 확인)
        AsyncReadComplete,          // 완료 콜백
        AsyncContext                // 콜백 컨텍스트
    );

    // STATUS_PENDING은 정상 (비동기 진행 중)
    if (status == STATUS_PENDING) {
        // 완료 대기 (또는 다른 작업 수행)
        // KeWaitForSingleObject(&AsyncContext->CompletionEvent, ...);
    }

    return status;
}
```

---

## 21.6 재진입 방지

### 21.6.1 재진입 문제

```c
// FltReadFile 호출 시 우리 필터의 콜백이 다시 호출될 수 있음!

FLT_POSTOP_CALLBACK_STATUS PostCreate(...)
{
    // FltReadFile 호출
    FltReadFile(Instance, FileObject, ...);
    // ↓
    // PreRead 콜백 호출됨! (우리 필터)
    // ↓
    // 다시 FltReadFile 호출하면? 무한 루프!
}
```

### 21.6.2 재진입 감지 및 방지

```c
// 방법 1: FltIsOperationSynchronous 사용

FLT_PREOP_CALLBACK_STATUS PreRead(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext
)
{
    // 동기 작업인지 확인
    if (FltIsOperationSynchronous(Data)) {
        // 우리 FltReadFile에서 발생한 요청일 가능성
        // 또는 다른 판단 로직 추가
    }

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 방법 2: TopLevelIrp 확인

FLT_PREOP_CALLBACK_STATUS PreRead(...)
{
    // TopLevelIrp가 설정되어 있으면 재진입
    if (IoGetTopLevelIrp() != NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

// 방법 3: 명시적 플래그 사용

// 스레드 로컬 저장소 또는 컨텍스트에 플래그 저장
typedef struct _FM_STREAM_CONTEXT {
    // ...
    volatile LONG ReadInProgress;
} FM_STREAM_CONTEXT;

NTSTATUS ReadFileWithReentryProtection(
    PFLT_INSTANCE Instance,
    PFILE_OBJECT FileObject,
    PFM_STREAM_CONTEXT StreamContext
)
{
    NTSTATUS status;

    // 재진입 확인
    if (InterlockedCompareExchange(&StreamContext->ReadInProgress, 1, 0) != 0) {
        // 이미 읽기 중 - 재진입!
        return STATUS_UNSUCCESSFUL;
    }

    // 읽기 수행
    status = FltReadFile(Instance, FileObject, ...);

    // 플래그 해제
    InterlockedExchange(&StreamContext->ReadInProgress, 0);

    return status;
}

// PreRead에서 확인
FLT_PREOP_CALLBACK_STATUS PreRead(...)
{
    PFM_STREAM_CONTEXT ctx = NULL;

    status = FltGetStreamContext(..., &ctx);
    if (NT_SUCCESS(status)) {
        if (ctx->ReadInProgress) {
            // 재진입 - 통과
            FltReleaseContext(ctx);
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
        }
        FltReleaseContext(ctx);
    }

    // 정상 처리
    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

---

## 21.7 성능 최적화

### 21.7.1 불필요한 읽기 피하기

```c
// 최적화 전략

// 1. 파일 확장자로 사전 필터링
BOOLEAN ShouldScanFile(PCUNICODE_STRING FileName)
{
    static const WCHAR* ScanExtensions[] = {
        L".txt", L".doc", L".docx", L".xls", L".xlsx",
        L".pdf", L".csv", L".xml", NULL
    };

    // 확장자 추출 및 비교
    // ...
}

// 2. 파일 크기 제한
if (fileSize.QuadPart > MAX_SCAN_SIZE) {
    return FLT_POSTOP_FINISHED_PROCESSING;
}

// 3. 이미 스캔한 파일 건너뛰기 (컨텍스트 활용)
if (streamContext->IsScanned) {
    return FLT_POSTOP_FINISHED_PROCESSING;
}

// 4. 캐시 활용 (해시 기반)
ULONG fileHash = CalculateFileHash(fileData);
if (IsHashInCache(fileHash)) {
    // 이전에 분석한 파일
    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

### 21.7.2 효율적인 버퍼 관리

```c
// Lookaside List로 빈번한 할당 최적화

NPAGED_LOOKASIDE_LIST g_BufferLookaside;

// 초기화 (DriverEntry)
NTSTATUS InitializeBufferPool(void)
{
    ExInitializeNPagedLookasideList(
        &g_BufferLookaside,
        NULL,           // 할당 함수 (기본)
        NULL,           // 해제 함수 (기본)
        0,              // 플래그
        CHUNK_SIZE,     // 엔트리 크기
        FM_TAG,         // 풀 태그
        0               // 깊이 (자동)
    );

    return STATUS_SUCCESS;
}

// 버퍼 획득
PVOID AllocateReadBuffer(void)
{
    return ExAllocateFromNPagedLookasideList(&g_BufferLookaside);
}

// 버퍼 반환
VOID FreeReadBuffer(PVOID Buffer)
{
    ExFreeToNPagedLookasideList(&g_BufferLookaside, Buffer);
}

// 정리 (Unload)
VOID CleanupBufferPool(void)
{
    ExDeleteNPagedLookasideList(&g_BufferLookaside);
}
```

---

## 21.8 오류 처리

### 21.8.1 일반적인 오류

```c
NTSTATUS HandleReadError(NTSTATUS Status)
{
    switch (Status) {

    case STATUS_SUCCESS:
        return Status;

    case STATUS_END_OF_FILE:
        // 파일 끝 도달 - 정상
        return STATUS_SUCCESS;

    case STATUS_FILE_LOCK_CONFLICT:
        // 다른 프로세스가 락 보유
        FmDbgPrint("File is locked");
        return Status;

    case STATUS_ACCESS_DENIED:
        // 접근 권한 없음
        FmDbgPrint("Access denied");
        return Status;

    case STATUS_INSUFFICIENT_RESOURCES:
        // 메모리 부족
        FmDbgPrint("Insufficient resources");
        return Status;

    case STATUS_FILE_IS_A_DIRECTORY:
        // 디렉터리
        return Status;

    default:
        FmDbgPrint("Read failed: 0x%08X", Status);
        return Status;
    }
}
```

### 21.8.2 재시도 로직

```c
#define MAX_RETRY_COUNT 3
#define RETRY_DELAY_MS  100

NTSTATUS ReadFileWithRetry(
    PFLT_INSTANCE Instance,
    PFILE_OBJECT FileObject,
    PVOID Buffer,
    ULONG Length,
    PULONG BytesRead
)
{
    NTSTATUS status;
    LARGE_INTEGER offset = { 0 };
    LARGE_INTEGER delay;

    delay.QuadPart = -10000LL * RETRY_DELAY_MS;  // 100ms

    for (int retry = 0; retry < MAX_RETRY_COUNT; retry++) {

        status = FltReadFile(
            Instance,
            FileObject,
            &offset,
            Length,
            Buffer,
            FLTFL_IO_OPERATION_NON_CACHED,
            BytesRead,
            NULL,
            NULL
        );

        if (NT_SUCCESS(status)) {
            return status;
        }

        // 재시도 가능한 오류인지 확인
        if (status != STATUS_FILE_LOCK_CONFLICT &&
            status != STATUS_SHARING_VIOLATION) {
            break;  // 재시도 불가
        }

        // 대기 후 재시도
        if (retry < MAX_RETRY_COUNT - 1) {
            KeDelayExecutionThread(KernelMode, FALSE, &delay);
        }
    }

    return status;
}
```

---

## 요약

이 챕터에서 학습한 내용:

1. **읽기 방법**: FltReadFile, ZwReadFile, IRP 버퍼 직접 접근
2. **FltReadFile 사용**: 기본 호출, 플래그, 동기/비동기
3. **청크 읽기**: 대용량 파일, 경계 처리
4. **버퍼 접근**: MDL, SystemBuffer, UserBuffer
5. **재진입 방지**: FltIsOperationSynchronous, 플래그
6. **성능 최적화**: Lookaside List, 사전 필터링

다음 챕터에서는 읽은 데이터에서 패턴을 매칭하는 방법을 학습합니다.

---

## 참고 자료

- [FltReadFile](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltreadfile)
- [FltQueryInformationFile](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/nf-fltkernel-fltqueryinformationfile)
- [I/O Buffer Access](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/accessing-user-buffers)
