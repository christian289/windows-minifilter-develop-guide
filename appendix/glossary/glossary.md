# 용어집 (Glossary)

## Windows Minifilter 드라이버 개발 핵심 용어

이 용어집은 Windows Minifilter 드라이버 개발에서 사용되는 주요 용어를 알파벳 순으로 정리합니다. 각 용어에 대해 영문, 한글, 설명, C# 비유(해당되는 경우)를 제공합니다.

---

## A

### Access Control List (ACL)
**한글**: 접근 제어 목록
**설명**: 객체에 대한 접근 권한을 정의하는 목록. DACL(임의적 ACL)과 SACL(시스템 ACL)로 구분됩니다.
**C# 비유**: `FileSystemAccessRule`을 통한 파일 권한 설정

### Altitude
**한글**: 고도
**설명**: Minifilter 드라이버의 스택 내 위치를 결정하는 숫자 값. 높은 고도의 필터가 먼저 I/O 요청을 받습니다.
**범위**: 0 ~ 409999 (Microsoft 할당)
**예시**: 안티바이러스(320000-329999), 암호화(140000-149999)

### APC (Asynchronous Procedure Call)
**한글**: 비동기 프로시저 호출
**설명**: 특정 스레드 컨텍스트에서 비동기적으로 실행되는 함수 호출 메커니즘.
**IRQL**: APC_LEVEL (1)

---

## B

### Blue Screen of Death (BSOD)
**한글**: 블루 스크린, 죽음의 블루 스크린
**설명**: Windows 커널에서 복구 불가능한 오류 발생 시 표시되는 화면. 버그 체크(Bug Check)라고도 합니다.
**원인**: IRQL 위반, NULL 포인터 참조, 메모리 손상 등

### Buffer Pool
**한글**: 버퍼 풀
**설명**: 자주 사용되는 크기의 메모리를 미리 할당해 두는 메모리 관리 기법.
**C# 비유**: `ArrayPool<T>`

### Bug Check Code
**한글**: 버그 체크 코드
**설명**: BSOD 발생 시 원인을 나타내는 고유 코드.
**예시**: 0x0A (IRQL_NOT_LESS_OR_EQUAL), 0xD1 (DRIVER_IRQL_NOT_LESS_OR_EQUAL)

---

## C

### Callback
**한글**: 콜백
**설명**: 특정 이벤트 발생 시 시스템이 호출하는 드라이버 제공 함수.
**Minifilter 콜백**: PreOperation, PostOperation
**C# 비유**: 이벤트 핸들러, 대리자(Delegate)

### Context
**한글**: 컨텍스트
**설명**: 객체에 연결된 드라이버 정의 데이터 구조. 객체의 수명 동안 정보를 유지합니다.
**유형**: Volume, Instance, File, Stream, StreamHandle
**C# 비유**: `HttpContext.Items`, `AsyncLocal<T>`

### Context Switch
**한글**: 컨텍스트 스위치
**설명**: CPU가 한 스레드에서 다른 스레드로 실행을 전환하는 과정.

---

## D

### DACL (Discretionary Access Control List)
**한글**: 임의적 접근 제어 목록
**설명**: 객체에 대한 사용자/그룹별 접근 권한을 정의하는 ACL.

### Deadlock
**한글**: 교착 상태
**설명**: 두 개 이상의 스레드가 서로의 리소스를 기다리며 영구적으로 차단되는 상태.
**C# 비유**: `lock` 중첩으로 인한 교착

### Device Object
**한글**: 장치 객체
**설명**: 드라이버가 관리하는 하드웨어 또는 논리적 장치를 나타내는 커널 구조체.
**구조체**: DEVICE_OBJECT

### DPC (Deferred Procedure Call)
**한글**: 지연된 프로시저 호출
**설명**: 인터럽트 처리 후 더 낮은 IRQL에서 실행되는 루틴.
**IRQL**: DISPATCH_LEVEL (2)

### Driver Verifier
**한글**: 드라이버 검증기
**설명**: 드라이버의 잘못된 동작을 감지하는 Windows 내장 도구.
**기능**: 풀 추적, 특별 풀, IRQL 검사, 교착 상태 감지

### DRM (Digital Rights Management)
**한글**: 디지털 저작권 관리
**설명**: 디지털 콘텐츠의 사용을 제어하고 보호하는 기술.

---

## E

### EPROCESS
**한글**: Executive 프로세스
**설명**: Windows 커널에서 프로세스를 나타내는 내부 구조체.

