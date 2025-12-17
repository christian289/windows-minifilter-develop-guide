# 부록 D: 디버깅 체크리스트

## Minifilter 드라이버 문제 해결 가이드

이 부록은 Minifilter 드라이버 개발 중 발생하는 일반적인 문제와 해결 방법을 체크리스트 형태로 정리합니다. 문제 유형별로 확인해야 할 사항을 단계적으로 제시합니다.

---

## D.1 드라이버 로드 실패

### 증상
- 드라이버가 시작되지 않음
- `fltmc load` 명령 실패
- 서비스 시작 오류

### 체크리스트

#### 1단계: 기본 확인
- [ ] INF 파일이 올바르게 설치되었는가?
- [ ] 드라이버 파일(.sys)이 올바른 위치에 있는가?
  ```cmd
  dir C:\Windows\System32\drivers\YourDriver.sys
  ```
- [ ] 드라이버가 올바르게 서명되었는가?
  ```cmd
  signtool verify /pa /v YourDriver.sys
  ```

#### 2단계: 서비스 확인
- [ ] 서비스가 등록되어 있는가?
  ```cmd
  sc query YourFilter
  ```
- [ ] 서비스 시작 유형이 올바른가?
  ```cmd
  sc qc YourFilter
  ```

#### 3단계: 필터 관리자 확인
- [ ] 필터 관리자가 실행 중인가?
  ```cmd
  sc query fltmgr
  ```
- [ ] 필터가 등록되어 있는가?
  ```cmd
  fltmc filters
  ```

#### 4단계: 이벤트 로그 확인
```powershell
Get-WinEvent -LogName System -MaxEvents 50 |
    Where-Object { $_.ProviderName -eq 'Service Control Manager' }
```

#### 5단계: WinDbg 분석
```
# 드라이버 로드 중단점 설정
kd> sxe ld:YourDriver

# 로드 후 심볼 확인
kd> lm m YourDriver

# DriverEntry 중단점
kd> bp YourDriver!DriverEntry
```

### 일반적인 원인과 해결책

| 원인 | 해결책 |
|------|--------|
| 서명 문제 | 테스트 서명 활성화 또는 유효한 서명 사용 |
| 고도 충돌 | INF에서 고유한 고도 값 지정 |
| 종속성 부족 | 필요한 라이브러리/드라이버 확인 |
| 레지스트리 오류 | 레지스트리 키 수동 확인 및 수정 |

---

## D.2 블루 스크린 (BSOD)

### 증상
- 시스템 충돌
- BSOD 오류 화면
- 자동 재부팅

### 체크리스트

#### 1단계: 덤프 파일 수집
- [ ] 덤프 파일 위치 확인
  ```
  C:\Windows\MEMORY.DMP (전체 덤프)
  C:\Windows\Minidump\*.dmp (미니 덤프)
  ```
- [ ] 덤프 설정 확인
  ```
  시스템 속성 → 고급 → 시작 및 복구 → 설정
  ```

#### 2단계: WinDbg로 덤프 분석
```
# 덤프 파일 열기
File → Open Crash Dump

# 자동 분석
kd> !analyze -v

# 스택 추적
kd> k

# 충돌 시점 컨텍스트
kd> .ecxr
kd> k
```

#### 3단계: 버그 체크 코드 확인

| 버그 체크 | 의미 | 일반적 원인 |
|-----------|------|------------|
| IRQL_NOT_LESS_OR_EQUAL (0x0A) | 잘못된 IRQL에서 메모리 접근 | 높은 IRQL에서 페이지 메모리 접근 |
| KMODE_EXCEPTION_NOT_HANDLED (0x1E) | 커널 예외 처리 안됨 | NULL 포인터, 범위 초과 |
| PAGE_FAULT_IN_NONPAGED_AREA (0x50) | 잘못된 메모리 접근 | 해제된 메모리 사용 |
| DRIVER_IRQL_NOT_LESS_OR_EQUAL (0xD1) | 드라이버 IRQL 위반 | Paged 메모리를 높은 IRQL에서 접근 |
| KERNEL_DATA_INPAGE_ERROR (0x7A) | 커널 데이터 읽기 실패 | I/O 오류, 손상된 메모리 |
| SYSTEM_SERVICE_EXCEPTION (0x3B) | 시스템 서비스 예외 | 잘못된 매개변수 |

