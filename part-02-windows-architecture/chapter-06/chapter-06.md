# Chapter 6: Windows 보안 모델

## 난이도: ⭐⭐⭐ (중급)

## 학습 목표

- Windows 보안 주체(Security Principal)와 SID 이해
- 액세스 토큰의 구조와 역할 파악
- 보안 설명자(Security Descriptor)와 ACL 이해
- 무결성 수준(Integrity Level) 개념 습득
- DRM 구현에 필요한 보안 지식 확보

---

## 들어가며

DRM 파일 보호를 구현할 때, "이 파일에 접근하려는 프로세스가 신뢰할 수 있는가?"를 판단해야 합니다. Windows 보안 모델을 이해하면 이러한 판단의 기준을 세울 수 있습니다.

이 장에서는 Windows의 핵심 보안 개념을 다룹니다. .NET의 `System.Security` 네임스페이스와 연관지어 설명합니다.

---

## 6.1 보안 주체와 SID

### 6.1.1 보안 주체 (Security Principal)

**보안 주체**는 보안 컨텍스트를 가질 수 있는 엔티티입니다:

```
보안 주체 종류:
├── 사용자 계정 (User Account)
│   └─ 예: DOMAIN\username, .\Administrator
├── 그룹 (Group)
│   └─ 예: Administrators, Users, Power Users
├── 컴퓨터 계정 (Computer Account)
│   └─ 예: DOMAIN\COMPUTER$
├── 서비스 계정 (Service Account)
│   └─ 예: LocalSystem, NetworkService, LocalService
└── 가상 계정
    └─ 예: NT SERVICE\ServiceName
```

### 6.1.2 SID (Security Identifier)

모든 보안 주체는 고유한 **SID**로 식별됩니다:

```
SID 구조:
S-R-I-S₁-S₂-...-Sₙ-RID

S-1-5-21-3623811015-3361044348-30300820-1013
│ │ │  └─────────────────────────────────┘ └──┘
│ │ │              도메인/머신 ID           상대 ID (RID)
│ │ └─ 식별자 권한 (5 = NT Authority)
│ └─ 리비전 (항상 1)
└─ SID 접두사 (항상 S)
```

### 잘 알려진 SID (Well-Known SIDs)

| SID | 이름 | 설명 |
|-----|------|------|
| S-1-0-0 | Nobody | 아무도 아님 |
| S-1-1-0 | Everyone | 모든 사용자 |
| S-1-5-18 | SYSTEM | LocalSystem 계정 |
| S-1-5-19 | LOCAL SERVICE | LocalService 계정 |
| S-1-5-20 | NETWORK SERVICE | NetworkService 계정 |
| S-1-5-32-544 | Administrators | 관리자 그룹 |
| S-1-5-32-545 | Users | 사용자 그룹 |

### C# 비교

```csharp
// C#에서 SID 다루기
// Working with SID in C#
using System.Security.Principal;

// 현재 사용자 SID
var currentUser = WindowsIdentity.GetCurrent();
Console.WriteLine($"User SID: {currentUser.User?.Value}");

// 잘 알려진 SID 생성
var adminSid = new SecurityIdentifier(WellKnownSidType.BuiltinAdministratorsSid, null);
Console.WriteLine($"Administrators SID: {adminSid.Value}");  // S-1-5-32-544

// 그룹 멤버십 확인
var principal = new WindowsPrincipal(currentUser);
bool isAdmin = principal.IsInRole(WindowsBuiltInRole.Administrator);
```

```c
// 커널에서 SID 다루기
// Working with SID in kernel
PSID userSid;
NTSTATUS status;

// 프로세스의 토큰에서 사용자 SID 얻기
// Get user SID from process token
PACCESS_TOKEN token = PsReferencePrimaryToken(PsGetCurrentProcess());

PTOKEN_USER tokenUser;
status = SeQueryInformationToken(token, TokenUser, (PVOID*)&tokenUser);

if (NT_SUCCESS(status)) {
    // tokenUser->User.Sid에 사용자 SID가 있음
    // SID 문자열로 변환 (디버깅용)
    UNICODE_STRING sidString;
    RtlConvertSidToUnicodeString(&sidString, tokenUser->User.Sid, TRUE);
    DbgPrint("User SID: %wZ\n", &sidString);
    RtlFreeUnicodeString(&sidString);

    ExFreePool(tokenUser);
}

PsDereferencePrimaryToken(token);
```