### ERESOURCE
**한글**: Executive 리소스
**설명**: 커널 모드에서 사용하는 Reader-Writer 잠금 메커니즘.
**C# 비유**: `ReaderWriterLockSlim`

### ETHREAD
**한글**: Executive 스레드
**설명**: Windows 커널에서 스레드를 나타내는 내부 구조체.

### ETW (Event Tracing for Windows)
**한글**: Windows 이벤트 추적
**설명**: 고성능 이벤트 로깅 및 추적 인프라.
**C# 비유**: `EventSource`

---

## F

### Fast I/O
**한글**: 빠른 I/O
**설명**: IRP를 생성하지 않고 직접 파일 시스템 캐시에 접근하는 최적화 경로.

### File Object
**한글**: 파일 객체
**설명**: 열린 파일 인스턴스를 나타내는 커널 구조체.
**구조체**: FILE_OBJECT

### Filter Manager (FltMgr)
**한글**: 필터 관리자
**설명**: Minifilter 드라이버를 관리하고 I/O 요청을 라우팅하는 Microsoft 제공 시스템 드라이버.

### FLT_CALLBACK_DATA
**한글**: 필터 콜백 데이터
**설명**: Minifilter 콜백에 전달되는 I/O 작업 정보 구조체.
**C# 비유**: 이벤트 인자(EventArgs)

---

## G

### Global Descriptor Table (GDT)
**한글**: 전역 서술자 테이블
**설명**: x86/x64 프로세서에서 메모리 세그먼트를 정의하는 데이터 구조.

---

## H

### HAL (Hardware Abstraction Layer)
**한글**: 하드웨어 추상화 계층
**설명**: 하드웨어 차이를 숨기고 일관된 인터페이스를 제공하는 계층.

### Handle
**한글**: 핸들
**설명**: 커널 객체에 대한 사용자 모드 참조.
**C# 비유**: `SafeHandle`, `IntPtr`

### Hyper-V
**한글**: 하이퍼-V
**설명**: Microsoft의 Type 1 하이퍼바이저 기반 가상화 기술.

---

## I

### I/O Manager
**한글**: I/O 관리자
**설명**: Windows 커널에서 I/O 작업을 조율하는 핵심 구성 요소.

### I/O Stack Location
**한글**: I/O 스택 위치
**설명**: IRP 내에서 각 드라이버별 매개변수를 저장하는 구조.
**구조체**: IO_STACK_LOCATION

### INF (Information File)
**한글**: 정보 파일
**설명**: 드라이버 설치 정보를 담은 텍스트 파일.
**확장자**: .inf

### Instance
**한글**: 인스턴스
**설명**: 볼륨에 연결된 Minifilter의 개별 존재. 하나의 필터가 여러 볼륨에 연결되면 각각이 인스턴스입니다.

### IRP (I/O Request Packet)
**한글**: I/O 요청 패킷
**설명**: Windows 커널에서 I/O 작업을 나타내는 데이터 구조.
**C# 비유**: 메시지 객체, 요청 컨텍스트

### IRQL (Interrupt Request Level)
**한글**: 인터럽트 요청 레벨
**설명**: 현재 실행 코드의 우선순위를 나타내는 값. 높은 IRQL에서는 낮은 IRQL의 작업이 차단됩니다.
**레벨**:
- PASSIVE_LEVEL (0): 일반 스레드
- APC_LEVEL (1): APC 실행
- DISPATCH_LEVEL (2): DPC, 스핀락
- DIRQL (3+): 인터럽트 서비스

---

## K

### KMDF (Kernel-Mode Driver Framework)
**한글**: 커널 모드 드라이버 프레임워크
**설명**: WDF(Windows Driver Framework)의 커널 모드 버전으로, 드라이버 개발을 단순화합니다.

### Kernel Mode
**한글**: 커널 모드
**설명**: 시스템 리소스에 완전한 접근 권한을 가진 실행 모드.
**Ring**: Ring 0
**C# 비유**: 해당 없음 (C#은 사용자 모드에서만 실행)

---

## L

### Legacy Filter
**한글**: 레거시 필터
**설명**: Minifilter 이전의 전통적인 파일 시스템 필터 드라이버 모델.
**대체**: Minifilter로 대체 권장

### Lookaside List
**한글**: Lookaside 리스트
**설명**: 고정 크기 메모리 블록의 빠른 할당/해제를 위한 캐시.
**유형**: Paged, NonPaged
**C# 비유**: `ObjectPool<T>`

---

## M

