# 부록 A: WDK API 레퍼런스

## Minifilter 핵심 API 가이드

이 부록은 Windows Minifilter 드라이버 개발에서 가장 자주 사용되는 WDK API를 정리한 레퍼런스입니다. 각 함수에 대해 .NET 개발자가 이해하기 쉽도록 C# 비유와 함께 설명합니다.

---

## A.1 필터 등록 및 해제 API

### FltRegisterFilter

Minifilter를 필터 관리자에 등록합니다.

```c
NTSTATUS FltRegisterFilter(
    _In_  PDRIVER_OBJECT         Driver,          // 드라이버 객체
    _In_  const FLT_REGISTRATION *Registration,   // 등록 구조체
    _Out_ PFLT_FILTER            *RetFilter       // 반환될 필터 핸들
);
```

**매개변수:**
| 매개변수 | 설명 |
|----------|------|
| Driver | DriverEntry에서 받은 드라이버 객체 포인터 |
| Registration | 콜백 및 옵션을 정의한 FLT_REGISTRATION 구조체 |
| RetFilter | 성공 시 필터 핸들을 받을 포인터 |

**반환값:**
- `STATUS_SUCCESS`: 성공
- `STATUS_FLT_NOT_INITIALIZED`: 필터 관리자 초기화 안됨
- `STATUS_INVALID_PARAMETER`: 잘못된 매개변수

**C# 비유:**
```csharp
// C#에서 DI 컨테이너에 서비스 등록하는 것과 유사
// C# equivalent: Similar to registering services in DI container
services.AddSingleton<IMyFilter, MyFilterImplementation>();
```

**사용 예:**
```c
NTSTATUS status;
PFLT_FILTER gFilter;

// Minifilter 등록
// Register the minifilter
status = FltRegisterFilter(
    DriverObject,
    &FilterRegistration,
    &gFilter);

if (!NT_SUCCESS(status)) {
    // 등록 실패 처리
    // Handle registration failure
    return status;
}
```

---

### FltUnregisterFilter

등록된 Minifilter를 해제합니다.

```c
VOID FltUnregisterFilter(
    _In_ PFLT_FILTER Filter    // 해제할 필터 핸들
);
```

**매개변수:**
| 매개변수 | 설명 |
|----------|------|
| Filter | FltRegisterFilter로 얻은 필터 핸들 |

**참고사항:**
- 이 함수는 동기적으로 실행됩니다
- 모든 미완료 I/O가 완료될 때까지 대기합니다
- 드라이버 언로드 시 반드시 호출해야 합니다

**사용 예:**
```c
NTSTATUS FilterUnload(FLT_FILTER_UNLOAD_FLAGS Flags)
{
    UNREFERENCED_PARAMETER(Flags);

    // 통신 포트 닫기
    // Close communication port
    if (gServerPort != NULL) {
        FltCloseCommunicationPort(gServerPort);
    }

    // 필터 해제
    // Unregister filter
    FltUnregisterFilter(gFilter);

    return STATUS_SUCCESS;
}
```

---

### FltStartFiltering

필터링을 시작합니다.

```c
NTSTATUS FltStartFiltering(
    _In_ PFLT_FILTER Filter    // 시작할 필터 핸들
);
```

**반환값:**
- `STATUS_SUCCESS`: 성공
- `STATUS_FLT_DELETING_OBJECT`: 필터가 삭제 중

**중요:** FltRegisterFilter 후, 모든 초기화가 완료된 뒤 호출해야 합니다.

```c
// 올바른 순서
// Correct sequence
status = FltRegisterFilter(DriverObject, &Registration, &gFilter);
if (!NT_SUCCESS(status)) return status;

// 추가 초기화 수행
// Perform additional initialization
status = InitializeCommunicationPort();
if (!NT_SUCCESS(status)) {
    FltUnregisterFilter(gFilter);
    return status;
}

// 필터링 시작
// Start filtering
status = FltStartFiltering(gFilter);
if (!NT_SUCCESS(status)) {
    FltCloseCommunicationPort(gServerPort);
    FltUnregisterFilter(gFilter);
    return status;
}
```

---

## A.2 컨텍스트 관리 API

### FltAllocateContext

컨텍스트 메모리를 할당합니다.

```c
NTSTATUS FltAllocateContext(
    _In_  PFLT_FILTER     Filter,         // 필터 핸들
    _In_  FLT_CONTEXT_TYPE ContextType,   // 컨텍스트 유형
    _In_  SIZE_T          ContextSize,    // 크기 (바이트)
    _In_  POOL_TYPE       PoolType,       // 메모리 풀 유형
    _Out_ PFLT_CONTEXT    *RetContext     // 반환될 컨텍스트
);
```

**컨텍스트 유형:**
| 유형 | 설명 | 연결 대상 |
|------|------|----------|
| FLT_VOLUME_CONTEXT | 볼륨 컨텍스트 | 볼륨 (C:, D: 등) |
| FLT_INSTANCE_CONTEXT | 인스턴스 컨텍스트 | 필터 인스턴스 |
| FLT_FILE_CONTEXT | 파일 컨텍스트 | 파일 객체 |
| FLT_STREAM_CONTEXT | 스트림 컨텍스트 | 파일 스트림 |
| FLT_STREAMHANDLE_CONTEXT | 스트림 핸들 컨텍스트 | 파일 핸들 |
| FLT_TRANSACTION_CONTEXT | 트랜잭션 컨텍스트 | 트랜잭션 |