#### 4단계: 상세 분석
```
# 버그 체크 코드별 분석
kd> !analyze -show <BugCheckCode>

# 메모리 풀 확인
kd> !pool <Address>

# 드라이버 소유권 확인
kd> !poolused 2

# IRP 분석
kd> !irp <Address>
```

### IRQL 관련 BSOD 해결

```c
// 문제 코드
// Problematic code
VOID ProblematicFunction(PVOID PagedBuffer)
{
    KIRQL oldIrql;
    KeAcquireSpinLock(&Lock, &oldIrql);  // IRQL = DISPATCH_LEVEL

    // 오류! Paged 메모리 접근
    // Error! Accessing paged memory
    RtlCopyMemory(Dest, PagedBuffer, Size);  // BSOD!

    KeReleaseSpinLock(&Lock, oldIrql);
}

// 수정된 코드
// Fixed code
VOID FixedFunction(PVOID PagedBuffer, SIZE_T Size)
{
    KIRQL oldIrql;
    PVOID nonPagedCopy;

    // 먼저 NonPaged로 복사
    // Copy to NonPaged first
    nonPagedCopy = ExAllocatePool2(POOL_FLAG_NON_PAGED, Size, TAG);
    if (nonPagedCopy == NULL) return;

    RtlCopyMemory(nonPagedCopy, PagedBuffer, Size);

    // 이제 스핀락 사용 가능
    // Now can use spinlock
    KeAcquireSpinLock(&Lock, &oldIrql);
    // nonPagedCopy 사용
    // Use nonPagedCopy
    KeReleaseSpinLock(&Lock, oldIrql);

    ExFreePoolWithTag(nonPagedCopy, TAG);
}
```

---

## D.3 메모리 누수

### 증상
- 시간이 지남에 따라 사용 가능한 메모리 감소
- 시스템 성능 저하
- 풀 태그 메모리 증가

### 체크리스트

#### 1단계: 풀 태그 모니터링
```
# poolmon.exe 사용 (WDK 도구)
poolmon -b -p

# 태그별 정렬
poolmon -i<Tag> -p

# WinDbg에서
kd> !poolused 2
```

#### 2단계: 드라이버 태그 확인
```
# 특정 태그 검색
kd> !poolfind <Tag>

# 풀 할당 분석
kd> !pool <Address>
```

#### 3단계: Driver Verifier 활성화
```powershell
# 풀 추적 활성화
verifier /flags 0x8 /driver YourDriver.sys

# 특별 풀 활성화 (오버런/언더런 감지)
verifier /flags 0x1 /driver YourDriver.sys

# 재부팅 후 확인
verifier /query
```

#### 4단계: 코드 리뷰
- [ ] 모든 `ExAllocatePool` 호출에 대응하는 `ExFreePool`이 있는가?
- [ ] 오류 경로에서 메모리가 해제되는가?
- [ ] 컨텍스트가 올바르게 해제되는가?

### 메모리 누수 추적 코드

```c
// 디버그 빌드에서 할당 추적
// Track allocations in debug build
#if DBG
typedef struct _ALLOC_TRACKER {
    LIST_ENTRY ListEntry;
    PVOID Address;
    SIZE_T Size;
    ULONG Tag;
    CHAR Function[64];
    ULONG Line;
} ALLOC_TRACKER, *PALLOC_TRACKER;

LIST_ENTRY gAllocList;
KSPIN_LOCK gAllocLock;

PVOID TrackedAlloc(
    SIZE_T Size,
    ULONG Tag,
    const char* Function,
    ULONG Line)
{
    PVOID ptr = ExAllocatePool2(POOL_FLAG_NON_PAGED, Size, Tag);
    if (ptr != NULL) {
        PALLOC_TRACKER tracker = ExAllocatePool2(
            POOL_FLAG_NON_PAGED,
            sizeof(ALLOC_TRACKER),
            'KART');

        if (tracker != NULL) {
            tracker->Address = ptr;
            tracker->Size = Size;
            tracker->Tag = Tag;
            RtlStringCbCopyA(tracker->Function, sizeof(tracker->Function), Function);
            tracker->Line = Line;

            KIRQL oldIrql;
            KeAcquireSpinLock(&gAllocLock, &oldIrql);
            InsertTailList(&gAllocList, &tracker->ListEntry);
            KeReleaseSpinLock(&gAllocLock, oldIrql);
        }
    }
    return ptr;
}

#define ALLOC(size, tag) TrackedAlloc(size, tag, __FUNCTION__, __LINE__)

// 언로드 시 누수 확인
// Check for leaks on unload
VOID CheckForLeaks()
{
    PLIST_ENTRY entry;
    KIRQL oldIrql;

    KeAcquireSpinLock(&gAllocLock, &oldIrql);

    while (!IsListEmpty(&gAllocList)) {
        entry = RemoveHeadList(&gAllocList);
        PALLOC_TRACKER tracker = CONTAINING_RECORD(entry, ALLOC_TRACKER, ListEntry);

        DbgPrint("LEAK: %p (%llu bytes) allocated at %s:%lu\n",
            tracker->Address,
            (ULONGLONG)tracker->Size,
            tracker->Function,
            tracker->Line);

        ExFreePoolWithTag(tracker, 'KART');
    }

    KeReleaseSpinLock(&gAllocLock, oldIrql);
}
#endif
```