---

## 6.2 액세스 토큰

### 6.2.1 토큰의 역할

**액세스 토큰(Access Token)**은 보안 주체의 자격 증명을 나타냅니다:

```
┌─────────────────────────────────────────────────────────────┐
│                     Access Token                             │
├─────────────────────────────────────────────────────────────┤
│ Token ID              : 고유 식별자                          │
├─────────────────────────────────────────────────────────────┤
│ User SID              : S-1-5-21-...-1001 (사용자)          │
├─────────────────────────────────────────────────────────────┤
│ Group SIDs:                                                  │
│   - S-1-5-32-544 (Administrators)                           │
│   - S-1-5-32-545 (Users)                                    │
│   - S-1-5-21-...-513 (Domain Users)                         │
│   - S-1-5-4 (Interactive)                                   │
│   - ...                                                      │
├─────────────────────────────────────────────────────────────┤
│ Privileges:                                                  │
│   - SeDebugPrivilege (디버그 권한)                          │
│   - SeBackupPrivilege (백업 권한)                           │
│   - SeRestorePrivilege (복원 권한)                          │
│   - SeShutdownPrivilege (종료 권한)                         │
│   - ...                                                      │
├─────────────────────────────────────────────────────────────┤
│ Integrity Level       : High (S-1-16-12288)                 │
├─────────────────────────────────────────────────────────────┤
│ Session ID            : 1                                    │
└─────────────────────────────────────────────────────────────┘
```

### 6.2.2 토큰 유형

```
토큰 유형:
├── Primary Token (기본 토큰)
│   └─ 프로세스에 할당됨
│   └─ 프로세스 생성 시 부모로부터 복사
│
└── Impersonation Token (가장 토큰)
    └─ 스레드에 할당됨
    └─ 다른 사용자로 가장(impersonate)할 때 사용
    └─ 서버가 클라이언트를 대신해 작업할 때
```

### 6.2.3 커널에서 토큰 접근

```c
// 프로세스 토큰 얻기
// Get process token
NTSTATUS GetProcessToken(PEPROCESS Process, PACCESS_TOKEN* Token)
{
    *Token = PsReferencePrimaryToken(Process);
    return STATUS_SUCCESS;
}

// 토큰에서 정보 조회
// Query information from token
NTSTATUS GetTokenUser(PACCESS_TOKEN Token, PSID* UserSid)
{
    NTSTATUS status;
    PTOKEN_USER tokenUser = NULL;

    status = SeQueryInformationToken(
        Token,
        TokenUser,
        (PVOID*)&tokenUser
    );

    if (NT_SUCCESS(status)) {
        // SID 복사
        ULONG sidLength = RtlLengthSid(tokenUser->User.Sid);
        *UserSid = ExAllocatePoolWithTag(PagedPool, sidLength, 'disU');

        if (*UserSid != NULL) {
            RtlCopySid(sidLength, *UserSid, tokenUser->User.Sid);
        }

        ExFreePool(tokenUser);
    }

    return status;
}

// 관리자 그룹 멤버십 확인
// Check Administrator group membership
BOOLEAN IsUserAdmin(PACCESS_TOKEN Token)
{
    BOOLEAN isAdmin = FALSE;

    // 잘 알려진 Administrators SID 생성
    // Create well-known Administrators SID
    SID_IDENTIFIER_AUTHORITY ntAuthority = SECURITY_NT_AUTHORITY;
    PSID adminSid;

    NTSTATUS status = RtlAllocateAndInitializeSid(
        &ntAuthority,
        2,                         // SubAuthority 개수
        SECURITY_BUILTIN_DOMAIN_RID,
        DOMAIN_ALIAS_RID_ADMINS,
        0, 0, 0, 0, 0, 0,
        &adminSid
    );

    if (NT_SUCCESS(status)) {
        // 토큰이 해당 SID를 포함하는지 확인
        if (!SeTokenIsAdmin(Token)) {
            // 그룹 멤버십 직접 확인
            PTOKEN_GROUPS groups;
            status = SeQueryInformationToken(Token, TokenGroups, (PVOID*)&groups);

            if (NT_SUCCESS(status)) {
                for (ULONG i = 0; i < groups->GroupCount; i++) {
                    if (RtlEqualSid(adminSid, groups->Groups[i].Sid)) {
                        isAdmin = TRUE;
                        break;
                    }
                }
                ExFreePool(groups);
            }
        } else {
            isAdmin = TRUE;
        }

        RtlFreeSid(adminSid);
    }

    return isAdmin;
}
```