**C# 비유:**
```csharp
// C#에서 HttpContext.Items에 데이터 저장하는 것과 유사
// C# equivalent: Similar to storing data in HttpContext.Items
HttpContext.Items["FileMetadata"] = new FileMetadata();
```

**사용 예:**
```c
NTSTATUS CreateStreamContext(
    _In_  PFLT_FILTER Filter,
    _Out_ PSTREAM_CONTEXT *StreamContext)
{
    NTSTATUS status;
    PSTREAM_CONTEXT context;

    // 스트림 컨텍스트 할당
    // Allocate stream context
    status = FltAllocateContext(
        Filter,
        FLT_STREAM_CONTEXT,
        sizeof(STREAM_CONTEXT),
        NonPagedPoolNx,
        (PFLT_CONTEXT*)&context);

    if (NT_SUCCESS(status)) {
        // 초기화
        // Initialize
        RtlZeroMemory(context, sizeof(STREAM_CONTEXT));
        context->IsScanned = FALSE;
        *StreamContext = context;
    }

    return status;
}
```

---

### FltSetStreamContext / FltSetFileContext

스트림 또는 파일에 컨텍스트를 설정합니다.

```c
NTSTATUS FltSetStreamContext(
    _In_      PFLT_INSTANCE     Instance,        // 인스턴스
    _In_      PFILE_OBJECT      FileObject,      // 파일 객체
    _In_      FLT_SET_CONTEXT_OPERATION Operation, // 작업 유형
    _In_      PFLT_CONTEXT      NewContext,      // 새 컨텍스트
    _Outptr_opt_ PFLT_CONTEXT   *OldContext      // 기존 컨텍스트 (옵션)
);
```

**작업 유형:**
| 작업 | 설명 |
|------|------|
| FLT_SET_CONTEXT_REPLACE_IF_EXISTS | 기존 것이 있으면 교체 |
| FLT_SET_CONTEXT_KEEP_IF_EXISTS | 기존 것이 있으면 유지 |

**사용 예:**
```c
NTSTATUS SetOrReplaceStreamContext(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ PSTREAM_CONTEXT NewContext)
{
    NTSTATUS status;
    PSTREAM_CONTEXT oldContext = NULL;

    // 컨텍스트 설정 (기존 것이 있으면 교체)
    // Set context (replace if exists)
    status = FltSetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        FLT_SET_CONTEXT_REPLACE_IF_EXISTS,
        NewContext,
        (PFLT_CONTEXT*)&oldContext);

    if (oldContext != NULL) {
        // 이전 컨텍스트 참조 해제
        // Release reference to old context
        FltReleaseContext(oldContext);
    }

    return status;
}
```

---

### FltGetStreamContext / FltGetFileContext

스트림 또는 파일에서 컨텍스트를 가져옵니다.

```c
NTSTATUS FltGetStreamContext(
    _In_  PFLT_INSTANCE   Instance,      // 인스턴스
    _In_  PFILE_OBJECT    FileObject,    // 파일 객체
    _Out_ PFLT_CONTEXT    *Context       // 반환될 컨텍스트
);
```

**중요:** 반환된 컨텍스트는 참조 카운트가 증가되므로, 사용 후 FltReleaseContext를 호출해야 합니다.

```c
NTSTATUS GetOrCreateStreamContext(
    _In_  PFLT_CALLBACK_DATA Data,
    _In_  PCFLT_RELATED_OBJECTS FltObjects,
    _Out_ PSTREAM_CONTEXT *StreamContext,
    _Out_ PBOOLEAN IsNew)
{
    NTSTATUS status;
    PSTREAM_CONTEXT context = NULL;

    *IsNew = FALSE;

    // 기존 컨텍스트 조회
    // Try to get existing context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&context);

    if (status == STATUS_NOT_FOUND) {
        // 새 컨텍스트 생성
        // Create new context
        status = FltAllocateContext(
            FltObjects->Filter,
            FLT_STREAM_CONTEXT,
            sizeof(STREAM_CONTEXT),
            NonPagedPoolNx,
            (PFLT_CONTEXT*)&context);

        if (NT_SUCCESS(status)) {
            RtlZeroMemory(context, sizeof(STREAM_CONTEXT));

            status = FltSetStreamContext(
                FltObjects->Instance,
                FltObjects->FileObject,
                FLT_SET_CONTEXT_KEEP_IF_EXISTS,
                context,
                NULL);

            if (NT_SUCCESS(status)) {
                *IsNew = TRUE;
            }
        }
    }

    *StreamContext = context;
    return status;
}
```

---

### FltReleaseContext

컨텍스트 참조를 해제합니다.

```c
VOID FltReleaseContext(
    _In_ PFLT_CONTEXT Context    // 해제할 컨텍스트
);
```

**참조 카운팅 규칙:**
- `FltAllocateContext`: 참조 카운트 = 1
- `FltGetXxxContext`: 참조 카운트 +1
- `FltSetXxxContext`: 참조 카운트 +1 (설정에 성공한 경우)
- `FltReleaseContext`: 참조 카운트 -1

