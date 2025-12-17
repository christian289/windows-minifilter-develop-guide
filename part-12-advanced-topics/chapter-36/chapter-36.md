# Chapter 36: 보안 강화 및 고급 기법

## 난이도: ★★★★★ (고급)

## 학습 목표
- 드라이버 자가 보호 메커니즘 구현
- 안티 탬퍼링 기법 이해
- 위협 탐지 및 대응 시스템 설계
- 보안 감사 및 규정 준수

## 36.1 드라이버 자가 보호

### 드라이버 파일 보호

```c
// SelfProtection.c
// 드라이버 자가 보호 구현
// Driver self-protection implementation

#include <fltKernel.h>

// 보호할 파일 목록
// Files to protect
typedef struct _PROTECTED_FILE {
    UNICODE_STRING FilePath;
    ULONG Flags;
} PROTECTED_FILE, *PPROTECTED_FILE;

#define PROTECT_FLAG_DRIVER     0x0001
#define PROTECT_FLAG_CONFIG     0x0002
#define PROTECT_FLAG_SERVICE    0x0004

// 보호 대상 파일들
// Protected files
PROTECTED_FILE g_ProtectedFiles[] = {
    { RTL_CONSTANT_STRING(L"\\SystemRoot\\System32\\drivers\\DrmFilter.sys"), PROTECT_FLAG_DRIVER },
    { RTL_CONSTANT_STRING(L"\\SystemRoot\\System32\\DrmFilterService.exe"), PROTECT_FLAG_SERVICE },
    { { 0 }, 0 }  // 종료 표시
};

// 보호 대상 파일인지 확인
// Check if file is protected
BOOLEAN
IsProtectedFile(
    _In_ PCUNICODE_STRING FilePath,
    _Out_opt_ PULONG ProtectionFlags
    )
{
    for (int i = 0; g_ProtectedFiles[i].FilePath.Buffer != NULL; i++) {
        if (RtlEqualUnicodeString(FilePath, &g_ProtectedFiles[i].FilePath, TRUE)) {
            if (ProtectionFlags != NULL) {
                *ProtectionFlags = g_ProtectedFiles[i].Flags;
            }
            return TRUE;
        }
    }

    return FALSE;
}

// 신뢰할 수 있는 프로세스인지 확인
// Check if process is trusted
BOOLEAN
IsTrustedProcess(
    VOID
    )
{
    PEPROCESS process = PsGetCurrentProcess();
    PUNICODE_STRING imageName = NULL;
    NTSTATUS status;
    BOOLEAN trusted = FALSE;

    status = SeLocateProcessImageName(process, &imageName);

    if (NT_SUCCESS(status) && imageName != NULL) {
        // Windows 업데이트 서비스
        // Windows Update service
        if (wcsstr(imageName->Buffer, L"TrustedInstaller.exe") != NULL) {
            trusted = TRUE;
        }
        // 시스템 프로세스
        // System process
        else if (wcsstr(imageName->Buffer, L"System") != NULL) {
            trusted = TRUE;
        }
        // 자체 서비스
        // Own service
        else if (wcsstr(imageName->Buffer, L"DrmFilterService.exe") != NULL) {
            trusted = TRUE;
        }

        ExFreePool(imageName);
    }

    return trusted;
}

// PreCreate에서 자가 보호
// Self-protection in PreCreate
FLT_PREOP_CALLBACK_STATUS
SelfProtectionPreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    PFLT_FILE_NAME_INFORMATION nameInfo;
    NTSTATUS status;
    ULONG protectionFlags;

    // 커널 모드 요청은 허용
    // Allow kernel mode requests
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 가져오기
    // Get file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    FltParseFileNameInformation(nameInfo);

    // 보호 대상 확인
    // Check if protected
    if (!IsProtectedFile(&nameInfo->Name, &protectionFlags)) {
        FltReleaseFileNameInformation(nameInfo);
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 접근 유형 확인
    // Check access type
    ACCESS_MASK desiredAccess = Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess;
    ULONG createDisposition = (Data->Iopb->Parameters.Create.Options >> 24) & 0xFF;

    // 삭제/덮어쓰기/수정 시도 감지
    // Detect delete/overwrite/modify attempts
    BOOLEAN isDangerous = FALSE;

    if (desiredAccess & (DELETE | FILE_WRITE_DATA | FILE_APPEND_DATA)) {
        isDangerous = TRUE;
    }

    if (createDisposition == FILE_SUPERSEDE ||
        createDisposition == FILE_OVERWRITE ||
        createDisposition == FILE_OVERWRITE_IF) {
        isDangerous = TRUE;
    }

    FltReleaseFileNameInformation(nameInfo);

    if (!isDangerous) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 신뢰할 수 있는 프로세스 확인
    // Check for trusted process
    if (IsTrustedProcess()) {
        DbgPrint("[DrmFilter] 신뢰 프로세스의 보호 파일 접근 허용\n");
        // [DrmFilter] Allowing trusted process access to protected file
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 접근 차단
    // Block access
    DbgPrint("[DrmFilter] 보호 파일 접근 차단: PID=%lu\n",
             HandleToUlong(PsGetCurrentProcessId()));
    // [DrmFilter] Blocking access to protected file

    Data->IoStatus.Status = STATUS_ACCESS_DENIED;
    Data->IoStatus.Information = 0;

    return FLT_PREOP_COMPLETE;
}

// PreSetInformation에서 자가 보호
// Self-protection in PreSetInformation
FLT_PREOP_CALLBACK_STATUS
SelfProtectionPreSetInfo(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    FILE_INFORMATION_CLASS infoClass;
    PFLT_FILE_NAME_INFORMATION nameInfo;
    NTSTATUS status;

    infoClass = Data->Iopb->Parameters.SetFileInformation.FileInformationClass;

    // 위험한 정보 클래스만 확인
    // Check only dangerous info classes
    if (infoClass != FileDispositionInformation &&
        infoClass != FileDispositionInformationEx &&
        infoClass != FileRenameInformation &&
        infoClass != FileRenameInformationEx) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 커널 모드 허용
    // Allow kernel mode
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 파일 이름 확인
    // Check file name
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    FltParseFileNameInformation(nameInfo);

    BOOLEAN isProtected = IsProtectedFile(&nameInfo->Name, NULL);

    FltReleaseFileNameInformation(nameInfo);

    if (!isProtected) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 신뢰 프로세스 확인
    // Check trusted process
    if (IsTrustedProcess()) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 차단
    // Block
    DbgPrint("[DrmFilter] 보호 파일 수정 시도 차단\n");
    // [DrmFilter] Blocking modification attempt on protected file

    Data->IoStatus.Status = STATUS_ACCESS_DENIED;
    return FLT_PREOP_COMPLETE;
}
```