---

## 6.3 보안 설명자와 ACL

### 6.3.1 보안 설명자 (Security Descriptor)

모든 보안 가능 객체(파일, 레지스트리 키, 프로세스 등)는 **보안 설명자**를 가집니다:

```
┌─────────────────────────────────────────────────────────────┐
│                  Security Descriptor                         │
├─────────────────────────────────────────────────────────────┤
│ Revision           : 1 (항상)                                │
├─────────────────────────────────────────────────────────────┤
│ Control Flags:                                               │
│   - SE_DACL_PRESENT                                         │
│   - SE_SACL_PRESENT                                         │
│   - SE_OWNER_DEFAULTED                                      │
│   - ...                                                      │
├─────────────────────────────────────────────────────────────┤
│ Owner SID          : S-1-5-21-...-1001 (소유자)             │
├─────────────────────────────────────────────────────────────┤
│ Group SID          : S-1-5-21-...-513 (기본 그룹)           │
├─────────────────────────────────────────────────────────────┤
│ DACL (Discretionary ACL):                                   │
│   누가 어떤 접근을 할 수 있는지/없는지                        │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ ACE 1: Allow Administrators - Full Control          │   │
│   │ ACE 2: Allow SYSTEM - Full Control                  │   │
│   │ ACE 3: Allow Owner - Full Control                   │   │
│   │ ACE 4: Allow Users - Read, Execute                  │   │
│   │ ACE 5: Deny Guest - All Access                      │   │
│   └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│ SACL (System ACL):                                          │
│   감사(Audit) 설정                                          │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Audit Success: Write access by Everyone             │   │
│   │ Audit Failure: All access by Everyone               │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 6.3.2 ACL과 ACE

```
DACL (Discretionary Access Control List)
├── ACE (Access Control Entry) 1
│   ├── Type: ACCESS_ALLOWED_ACE_TYPE
│   ├── SID: S-1-5-32-544 (Administrators)
│   └── Access Mask: 0x1F01FF (Full Control)
├── ACE 2
│   ├── Type: ACCESS_ALLOWED_ACE_TYPE
│   ├── SID: S-1-5-32-545 (Users)
│   └── Access Mask: 0x1200A9 (Read & Execute)
├── ACE 3
│   ├── Type: ACCESS_DENIED_ACE_TYPE
│   ├── SID: S-1-5-7 (Anonymous)
│   └── Access Mask: 0x1F01FF (All)
└── ...
```

### 6.3.3 파일 접근 권한 비트

| 권한 | 값 | 설명 |
|------|-----|------|
| FILE_READ_DATA | 0x0001 | 파일 데이터 읽기 |
| FILE_WRITE_DATA | 0x0002 | 파일 데이터 쓰기 |
| FILE_APPEND_DATA | 0x0004 | 데이터 추가 |
| FILE_READ_EA | 0x0008 | 확장 속성 읽기 |
| FILE_WRITE_EA | 0x0010 | 확장 속성 쓰기 |
| FILE_EXECUTE | 0x0020 | 실행 |
| FILE_READ_ATTRIBUTES | 0x0080 | 속성 읽기 |
| FILE_WRITE_ATTRIBUTES | 0x0100 | 속성 쓰기 |
| DELETE | 0x00010000 | 삭제 |
| READ_CONTROL | 0x00020000 | 보안 설명자 읽기 |
| WRITE_DAC | 0x00040000 | DACL 수정 |
| WRITE_OWNER | 0x00080000 | 소유자 변경 |

### C# 비교

```csharp
// C#에서 파일 ACL 확인
// Check file ACL in C#
using System.Security.AccessControl;