```c
// 올바른 패턴
// Correct pattern
PSTREAM_CONTEXT ctx = NULL;
status = FltGetStreamContext(Instance, FileObject, &ctx);
if (NT_SUCCESS(status)) {
    // 컨텍스트 사용
    // Use context
    DoSomethingWithContext(ctx);

    // 반드시 해제
    // Must release
    FltReleaseContext(ctx);
}
```

---

## A.3 I/O 작업 API

### FltGetFileNameInformation

파일 이름 정보를 가져옵니다.

```c
NTSTATUS FltGetFileNameInformation(
    _In_  PFLT_CALLBACK_DATA         CallbackData,  // 콜백 데이터
    _In_  FLT_FILE_NAME_OPTIONS      NameOptions,   // 이름 옵션
    _Out_ PFLT_FILE_NAME_INFORMATION *FileNameInfo  // 파일 이름 정보
);
```

**이름 옵션 플래그:**
| 옵션 | 설명 |
|------|------|
| FLT_FILE_NAME_NORMALIZED | 정규화된 이름 (심볼릭 링크 해석) |
| FLT_FILE_NAME_OPENED | 열린 이름 (원래 요청된 이름) |
| FLT_FILE_NAME_SHORT | 8.3 형식 짧은 이름 |
| FLT_FILE_NAME_QUERY_DEFAULT | 캐시 사용 |
| FLT_FILE_NAME_QUERY_CACHE_ONLY | 캐시만 사용 |
| FLT_FILE_NAME_QUERY_FILESYSTEM_ONLY | 파일 시스템만 쿼리 |

**사용 예:**
```c
NTSTATUS GetNormalizedFileName(
    _In_  PFLT_CALLBACK_DATA Data,
    _Out_ PUNICODE_STRING FileName)
{
    NTSTATUS status;
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;

    // 정규화된 파일 이름 가져오기
    // Get normalized file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 이름 파싱
    // Parse the name
    status = FltParseFileNameInformation(nameInfo);
    if (NT_SUCCESS(status)) {
        // 이름 복사 (FinalComponent = 파일명만, Name = 전체 경로)
        // Copy name (FinalComponent = filename only, Name = full path)
        RtlCopyUnicodeString(FileName, &nameInfo->Name);
    }

    // 해제
    // Release
    FltReleaseFileNameInformation(nameInfo);

    return status;
}
```

---

### FltParseFileNameInformation

파일 이름 정보를 파싱합니다.

```c
NTSTATUS FltParseFileNameInformation(
    _Inout_ PFLT_FILE_NAME_INFORMATION FileNameInfo
);
```

파싱 후 사용 가능한 필드:

| 필드 | 설명 | 예시 |
|------|------|------|
| Volume | 볼륨 이름 | `\Device\HarddiskVolume1` |
| Share | 공유 이름 (네트워크) | `\Server\Share` |
| ParentDir | 상위 디렉터리 | `\Windows\System32\` |
| FinalComponent | 파일/디렉터리 이름 | `notepad.exe` |
| Stream | 스트림 이름 | `:$DATA` |
| Extension | 확장자 | `exe` |

---

### FltReadFile / FltWriteFile

파일을 읽거나 씁니다.

```c
NTSTATUS FltReadFile(
    _In_      PFLT_INSTANCE   InitiatingInstance,  // 인스턴스
    _In_      PFILE_OBJECT    FileObject,          // 파일 객체
    _In_opt_  PLARGE_INTEGER  ByteOffset,          // 읽기 시작 오프셋
    _In_      ULONG           Length,              // 읽을 바이트 수
    _Out_     PVOID           Buffer,              // 버퍼
    _In_      FLT_IO_OPERATION_FLAGS Flags,        // 플래그
    _Out_opt_ PULONG          BytesRead,           // 실제 읽은 바이트 수
    _In_opt_  PFLT_COMPLETED_ASYNC_IO_CALLBACK CallbackRoutine, // 비동기 콜백
    _In_opt_  PVOID           CallbackContext      // 콜백 컨텍스트
);
```

**플래그:**
| 플래그 | 설명 |
|--------|------|
| FLTFL_IO_OPERATION_NON_CACHED | 캐시 우회 |
| FLTFL_IO_OPERATION_PAGING | 페이징 I/O |
| FLTFL_IO_OPERATION_DO_NOT_UPDATE_BYTE_OFFSET | 오프셋 업데이트 안함 |

**사용 예:**
```c
NTSTATUS ReadFileHeader(
    _In_  PFLT_INSTANCE Instance,
    _In_  PFILE_OBJECT FileObject,
    _Out_ PFILE_HEADER Header)
{
    NTSTATUS status;
    LARGE_INTEGER offset = {0};
    ULONG bytesRead = 0;

    // 파일 헤더 읽기 (처음 256바이트)
    // Read file header (first 256 bytes)
    status = FltReadFile(
        Instance,
        FileObject,
        &offset,                          // 오프셋 0
        sizeof(FILE_HEADER),              // 읽을 크기
        Header,                           // 버퍼
        FLTFL_IO_OPERATION_NON_CACHED,    // 캐시 우회
        &bytesRead,
        NULL,                             // 동기 호출
        NULL);

    if (NT_SUCCESS(status) && bytesRead < sizeof(FILE_HEADER)) {
        status = STATUS_END_OF_FILE;
    }

    return status;
}
```

---

### FltSetInformationFile

파일 정보를 설정합니다.

```c
NTSTATUS FltSetInformationFile(
    _In_ PFLT_INSTANCE            Instance,         // 인스턴스
    _In_ PFILE_OBJECT             FileObject,       // 파일 객체
    _In_ PVOID                    FileInformation,  // 설정할 정보
    _In_ ULONG                    Length,           // 정보 크기
    _In_ FILE_INFORMATION_CLASS   FileInformationClass // 정보 클래스
);
```

**자주 사용되는 정보 클래스:**
| 클래스 | 설명 | 구조체 |
|--------|------|--------|
| FileBasicInformation | 기본 속성 (생성/수정 시간 등) | FILE_BASIC_INFORMATION |
| FileDispositionInformation | 삭제 마킹 | FILE_DISPOSITION_INFORMATION |
| FileEndOfFileInformation | 파일 크기 | FILE_END_OF_FILE_INFORMATION |
| FileRenameInformation | 이름 변경 | FILE_RENAME_INFORMATION |

**사용 예:**
```c
NTSTATUS SetFileHidden(
    _In_ PFLT_INSTANCE Instance,
    _In_ PFILE_OBJECT FileObject)
{
    FILE_BASIC_INFORMATION basicInfo = {0};

    // 숨김 속성 설정
    // Set hidden attribute
    basicInfo.FileAttributes = FILE_ATTRIBUTE_HIDDEN;

    return FltSetInformationFile(
        Instance,
        FileObject,
        &basicInfo,
        sizeof(basicInfo),
        FileBasicInformation);
}
```

---

## A.4 통신 포트 API

### FltCreateCommunicationPort

사용자 모드와 통신할 포트를 생성합니다.

```c
NTSTATUS FltCreateCommunicationPort(
    _In_     PFLT_FILTER                Filter,          // 필터 핸들
    _Out_    PFLT_PORT                  *ServerPort,     // 서버 포트
    _In_     POBJECT_ATTRIBUTES         ObjectAttributes, // 객체 속성
    _In_opt_ PVOID                      ServerPortCookie, // 서버 쿠키
    _In_     PFLT_CONNECT_NOTIFY        ConnectNotifyCallback,    // 연결 콜백
    _In_     PFLT_DISCONNECT_NOTIFY     DisconnectNotifyCallback, // 연결 해제 콜백
    _In_opt_ PFLT_MESSAGE_NOTIFY        MessageNotifyCallback,    // 메시지 콜백
    _In_     LONG                       MaxConnections    // 최대 연결 수
);
```

**C# 비유:**
```csharp
// C#의 Named Pipe Server와 유사
// Similar to C# Named Pipe Server
var server = new NamedPipeServerStream("MyPipe", PipeDirection.InOut);
await server.WaitForConnectionAsync();
```

**사용 예:**
```c
NTSTATUS CreateCommunicationPort(_In_ PFLT_FILTER Filter)
{
    NTSTATUS status;
    UNICODE_STRING portName;
    OBJECT_ATTRIBUTES oa;
    PSECURITY_DESCRIPTOR sd;

    // 보안 설명자 생성 (관리자만 접근)
    // Create security descriptor (admin access only)
    status = FltBuildDefaultSecurityDescriptor(&sd, FLT_PORT_ALL_ACCESS);
    if (!NT_SUCCESS(status)) return status;

    RtlInitUnicodeString(&portName, L"\\DrmFilterPort");

    InitializeObjectAttributes(
        &oa,
        &portName,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        sd);

    // 통신 포트 생성
    // Create communication port
    status = FltCreateCommunicationPort(
        Filter,
        &gServerPort,
        &oa,
        NULL,                    // 서버 쿠키
        ClientConnectCallback,   // 연결 시 콜백
        ClientDisconnectCallback,// 연결 해제 시 콜백
        ClientMessageCallback,   // 메시지 수신 시 콜백
        10);                     // 최대 10개 연결

    FltFreeSecurityDescriptor(sd);

    return status;
}
```

---

### FltSendMessage

커널에서 사용자 모드로 메시지를 보냅니다.

```c
NTSTATUS FltSendMessage(
    _In_      PFLT_FILTER    Filter,           // 필터 핸들
    _In_      PFLT_PORT      *ClientPort,      // 클라이언트 포트
    _In_      PVOID          SenderBuffer,     // 보낼 데이터
    _In_      ULONG          SenderBufferLength, // 데이터 크기
    _Out_opt_ PVOID          ReplyBuffer,      // 응답 버퍼
    _Inout_opt_ PULONG       ReplyLength,      // 응답 크기
    _In_opt_  PLARGE_INTEGER Timeout           // 타임아웃
);
```

**타임아웃:**
- NULL: 무한 대기
- 0: 즉시 반환 (비차단)
- 양수: 지정된 시간(100나노초 단위)만큼 대기
- 음수: 상대 시간

```c
NTSTATUS SendBlockNotification(
    _In_ PCUNICODE_STRING FilePath,
    _In_ ULONG ProcessId)
{
    BLOCK_NOTIFICATION notification;
    BLOCK_RESPONSE response;
    ULONG responseLength = sizeof(response);
    LARGE_INTEGER timeout;

    // 5초 타임아웃
    // 5 second timeout
    timeout.QuadPart = -50000000LL;  // 100ns 단위, 음수는 상대 시간

    notification.ProcessId = ProcessId;
    RtlCopyMemory(notification.FilePath, FilePath->Buffer,
                  min(FilePath->Length, sizeof(notification.FilePath) - sizeof(WCHAR)));

    // 메시지 전송 및 응답 대기
    // Send message and wait for response
    return FltSendMessage(
        gFilter,
        &gClientPort,
        &notification,
        sizeof(notification),
        &response,
        &responseLength,
        &timeout);
}
```

---

## A.5 메모리 관리 API

### ExAllocatePool2 / ExAllocatePoolWithTag

커널 메모리를 할당합니다.

```c
// Windows 10 2004 이상 권장
// Recommended for Windows 10 2004+
PVOID ExAllocatePool2(
    _In_ POOL_FLAGS Flags,    // 풀 플래그
    _In_ SIZE_T     NumberOfBytes, // 크기
    _In_ ULONG      Tag       // 태그
);