### 레지스트리 보호

```c
// RegistryProtection.c
// 레지스트리 자가 보호
// Registry self-protection

#include <ntddk.h>

// 레지스트리 콜백 컨텍스트
// Registry callback context
LARGE_INTEGER g_RegistryCookie = { 0 };

// 보호할 레지스트리 키
// Registry keys to protect
const UNICODE_STRING g_ProtectedKeys[] = {
    RTL_CONSTANT_STRING(L"\\Registry\\Machine\\System\\CurrentControlSet\\Services\\DrmFilter"),
    RTL_CONSTANT_STRING(L"\\Registry\\Machine\\Software\\DrmFilter"),
    { 0, 0, NULL }
};

// 레지스트리 콜백 함수
// Registry callback function
NTSTATUS
RegistryCallback(
    _In_ PVOID CallbackContext,
    _In_opt_ PVOID Argument1,
    _In_opt_ PVOID Argument2
    )
{
    REG_NOTIFY_CLASS notifyClass = (REG_NOTIFY_CLASS)(ULONG_PTR)Argument1;
    NTSTATUS status = STATUS_SUCCESS;

    UNREFERENCED_PARAMETER(CallbackContext);

    switch (notifyClass) {
        case RegNtPreDeleteKey:
        case RegNtPreSetValueKey:
        case RegNtPreDeleteValueKey:
        case RegNtPreRenameKey:
        {
            // 키 경로 가져오기
            // Get key path
            PREG_DELETE_KEY_INFORMATION deleteInfo = NULL;
            PREG_SET_VALUE_KEY_INFORMATION setInfo = NULL;
            PVOID object = NULL;

            switch (notifyClass) {
                case RegNtPreDeleteKey:
                    deleteInfo = (PREG_DELETE_KEY_INFORMATION)Argument2;
                    object = deleteInfo->Object;
                    break;
                case RegNtPreSetValueKey:
                    setInfo = (PREG_SET_VALUE_KEY_INFORMATION)Argument2;
                    object = setInfo->Object;
                    break;
                // ... 기타 케이스
            }

            if (object != NULL) {
                PUNICODE_STRING keyPath = NULL;

                // 객체 이름 가져오기
                // Get object name
                status = CmCallbackGetKeyObjectIDEx(
                    &g_RegistryCookie,
                    object,
                    NULL,
                    &keyPath,
                    0
                );

                if (NT_SUCCESS(status) && keyPath != NULL) {
                    // 보호 키 확인
                    // Check protected key
                    for (int i = 0; g_ProtectedKeys[i].Buffer != NULL; i++) {
                        if (RtlPrefixUnicodeString(&g_ProtectedKeys[i], keyPath, TRUE)) {
                            // 신뢰 프로세스 확인
                            // Check trusted process
                            if (!IsTrustedProcess()) {
                                DbgPrint("[DrmFilter] 레지스트리 보호 키 접근 차단\n");
                                // [DrmFilter] Blocking access to protected registry key
                                status = STATUS_ACCESS_DENIED;
                            }
                            break;
                        }
                    }

                    CmCallbackReleaseKeyObjectIDEx(keyPath);
                }
            }
            break;
        }

        default:
            break;
    }

    return status;
}

// 레지스트리 보호 초기화
// Initialize registry protection
NTSTATUS
InitializeRegistryProtection(
    VOID
    )
{
    UNICODE_STRING altitude = RTL_CONSTANT_STRING(L"365000");

    return CmRegisterCallbackEx(
        RegistryCallback,
        &altitude,
        NULL,               // DriverObject
        NULL,               // Context
        &g_RegistryCookie,
        NULL
    );
}

// 레지스트리 보호 종료
// Cleanup registry protection
VOID
CleanupRegistryProtection(
    VOID
    )
{
    if (g_RegistryCookie.QuadPart != 0) {
        CmUnRegisterCallback(g_RegistryCookie);
        g_RegistryCookie.QuadPart = 0;
    }
}
```