var fileInfo = new FileInfo(@"C:\test.txt");
var security = fileInfo.GetAccessControl();

// DACL의 각 규칙 확인
foreach (FileSystemAccessRule rule in
    security.GetAccessRules(true, true, typeof(NTAccount)))
{
    Console.WriteLine($"Identity: {rule.IdentityReference}");
    Console.WriteLine($"Type: {rule.AccessControlType}");
    Console.WriteLine($"Rights: {rule.FileSystemRights}");
    Console.WriteLine();
}
```

---

## 6.4 무결성 수준 (Integrity Level)

### 6.4.1 무결성 수준 계층

**무결성 수준(Integrity Level)**은 프로세스의 신뢰도를 나타냅니다:

```
높은 신뢰도 ↑
┌─────────────────────────────────────────────────────────────┐
│ System (S-1-16-16384)                                       │
│   - SYSTEM 계정으로 실행되는 프로세스                        │
│   - 서비스, 커널 구성 요소                                   │
├─────────────────────────────────────────────────────────────┤
│ High (S-1-16-12288)                                         │
│   - 관리자 권한으로 실행                                     │
│   - "관리자 권한으로 실행" 시                                │
├─────────────────────────────────────────────────────────────┤
│ Medium (S-1-16-8192)                                        │
│   - 표준 사용자 프로세스                                     │
│   - 대부분의 데스크톱 애플리케이션                           │
├─────────────────────────────────────────────────────────────┤
│ Low (S-1-16-4096)                                           │
│   - 보호 모드 Internet Explorer                              │
│   - 샌드박스된 프로세스                                      │
│   - 제한된 쓰기 권한                                         │
├─────────────────────────────────────────────────────────────┤
│ Untrusted (S-1-16-0)                                        │
│   - 신뢰할 수 없는 프로세스                                  │
│   - 거의 사용되지 않음                                       │
└─────────────────────────────────────────────────────────────┘
낮은 신뢰도 ↓
```

### 6.4.2 무결성 수준 규칙

**"No Write Up"** 원칙: 낮은 무결성 프로세스는 높은 무결성 객체에 쓸 수 없습니다.

```
시나리오:
┌─────────────────┐          ┌─────────────────┐
│ 프로세스 (Low)  │ ──쓰기──▶ │ 파일 (Medium)   │
│ 무결성: Low     │    ❌     │ 무결성: Medium  │
└─────────────────┘ (차단됨)  └─────────────────┘

┌─────────────────┐          ┌─────────────────┐
│ 프로세스 (Medium)│ ──쓰기──▶ │ 파일 (Medium)   │
│ 무결성: Medium  │    ✅     │ 무결성: Medium  │
└─────────────────┘ (허용됨)  └─────────────────┘

┌─────────────────┐          ┌─────────────────┐
│ 프로세스 (High) │ ──읽기──▶ │ 파일 (Low)      │
│ 무결성: High    │    ✅     │ 무결성: Low     │
└─────────────────┘ (허용됨)  └─────────────────┘
```

### 6.4.3 커널에서 무결성 수준 확인

```c
// 프로세스의 무결성 수준 얻기
// Get process integrity level
NTSTATUS GetProcessIntegrityLevel(
    PEPROCESS Process,
    PULONG IntegrityLevel
)
{
    NTSTATUS status;
    PACCESS_TOKEN token;
    PTOKEN_MANDATORY_LABEL label = NULL;

    token = PsReferencePrimaryToken(Process);

    status = SeQueryInformationToken(
        token,
        TokenIntegrityLevel,
        (PVOID*)&label
    );

    if (NT_SUCCESS(status)) {
        // SID에서 무결성 수준 추출
        // Extract integrity level from SID
        PSID integrityLevelSid = label->Label.Sid;
        ULONG subAuthCount = *RtlSubAuthorityCountSid(integrityLevelSid);

        if (subAuthCount > 0) {
            *IntegrityLevel = *RtlSubAuthoritySid(
                integrityLevelSid,
                subAuthCount - 1
            );
        }

        ExFreePool(label);
    }

    PsDereferencePrimaryToken(token);
    return status;
}