// 레거시 (Windows 10 이전)
// Legacy (pre-Windows 10)
PVOID ExAllocatePoolWithTag(
    _In_ POOL_TYPE PoolType,  // 풀 유형
    _In_ SIZE_T    NumberOfBytes, // 크기
    _In_ ULONG     Tag        // 태그
);
```

**풀 플래그 (ExAllocatePool2):**
| 플래그 | 설명 |
|--------|------|
| POOL_FLAG_NON_PAGED | 비페이지 풀 |
| POOL_FLAG_PAGED | 페이지 풀 |
| POOL_FLAG_CACHE_ALIGNED | 캐시 라인 정렬 |
| POOL_FLAG_UNINITIALIZED | 초기화 안함 (성능 최적화) |

**C# 비유:**
```csharp
// C#에서 배열 할당과 유사하지만, 메모리 풀 선택이 필요
// Similar to C# array allocation, but requires pool selection
byte[] buffer = new byte[1024];  // 자동 관리됨
// Automatic management in C#
```

**사용 예:**
```c
// 태그 정의 (역순으로 읽으면 'DRMP' = DRM Pool)
// Tag definition (reads 'DRMP' backwards)
#define TAG_DRM_POOL 'PMRD'

PVOID AllocateNonPagedBuffer(SIZE_T Size)
{
    // NonPaged 메모리 할당 (IRQL >= DISPATCH_LEVEL에서 사용 가능)
    // Allocate NonPaged memory (usable at IRQL >= DISPATCH_LEVEL)
    return ExAllocatePool2(
        POOL_FLAG_NON_PAGED,
        Size,
        TAG_DRM_POOL);
}

PVOID AllocatePagedBuffer(SIZE_T Size)
{
    // Paged 메모리 할당 (PASSIVE_LEVEL에서만 사용)
    // Allocate Paged memory (only usable at PASSIVE_LEVEL)
    return ExAllocatePool2(
        POOL_FLAG_PAGED,
        Size,
        TAG_DRM_POOL);
}
```

---

### ExFreePoolWithTag

할당된 메모리를 해제합니다.

```c
VOID ExFreePoolWithTag(
    _In_ PVOID P,      // 해제할 포인터
    _In_ ULONG Tag     // 할당 시 사용한 태그
);
```

```c
VOID FreeBuffer(_In_ PVOID Buffer)
{
    if (Buffer != NULL) {
        ExFreePoolWithTag(Buffer, TAG_DRM_POOL);
    }
}
```

---

### ExInitializeNPagedLookasideList / ExAllocateFromNPagedLookasideList

Lookaside List를 초기화하고 사용합니다 (성능 최적화용).

```c
VOID ExInitializeNPagedLookasideList(
    _Out_    PNPAGED_LOOKASIDE_LIST Lookaside, // Lookaside 구조체
    _In_opt_ PALLOCATE_FUNCTION     Allocate,  // 커스텀 할당 함수
    _In_opt_ PFREE_FUNCTION         Free,      // 커스텀 해제 함수
    _In_     ULONG                  Flags,     // 플래그
    _In_     SIZE_T                 Size,      // 요소 크기
    _In_     ULONG                  Tag,       // 태그
    _In_     USHORT                 Depth      // 깊이 (0 = 시스템 결정)
);

PVOID ExAllocateFromNPagedLookasideList(
    _Inout_ PNPAGED_LOOKASIDE_LIST Lookaside
);

VOID ExFreeToNPagedLookasideList(
    _Inout_ PNPAGED_LOOKASIDE_LIST Lookaside,
    _In_    PVOID                  Entry
);
```

**C# 비유:**
```csharp
// C#의 ObjectPool과 유사
// Similar to C# ObjectPool
var pool = new ObjectPool<MyObject>(() => new MyObject());
var obj = pool.Get();
pool.Return(obj);
```

**사용 예:**
```c
NPAGED_LOOKASIDE_LIST gContextLookaside;