---

## D.4 교착 상태 (Deadlock)

### 증상
- 시스템 또는 특정 프로세스 멈춤
- 응답 없음
- I/O 요청이 완료되지 않음

### 체크리스트

#### 1단계: 교착 상태 감지
```
# WinDbg에서 잠긴 스레드 확인
kd> !locks

# 모든 스레드 스택 확인
kd> !process 0 17

# 특정 프로세스의 스레드
kd> !process <ProcessAddress> 17
```

#### 2단계: 잠금 분석
```
# ERESOURCE 분석
kd> !locks -v

# 스핀락 분석
kd> !qlocks

# 대기 체인 분석
kd> !thread <ThreadAddress>
kd> .thread <ThreadAddress>
kd> k
```

#### 3단계: Driver Verifier 교착 상태 감지
```powershell
# 교착 상태 감지 활성화
verifier /flags 0x20 /driver YourDriver.sys

# 확인
verifier /query
```

### 교착 상태 방지 패턴

```c
// 잘못된 패턴: 잠금 순서 불일치
// Wrong pattern: Inconsistent lock order
// Thread 1: LockA → LockB
// Thread 2: LockB → LockA (교착 상태!)

// 올바른 패턴: 일관된 잠금 순서
// Correct pattern: Consistent lock order
// 항상 LockA → LockB 순서로 획득
// Always acquire LockA → LockB order

typedef struct _LOCK_CONTEXT {
    ERESOURCE LockA;      // 레벨 1 (항상 먼저)
    ERESOURCE LockB;      // 레벨 2 (항상 나중에)
} LOCK_CONTEXT;

VOID AcquireBothLocks(PLOCK_CONTEXT Ctx)
{
    // 항상 같은 순서로 획득
    // Always acquire in same order
    ExAcquireResourceExclusiveLite(&Ctx->LockA, TRUE);
    ExAcquireResourceExclusiveLite(&Ctx->LockB, TRUE);
}

VOID ReleaseBothLocks(PLOCK_CONTEXT Ctx)
{
    // 역순으로 해제
    // Release in reverse order
    ExReleaseResourceLite(&Ctx->LockB);
    ExReleaseResourceLite(&Ctx->LockA);
}
```

### 타임아웃 패턴

```c
// 타임아웃을 사용한 잠금 획득
// Lock acquisition with timeout
BOOLEAN TryAcquireLockWithTimeout(
    PKSPIN_LOCK Lock,
    PKIRQL OldIrql,
    ULONG TimeoutMs)
{
    LARGE_INTEGER timeout;
    LARGE_INTEGER start, current;
    BOOLEAN acquired = FALSE;

    timeout.QuadPart = -(LONGLONG)TimeoutMs * 10000;
    KeQuerySystemTime(&start);

    while (!acquired) {
        acquired = KeTryToAcquireSpinLockAtDpcLevel(Lock);
        if (acquired) {
            break;
        }

        KeQuerySystemTime(&current);
        if ((current.QuadPart - start.QuadPart) > -timeout.QuadPart) {
            // 타임아웃
            // Timeout
            DbgPrint("Lock acquisition timeout!\n");
            return FALSE;
        }

        // 잠시 대기
        // Brief wait
        KeStallExecutionProcessor(1);
    }

    return TRUE;
}
```

---

## D.5 성능 문제

### 증상
- 시스템 응답 느림
- 높은 CPU 사용률
- 파일 I/O 느림