## 36.2 안티 탬퍼링

### 무결성 검증

```c
// IntegrityCheck.c
// 드라이버 무결성 검증
// Driver integrity verification

#include <fltKernel.h>
#include <bcrypt.h>

// 무결성 정보
// Integrity information
typedef struct _INTEGRITY_INFO {
    UCHAR ExpectedHash[32];  // SHA-256
    ULONG CodeSize;
    PVOID CodeBase;
    BOOLEAN Verified;
} INTEGRITY_INFO, *PINTEGRITY_INFO;

INTEGRITY_INFO g_IntegrityInfo = { 0 };

// SHA-256 해시 계산
// Calculate SHA-256 hash
NTSTATUS
CalculateSha256(
    _In_reads_bytes_(DataSize) PVOID Data,
    _In_ ULONG DataSize,
    _Out_writes_(32) PUCHAR Hash
    )
{
    NTSTATUS status;
    BCRYPT_ALG_HANDLE algHandle = NULL;
    BCRYPT_HASH_HANDLE hashHandle = NULL;
    ULONG hashLength = 32;
    ULONG resultLength;

    status = BCryptOpenAlgorithmProvider(
        &algHandle,
        BCRYPT_SHA256_ALGORITHM,
        NULL,
        0
    );

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    status = BCryptCreateHash(
        algHandle,
        &hashHandle,
        NULL,
        0,
        NULL,
        0,
        0
    );

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    status = BCryptHashData(
        hashHandle,
        (PUCHAR)Data,
        DataSize,
        0
    );

    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    status = BCryptFinishHash(
        hashHandle,
        Hash,
        hashLength,
        0
    );

Cleanup:
    if (hashHandle != NULL) {
        BCryptDestroyHash(hashHandle);
    }
    if (algHandle != NULL) {
        BCryptCloseAlgorithmProvider(algHandle, 0);
    }

    return status;
}

// 드라이버 코드 무결성 검증
// Verify driver code integrity
NTSTATUS
VerifyDriverIntegrity(
    _In_ PDRIVER_OBJECT DriverObject
    )
{
    NTSTATUS status;
    UCHAR currentHash[32];

    // 드라이버 이미지 기본 주소 가져오기
    // Get driver image base address
    PVOID driverBase = DriverObject->DriverStart;
    ULONG driverSize = DriverObject->DriverSize;

    if (driverBase == NULL || driverSize == 0) {
        return STATUS_INVALID_PARAMETER;
    }

    // 현재 해시 계산
    // Calculate current hash
    status = CalculateSha256(driverBase, driverSize, currentHash);

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DrmFilter] 해시 계산 실패: 0x%08X\n", status);
        // [DrmFilter] Hash calculation failed
        return status;
    }

    // 저장된 해시와 비교
    // Compare with stored hash
    if (RtlCompareMemory(currentHash, g_IntegrityInfo.ExpectedHash, 32) != 32) {
        DbgPrint("[DrmFilter] 무결성 검증 실패! 드라이버가 변조됨\n");
        // [DrmFilter] Integrity check failed! Driver tampered

        // 경고 이벤트 생성
        // Generate alert event
        GenerateTamperAlert(TAMPER_TYPE_CODE_MODIFIED);

        return STATUS_INVALID_IMAGE_HASH;
    }

    DbgPrint("[DrmFilter] 무결성 검증 성공\n");
    // [DrmFilter] Integrity check passed

    g_IntegrityInfo.Verified = TRUE;

    return STATUS_SUCCESS;
}

// 주기적 무결성 검사
// Periodic integrity check
VOID
PeriodicIntegrityCheck(
    _In_ PVOID Context
    )
{
    PDRIVER_OBJECT driverObject = (PDRIVER_OBJECT)Context;
    NTSTATUS status;

    while (TRUE) {
        // 5분마다 검사
        // Check every 5 minutes
        LARGE_INTEGER delay;
        delay.QuadPart = -3000000000LL;  // 5분

        KeDelayExecutionThread(KernelMode, FALSE, &delay);

        // 종료 확인
        // Check for shutdown
        if (g_ShuttingDown) {
            break;
        }

        status = VerifyDriverIntegrity(driverObject);

        if (!NT_SUCCESS(status)) {
            // 탬퍼링 감지 시 조치
            // Action on tampering detection
            HandleTamperingDetected();
        }
    }

    PsTerminateSystemThread(STATUS_SUCCESS);
}
```