NTSTATUS InitializeLookaside()
{
    // Lookaside List 초기화
    // Initialize Lookaside List
    ExInitializeNPagedLookasideList(
        &gContextLookaside,
        NULL,                    // 기본 할당 함수
        NULL,                    // 기본 해제 함수
        0,                       // 플래그
        sizeof(MY_CONTEXT),      // 요소 크기
        TAG_DRM_POOL,
        0);                      // 시스템이 깊이 결정

    return STATUS_SUCCESS;
}

PVOID AllocateContext()
{
    return ExAllocateFromNPagedLookasideList(&gContextLookaside);
}

VOID FreeContext(_In_ PVOID Context)
{
    ExFreeToNPagedLookasideList(&gContextLookaside, Context);
}

VOID CleanupLookaside()
{
    ExDeleteNPagedLookasideList(&gContextLookaside);
}
```

---

## A.6 동기화 API

### KeInitializeSpinLock / KeAcquireSpinLock / KeReleaseSpinLock

스핀 락을 사용합니다.

```c
VOID KeInitializeSpinLock(_Out_ PKSPIN_LOCK SpinLock);

VOID KeAcquireSpinLock(
    _Inout_ PKSPIN_LOCK SpinLock,
    _Out_   PKIRQL      OldIrql
);

VOID KeReleaseSpinLock(
    _Inout_ PKSPIN_LOCK SpinLock,
    _In_    KIRQL       NewIrql
);
```

**C# 비유:**
```csharp
// C#의 lock과 유사하지만 더 저수준
// Similar to C# lock but lower level
lock (syncObject)
{
    // critical section
}
```

**사용 예:**
```c
typedef struct _PROTECTED_LIST {
    KSPIN_LOCK Lock;
    LIST_ENTRY Head;
    ULONG Count;
} PROTECTED_LIST, *PPROTECTED_LIST;

VOID InitializeProtectedList(_Out_ PPROTECTED_LIST List)
{
    KeInitializeSpinLock(&List->Lock);
    InitializeListHead(&List->Head);
    List->Count = 0;
}

VOID AddToProtectedList(
    _Inout_ PPROTECTED_LIST List,
    _In_    PLIST_ENTRY     Entry)
{
    KIRQL oldIrql;

    // 스핀 락 획득 (IRQL이 DISPATCH_LEVEL로 상승)
    // Acquire spin lock (IRQL raised to DISPATCH_LEVEL)
    KeAcquireSpinLock(&List->Lock, &oldIrql);

    InsertTailList(&List->Head, Entry);
    List->Count++;

    // 스핀 락 해제
    // Release spin lock
    KeReleaseSpinLock(&List->Lock, oldIrql);
}
```

---

### ExInitializeResourceLite / ExAcquireResourceExclusiveLite / ExAcquireResourceSharedLite

ERESOURCE (Reader-Writer 락)를 사용합니다.

```c
NTSTATUS ExInitializeResourceLite(_Out_ PERESOURCE Resource);

BOOLEAN ExAcquireResourceExclusiveLite(
    _Inout_ PERESOURCE Resource,
    _In_    BOOLEAN    Wait
);

BOOLEAN ExAcquireResourceSharedLite(
    _Inout_ PERESOURCE Resource,
    _In_    BOOLEAN    Wait
);

VOID ExReleaseResourceLite(_Inout_ PERESOURCE Resource);

NTSTATUS ExDeleteResourceLite(_Inout_ PERESOURCE Resource);
```

**C# 비유:**
```csharp
// C#의 ReaderWriterLockSlim과 동일한 개념
// Same concept as C# ReaderWriterLockSlim
var rwLock = new ReaderWriterLockSlim();

rwLock.EnterReadLock();
// 읽기 작업
rwLock.ExitReadLock();

rwLock.EnterWriteLock();
// 쓰기 작업
rwLock.ExitWriteLock();
```

**사용 예:**
```c
typedef struct _POLICY_CACHE {
    ERESOURCE Lock;
    PVOID Data;
    ULONG DataSize;
} POLICY_CACHE, *PPOLICY_CACHE;

NTSTATUS InitializePolicyCache(_Out_ PPOLICY_CACHE Cache)
{
    RtlZeroMemory(Cache, sizeof(POLICY_CACHE));
    return ExInitializeResourceLite(&Cache->Lock);
}

PVOID ReadPolicyCache(_In_ PPOLICY_CACHE Cache)
{
    PVOID data = NULL;

    // 공유(읽기) 락 획득
    // Acquire shared (read) lock
    ExAcquireResourceSharedLite(&Cache->Lock, TRUE);

    data = Cache->Data;

    ExReleaseResourceLite(&Cache->Lock);

    return data;
}

