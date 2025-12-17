# 부록 B: NTSTATUS 코드 레퍼런스

## Windows 커널 상태 코드 완벽 가이드

이 부록은 Minifilter 드라이버 개발에서 자주 접하는 NTSTATUS 코드를 정리한 레퍼런스입니다. 각 코드의 의미와 적절한 처리 방법을 설명합니다.

---

## B.1 NTSTATUS 구조 이해

### 상태 코드 형식

NTSTATUS는 32비트 값으로, 다음과 같은 구조를 가집니다:

```
 31 30 29 28                               0
┌──┬──┬──┬─────────────────────────────────┐
│S │C │R │       상태 코드 값               │
└──┴──┴──┴─────────────────────────────────┘
```

| 비트 | 이름 | 설명 |
|------|------|------|
| 31-30 | Severity | 심각도 (00=성공, 01=정보, 10=경고, 11=오류) |
| 29 | Customer | 0=Microsoft 정의, 1=사용자 정의 |
| 28 | Reserved | 예약됨 |
| 27-16 | Facility | 기능 영역 코드 |
| 15-0 | Code | 상태 코드 |

### 심각도 분류

| 값 | 심각도 | 의미 | 매크로 |
|----|--------|------|--------|
| 0x00000000 | Success | 작업 성공 | NT_SUCCESS(Status) |
| 0x40000000 | Informational | 정보성 (성공에 가까움) | NT_INFORMATION(Status) |
| 0x80000000 | Warning | 경고 (부분 성공) | NT_WARNING(Status) |
| 0xC0000000 | Error | 오류 (실패) | !NT_SUCCESS(Status) |

### C# 비유

```csharp
// C#에서 예외 처리와 유사한 개념
// Similar concept to C# exception handling

// C#
try {
    result = SomeOperation();
}
catch (FileNotFoundException) { ... }
catch (UnauthorizedAccessException) { ... }

// C (NTSTATUS)
status = SomeOperation();
if (status == STATUS_OBJECT_NAME_NOT_FOUND) { ... }
else if (status == STATUS_ACCESS_DENIED) { ... }
```

---

## B.2 성공 코드 (Success)

### STATUS_SUCCESS (0x00000000)
**설명:** 작업이 성공적으로 완료됨

```c
NTSTATUS status = FltRegisterFilter(DriverObject, &Reg, &Filter);
if (NT_SUCCESS(status)) {
    // 필터 등록 성공
    // Filter registration succeeded
}
```

### STATUS_PENDING (0x00000103)
**설명:** 작업이 비동기로 진행 중

```c
// I/O 요청을 비동기로 완료
// Complete I/O request asynchronously
Data->IoStatus.Status = STATUS_PENDING;
return FLT_PREOP_PENDING;
```

### STATUS_REPARSE (0x00000104)
**설명:** 경로를 다시 파싱해야 함 (심볼릭 링크, 마운트 포인트)

```c
// 리파스 포인트 처리
// Handle reparse point
if (status == STATUS_REPARSE) {
    // 새 경로로 재시도
    // Retry with new path
}
```

### STATUS_MORE_ENTRIES (0x00000105)
**설명:** 더 많은 데이터가 있음 (열거 작업에서)

```c
// 디렉터리 열거
// Directory enumeration
while (status == STATUS_MORE_ENTRIES) {
    status = FltGetNextEntry(&entries);
}
```

---

## B.3 정보 코드 (Informational)

### STATUS_BUFFER_OVERFLOW (0x80000005)
**설명:** 버퍼 크기 부족, 데이터가 잘림

```c
NTSTATUS GetFileName(PFLT_CALLBACK_DATA Data, PUNICODE_STRING FileName)
{
    UCHAR buffer[256];
    ULONG returnedLength;
    NTSTATUS status;

    status = FltQueryInformationFile(
        Data->Iopb->TargetInstance,
        Data->Iopb->TargetFileObject,
        buffer,
        sizeof(buffer),
        FileNameInformation,
        &returnedLength);

    if (status == STATUS_BUFFER_OVERFLOW) {
        // 더 큰 버퍼로 재시도
        // Retry with larger buffer
        PVOID largerBuffer = ExAllocatePool2(
            POOL_FLAG_PAGED, returnedLength, TAG_POOL);
        // ...
    }

    return status;
}
```

### STATUS_NO_MORE_FILES (0x80000006)
**설명:** 디렉터리 열거 완료 (더 이상 파일 없음)