### Major Function
**한글**: 주요 함수
**설명**: IRP의 작업 유형을 나타내는 코드.
**예시**: IRP_MJ_CREATE (0x00), IRP_MJ_READ (0x03), IRP_MJ_WRITE (0x04)

### MDL (Memory Descriptor List)
**한글**: 메모리 설명자 목록
**설명**: 물리 메모리 페이지를 설명하는 구조체. DMA 및 직접 I/O에 사용됩니다.

### Minifilter
**한글**: 미니필터
**설명**: Filter Manager를 통해 등록되는 최신 파일 시스템 필터 드라이버 모델.
**장점**: 간단한 개발, 안정성, 이식성

### Minor Function
**한글**: 부 함수
**설명**: Major Function의 하위 작업을 나타내는 코드.
**예시**: IRP_MN_QUERY_DIRECTORY

---

## N

### NonPaged Pool
**한글**: 비페이지 풀
**설명**: 항상 물리 메모리에 상주하는 커널 메모리 풀. 어떤 IRQL에서도 접근 가능합니다.
**사용**: DISPATCH_LEVEL 이상에서 접근해야 하는 데이터

### NTSTATUS
**한글**: NT 상태
**설명**: Windows 커널 API의 표준 반환 값 유형. 32비트 값으로 성공/실패와 상세 정보를 포함합니다.
**C# 비유**: `HRESULT`, 예외 코드

---

## O

### Object Manager
**한글**: 객체 관리자
**설명**: Windows 커널 객체를 생성, 관리, 삭제하는 구성 요소.

### Oplock (Opportunistic Lock)
**한글**: 기회적 잠금
**설명**: 네트워크 파일 시스템에서 캐싱 최적화를 위한 잠금 메커니즘.

---

## P

### Paged Pool
**한글**: 페이지 풀
**설명**: 필요 시 디스크로 스왑될 수 있는 커널 메모리 풀. PASSIVE_LEVEL에서만 접근해야 합니다.

### PnP (Plug and Play)
**한글**: 플러그 앤 플레이
**설명**: 하드웨어 자동 감지 및 구성 기술.

### Pool Tag
**한글**: 풀 태그
**설명**: 메모리 할당을 식별하는 4바이트 값. 메모리 누수 디버깅에 사용됩니다.
**예시**: 'Drm ' = 0x206D7244

### PostOperation Callback
**한글**: 후처리 콜백
**설명**: I/O 작업이 완료된 후 호출되는 Minifilter 콜백.
**반환값**: FLT_POSTOP_CALLBACK_STATUS

### PreOperation Callback
**한글**: 전처리 콜백
**설명**: I/O 작업이 하위 스택으로 전달되기 전에 호출되는 Minifilter 콜백.
**반환값**: FLT_PREOP_CALLBACK_STATUS

---

## R

### Reference Count
**한글**: 참조 횟수
**설명**: 객체가 참조되는 횟수. 0이 되면 객체가 해제됩니다.
**C# 비유**: 참조 카운팅 (C#은 GC 사용)

### Reparse Point
**한글**: 리파스 포인트
**설명**: 파일 시스템 경로를 다른 위치로 리다이렉트하는 메커니즘.
**용도**: 심볼릭 링크, 마운트 포인트, OneDrive 등

---

## S

### SACL (System Access Control List)
**한글**: 시스템 접근 제어 목록
**설명**: 객체 접근에 대한 감사 정책을 정의하는 ACL.

### Security Descriptor
**한글**: 보안 설명자
**설명**: 객체의 소유자, ACL 등 보안 정보를 포함하는 구조체.

### Spin Lock
**한글**: 스핀 락
**설명**: 락을 얻을 때까지 CPU를 점유하며 대기하는 동기화 메커니즘.
**IRQL**: DISPATCH_LEVEL로 상승
**C# 비유**: `SpinLock`

### Stream
**한글**: 스트림
**설명**: NTFS에서 파일 데이터의 개별 흐름. 기본 데이터 스트림($DATA) 외에 대체 데이터 스트림(ADS)을 가질 수 있습니다.

### SSN (Social Security Number)
**한글**: 주민등록번호 (한국), 사회보장번호 (미국)
**설명**: 개인 식별 번호. DRM/DLP에서 민감 정보로 보호됩니다.
**한국 패턴**: XXXXXX-XXXXXXX

---

## T

### TDI (Transport Driver Interface)
**한글**: 전송 드라이버 인터페이스
**설명**: 네트워크 프로토콜 드라이버 인터페이스 (레거시).
**대체**: WFP(Windows Filtering Platform)

---

## U