### 프로세스 보호

```c
// ProcessProtection.c
// 프로세스 종료 방지
// Process termination protection

#include <ntddk.h>

// 프로세스 알림 콜백
// Process notification callback
VOID
ProcessNotifyCallback(
    _Inout_ PEPROCESS Process,
    _In_ HANDLE ProcessId,
    _Inout_opt_ PPS_CREATE_NOTIFY_INFO CreateInfo
    )
{
    if (CreateInfo == NULL) {
        // 프로세스 종료
        // Process termination
        return;
    }

    // 프로세스 생성
    // Process creation
    if (CreateInfo->ImageFileName != NULL) {
        // 의심스러운 도구 감지
        // Detect suspicious tools
        if (wcsstr(CreateInfo->ImageFileName->Buffer, L"ProcessHacker") != NULL ||
            wcsstr(CreateInfo->ImageFileName->Buffer, L"procexp") != NULL) {
            DbgPrint("[DrmFilter] 의심스러운 도구 실행 감지: %wZ\n",
                     CreateInfo->ImageFileName);
            // [DrmFilter] Suspicious tool execution detected

            // 알림 전송
            // Send alert
            GenerateSecurityAlert(ALERT_SUSPICIOUS_TOOL, ProcessId);
        }
    }
}

// OB 콜백으로 프로세스 핸들 보호
// Protect process handles with OB callback
OB_PREOP_CALLBACK_STATUS
ProcessHandlePreCallback(
    _In_ PVOID RegistrationContext,
    _Inout_ POB_PRE_OPERATION_INFORMATION OperationInfo
    )
{
    UNREFERENCED_PARAMETER(RegistrationContext);

    // 자신의 프로세스 핸들 오픈 감지
    // Detect own process handle open
    if (OperationInfo->ObjectType != *PsProcessType) {
        return OB_PREOP_SUCCESS;
    }

    PEPROCESS targetProcess = (PEPROCESS)OperationInfo->Object;
    PUNICODE_STRING imageName = NULL;

    // 대상 프로세스가 우리 서비스인지 확인
    // Check if target process is our service
    if (NT_SUCCESS(SeLocateProcessImageName(targetProcess, &imageName))) {
        if (wcsstr(imageName->Buffer, L"DrmFilterService.exe") != NULL) {
            // 요청자가 신뢰할 수 있는지 확인
            // Check if requester is trusted
            if (!IsTrustedProcess()) {
                // 위험한 접근 권한 제거
                // Remove dangerous access rights
                if (OperationInfo->Operation == OB_OPERATION_HANDLE_CREATE) {
                    OperationInfo->Parameters->CreateHandleInformation.DesiredAccess &=
                        ~(PROCESS_TERMINATE | PROCESS_VM_WRITE | PROCESS_VM_OPERATION);
                }

                DbgPrint("[DrmFilter] 서비스 프로세스 핸들 접근 제한\n");
                // [DrmFilter] Restricting service process handle access
            }
        }

        ExFreePool(imageName);
    }

    return OB_PREOP_SUCCESS;
}

// OB 콜백 등록
// Register OB callback
PVOID g_ObCallbackHandle = NULL;

NTSTATUS
RegisterProcessProtection(
    VOID
    )
{
    OB_CALLBACK_REGISTRATION callbackReg;
    OB_OPERATION_REGISTRATION operationReg;
    UNICODE_STRING altitude = RTL_CONSTANT_STRING(L"365000");

    RtlZeroMemory(&callbackReg, sizeof(callbackReg));
    RtlZeroMemory(&operationReg, sizeof(operationReg));

    operationReg.ObjectType = PsProcessType;
    operationReg.Operations = OB_OPERATION_HANDLE_CREATE | OB_OPERATION_HANDLE_DUPLICATE;
    operationReg.PreOperation = ProcessHandlePreCallback;
    operationReg.PostOperation = NULL;

    callbackReg.Version = OB_FLT_REGISTRATION_VERSION;
    callbackReg.OperationRegistrationCount = 1;
    callbackReg.RegistrationContext = NULL;
    callbackReg.OperationRegistration = &operationReg;
    callbackReg.Altitude = altitude;

    return ObRegisterCallbacks(&callbackReg, &g_ObCallbackHandle);
}
```

## 36.3 위협 탐지

### 이상 행위 탐지