### 체크리스트

#### 1단계: 성능 카운터 확인
```powershell
# 디스크 성능
Get-Counter '\PhysicalDisk(*)\Disk Reads/sec'
Get-Counter '\PhysicalDisk(*)\Disk Writes/sec'
Get-Counter '\PhysicalDisk(*)\Avg. Disk sec/Read'

# 프로세서
Get-Counter '\Processor(_Total)\% Processor Time'
```

#### 2단계: WPR/WPA 프로파일링
```cmd
# 성능 추적 시작
wpr -start GeneralProfile -filemode

# 문제 재현

# 추적 중지
wpr -stop profile.etl

# WPA로 분석
wpa profile.etl
```

#### 3단계: WinDbg CPU 프로파일링
```
# CPU 사용률 높은 스레드 찾기
kd> !running -t

# 스택 샘플링
kd> !runaway 7
```

### 성능 최적화 체크리스트

- [ ] 불필요한 IRP를 조기에 통과시키는가?
- [ ] 캐싱을 활용하고 있는가?
- [ ] 잠금 경합이 최소화되어 있는가?
- [ ] 페이징 I/O를 건너뛰고 있는가?
- [ ] 비동기 처리를 활용하고 있는가?

```c
// 성능 최적화된 PreCreate 예시
// Performance-optimized PreCreate example
FLT_PREOP_CALLBACK_STATUS FastPreCreate(
    PFLT_CALLBACK_DATA Data,
    PCFLT_RELATED_OBJECTS FltObjects,
    PVOID *CompletionContext)
{
    // 1. 빠른 필터링 - 관심 없는 작업 즉시 통과
    // 1. Fast filtering - immediately pass uninteresting operations

    // 커널 모드 요청 통과
    // Pass kernel mode requests
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 볼륨 열기 통과
    // Pass volume opens
    if (FlagOn(FltObjects->FileObject->Flags, FO_VOLUME_OPEN)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 2. 캐시 확인
    // 2. Check cache
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    NTSTATUS status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_CACHE_ONLY,
        &nameInfo);

    if (NT_SUCCESS(status)) {
        // 캐시에서 빠르게 결정
        // Quick decision from cache
        BOOLEAN isProtected = CheckCachedPolicy(&nameInfo->Name);
        FltReleaseFileNameInformation(nameInfo);

        if (!isProtected) {
            return FLT_PREOP_SUCCESS_NO_CALLBACK;
        }
    }

    // 3. 필요한 경우에만 상세 처리
    // 3. Detailed processing only when needed
    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

---

## D.6 필터 관리자 문제

### 증상
- 필터가 볼륨에 연결되지 않음
- 인스턴스 설정 실패
- 컨텍스트 작업 실패

### 체크리스트

#### 1단계: 필터 상태 확인
```cmd
# 등록된 필터 목록
fltmc filters

# 볼륨별 인스턴스
fltmc instances

# 볼륨 정보
fltmc volumes
```

#### 2단계: 인스턴스 확인
```cmd
# 특정 볼륨의 인스턴스
fltmc instances -v C:

# 인스턴스 분리
fltmc detach YourFilter C:

# 인스턴스 연결
fltmc attach YourFilter C:
```

#### 3단계: WinDbg 분석
```
# 필터 정보
kd> !fltkd.filter <FilterAddress>

# 볼륨 정보
kd> !fltkd.volume <VolumeAddress>

# 인스턴스 정보
kd> !fltkd.instance <InstanceAddress>