VOID UpdatePolicyCache(
    _Inout_ PPOLICY_CACHE Cache,
    _In_    PVOID         NewData,
    _In_    ULONG         Size)
{
    // 배타적(쓰기) 락 획득
    // Acquire exclusive (write) lock
    ExAcquireResourceExclusiveLite(&Cache->Lock, TRUE);

    if (Cache->Data != NULL) {
        ExFreePoolWithTag(Cache->Data, TAG_DRM_POOL);
    }

    Cache->Data = NewData;
    Cache->DataSize = Size;

    ExReleaseResourceLite(&Cache->Lock);
}
```

---

## A.7 유틸리티 API

### RtlInitUnicodeString / RtlCopyUnicodeString

유니코드 문자열을 초기화하고 복사합니다.

```c
VOID RtlInitUnicodeString(
    _Out_    PUNICODE_STRING DestinationString,
    _In_opt_ PCWSTR          SourceString
);

VOID RtlCopyUnicodeString(
    _Out_ PUNICODE_STRING  DestinationString,
    _In_  PCUNICODE_STRING SourceString
);
```

**UNICODE_STRING 구조체:**
```c
typedef struct _UNICODE_STRING {
    USHORT Length;        // 바이트 단위 현재 길이 (NULL 제외)
    USHORT MaximumLength; // 바이트 단위 최대 크기
    PWCH   Buffer;        // 문자열 버퍼
} UNICODE_STRING;
```

**C# 비유:**
```csharp
// C#의 string과 다르게, 버퍼 관리를 직접 해야 함
// Unlike C# string, buffer management is manual
string str = "Hello";  // C# - 자동 관리
// C# - automatic management