```c
// AnomalyDetection.c
// 이상 행위 탐지 시스템
// Anomaly detection system

#include <fltKernel.h>

// 프로세스별 행위 통계
// Per-process behavior statistics
typedef struct _PROCESS_BEHAVIOR {
    LIST_ENTRY ListEntry;
    HANDLE ProcessId;
    UNICODE_STRING ProcessName;

    // 접근 통계
    // Access statistics
    volatile LONG FileAccessCount;
    volatile LONG WriteCount;
    volatile LONG DeleteCount;
    volatile LONG RenameCount;

    // 시간 윈도우
    // Time window
    LARGE_INTEGER WindowStart;
    LARGE_INTEGER LastActivity;

    // 임계값 초과 여부
    // Threshold exceeded
    BOOLEAN ThresholdExceeded;
} PROCESS_BEHAVIOR, *PPROCESS_BEHAVIOR;

// 행위 임계값
// Behavior thresholds
#define THRESHOLD_FILE_ACCESS_PER_MINUTE    1000
#define THRESHOLD_WRITE_PER_MINUTE          500
#define THRESHOLD_DELETE_PER_MINUTE         100
#define THRESHOLD_RENAME_PER_MINUTE         100

// 행위 테이블
// Behavior table
LIST_ENTRY g_BehaviorList;
EX_PUSH_LOCK g_BehaviorLock;

// 프로세스 행위 가져오기 또는 생성
// Get or create process behavior
PPROCESS_BEHAVIOR
GetOrCreateProcessBehavior(
    _In_ HANDLE ProcessId
    )
{
    PPROCESS_BEHAVIOR behavior = NULL;
    PLIST_ENTRY entry;

    FltAcquirePushLockShared(&g_BehaviorLock);

    // 기존 항목 검색
    // Search existing entry
    for (entry = g_BehaviorList.Flink;
         entry != &g_BehaviorList;
         entry = entry->Flink) {

        behavior = CONTAINING_RECORD(entry, PROCESS_BEHAVIOR, ListEntry);
        if (behavior->ProcessId == ProcessId) {
            FltReleasePushLock(&g_BehaviorLock);
            return behavior;
        }
    }

    FltReleasePushLock(&g_BehaviorLock);

    // 새 항목 생성
    // Create new entry
    behavior = ExAllocatePool2(
        POOL_FLAG_NON_PAGED,
        sizeof(PROCESS_BEHAVIOR),
        'heBP'
    );

    if (behavior == NULL) {
        return NULL;
    }

    RtlZeroMemory(behavior, sizeof(PROCESS_BEHAVIOR));
    behavior->ProcessId = ProcessId;
    KeQuerySystemTime(&behavior->WindowStart);
    behavior->LastActivity = behavior->WindowStart;

    // 프로세스 이름 가져오기
    // Get process name
    PEPROCESS process;
    if (NT_SUCCESS(PsLookupProcessByProcessId(ProcessId, &process))) {
        PUNICODE_STRING imageName;
        if (NT_SUCCESS(SeLocateProcessImageName(process, &imageName))) {
            behavior->ProcessName.Buffer = ExAllocatePool2(
                POOL_FLAG_NON_PAGED,
                imageName->Length + sizeof(WCHAR),
                'mNpB'
            );
            if (behavior->ProcessName.Buffer != NULL) {
                RtlCopyUnicodeString(&behavior->ProcessName, imageName);
            }
            ExFreePool(imageName);
        }
        ObDereferenceObject(process);
    }

    FltAcquirePushLockExclusive(&g_BehaviorLock);
    InsertTailList(&g_BehaviorList, &behavior->ListEntry);
    FltReleasePushLock(&g_BehaviorLock);

    return behavior;
}

// 행위 기록 및 분석
// Record and analyze behavior
BOOLEAN
AnalyzeBehavior(
    _In_ HANDLE ProcessId,
    _In_ ULONG Operation
    )
{
    PPROCESS_BEHAVIOR behavior;
    LARGE_INTEGER currentTime;
    LONGLONG elapsedMs;

    behavior = GetOrCreateProcessBehavior(ProcessId);
    if (behavior == NULL) {
        return FALSE;
    }

    KeQuerySystemTime(&currentTime);
    behavior->LastActivity = currentTime;

    // 시간 윈도우 확인 (1분)
    // Check time window (1 minute)
    elapsedMs = (currentTime.QuadPart - behavior->WindowStart.QuadPart) / 10000;

    if (elapsedMs > 60000) {
        // 윈도우 리셋
        // Reset window
        behavior->WindowStart = currentTime;
        behavior->FileAccessCount = 0;
        behavior->WriteCount = 0;
        behavior->DeleteCount = 0;
        behavior->RenameCount = 0;
        behavior->ThresholdExceeded = FALSE;
    }

    // 카운터 증가
    // Increment counter
    InterlockedIncrement(&behavior->FileAccessCount);

    switch (Operation) {
        case IRP_MJ_WRITE:
            InterlockedIncrement(&behavior->WriteCount);
            break;
        case IRP_MJ_SET_INFORMATION:
            // 삭제 또는 이름 변경으로 가정
            // Assume delete or rename
            InterlockedIncrement(&behavior->DeleteCount);
            break;
    }

    // 임계값 확인
    // Check thresholds
    BOOLEAN anomalyDetected = FALSE;

    if (behavior->FileAccessCount > THRESHOLD_FILE_ACCESS_PER_MINUTE) {
        DbgPrint("[DrmFilter] 이상 탐지: PID=%lu - 과도한 파일 접근 (%ld/분)\n",
                 HandleToUlong(ProcessId), behavior->FileAccessCount);
        // [DrmFilter] Anomaly detected: Excessive file access
        anomalyDetected = TRUE;
    }

    if (behavior->WriteCount > THRESHOLD_WRITE_PER_MINUTE) {
        DbgPrint("[DrmFilter] 이상 탐지: PID=%lu - 과도한 쓰기 (%ld/분)\n",
                 HandleToUlong(ProcessId), behavior->WriteCount);
        // [DrmFilter] Anomaly detected: Excessive writes
        anomalyDetected = TRUE;
    }

    if (behavior->DeleteCount > THRESHOLD_DELETE_PER_MINUTE) {
        DbgPrint("[DrmFilter] 이상 탐지: PID=%lu - 과도한 삭제 (%ld/분)\n",
                 HandleToUlong(ProcessId), behavior->DeleteCount);
        // [DrmFilter] Anomaly detected: Excessive deletes
        anomalyDetected = TRUE;
    }

    if (anomalyDetected && !behavior->ThresholdExceeded) {
        behavior->ThresholdExceeded = TRUE;

        // 보안 이벤트 생성
        // Generate security event
        GenerateAnomalyAlert(
            ProcessId,
            &behavior->ProcessName,
            behavior->FileAccessCount,
            behavior->WriteCount,
            behavior->DeleteCount
        );
    }

    return anomalyDetected;
}

// 랜섬웨어 패턴 탐지
// Ransomware pattern detection
BOOLEAN
DetectRansomwarePattern(
    _In_ HANDLE ProcessId,
    _In_ PCUNICODE_STRING FilePath,
    _In_ ULONG Operation
    )
{
    PPROCESS_BEHAVIOR behavior;

    behavior = GetOrCreateProcessBehavior(ProcessId);
    if (behavior == NULL) {
        return FALSE;
    }

    // 랜섬웨어 지표:
    // Ransomware indicators:
    // 1. 짧은 시간에 많은 파일 수정
    // 1. Many file modifications in short time
    // 2. 확장자 변경 패턴
    // 2. Extension change pattern
    // 3. 암호화 파일 헤더 감지
    // 3. Encrypted file header detection

    BOOLEAN suspicious = FALSE;

    // 높은 쓰기율 + 이름 변경 조합
    // High write rate + rename combination
    if (behavior->WriteCount > 100 && behavior->RenameCount > 50) {
        suspicious = TRUE;

        DbgPrint("[DrmFilter] 랜섬웨어 의심 패턴: PID=%lu\n",
                 HandleToUlong(ProcessId));
        // [DrmFilter] Suspected ransomware pattern

        // 확장자 변경 패턴 확인
        // Check extension change pattern
        // .encrypted, .locked, .crypto 등
    }

    return suspicious;
}
```