// 무결성 수준 상수
// Integrity level constants
#define SECURITY_MANDATORY_UNTRUSTED_RID    0x0000
#define SECURITY_MANDATORY_LOW_RID          0x1000
#define SECURITY_MANDATORY_MEDIUM_RID       0x2000
#define SECURITY_MANDATORY_HIGH_RID         0x3000
#define SECURITY_MANDATORY_SYSTEM_RID       0x4000

// 사용 예
// Usage example
ULONG integrityLevel;
if (NT_SUCCESS(GetProcessIntegrityLevel(PsGetCurrentProcess(), &integrityLevel))) {
    if (integrityLevel >= SECURITY_MANDATORY_HIGH_RID) {
        DbgPrint("High integrity process\n");
    } else if (integrityLevel >= SECURITY_MANDATORY_MEDIUM_RID) {
        DbgPrint("Medium integrity process\n");
    } else {
        DbgPrint("Low integrity process\n");
    }
}
```

---

## 6.5 DRM에서의 보안 활용

### 6.5.1 신뢰할 수 있는 프로세스 판별

DRM 구현 시 특정 프로세스만 보호된 파일에 접근하도록 허용해야 합니다:

```c
// 프로세스 신뢰도 검사 예시
// Example of process trust verification
NTSTATUS CheckProcessTrust(
    PFLT_CALLBACK_DATA Data,
    PBOOLEAN IsTrusted
)
{
    NTSTATUS status = STATUS_SUCCESS;
    PEPROCESS process = NULL;
    PACCESS_TOKEN token = NULL;
    PUNICODE_STRING imageName = NULL;

    *IsTrusted = FALSE;

    // 1. 요청한 프로세스 얻기
    // Get requesting process
    process = IoThreadToProcess(Data->Thread);

    // 2. 프로세스 이미지 이름 확인
    // Check process image name
    status = SeLocateProcessImageName(process, &imageName);
    if (NT_SUCCESS(status)) {
        // 신뢰할 수 있는 프로세스 목록과 비교
        // Compare with trusted process list
        if (IsInTrustedList(imageName)) {
            *IsTrusted = TRUE;
        }
        ExFreePool(imageName);
    }

    // 3. 서명 확인 (선택)
    // Verify signature (optional)
    if (!*IsTrusted) {
        // 프로세스 이미지의 디지털 서명 확인
        // ...
    }

    // 4. 무결성 수준 확인
    // Check integrity level
    if (*IsTrusted) {
        token = PsReferencePrimaryToken(process);
        ULONG integrityLevel;
        // 낮은 무결성 프로세스는 신뢰하지 않음
        if (GetIntegrityLevel(token, &integrityLevel) >= 0 &&
            integrityLevel < SECURITY_MANDATORY_MEDIUM_RID) {
            *IsTrusted = FALSE;
        }
        PsDereferencePrimaryToken(token);
    }

    return status;
}
```

### 6.5.2 예외 프로세스 처리

특정 시스템 프로세스는 예외 처리가 필요합니다:

```c
// 예외 프로세스 목록
// Exception process list
static const UNICODE_STRING TrustedProcesses[] = {
    RTL_CONSTANT_STRING(L"\\Windows\\System32\\svchost.exe"),
    RTL_CONSTANT_STRING(L"\\Windows\\System32\\services.exe"),
    RTL_CONSTANT_STRING(L"\\Windows\\explorer.exe"),
    // ... 신뢰할 수 있는 프로세스 목록
};