# 콜백 목록
kd> !fltkd.callbacks <FilterAddress>
```

### InstanceSetup 실패 디버깅

```c
NTSTATUS InstanceSetupCallback(
    PCFLT_RELATED_OBJECTS FltObjects,
    FLT_INSTANCE_SETUP_FLAGS Flags,
    DEVICE_TYPE VolumeDeviceType,
    FLT_FILESYSTEM_TYPE VolumeFilesystemType)
{
    // 디버그 출력
    // Debug output
    DbgPrint("[DRM] InstanceSetup: DeviceType=%d, FsType=%d, Flags=0x%X\n",
        VolumeDeviceType, VolumeFilesystemType, Flags);

    // 네트워크 드라이브 제외
    // Exclude network drives
    if (VolumeDeviceType == FILE_DEVICE_NETWORK_FILE_SYSTEM) {
        DbgPrint("[DRM] Skipping network volume\n");
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    // RAW 파일시스템 제외
    // Exclude RAW filesystem
    if (VolumeFilesystemType == FLT_FSTYPE_RAW) {
        DbgPrint("[DRM] Skipping RAW volume\n");
        return STATUS_FLT_DO_NOT_ATTACH;
    }

    DbgPrint("[DRM] Attaching to volume\n");
    return STATUS_SUCCESS;
}
```

---

## D.7 통신 포트 문제

### 증상
- 사용자 모드 앱이 연결할 수 없음
- 메시지 송수신 실패
- 타임아웃

### 체크리스트

#### 1단계: 포트 확인
```cmd
# WinObj로 포트 확인 (Sysinternals)
WinObj.exe
# \FileSystem\Filters\<PortName> 확인
```

#### 2단계: 권한 확인
- [ ] 보안 설명자가 올바른가?
- [ ] 앱이 필요한 권한을 가지고 있는가?

#### 3단계: WinDbg 분석
```
# 포트 목록
kd> !fltkd.portlist

# 연결된 클라이언트
kd> !fltkd.port <PortAddress>
```

### 통신 포트 디버깅 코드

```c
// 상세한 오류 로깅이 포함된 포트 생성
// Port creation with detailed error logging
NTSTATUS CreateDebugPort(PFLT_FILTER Filter)
{
    NTSTATUS status;
    UNICODE_STRING portName;
    OBJECT_ATTRIBUTES oa;
    PSECURITY_DESCRIPTOR sd = NULL;

    DbgPrint("[DRM] Creating communication port...\n");

    // 보안 설명자 생성
    // Create security descriptor
    status = FltBuildDefaultSecurityDescriptor(&sd, FLT_PORT_ALL_ACCESS);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[DRM] FltBuildDefaultSecurityDescriptor failed: 0x%X\n", status);
        return status;
    }

    RtlInitUnicodeString(&portName, L"\\DrmFilterPort");

    InitializeObjectAttributes(
        &oa,
        &portName,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        sd);

    status = FltCreateCommunicationPort(
        Filter,
        &gServerPort,
        &oa,
        NULL,
        ClientConnectCallback,
        ClientDisconnectCallback,
        ClientMessageCallback,
        10);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DRM] FltCreateCommunicationPort failed: 0x%X\n", status);
    } else {
        DbgPrint("[DRM] Communication port created successfully\n");
    }

    FltFreeSecurityDescriptor(sd);
    return status;
}

// 연결 콜백 디버깅
// Connection callback debugging
NTSTATUS ClientConnectCallback(
    PFLT_PORT ClientPort,
    PVOID ServerPortCookie,
    PVOID ConnectionContext,
    ULONG SizeOfContext,
    PVOID *ConnectionPortCookie)
{
    DbgPrint("[DRM] Client connecting: Port=%p, ContextSize=%lu\n",
        ClientPort, SizeOfContext);

    if (gClientPort != NULL) {
        DbgPrint("[DRM] Rejecting - already connected\n");
        return STATUS_ALREADY_REGISTERED;
    }

    gClientPort = ClientPort;
    DbgPrint("[DRM] Client connected successfully\n");

    return STATUS_SUCCESS;
}
```

---

## D.8 일반적인 실수와 해결책

### 실수 1: IRQL 위반

```c
// 잘못됨: DISPATCH_LEVEL에서 Paged 메모리 사용
// Wrong: Using Paged memory at DISPATCH_LEVEL
VOID WrongFunction()
{
    KIRQL oldIrql;
    KeAcquireSpinLock(&Lock, &oldIrql);  // DISPATCH_LEVEL

    // 오류! Paged 풀 할당
    // Error! Paged pool allocation
    PVOID buffer = ExAllocatePool2(POOL_FLAG_PAGED, 100, TAG);

    KeReleaseSpinLock(&Lock, oldIrql);
}

// 올바름: NonPaged 메모리 사용
// Correct: Using NonPaged memory
VOID CorrectFunction()
{
    KIRQL oldIrql;
    KeAcquireSpinLock(&Lock, &oldIrql);

    // NonPaged 풀 사용
    // Use NonPaged pool
    PVOID buffer = ExAllocatePool2(POOL_FLAG_NON_PAGED, 100, TAG);

    KeReleaseSpinLock(&Lock, oldIrql);
}
```

### 실수 2: 컨텍스트 참조 누수

```c
// 잘못됨: 참조 해제 누락
// Wrong: Missing reference release
VOID WrongContextUsage()
{
    PSTREAM_CONTEXT ctx;
    FltGetStreamContext(Instance, FileObject, &ctx);

    if (ctx != NULL) {
        // 컨텍스트 사용
        // Use context
        DoSomething(ctx);
        // 참조 해제 누락!
        // Missing reference release!
    }
}