```c
// IRP_MJ_DIRECTORY_CONTROL 처리
// Handle IRP_MJ_DIRECTORY_CONTROL
if (status == STATUS_NO_MORE_FILES) {
    // 열거 완료 - 정상 상황
    // Enumeration complete - normal condition
    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

---

## B.4 경고 코드 (Warning)

### STATUS_DATATYPE_MISALIGNMENT (0x80000002)
**설명:** 데이터 정렬 오류

```c
// 구조체 정렬 확인
// Check structure alignment
#pragma pack(push, 8)
typedef struct _ALIGNED_STRUCT {
    ULONG64 Value;  // 8바이트 정렬 필요
} ALIGNED_STRUCT;
#pragma pack(pop)
```

### STATUS_DEVICE_PAPER_EMPTY (0x8000000E)
**설명:** 프린터 용지 없음 (프린터 드라이버)

---

## B.5 일반 오류 코드 (Error - Common)

### STATUS_UNSUCCESSFUL (0xC0000001)
**설명:** 일반적인 실패

```c
// 특정 오류 코드를 사용할 수 없을 때
// When specific error code is not available
if (operationFailed) {
    return STATUS_UNSUCCESSFUL;
}
```

### STATUS_NOT_IMPLEMENTED (0xC0000002)
**설명:** 요청된 기능이 구현되지 않음

```c
FLT_PREOP_CALLBACK_STATUS PreNotImplemented(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext)
{
    // 지원하지 않는 작업
    // Unsupported operation
    Data->IoStatus.Status = STATUS_NOT_IMPLEMENTED;
    return FLT_PREOP_COMPLETE;
}
```

### STATUS_INVALID_PARAMETER (0xC000000D)
**설명:** 잘못된 매개변수

```c
// 매개변수 검증
// Parameter validation
if (Buffer == NULL || BufferSize == 0) {
    return STATUS_INVALID_PARAMETER;
}
```

### STATUS_NO_SUCH_FILE (0xC000000F)
**설명:** 파일이 존재하지 않음

### STATUS_NO_SUCH_DEVICE (0xC000000E)
**설명:** 장치가 존재하지 않음

### STATUS_INVALID_DEVICE_REQUEST (0xC0000010)
**설명:** 잘못된 장치 요청

### STATUS_END_OF_FILE (0xC0000011)
**설명:** 파일 끝에 도달

```c
// 파일 읽기 시 EOF 처리
// Handle EOF when reading file
NTSTATUS ReadEntireFile(...)
{
    status = FltReadFile(...);
    if (status == STATUS_END_OF_FILE) {
        // 파일 끝 - 정상 종료
        // End of file - normal termination
        break;
    }
}
```

### STATUS_NO_MEMORY (0xC0000017)
**설명:** 메모리 부족

```c
PVOID buffer = ExAllocatePool2(POOL_FLAG_NON_PAGED, size, TAG_POOL);
if (buffer == NULL) {
    return STATUS_NO_MEMORY;
}
```

---

## B.6 접근 관련 오류 코드

### STATUS_ACCESS_DENIED (0xC0000022)
**설명:** 접근이 거부됨

```c
FLT_PREOP_CALLBACK_STATUS PreCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext)
{
    // DRM 정책에 따라 접근 차단
    // Block access according to DRM policy
    if (IsProtectedFile(fileName) && !IsAuthorizedProcess(processId)) {
        Data->IoStatus.Status = STATUS_ACCESS_DENIED;
        Data->IoStatus.Information = 0;
        return FLT_PREOP_COMPLETE;
    }

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

### STATUS_ACCESS_VIOLATION (0xC0000005)
**설명:** 메모리 접근 위반 (잘못된 포인터)

```c
__try {
    // 사용자 버퍼 접근
    // Access user buffer
    ProbeForRead(UserBuffer, BufferSize, sizeof(UCHAR));
    RtlCopyMemory(KernelBuffer, UserBuffer, BufferSize);
}
__except (EXCEPTION_EXECUTE_HANDLER) {
    return STATUS_ACCESS_VIOLATION;
}
```

### STATUS_SHARING_VIOLATION (0xC0000043)
**설명:** 공유 위반 (파일이 다른 프로세스에 의해 잠김)

```c
// 파일 열기 실패 시 재시도 로직
// Retry logic on file open failure
if (status == STATUS_SHARING_VIOLATION) {
    // 잠시 대기 후 재시도
    // Wait briefly and retry
    LARGE_INTEGER delay;
    delay.QuadPart = -10000000LL;  // 1초
    KeDelayExecutionThread(KernelMode, FALSE, &delay);
    // 재시도...
}
```

### STATUS_FILE_LOCK_CONFLICT (0xC0000054)
**설명:** 파일 잠금 충돌

### STATUS_LOCK_NOT_GRANTED (0xC0000055)
**설명:** 잠금을 획득할 수 없음

---

## B.7 파일/디렉터리 오류 코드

### STATUS_OBJECT_NAME_NOT_FOUND (0xC0000034)
**설명:** 객체(파일/디렉터리) 이름을 찾을 수 없음

```c
// 파일 존재 여부 확인
// Check file existence
status = FltQueryInformationFile(...);
if (status == STATUS_OBJECT_NAME_NOT_FOUND) {
    // 파일이 존재하지 않음
    // File does not exist
}
```

### STATUS_OBJECT_NAME_COLLISION (0xC0000035)
**설명:** 이름 충돌 (같은 이름의 파일이 이미 존재)

### STATUS_OBJECT_NAME_INVALID (0xC0000033)
**설명:** 잘못된 객체 이름

### STATUS_OBJECT_PATH_NOT_FOUND (0xC000003A)
**설명:** 경로를 찾을 수 없음

```c
// 경로 vs 파일 오류 구분
// Distinguish path vs file error
if (status == STATUS_OBJECT_PATH_NOT_FOUND) {
    // 중간 디렉터리가 없음 (예: C:\Foo\Bar에서 C:\Foo가 없음)
    // Intermediate directory missing (e.g., C:\Foo missing in C:\Foo\Bar)
} else if (status == STATUS_OBJECT_NAME_NOT_FOUND) {
    // 최종 파일/디렉터리가 없음
    // Final file/directory missing
}
```

### STATUS_OBJECT_PATH_INVALID (0xC0000039)
**설명:** 잘못된 경로

### STATUS_OBJECT_PATH_SYNTAX_BAD (0xC000003B)
**설명:** 경로 구문 오류

### STATUS_DELETE_PENDING (0xC0000056)
**설명:** 파일 삭제 대기 중

```c
// 삭제 대기 중인 파일 처리
// Handle file pending deletion
if (status == STATUS_DELETE_PENDING) {
    // 파일에 접근할 수 없음 - 삭제 예정
    // Cannot access file - deletion pending
    return FLT_PREOP_COMPLETE;
}
```

### STATUS_DIRECTORY_NOT_EMPTY (0xC0000101)
**설명:** 디렉터리가 비어있지 않음 (삭제 불가)

### STATUS_NOT_A_DIRECTORY (0xC0000103)
**설명:** 디렉터리가 아님

### STATUS_FILE_IS_A_DIRECTORY (0xC00000BA)
**설명:** 파일이 아닌 디렉터리임

### STATUS_FILE_RENAMED (0xC00000D5)
**설명:** 파일이 이름 변경됨

### STATUS_FILE_DELETED (0xC0000123)
**설명:** 파일이 삭제됨

---

## B.8 버퍼/크기 오류 코드

### STATUS_BUFFER_TOO_SMALL (0xC0000023)
**설명:** 버퍼 크기가 너무 작음

```c
// 필요한 버퍼 크기 확인 패턴
// Pattern to check required buffer size
ULONG requiredSize = 0;
status = FltGetFileNameInformation(Data, Options, &nameInfo);

if (status == STATUS_BUFFER_TOO_SMALL) {
    // 더 큰 버퍼 할당
    // Allocate larger buffer
}
```

### STATUS_INFO_LENGTH_MISMATCH (0xC0000004)
**설명:** 정보 길이가 맞지 않음

```c
// 구조체 크기 검증
// Validate structure size
if (InputBufferLength < sizeof(MY_INPUT_STRUCT)) {
    return STATUS_INFO_LENGTH_MISMATCH;
}
```

### STATUS_INTEGER_OVERFLOW (0xC0000095)
**설명:** 정수 오버플로우

```c
// 안전한 크기 계산
// Safe size calculation
ULONG newSize;
status = RtlULongAdd(existingSize, additionalSize, &newSize);
if (status == STATUS_INTEGER_OVERFLOW) {
    return status;
}
```

---

## B.9 FLT 관련 오류 코드

### STATUS_FLT_NO_HANDLER_DEFINED (0xC01C0001)
**설명:** 핸들러가 정의되지 않음

### STATUS_FLT_CONTEXT_ALREADY_DEFINED (0xC01C0002)
**설명:** 컨텍스트가 이미 정의됨

```c
// 컨텍스트 설정 시 기존 것 유지
// Keep existing context when setting
status = FltSetStreamContext(
    Instance,
    FileObject,
    FLT_SET_CONTEXT_KEEP_IF_EXISTS,  // 이미 있으면 유지
    NewContext,
    &ExistingContext);

if (status == STATUS_FLT_CONTEXT_ALREADY_DEFINED) {
    // 기존 컨텍스트 사용
    // Use existing context
    FltReleaseContext(NewContext);
}
```

### STATUS_FLT_INVALID_CONTEXT_REGISTRATION (0xC01C0003)
**설명:** 잘못된 컨텍스트 등록

### STATUS_FLT_NAME_CACHE_MISS (0xC01C0004)
**설명:** 이름 캐시 미스

```c
// 파일시스템 직접 쿼리
// Query filesystem directly
status = FltGetFileNameInformation(
    Data,
    FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_FILESYSTEM_ONLY,
    &nameInfo);
```

### STATUS_FLT_NO_DEVICE_OBJECT (0xC01C0005)
**설명:** 장치 객체 없음

### STATUS_FLT_VOLUME_NOT_FOUND (0xC01C0006)
**설명:** 볼륨을 찾을 수 없음

### STATUS_FLT_INSTANCE_NOT_FOUND (0xC01C0007)
**설명:** 인스턴스를 찾을 수 없음

### STATUS_FLT_FILTER_NOT_FOUND (0xC01C0008)
**설명:** 필터를 찾을 수 없음

### STATUS_FLT_VOLUME_ALREADY_MOUNTED (0xC01C0009)
**설명:** 볼륨이 이미 마운트됨

### STATUS_FLT_INSTANCE_ALTITUDE_COLLISION (0xC01C000A)
**설명:** 인스턴스 고도 충돌

```c
// 고도 충돌 해결
// Resolve altitude collision
// INF 파일에서 고유한 고도 값 지정
// Specify unique altitude value in INF file
```

### STATUS_FLT_INSTANCE_NAME_COLLISION (0xC01C000B)
**설명:** 인스턴스 이름 충돌

### STATUS_FLT_NOT_INITIALIZED (0xC01C000C)
**설명:** 필터 관리자가 초기화되지 않음

### STATUS_FLT_FILTER_NOT_READY (0xC01C000D)
**설명:** 필터가 준비되지 않음

### STATUS_FLT_CBDQ_DISABLED (0xC01C000E)
**설명:** 콜백 데이터 큐 비활성화됨

### STATUS_FLT_DO_NOT_ATTACH (0xC01C000F)
**설명:** 인스턴스를 연결하지 않음

```c
NTSTATUS InstanceSetupCallback(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_SETUP_FLAGS Flags,
    DEVICE_TYPE VolumeDeviceType,
    FLT_FILESYSTEM_TYPE VolumeFilesystemType)
{
    // 네트워크 드라이브에는 연결하지 않음
    // Do not attach to network drives
    if (VolumeDeviceType == FILE_DEVICE_NETWORK_FILE_SYSTEM) {
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    return STATUS_SUCCESS;
}
```

### STATUS_FLT_DO_NOT_DETACH (0xC01C0010)
**설명:** 인스턴스를 분리하지 않음

### STATUS_FLT_DELETING_OBJECT (0xC01C0011)
**설명:** 객체가 삭제 중

### STATUS_FLT_MUST_BE_NONPAGED_POOL (0xC01C0012)
**설명:** NonPaged 풀이어야 함

---

## B.10 I/O 오류 코드

### STATUS_IO_DEVICE_ERROR (0xC0000185)
**설명:** 장치 I/O 오류

### STATUS_DEVICE_NOT_READY (0xC00000A3)
**설명:** 장치가 준비되지 않음

### STATUS_DISK_FULL (0xC000007F)
**설명:** 디스크 공간 부족

```c
// 디스크 공간 확인
// Check disk space
if (status == STATUS_DISK_FULL) {
    // 사용자에게 디스크 공간 부족 알림
    // Notify user of insufficient disk space
    SendNotificationToUser(NOTIFICATION_DISK_FULL, volumeName);
}
```

### STATUS_MEDIA_WRITE_PROTECTED (0xC00000A2)
**설명:** 미디어가 쓰기 보호됨

### STATUS_DEVICE_OFF_LINE (0x80000010)
**설명:** 장치가 오프라인

### STATUS_CANCELLED (0xC0000120)
**설명:** 요청이 취소됨

```c
// I/O 취소 처리
// Handle I/O cancellation
FLT_PREOP_CALLBACK_STATUS PreRead(...)
{
    // 취소 루틴 등록
    // Register cancel routine
    FltSetCancelCompletion(Data, CancelRoutine);

    // ...

    // 취소 확인
    // Check for cancellation
    if (FltCancellableWaitForSingleObject(&event, Data, TRUE, NULL)
        == STATUS_CANCELLED) {
        return FLT_PREOP_COMPLETE;
    }
}
```

### STATUS_IO_TIMEOUT (0xC00000B5)
**설명:** I/O 시간 초과

---

## B.11 보안 오류 코드

### STATUS_PRIVILEGE_NOT_HELD (0xC0000061)
**설명:** 필요한 권한이 없음

```c
// 권한 확인
// Check privilege
SECURITY_SUBJECT_CONTEXT subjectContext;
SeCaptureSubjectContext(&subjectContext);

BOOLEAN hasPrivilege = SePrivilegeCheck(
    &privilegeSet,
    &subjectContext,
    UserMode);

SeReleaseSubjectContext(&subjectContext);

if (!hasPrivilege) {
    return STATUS_PRIVILEGE_NOT_HELD;
}
```

### STATUS_BAD_IMPERSONATION_LEVEL (0xC00000A5)
**설명:** 잘못된 가장 수준

### STATUS_LOGON_FAILURE (0xC000006D)
**설명:** 로그온 실패

### STATUS_ACCOUNT_DISABLED (0xC0000072)
**설명:** 계정 비활성화됨

### STATUS_WRONG_PASSWORD (0xC000006A)
**설명:** 잘못된 비밀번호

---

## B.12 오류 처리 패턴

### 기본 오류 처리

```c
NTSTATUS DoOperation(...)
{
    NTSTATUS status;

    status = Step1();
    if (!NT_SUCCESS(status)) {
        // 로깅 및 정리
        // Logging and cleanup
        LogError("Step1 failed: 0x%08X", status);
        return status;
    }

    status = Step2();
    if (!NT_SUCCESS(status)) {
        // Step1 정리 필요
        // Need to clean up Step1
        CleanupStep1();
        return status;
    }

    return STATUS_SUCCESS;
}
```

### goto를 사용한 정리 패턴

```c
NTSTATUS DoComplexOperation(...)
{
    NTSTATUS status;
    PVOID resource1 = NULL;
    PVOID resource2 = NULL;
    HANDLE handle = NULL;

    // 리소스 1 할당
    // Allocate resource 1
    resource1 = ExAllocatePool2(POOL_FLAG_PAGED, size1, TAG);
    if (resource1 == NULL) {
        status = STATUS_NO_MEMORY;
        goto Cleanup;
    }

    // 리소스 2 할당
    // Allocate resource 2
    resource2 = ExAllocatePool2(POOL_FLAG_PAGED, size2, TAG);
    if (resource2 == NULL) {
        status = STATUS_NO_MEMORY;
        goto Cleanup;
    }

    // 핸들 열기
    // Open handle
    status = ZwOpenFile(&handle, ...);
    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    // 작업 수행
    // Perform operation
    status = DoActualWork(resource1, resource2, handle);

Cleanup:
    if (handle != NULL) {
        ZwClose(handle);
    }
    if (resource2 != NULL) {
        ExFreePoolWithTag(resource2, TAG);
    }
    if (resource1 != NULL) {
        ExFreePoolWithTag(resource1, TAG);
    }

    return status;
}
```

### __try/__finally를 사용한 정리 패턴

```c
NTSTATUS DoOperationWithSEH(...)
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID buffer = NULL;
    ERESOURCE resourceLock;
    BOOLEAN resourceAcquired = FALSE;

    __try {
        buffer = ExAllocatePool2(POOL_FLAG_PAGED, size, TAG);
        if (buffer == NULL) {
            status = STATUS_NO_MEMORY;
            __leave;  // __finally로 이동
        }

        ExAcquireResourceExclusiveLite(&resourceLock, TRUE);
        resourceAcquired = TRUE;

        status = DoWork(buffer);
        if (!NT_SUCCESS(status)) {
            __leave;
        }
    }
    __finally {
        // 항상 실행됨
        // Always executed
        if (resourceAcquired) {
            ExReleaseResourceLite(&resourceLock);
        }
        if (buffer != NULL) {
            ExFreePoolWithTag(buffer, TAG);
        }
    }

    return status;
}
```

---

## B.13 NTSTATUS 참조 표

### 파일 시스템 관련 주요 코드

| 코드 | 값 | 설명 |
|------|---|------|
| STATUS_SUCCESS | 0x00000000 | 성공 |
| STATUS_ACCESS_DENIED | 0xC0000022 | 접근 거부 |
| STATUS_OBJECT_NAME_NOT_FOUND | 0xC0000034 | 객체 없음 |
| STATUS_OBJECT_PATH_NOT_FOUND | 0xC000003A | 경로 없음 |
| STATUS_SHARING_VIOLATION | 0xC0000043 | 공유 위반 |
| STATUS_DELETE_PENDING | 0xC0000056 | 삭제 대기 중 |
| STATUS_FILE_IS_A_DIRECTORY | 0xC00000BA | 디렉터리임 |
| STATUS_NOT_A_DIRECTORY | 0xC0000103 | 디렉터리 아님 |
| STATUS_END_OF_FILE | 0xC0000011 | 파일 끝 |
| STATUS_DISK_FULL | 0xC000007F | 디스크 가득 참 |

### Minifilter 관련 주요 코드

| 코드 | 값 | 설명 |
|------|---|------|
| STATUS_FLT_DO_NOT_ATTACH | 0xC01C000F | 연결 안함 |
| STATUS_FLT_DO_NOT_DETACH | 0xC01C0010 | 분리 안함 |
| STATUS_FLT_CONTEXT_ALREADY_DEFINED | 0xC01C0002 | 컨텍스트 있음 |
| STATUS_FLT_DELETING_OBJECT | 0xC01C0011 | 삭제 중 |
| STATUS_FLT_NOT_INITIALIZED | 0xC01C000C | 초기화 안됨 |

### 메모리 관련 주요 코드

| 코드 | 값 | 설명 |
|------|---|------|
| STATUS_NO_MEMORY | 0xC0000017 | 메모리 부족 |
| STATUS_INSUFFICIENT_RESOURCES | 0xC000009A | 리소스 부족 |
| STATUS_BUFFER_TOO_SMALL | 0xC0000023 | 버퍼 작음 |
| STATUS_BUFFER_OVERFLOW | 0x80000005 | 버퍼 오버플로 |

---

## B.14 디버깅 도우미

### WinDbg에서 NTSTATUS 확인

```
# 오류 코드 조회
# Look up error code
kd> !error 0xC0000022
Error code: (NTSTATUS) 0xc0000022 (3221225506) - {Access Denied}

kd> !error c0000034
Error code: (NTSTATUS) 0xc0000034 (3221225524) - {Object Name not Found}
```

### ntstatus.h 검색

```c
// ntstatus.h 파일에서 코드 정의 확인
// Check code definition in ntstatus.h
#define STATUS_ACCESS_DENIED             ((NTSTATUS)0xC0000022L)
```

### 커스텀 오류 코드 정의

```c
// 사용자 정의 오류 코드 (Customer 비트 설정)
// Custom error codes (Customer bit set)
#define STATUS_DRM_POLICY_VIOLATION      ((NTSTATUS)0xE0000001L)
#define STATUS_DRM_LICENSE_EXPIRED       ((NTSTATUS)0xE0000002L)
#define STATUS_DRM_UNAUTHORIZED_ACCESS   ((NTSTATUS)0xE0000003L)
```

---

## 요약

NTSTATUS 코드를 올바르게 이해하고 처리하는 것은 안정적인 Minifilter 드라이버 개발의 핵심입니다:

1. **NT_SUCCESS 매크로** 사용으로 성공 여부 확인
2. **특정 오류 코드**에 대한 적절한 처리 로직 구현
3. **리소스 정리** 패턴을 일관되게 적용
4. **WinDbg의 !error** 명령으로 디버깅 시 코드 의미 확인

파일 시스템 드라이버에서는 특히 `STATUS_ACCESS_DENIED`, `STATUS_OBJECT_NAME_NOT_FOUND`, `STATUS_SHARING_VIOLATION` 등의 코드가 자주 발생하므로, 이들에 대한 처리 로직을 미리 준비해 두는 것이 좋습니다.