// C에서는:
UNICODE_STRING str;
WCHAR buffer[256];
str.Buffer = buffer;
str.MaximumLength = sizeof(buffer);
RtlCopyUnicodeString(&str, &sourceStr);
```

---

### RtlCompareUnicodeString

유니코드 문자열을 비교합니다.

```c
LONG RtlCompareUnicodeString(
    _In_ PCUNICODE_STRING String1,
    _In_ PCUNICODE_STRING String2,
    _In_ BOOLEAN          CaseInSensitive  // TRUE = 대소문자 무시
);
```

**반환값:**
- 음수: String1 < String2
- 0: String1 == String2
- 양수: String1 > String2

---

### FsRtlIsNameInExpression

와일드카드를 사용하여 파일 이름을 매칭합니다.

```c
BOOLEAN FsRtlIsNameInExpression(
    _In_     PCUNICODE_STRING Expression,      // 패턴 (와일드카드 포함)
    _In_     PCUNICODE_STRING Name,            // 대상 이름
    _In_     BOOLEAN          IgnoreCase,      // 대소문자 무시
    _In_opt_ PWCH             UpcaseTable      // 대문자 변환 테이블
);
```

**와일드카드:**
- `*`: 0개 이상의 모든 문자
- `?`: 정확히 1개의 문자
- `DOS_STAR (<)`: DOS 스타일 * (확장자 전까지)
- `DOS_QM (>)`: DOS 스타일 ?
- `DOS_DOT (")`: DOS 스타일 .

**사용 예:**
```c
BOOLEAN IsProtectedFile(_In_ PCUNICODE_STRING FileName)
{
    UNICODE_STRING pattern;

    // .docx 파일 확인
    // Check for .docx files
    RtlInitUnicodeString(&pattern, L"*.DOCX");

    if (FsRtlIsNameInExpression(&pattern, FileName, TRUE, NULL)) {
        return TRUE;
    }

    // .xlsx 파일 확인
    // Check for .xlsx files
    RtlInitUnicodeString(&pattern, L"*.XLSX");

    return FsRtlIsNameInExpression(&pattern, FileName, TRUE, NULL);
}
```

---

## A.8 레지스트리 API

### ZwOpenKey / ZwQueryValueKey / ZwSetValueKey

레지스트리를 읽고 씁니다.

```c
NTSTATUS ZwOpenKey(
    _Out_ PHANDLE            KeyHandle,
    _In_  ACCESS_MASK        DesiredAccess,
    _In_  POBJECT_ATTRIBUTES ObjectAttributes
);

NTSTATUS ZwQueryValueKey(
    _In_      HANDLE                      KeyHandle,
    _In_      PUNICODE_STRING             ValueName,
    _In_      KEY_VALUE_INFORMATION_CLASS KeyValueInformationClass,
    _Out_opt_ PVOID                       KeyValueInformation,
    _In_      ULONG                       Length,
    _Out_     PULONG                      ResultLength
);

NTSTATUS ZwSetValueKey(
    _In_     HANDLE          KeyHandle,
    _In_     PUNICODE_STRING ValueName,
    _In_opt_ ULONG           TitleIndex,
    _In_     ULONG           Type,
    _In_opt_ PVOID           Data,
    _In_     ULONG           DataSize
);
```

**사용 예:**
```c
NTSTATUS ReadDwordFromRegistry(
    _In_  PCWSTR  KeyPath,
    _In_  PCWSTR  ValueName,
    _Out_ PULONG  Value)
{
    NTSTATUS status;
    HANDLE keyHandle = NULL;
    UNICODE_STRING keyName, valueName;
    OBJECT_ATTRIBUTES oa;
    ULONG resultLength;
    UCHAR buffer[sizeof(KEY_VALUE_PARTIAL_INFORMATION) + sizeof(ULONG)];
    PKEY_VALUE_PARTIAL_INFORMATION info = (PKEY_VALUE_PARTIAL_INFORMATION)buffer;

    RtlInitUnicodeString(&keyName, KeyPath);
    InitializeObjectAttributes(&oa, &keyName,
        OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    // 키 열기
    // Open key
    status = ZwOpenKey(&keyHandle, KEY_READ, &oa);
    if (!NT_SUCCESS(status)) return status;

    RtlInitUnicodeString(&valueName, ValueName);

    // 값 읽기
    // Read value
    status = ZwQueryValueKey(
        keyHandle,
        &valueName,
        KeyValuePartialInformation,
        info,
        sizeof(buffer),
        &resultLength);

    if (NT_SUCCESS(status) && info->Type == REG_DWORD) {
        *Value = *(PULONG)info->Data;
    }

    ZwClose(keyHandle);
    return status;
}
```

---

## A.9 프로세스/스레드 API

### PsGetCurrentProcessId / PsGetCurrentThreadId

현재 프로세스/스레드 ID를 가져옵니다.

```c
HANDLE PsGetCurrentProcessId(VOID);
HANDLE PsGetCurrentThreadId(VOID);
```

---

### PsLookupProcessByProcessId

프로세스 ID로 EPROCESS를 가져옵니다.

```c
NTSTATUS PsLookupProcessByProcessId(
    _In_  HANDLE    ProcessId,
    _Out_ PEPROCESS *Process
);
```

**중요:** 반환된 EPROCESS는 `ObDereferenceObject`로 해제해야 합니다.

```c
NTSTATUS GetProcessImageName(
    _In_  HANDLE ProcessId,
    _Out_ PUNICODE_STRING ImageName)
{
    NTSTATUS status;
    PEPROCESS process;

    // 프로세스 객체 조회
    // Look up process object
    status = PsLookupProcessByProcessId(ProcessId, &process);
    if (!NT_SUCCESS(status)) return status;

    // 이미지 파일 이름 가져오기
    // Get image file name
    status = SeLocateProcessImageName(process, ImageName);

    // 참조 해제
    // Dereference
    ObDereferenceObject(process);

    return status;
}
```

---

### KeGetCurrentIrql

현재 IRQL을 가져옵니다.

```c
KIRQL KeGetCurrentIrql(VOID);
```

**IRQL 레벨:**
| 레벨 | 값 | 설명 |
|------|---|------|
| PASSIVE_LEVEL | 0 | 일반 스레드 코드 |
| APC_LEVEL | 1 | APC 실행 중 |
| DISPATCH_LEVEL | 2 | DPC, 스핀 락 |
| DIRQL | 3-31 | 인터럽트 서비스 |

```c
// IRQL 확인 후 적절한 메모리 풀 사용
// Check IRQL and use appropriate memory pool
PVOID SafeAllocate(SIZE_T Size)
{
    if (KeGetCurrentIrql() > APC_LEVEL) {
        // DISPATCH_LEVEL 이상에서는 NonPaged만 사용 가능
        // Only NonPaged can be used at DISPATCH_LEVEL or above
        return ExAllocatePool2(POOL_FLAG_NON_PAGED, Size, TAG_DRM_POOL);
    } else {
        // PASSIVE/APC_LEVEL에서는 Paged 사용 가능
        // Paged can be used at PASSIVE/APC_LEVEL
        return ExAllocatePool2(POOL_FLAG_PAGED, Size, TAG_DRM_POOL);
    }
}
```

---

## A.10 API 참조 표

### 자주 사용되는 상태 코드

| 코드 | 값 | 설명 |
|------|---|------|
| STATUS_SUCCESS | 0x00000000 | 성공 |
| STATUS_UNSUCCESSFUL | 0xC0000001 | 일반 실패 |
| STATUS_NOT_FOUND | 0xC0000225 | 찾을 수 없음 |
| STATUS_ACCESS_DENIED | 0xC0000022 | 접근 거부 |
| STATUS_BUFFER_TOO_SMALL | 0xC0000023 | 버퍼 부족 |
| STATUS_INSUFFICIENT_RESOURCES | 0xC000009A | 리소스 부족 |
| STATUS_INVALID_PARAMETER | 0xC000000D | 잘못된 매개변수 |

### I/O 상태 매크로

```c
// 성공 확인
// Check for success
if (NT_SUCCESS(status)) { ... }

// 오류 확인
// Check for error
if (!NT_SUCCESS(status)) { ... }

// 정보/경고 확인
// Check for information/warning
if (NT_INFORMATION(status)) { ... }
if (NT_WARNING(status)) { ... }
```

---

## 요약

이 부록에서 다룬 WDK API는 Minifilter 개발의 핵심입니다:

1. **필터 등록**: FltRegisterFilter, FltStartFiltering, FltUnregisterFilter
2. **컨텍스트**: FltAllocateContext, FltSetStreamContext, FltGetStreamContext
3. **I/O 작업**: FltGetFileNameInformation, FltReadFile, FltWriteFile
4. **통신**: FltCreateCommunicationPort, FltSendMessage
5. **메모리**: ExAllocatePool2, ExFreePoolWithTag, Lookaside List
6. **동기화**: 스핀 락, ERESOURCE

각 API를 사용할 때는 IRQL 제한과 메모리 풀 선택에 주의하세요.