// 백업/복원 권한 확인
// Check backup/restore privilege
BOOLEAN HasBackupPrivilege(PACCESS_TOKEN Token)
{
    LUID backupLuid = { SE_BACKUP_PRIVILEGE, 0 };
    BOOLEAN hasPrivilege = FALSE;

    // SePrivilegeCheck 또는 유사 함수 사용
    // ...

    return hasPrivilege;
}
```

---

## 6.6 커널 보안 API 요약

### 주요 함수

```c
// 프로세스 토큰 얻기
// Get process token
PACCESS_TOKEN PsReferencePrimaryToken(PEPROCESS Process);
VOID PsDereferencePrimaryToken(PACCESS_TOKEN Token);

// 토큰 정보 조회
// Query token information
NTSTATUS SeQueryInformationToken(
    PACCESS_TOKEN Token,
    TOKEN_INFORMATION_CLASS TokenInformationClass,
    PVOID *TokenInformation
);

// 프로세스 이미지 이름 얻기
// Get process image name
NTSTATUS SeLocateProcessImageName(
    PEPROCESS Process,
    PUNICODE_STRING *ImageFileName
);

// SID 비교
// Compare SIDs
BOOLEAN RtlEqualSid(PSID Sid1, PSID Sid2);

// 접근 검사 (고급)
// Access check (advanced)
BOOLEAN SeAccessCheck(
    PSECURITY_DESCRIPTOR SecurityDescriptor,
    PSECURITY_SUBJECT_CONTEXT SubjectSecurityContext,
    BOOLEAN SubjectContextLocked,
    ACCESS_MASK DesiredAccess,
    ACCESS_MASK PreviouslyGrantedAccess,
    PPRIVILEGE_SET *Privileges,
    PGENERIC_MAPPING GenericMapping,
    KPROCESSOR_MODE AccessMode,
    PACCESS_MASK GrantedAccess,
    PNTSTATUS AccessStatus
);
```

### TOKEN_INFORMATION_CLASS 열거형

| 값 | 설명 |
|----|------|
| TokenUser | 사용자 SID |
| TokenGroups | 그룹 SID 목록 |
| TokenPrivileges | 권한 목록 |
| TokenOwner | 기본 소유자 SID |
| TokenIntegrityLevel | 무결성 수준 |
| TokenElevationType | 상승 유형 |
| TokenSessionId | 세션 ID |

---

## 정리

### 핵심 개념

| 개념 | 설명 |
|------|------|
| **SID** | 보안 주체의 고유 식별자 |
| **Access Token** | 프로세스/스레드의 보안 자격 증명 |
| **Security Descriptor** | 객체의 보안 정보 (소유자, ACL 등) |
| **DACL** | 접근 제어 목록 (누가 무엇을 할 수 있는지) |
| **SACL** | 감사 설정 |
| **Integrity Level** | 프로세스의 신뢰 수준 |

### DRM 구현 체크리스트

- [ ] 요청 프로세스의 이미지 이름 확인
- [ ] 프로세스의 무결성 수준 확인
- [ ] 필요시 디지털 서명 확인
- [ ] 예외 프로세스 목록 관리
- [ ] 백업/복원 권한 고려

### 다음 장 미리보기

Part 3에서는 개발 환경 구축을 다룹니다:
- Visual Studio 2022 + WDK 설치
- 테스트 VM 구성
- 커널 디버깅 연결 설정

---

## 연습 문제

### 1. SID 해석

다음 SID는 무엇을 나타내나요?
- S-1-5-18
- S-1-5-32-544

### 2. 무결성 수준

Medium 무결성 프로세스가 High 무결성 파일에 쓰기를 시도하면 어떻게 되나요?

### 3. 토큰 정보

프로세스가 관리자 그룹에 속하는지 확인하려면 어떤 TOKEN_INFORMATION_CLASS를 사용해야 하나요?

---

## 참고 자료

- [Access Control](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-control)
- [Security Identifiers](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/security-identifiers)
- [Mandatory Integrity Control](https://docs.microsoft.com/en-us/windows/win32/secauthz/mandatory-integrity-control)
- [Windows Security Model](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/windows-security-model)