### 보안 이벤트 로깅

```c
// SecurityLogging.c
// 보안 이벤트 로깅
// Security event logging

#include <fltKernel.h>
#include <evntprov.h>

// 보안 이벤트 유형
// Security event types
typedef enum _SECURITY_EVENT_TYPE {
    SEC_EVENT_FILE_BLOCKED,
    SEC_EVENT_POLICY_VIOLATION,
    SEC_EVENT_TAMPER_DETECTED,
    SEC_EVENT_ANOMALY_DETECTED,
    SEC_EVENT_RANSOMWARE_SUSPECTED,
    SEC_EVENT_UNAUTHORIZED_ACCESS
} SECURITY_EVENT_TYPE;

// 보안 이벤트 구조체
// Security event structure
typedef struct _SECURITY_EVENT {
    SECURITY_EVENT_TYPE Type;
    LARGE_INTEGER Timestamp;
    HANDLE ProcessId;
    WCHAR ProcessName[260];
    WCHAR FilePath[260];
    ULONG Operation;
    NTSTATUS ResultStatus;
    WCHAR Details[512];
} SECURITY_EVENT, *PSECURITY_EVENT;

// ETW 프로바이더로 이벤트 기록
// Log event to ETW provider
VOID
LogSecurityEvent(
    _In_ PSECURITY_EVENT Event
    )
{
    // ETW 이벤트 기록
    // Log ETW event
    EVENT_DESCRIPTOR eventDescriptor;
    EVENT_DATA_DESCRIPTOR dataDescriptor[5];

    RtlZeroMemory(&eventDescriptor, sizeof(eventDescriptor));
    eventDescriptor.Id = (USHORT)Event->Type;
    eventDescriptor.Version = 1;
    eventDescriptor.Level = TRACE_LEVEL_WARNING;

    EventDataDescCreate(&dataDescriptor[0], &Event->Timestamp, sizeof(LARGE_INTEGER));
    EventDataDescCreate(&dataDescriptor[1], &Event->ProcessId, sizeof(HANDLE));
    EventDataDescCreate(&dataDescriptor[2], Event->ProcessName, sizeof(Event->ProcessName));
    EventDataDescCreate(&dataDescriptor[3], Event->FilePath, sizeof(Event->FilePath));
    EventDataDescCreate(&dataDescriptor[4], Event->Details, sizeof(Event->Details));

    EtwWrite(g_EtwRegHandle, &eventDescriptor, NULL, 5, dataDescriptor);
}

// Windows 이벤트 로그에 기록
// Write to Windows Event Log
VOID
WriteToEventLog(
    _In_ PSECURITY_EVENT Event
    )
{
    IO_ERROR_LOG_PACKET* errorLogEntry;
    ULONG packetSize;

    packetSize = sizeof(IO_ERROR_LOG_PACKET) + sizeof(Event->Details);

    if (packetSize > ERROR_LOG_MAXIMUM_SIZE) {
        packetSize = ERROR_LOG_MAXIMUM_SIZE;
    }

    errorLogEntry = IoAllocateErrorLogEntry(
        g_DriverObject,
        (UCHAR)packetSize
    );

    if (errorLogEntry == NULL) {
        return;
    }

    errorLogEntry->ErrorCode = STATUS_LOG_FILE_FULL;  // 이벤트 ID로 사용
    errorLogEntry->UniqueErrorValue = Event->Type;
    errorLogEntry->FinalStatus = Event->ResultStatus;
    errorLogEntry->DumpDataSize = 0;

    IoWriteErrorLogEntry(errorLogEntry);
}

// 보안 이벤트 생성 헬퍼
// Security event generation helper
VOID
GenerateSecurityEvent(
    _In_ SECURITY_EVENT_TYPE Type,
    _In_ HANDLE ProcessId,
    _In_opt_ PCUNICODE_STRING FilePath,
    _In_ ULONG Operation,
    _In_opt_ PCWSTR Details
    )
{
    SECURITY_EVENT event = { 0 };

    event.Type = Type;
    KeQuerySystemTime(&event.Timestamp);
    event.ProcessId = ProcessId;
    event.Operation = Operation;

    // 프로세스 이름 가져오기
    // Get process name
    PEPROCESS process;
    if (NT_SUCCESS(PsLookupProcessByProcessId(ProcessId, &process))) {
        PUNICODE_STRING imageName;
        if (NT_SUCCESS(SeLocateProcessImageName(process, &imageName))) {
            RtlStringCbCopyW(event.ProcessName, sizeof(event.ProcessName),
                             imageName->Buffer);
            ExFreePool(imageName);
        }
        ObDereferenceObject(process);
    }

    // 파일 경로
    // File path
    if (FilePath != NULL && FilePath->Buffer != NULL) {
        USHORT copyLen = min(FilePath->Length, sizeof(event.FilePath) - sizeof(WCHAR));
        RtlCopyMemory(event.FilePath, FilePath->Buffer, copyLen);
    }

    // 상세 정보
    // Details
    if (Details != NULL) {
        RtlStringCbCopyW(event.Details, sizeof(event.Details), Details);
    }

    // 로깅
    // Logging
    LogSecurityEvent(&event);
    WriteToEventLog(&event);

    // 사용자 모드로 전송
    // Send to user mode
    SendSecurityEventToUserMode(&event);
}
```