### UMDF (User-Mode Driver Framework)
**한글**: 사용자 모드 드라이버 프레임워크
**설명**: 사용자 모드에서 실행되는 드라이버를 위한 프레임워크.

### User Mode
**한글**: 사용자 모드
**설명**: 제한된 시스템 리소스 접근 권한을 가진 실행 모드.
**Ring**: Ring 3
**C# 비유**: 모든 .NET 애플리케이션

---

## V

### Virtual Memory
**한글**: 가상 메모리
**설명**: 물리 메모리와 디스크를 조합하여 제공되는 추상화된 메모리 공간.

### Volume
**한글**: 볼륨
**설명**: 파일 시스템이 포맷된 저장 장치 파티션.
**예시**: C:, D:

---

## W

### WDK (Windows Driver Kit)
**한글**: Windows 드라이버 키트
**설명**: Windows 드라이버 개발에 필요한 도구, 라이브러리, 문서 모음.

### WDM (Windows Driver Model)
**한글**: Windows 드라이버 모델
**설명**: Windows 2000부터 사용된 커널 모드 드라이버 아키텍처.

### WHQL (Windows Hardware Quality Labs)
**한글**: Windows 하드웨어 품질 연구소
**설명**: Microsoft의 드라이버/하드웨어 인증 프로그램.
**인증**: 디지털 서명, 호환성 검증

### WinDbg
**한글**: 윈도우 디버거
**설명**: Microsoft의 커널 및 사용자 모드 디버거.

### Work Item
**한글**: 작업 항목
**설명**: 시스템 스레드에서 비동기로 실행되도록 큐에 추가되는 작업.
**C# 비유**: `Task`, `ThreadPool.QueueUserWorkItem`

---

## Z

### Zero-Copy
**한글**: 제로 카피
**설명**: 데이터 복사 없이 버퍼를 직접 공유하는 최적화 기법.

### ZwXxx Functions
**한글**: Zw 함수
**설명**: 커널 모드에서 네이티브 시스템 서비스를 호출하는 함수.
**예시**: ZwCreateFile, ZwReadFile, ZwClose
**차이**: NtXxx (사용자 모드) vs ZwXxx (커널 모드)

---

## 약어 모음

| 약어 | 전체 명칭 | 한글 |
|------|----------|------|
| ACL | Access Control List | 접근 제어 목록 |
| APC | Asynchronous Procedure Call | 비동기 프로시저 호출 |
| BSOD | Blue Screen of Death | 블루 스크린 |
| DPC | Deferred Procedure Call | 지연된 프로시저 호출 |
| DRM | Digital Rights Management | 디지털 저작권 관리 |
| ETW | Event Tracing for Windows | Windows 이벤트 추적 |
| HAL | Hardware Abstraction Layer | 하드웨어 추상화 계층 |
| IRP | I/O Request Packet | I/O 요청 패킷 |
| IRQL | Interrupt Request Level | 인터럽트 요청 레벨 |
| MDL | Memory Descriptor List | 메모리 설명자 목록 |
| PnP | Plug and Play | 플러그 앤 플레이 |
| WDK | Windows Driver Kit | Windows 드라이버 키트 |
| WDM | Windows Driver Model | Windows 드라이버 모델 |
| WHQL | Windows Hardware Quality Labs | Windows 하드웨어 품질 연구소 |

---

## FLT_PREOP 반환값

| 반환값 | 설명 |
|--------|------|
| FLT_PREOP_SUCCESS_WITH_CALLBACK | 계속 진행, PostOperation 콜백 호출 |
| FLT_PREOP_SUCCESS_NO_CALLBACK | 계속 진행, PostOperation 콜백 안함 |
| FLT_PREOP_PENDING | 작업 보류 (비동기 처리) |
| FLT_PREOP_COMPLETE | 즉시 완료 (하위로 전달 안함) |
| FLT_PREOP_SYNCHRONIZE | 동기화 필요 (PostOperation이 원래 스레드에서) |
| FLT_PREOP_DISALLOW_FASTIO | Fast I/O 거부 (IRP로 재시도) |

---

## FLT_POSTOP 반환값

| 반환값 | 설명 |
|--------|------|
| FLT_POSTOP_FINISHED_PROCESSING | 처리 완료 |
| FLT_POSTOP_MORE_PROCESSING_REQUIRED | 추가 처리 필요 (비동기) |

---

이 용어집은 Minifilter 드라이버 개발에서 자주 접하는 용어를 이해하는 데 도움이 됩니다. 새로운 개념을 만날 때마다 이 용어집을 참조하세요.