// 올바름: 항상 참조 해제
// Correct: Always release reference
VOID CorrectContextUsage()
{
    PSTREAM_CONTEXT ctx = NULL;
    NTSTATUS status = FltGetStreamContext(Instance, FileObject, &ctx);

    if (NT_SUCCESS(status) && ctx != NULL) {
        DoSomething(ctx);
        FltReleaseContext(ctx);  // 반드시 해제
    }
}
```

### 실수 3: 파일 이름 정보 누수

```c
// 잘못됨: 오류 경로에서 해제 누락
// Wrong: Missing release in error path
NTSTATUS WrongNameUsage(PFLT_CALLBACK_DATA Data)
{
    PFLT_FILE_NAME_INFORMATION nameInfo;

    FltGetFileNameInformation(Data, FLT_FILE_NAME_NORMALIZED, &nameInfo);

    if (SomeCondition()) {
        return STATUS_UNSUCCESSFUL;  // nameInfo 누수!
    }

    FltReleaseFileNameInformation(nameInfo);
    return STATUS_SUCCESS;
}

// 올바름: 모든 경로에서 해제
// Correct: Release in all paths
NTSTATUS CorrectNameUsage(PFLT_CALLBACK_DATA Data)
{
    PFLT_FILE_NAME_INFORMATION nameInfo = NULL;
    NTSTATUS status;

    status = FltGetFileNameInformation(Data, FLT_FILE_NAME_NORMALIZED, &nameInfo);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    if (SomeCondition()) {
        status = STATUS_UNSUCCESSFUL;
        goto Cleanup;
    }

    // 작업 수행
    // Do work

Cleanup:
    if (nameInfo != NULL) {
        FltReleaseFileNameInformation(nameInfo);
    }
    return status;
}
```

### 실수 4: 잘못된 버퍼 접근

```c
// 잘못됨: 사용자 버퍼 직접 접근
// Wrong: Direct user buffer access
VOID WrongBufferAccess(PVOID UserBuffer, ULONG Size)
{
    // 오류! 검증 없이 사용자 버퍼 접근
    // Error! Accessing user buffer without validation
    RtlCopyMemory(KernelBuffer, UserBuffer, Size);
}

// 올바름: __try/__except로 보호
// Correct: Protected with __try/__except
NTSTATUS CorrectBufferAccess(PVOID UserBuffer, ULONG Size)
{
    __try {
        ProbeForRead(UserBuffer, Size, sizeof(UCHAR));
        RtlCopyMemory(KernelBuffer, UserBuffer, Size);
        return STATUS_SUCCESS;
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        return GetExceptionCode();
    }
}
```

---

## D.9 디버깅 도구 요약

### 필수 도구

| 도구 | 용도 |
|------|------|
| WinDbg | 커널 디버깅, 덤프 분석 |
| Driver Verifier | 드라이버 검증 |
| PoolMon | 메모리 풀 모니터링 |
| WPR/WPA | 성능 프로파일링 |
| fltmc | Minifilter 관리 |
| WinObj | 커널 객체 확인 |

### WinDbg 확장 명령

```
!fltkd.filters      - 필터 목록
!fltkd.filter       - 필터 상세
!fltkd.volumes      - 볼륨 목록
!fltkd.instances    - 인스턴스 목록
!fltkd.callbacks    - 콜백 목록
!analyze -v         - 크래시 분석
!pool               - 풀 분석
!locks              - 잠금 분석
!irp                - IRP 분석
```

---

## 요약

Minifilter 디버깅에서 가장 중요한 것은:

1. **체계적인 접근**: 증상 → 데이터 수집 → 분석 → 해결
2. **적절한 도구 사용**: WinDbg, Driver Verifier, PoolMon
3. **예방적 코딩**: 검증, 오류 처리, 로깅
4. **IRQL 인식**: 현재 IRQL에서 허용되는 작업 이해

이 체크리스트를 참조하여 문제를 신속하게 진단하고 해결하세요.