## 36.4 규정 준수

### 감사 추적

```c
// AuditTrail.c
// 감사 추적 구현
// Audit trail implementation

#include <fltKernel.h>

// 감사 레코드
// Audit record
typedef struct _AUDIT_RECORD {
    LARGE_INTEGER Timestamp;
    GUID SessionId;
    HANDLE ProcessId;
    HANDLE ThreadId;
    WCHAR UserName[64];
    WCHAR DomainName[64];
    WCHAR FilePath[260];
    ULONG Operation;
    ULONG AccessMask;
    NTSTATUS Result;
    ULONG RecordSize;
} AUDIT_RECORD, *PAUDIT_RECORD;

// 감사 파일 핸들
// Audit file handle
HANDLE g_AuditFileHandle = NULL;
ERESOURCE g_AuditFileLock;

// 감사 파일 초기화
// Initialize audit file
NTSTATUS
InitializeAuditFile(
    _In_ PCUNICODE_STRING AuditFilePath
    )
{
    NTSTATUS status;
    OBJECT_ATTRIBUTES oa;
    IO_STATUS_BLOCK ioStatus;

    ExInitializeResourceLite(&g_AuditFileLock);

    InitializeObjectAttributes(
        &oa,
        (PUNICODE_STRING)AuditFilePath,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        NULL
    );

    status = ZwCreateFile(
        &g_AuditFileHandle,
        FILE_APPEND_DATA | SYNCHRONIZE,
        &oa,
        &ioStatus,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        FILE_SHARE_READ,
        FILE_OPEN_IF,
        FILE_SYNCHRONOUS_IO_NONALERT | FILE_NON_DIRECTORY_FILE,
        NULL,
        0
    );

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DrmFilter] 감사 파일 열기 실패: 0x%08X\n", status);
        // [DrmFilter] Failed to open audit file
    }

    return status;
}

// 감사 레코드 기록
// Write audit record
NTSTATUS
WriteAuditRecord(
    _In_ PAUDIT_RECORD Record
    )
{
    NTSTATUS status;
    IO_STATUS_BLOCK ioStatus;

    if (g_AuditFileHandle == NULL) {
        return STATUS_INVALID_HANDLE;
    }

    // 락 획득
    // Acquire lock
    ExAcquireResourceExclusiveLite(&g_AuditFileLock, TRUE);

    // 레코드 쓰기
    // Write record
    status = ZwWriteFile(
        g_AuditFileHandle,
        NULL,
        NULL,
        NULL,
        &ioStatus,
        Record,
        sizeof(AUDIT_RECORD),
        NULL,
        NULL
    );

    ExReleaseResourceLite(&g_AuditFileLock);

    return status;
}

// 감사 레코드 생성
// Create audit record
VOID
CreateAuditRecord(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCUNICODE_STRING FilePath,
    _In_ NTSTATUS Result
    )
{
    AUDIT_RECORD record = { 0 };
    PACCESS_TOKEN token;
    PTOKEN_USER tokenUser;
    ULONG returnLength;

    KeQuerySystemTime(&record.Timestamp);
    record.ProcessId = PsGetCurrentProcessId();
    record.ThreadId = PsGetCurrentThreadId();
    record.Operation = Data->Iopb->MajorFunction;
    record.Result = Result;

    // 파일 경로
    // File path
    if (FilePath != NULL && FilePath->Buffer != NULL) {
        USHORT copyLen = min(FilePath->Length, sizeof(record.FilePath) - sizeof(WCHAR));
        RtlCopyMemory(record.FilePath, FilePath->Buffer, copyLen);
    }

    // 사용자 정보 가져오기
    // Get user information
    token = PsReferencePrimaryToken(PsGetCurrentProcess());
    if (token != NULL) {
        if (NT_SUCCESS(SeQueryInformationToken(token, TokenUser, (PVOID*)&tokenUser))) {
            // SID를 사용자 이름으로 변환
            // Convert SID to user name
            // 실제 구현에서는 SecLookupAccountSid 사용
            // In real implementation, use SecLookupAccountSid

            ExFreePool(tokenUser);
        }
        PsDereferencePrimaryToken(token);
    }

    // 접근 마스크
    // Access mask
    if (Data->Iopb->MajorFunction == IRP_MJ_CREATE) {
        record.AccessMask = Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess;
    }

    record.RecordSize = sizeof(AUDIT_RECORD);

    // 레코드 기록
    // Write record
    WriteAuditRecord(&record);
}

// GDPR/HIPAA 준수 데이터 마스킹
// GDPR/HIPAA compliant data masking
VOID
MaskSensitiveData(
    _Inout_updates_bytes_(DataSize) PVOID Data,
    _In_ ULONG DataSize,
    _In_ ULONG SensitiveOffset,
    _In_ ULONG SensitiveLength
    )
{
    if (SensitiveOffset + SensitiveLength > DataSize) {
        return;
    }

    // 민감 데이터 마스킹 (별표로 대체)
    // Mask sensitive data (replace with asterisks)
    PUCHAR sensitiveData = (PUCHAR)Data + SensitiveOffset;

    for (ULONG i = 0; i < SensitiveLength; i++) {
        sensitiveData[i] = '*';
    }
}
```

## 36.5 정리

이 챕터에서 학습한 내용:

1. **드라이버 자가 보호**
   - 드라이버 파일 보호
   - 레지스트리 키 보호
   - 신뢰 프로세스 검증

2. **안티 탬퍼링**
   - 무결성 검증
   - 프로세스 핸들 보호
   - OB 콜백 활용

3. **위협 탐지**
   - 이상 행위 분석
   - 랜섬웨어 패턴 감지
   - 임계값 기반 탐지

4. **보안 로깅**
   - ETW 이벤트
   - Windows 이벤트 로그
   - 보안 이벤트 전송

5. **규정 준수**
   - 감사 추적
   - 데이터 마스킹
   - GDPR/HIPAA 고려

---

## 책 마무리

이로써 **Windows Minifilter 드라이버 개발 가이드**의 본문이 완료되었습니다.

이 책에서는 .NET 개발자가 C 언어와 커널 모드 프로그래밍을 배워 DRM 보호 기능을 갖춘 Minifilter 드라이버를 개발할 수 있도록 안내했습니다.

### 학습 경로 요약

1. **Part 1-3**: C 언어 기초와 Windows 아키텍처
2. **Part 4**: WinDbg 디버깅 마스터
3. **Part 5-6**: 첫 드라이버와 Minifilter 기초
4. **Part 7**: 콘텐츠 분석 (SSN 패턴 탐지)
5. **Part 8**: DRM 보호 구현
6. **Part 9**: 사용자 모드 서비스 연동
7. **Part 10**: 테스트와 성능 최적화
8. **Part 11**: 서명과 배포
9. **Part 12**: 가상화와 보안 강화

### 다음 단계

- 부록에서 추가 참고 자료 확인
- 실제 프로젝트에 적용
- 커뮤니티 참여 및 피드백

**행운을 빕니다!**
