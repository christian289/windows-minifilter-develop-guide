# Chapter 11: 커널 구조체 분석 실습

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- EPROCESS, ETHREAD 구조체로 프로세스/스레드 정보를 분석합니다
- DRIVER_OBJECT로 드라이버 구조를 이해합니다
- IRP 구조체로 I/O 요청 흐름을 추적합니다
- FLT_CALLBACK_DATA로 Minifilter 콜백 데이터를 분석합니다
- 실제 디버깅에서 구조체 정보를 활용합니다

## 도입: 구조체는 커널의 언어

Windows 커널은 모든 것을 **구조체**로 표현합니다. 프로세스는 EPROCESS, 스레드는 ETHREAD, I/O 요청은 IRP입니다. .NET 개발자에게 이것은 마치 Entity Framework의 엔티티와 같습니다. 구조체를 읽을 수 있으면 커널의 동작을 이해할 수 있습니다.

```csharp
// C# 비유
public class Process  // ≈ EPROCESS
{
    public int ProcessId { get; }
    public string ImageName { get; }
    public List<Thread> Threads { get; }
}

public class Thread  // ≈ ETHREAD
{
    public int ThreadId { get; }
    public Process OwningProcess { get; }
    public ThreadState State { get; }
}
```

---

## 11.1 프로세스 구조체: EPROCESS

### 11.1.1 EPROCESS 개요

EPROCESS(Executive Process)는 Windows 커널에서 프로세스를 표현하는 가장 핵심적인 구조체입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         EPROCESS 구조체                              │
├─────────────────────────────────────────────────────────────────────┤
│  구성 요소              설명                                         │
├─────────────────────────────────────────────────────────────────────┤
│  KPROCESS (Pcb)         스케줄러가 사용하는 커널 프로세스 블록      │
│  UniqueProcessId        프로세스 ID (PID)                           │
│  ActiveProcessLinks     전역 프로세스 리스트 연결                   │
│  ImageFileName          실행 파일 이름 (15자 제한)                  │
│  Peb                    Process Environment Block (사용자 모드)     │
│  Token                  보안 토큰                                   │
│  ThreadListHead         프로세스 내 스레드 리스트                   │
│  VadRoot                가상 주소 디스크립터 트리 (메모리 맵)       │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.1.2 EPROCESS 구조체 탐색

```
// EPROCESS 정의 보기
0: kd> dt nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : Ptr64 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY
   +0x458 RundownProtect   : _EX_RUNDOWN_REF
   +0x460 Flags2           : Uint4B
   +0x464 Flags            : Uint4B
   +0x468 CreateTime       : _LARGE_INTEGER
   +0x470 ProcessQuotaUsage : [2] Uint8B
   +0x480 ProcessQuotaPeak : [2] Uint8B
   +0x490 PeakVirtualSize  : Uint8B
   +0x498 VirtualSize      : Uint8B
   +0x4a0 SessionProcessLinks : _LIST_ENTRY
   +0x4b0 ExceptionPortData : Ptr64 Void
   +0x4b8 Token            : _EX_FAST_REF
   ...
   +0x5a8 ImageFileName    : [15] UChar
   ...
```

### 11.1.3 현재 프로세스 분석하기

```
// 현재 프로세스 확인
0: kd> !process -1 0
PROCESS ffff8a0123456000
    SessionId: 1  Cid: 1234    Peb: 000007ff`fffde000  ParentCid: 0abc
    DirBase: 12345678  ObjectTable: fffffa80`12345678  HandleCount: 200
    Image: notepad.exe

// EPROCESS 주소 획득: ffff8a0123456000

// 구조체 상세 보기
0: kd> dt nt!_EPROCESS ffff8a0123456000
   +0x000 Pcb              : _KPROCESS
   +0x440 UniqueProcessId  : 0x00000000`00001234 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY [ 0xffff8a01`23457448 - 0xffff8a01`23455448 ]
   +0x468 CreateTime       : _LARGE_INTEGER 0x01d9abcd`12345678
   +0x4b8 Token            : _EX_FAST_REF
   +0x5a8 ImageFileName    : [15]  "notepad.exe"
   ...

// 특정 필드만 보기
0: kd> dt nt!_EPROCESS ffff8a0123456000 UniqueProcessId ImageFileName
   +0x440 UniqueProcessId : 0x00000000`00001234 Void
   +0x5a8 ImageFileName   : [15]  "notepad.exe"
```

### 11.1.4 프로세스 리스트 순회

Windows는 모든 프로세스를 이중 연결 리스트(doubly-linked list)로 관리합니다.

```
// ActiveProcessLinks를 통해 프로세스 순회
0: kd> dt nt!_EPROCESS ffff8a0123456000 ActiveProcessLinks
   +0x448 ActiveProcessLinks : _LIST_ENTRY [ 0xffff8a01`23457448 - 0xffff8a01`23455448 ]

// Flink(Forward Link) = 다음 프로세스
// Blink(Backward Link) = 이전 프로세스

// 다음 프로세스 EPROCESS 주소 계산
// Flink 주소 - ActiveProcessLinks 오프셋(0x448) = EPROCESS 시작 주소
0: kd> ? 0xffff8a01`23457448 - 0x448
Evaluate expression: -8108795674512 = ffff8a01`23457000

// 다음 프로세스 확인
0: kd> dt nt!_EPROCESS ffff8a01`23457000 ImageFileName
   +0x5a8 ImageFileName : [15]  "explorer.exe"
```

```c
// C 코드로 표현한 프로세스 순회
// 커널 드라이버에서 모든 프로세스 열거하기

// 방법 1: PsGetNextProcess (권장)
PEPROCESS Process = NULL;
while ((Process = PsGetNextProcess(Process)) != NULL) {
    // Process 사용
    HANDLE pid = PsGetProcessId(Process);
    // ...
}

// 방법 2: ActiveProcessLinks 순회 (수동)
// 비추천 - 레이스 컨디션 위험
```

### 11.1.5 프로세스 관련 유용한 명령어

```
// 모든 프로세스 목록
!process 0 0

// 특정 이름의 프로세스 찾기
!process 0 0 notepad.exe

// 프로세스 상세 정보 (레벨 0-7)
!process ffff8a0123456000 7     // 최대 상세 정보

// 프로세스의 핸들 테이블
!handle 0 3 ffff8a0123456000    // 모든 핸들 상세 정보

// 프로세스의 보안 토큰
!token ffff8a0123456000         // 토큰 정보 (주소 필요)
```

---

## 11.2 스레드 구조체: ETHREAD

### 11.2.1 ETHREAD 개요

ETHREAD(Executive Thread)는 스레드를 표현합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ETHREAD 구조체                               │
├─────────────────────────────────────────────────────────────────────┤
│  구성 요소              설명                                         │
├─────────────────────────────────────────────────────────────────────┤
│  KTHREAD (Tcb)          스케줄러가 사용하는 커널 스레드 블록        │
│  Cid.UniqueProcess      소유 프로세스 ID                            │
│  Cid.UniqueThread       스레드 ID                                   │
│  ThreadsProcess         소유 EPROCESS 포인터                        │
│  StartAddress           스레드 시작 주소                            │
│  Win32StartAddress      사용자 모드 시작 주소                       │
│  Teb                    Thread Environment Block                    │
│  IrpList                대기 중인 IRP 리스트                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2.2 ETHREAD 구조체 탐색

```
// ETHREAD 정의 보기
0: kd> dt nt!_ETHREAD
   +0x000 Tcb              : _KTHREAD
   +0x5e0 CreateTime       : _LARGE_INTEGER
   +0x5e8 ExitTime         : _LARGE_INTEGER
   +0x5f8 PostBlockList    : _LIST_ENTRY
   +0x608 TerminationPort  : Ptr64 _TERMINATION_PORT
   +0x610 ActiveTimerListLock : Uint8B
   +0x618 ActiveTimerListHead : _LIST_ENTRY
   +0x628 Cid              : _CLIENT_ID
   +0x638 KeyedWaitSemaphore : _KSEMAPHORE
   +0x658 ClientSecurity   : _PS_CLIENT_SECURITY_CONTEXT
   +0x660 IrpList          : _LIST_ENTRY
   +0x670 TopLevelIrp      : Uint8B
   ...

// _CLIENT_ID 구조
0: kd> dt nt!_CLIENT_ID
   +0x000 UniqueProcess    : Ptr64 Void
   +0x008 UniqueThread     : Ptr64 Void
```

### 11.2.3 현재 스레드 분석

```
// 현재 스레드 확인
0: kd> !thread -1 0
THREAD ffff8a0123456780  Cid 1234.5678  Teb: 000000007ff8e000 Win32Thread: 0000000000000000 RUNNING on processor 0
Not impersonating
DeviceMap                 fffffa8012345678
Owning Process            ffff8a0123456000       Image:         notepad.exe
Attached Process          N/A            Image:         N/A
Wait Start TickCount      12345678       Ticks: 1234
Context Switch Count      5678           IdealProcessor: 0
UserTime                  00:00:00.015
KernelTime                00:00:00.031
...

// ETHREAD 주소: ffff8a0123456780

// 구조체 상세 보기
0: kd> dt nt!_ETHREAD ffff8a0123456780
   +0x628 Cid              : _CLIENT_ID
   +0x648 ThreadsProcess   : 0xffff8a01`23456000 _EPROCESS
   ...

// 스레드가 속한 프로세스 확인
0: kd> dt nt!_ETHREAD ffff8a0123456780 ThreadsProcess
   +0x648 ThreadsProcess : 0xffff8a01`23456000 _EPROCESS

0: kd> dt nt!_EPROCESS 0xffff8a01`23456000 ImageFileName
   +0x5a8 ImageFileName : [15]  "notepad.exe"
```

### 11.2.4 KTHREAD의 중요 필드

ETHREAD의 첫 번째 멤버인 KTHREAD(Kernel Thread)에는 스케줄링 정보가 있습니다.

```
// KTHREAD 주요 필드
0: kd> dt nt!_KTHREAD
   +0x000 Header           : _DISPATCHER_HEADER
   +0x018 CycleTime        : Uint8B
   +0x098 InitialStack     : Ptr64 Void
   +0x0a0 StackLimit       : Ptr64 Void
   +0x0a8 StackBase        : Ptr64 Void
   +0x0b0 ThreadLock       : Uint8B
   +0x184 State            : UChar                 // 스레드 상태
   +0x1c4 Priority         : Char                  // 우선순위
   +0x1c5 BasePriority     : Char
   +0x25c WaitReason       : UChar                 // 대기 이유
   ...

// 스레드 상태 값
// 0 = Initialized, 1 = Ready, 2 = Running, 3 = Standby
// 4 = Terminated, 5 = Waiting, 6 = Transition, 7 = DeferredReady

// 스레드 상태 확인
0: kd> dt nt!_KTHREAD ffff8a0123456780 State
   +0x184 State : 0x2 ''    // 2 = Running
```

---

## 11.3 드라이버 구조체: DRIVER_OBJECT

### 11.3.1 DRIVER_OBJECT 개요

DRIVER_OBJECT는 로드된 드라이버를 표현합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       DRIVER_OBJECT 구조체                           │
├─────────────────────────────────────────────────────────────────────┤
│  필드                   설명                                         │
├─────────────────────────────────────────────────────────────────────┤
│  DriverName             드라이버 이름 (\Driver\MyDriver)            │
│  DeviceObject           디바이스 오브젝트 체인                      │
│  MajorFunction[]        IRP 디스패치 루틴 배열 (28개)               │
│  DriverExtension        드라이버 확장 정보                          │
│  DriverStart            드라이버 메모리 시작 주소                   │
│  DriverSize             드라이버 크기                               │
│  DriverUnload           언로드 콜백                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.3.2 DRIVER_OBJECT 구조체 탐색

```
// DRIVER_OBJECT 정의 보기
0: kd> dt nt!_DRIVER_OBJECT
   +0x000 Type             : Int2B
   +0x002 Size             : Int2B
   +0x008 DeviceObject     : Ptr64 _DEVICE_OBJECT
   +0x010 Flags            : Uint4B
   +0x018 DriverStart      : Ptr64 Void
   +0x020 DriverSize       : Uint4B
   +0x028 DriverSection    : Ptr64 Void
   +0x030 DriverExtension  : Ptr64 _DRIVER_EXTENSION
   +0x038 DriverName       : _UNICODE_STRING
   +0x048 HardwareDatabase : Ptr64 _UNICODE_STRING
   +0x050 FastIoDispatch   : Ptr64 _FAST_IO_DISPATCH
   +0x058 DriverInit       : Ptr64     long
   +0x060 DriverStartIo    : Ptr64     void
   +0x068 DriverUnload     : Ptr64     void
   +0x070 MajorFunction    : [28] Ptr64     long
```

### 11.3.3 드라이버 오브젝트 찾기

```
// 드라이버 목록 확인
0: kd> !drvobj \Driver\Ntfs 7
Driver object (fffff80123456000) is for:
 \Driver\Ntfs
Driver Extension List: (id , addr)

Device Object list:
fffff80123457000
...

// 특정 드라이버 객체 분석
0: kd> dt nt!_DRIVER_OBJECT fffff80123456000
   +0x018 DriverStart      : 0xfffff801`12340000 Void
   +0x020 DriverSize       : 0x50000
   +0x038 DriverName       : _UNICODE_STRING "\Driver\Ntfs"
   +0x068 DriverUnload     : 0xfffff801`12345000     void  +0
   +0x070 MajorFunction    : [28] 0xfffff801`12341000     long  +0

// 드라이버 이름 출력
0: kd> dS fffff80123456000+0x38
"\Driver\Ntfs"
```

### 11.3.4 MajorFunction 배열 분석

드라이버의 IRP 핸들러를 확인할 수 있습니다.

```
// MajorFunction 배열 확인 (IRP 핸들러)
0: kd> dqs fffff80123456000+0x70 L1c
fffff801`23456070  fffff801`12341000 Ntfs!NtfsFsdCreate        // IRP_MJ_CREATE
fffff801`23456078  fffff801`12342000 Ntfs!NtfsFsdClose         // IRP_MJ_CLOSE
fffff801`23456080  fffff801`12343000 Ntfs!NtfsFsdRead          // IRP_MJ_READ
fffff801`23456088  fffff801`12344000 Ntfs!NtfsFsdWrite         // IRP_MJ_WRITE
...

// IRP_MJ_XXX 인덱스
// 0x00 = IRP_MJ_CREATE
// 0x01 = IRP_MJ_CREATE_NAMED_PIPE
// 0x02 = IRP_MJ_CLOSE
// 0x03 = IRP_MJ_READ
// 0x04 = IRP_MJ_WRITE
// ...
```

```c
// C 코드에서 MajorFunction 설정
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    // 모든 IRP를 기본 핸들러로 초기화
    for (int i = 0; i < IRP_MJ_MAXIMUM_FUNCTION; i++) {
        DriverObject->MajorFunction[i] = DefaultDispatch;
    }

    // 필요한 IRP만 커스텀 핸들러 설정
    DriverObject->MajorFunction[IRP_MJ_CREATE] = MyCreateDispatch;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = MyCloseDispatch;
    DriverObject->MajorFunction[IRP_MJ_READ] = MyReadDispatch;

    return STATUS_SUCCESS;
}
```

---

## 11.4 I/O 요청 구조체: IRP

### 11.4.1 IRP 개요

IRP(I/O Request Packet)는 모든 I/O 작업을 표현합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                           IRP 구조체                                 │
├─────────────────────────────────────────────────────────────────────┤
│  필드                   설명                                         │
├─────────────────────────────────────────────────────────────────────┤
│  MdlAddress             Memory Descriptor List (버퍼)               │
│  Flags                  IRP 플래그                                  │
│  IoStatus               I/O 완료 상태                               │
│  RequestorMode          요청 모드 (User/Kernel)                     │
│  Tail.Overlay.Thread    요청 스레드                                 │
│  UserBuffer             사용자 버퍼 (Direct I/O)                    │
│  CurrentLocation        현재 스택 위치                              │
│  StackCount             총 스택 수                                  │
│  Stack[]                IO_STACK_LOCATION 배열                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.4.2 IRP 구조체 탐색

```
// IRP 정의 보기
0: kd> dt nt!_IRP
   +0x000 Type             : Int2B
   +0x002 Size             : Uint2B
   +0x008 MdlAddress       : Ptr64 _MDL
   +0x010 Flags            : Uint4B
   +0x018 AssociatedIrp    : <anonymous-tag>
   +0x020 ThreadListEntry  : _LIST_ENTRY
   +0x030 IoStatus         : _IO_STATUS_BLOCK
   +0x040 RequestorMode    : Char
   +0x041 PendingReturned  : UChar
   +0x042 StackCount       : Char
   +0x043 CurrentLocation  : Char
   +0x044 Cancel           : UChar
   +0x045 CancelIrql       : UChar
   +0x048 UserBuffer       : Ptr64 Void
   +0x050 Tail             : <anonymous-tag>
```

### 11.4.3 IRP 분석 실습

```
// IRP 분석 (!irp 확장 명령어)
0: kd> !irp ffff8a0123457000
Irp is active with 5 stacks 3 is current (= 0xffff8a0123457138)
 No Mdl: No System Buffer: Thread ffff8a0123456780:  Irp stack trace.
     cmd  flg cl Device   File     Completion-Context
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000

            Args: 00000000 00000000 00000000 00000000
 [N/A(0), N/A(0)]
            0  0 00000000 00000000 00000000-00000000

            Args: 00000000 00000000 00000000 00000000
>[IRP_MJ_CREATE(0), N/A(0)]
            0  1 ffff8a01234580a0 ffff8a0123459000 00000000-00000000
                       \FileSystem\Ntfs
            Args: 00000000 01200000 00000000 00000000
...

// 출력 해석:
// - 5개 스택, 현재 3번째 위치
// - IRP_MJ_CREATE 요청
// - \FileSystem\Ntfs 디바이스로 전달 중
```

### 11.4.4 IO_STACK_LOCATION 분석

각 스택 레벨에는 IO_STACK_LOCATION이 있습니다.

```
// IO_STACK_LOCATION 정의
0: kd> dt nt!_IO_STACK_LOCATION
   +0x000 MajorFunction    : UChar
   +0x001 MinorFunction    : UChar
   +0x002 Flags            : UChar
   +0x003 Control          : UChar
   +0x008 Parameters       : <anonymous-tag>
   +0x028 DeviceObject     : Ptr64 _DEVICE_OBJECT
   +0x030 FileObject       : Ptr64 _FILE_OBJECT
   +0x038 CompletionRoutine : Ptr64     long
   +0x040 Context          : Ptr64 Void

// 현재 스택 위치 분석
0: kd> dt nt!_IO_STACK_LOCATION ffff8a0123457138
   +0x000 MajorFunction    : 0 ''          // IRP_MJ_CREATE
   +0x001 MinorFunction    : 0 ''
   +0x028 DeviceObject     : 0xffff8a01`234580a0 _DEVICE_OBJECT
   +0x030 FileObject       : 0xffff8a01`23459000 _FILE_OBJECT
```

### 11.4.5 IRP 플래그 해석

```
// IRP Flags 확인
0: kd> dt nt!_IRP ffff8a0123457000 Flags
   +0x010 Flags : 0x60000

// 주요 플래그 값:
// 0x00000001  IRP_NOCACHE             - 캐시 우회
// 0x00000002  IRP_PAGING_IO           - 페이징 I/O
// 0x00000004  IRP_MOUNT_COMPLETION    - 마운트 완료
// 0x00000008  IRP_SYNCHRONOUS_API     - 동기 API
// 0x00000010  IRP_ASSOCIATED_IRP      - 연관 IRP
// 0x00000020  IRP_BUFFERED_IO         - 버퍼드 I/O
// 0x00000040  IRP_DEALLOCATE_BUFFER   - 버퍼 해제 필요
// 0x00020000  IRP_CREATE_OPERATION    - 생성 작업
// 0x00040000  IRP_READ_OPERATION      - 읽기 작업
// 0x00080000  IRP_WRITE_OPERATION     - 쓰기 작업
```

---

## 11.5 파일 구조체: FILE_OBJECT

### 11.5.1 FILE_OBJECT 개요

FILE_OBJECT는 열린 파일 인스턴스를 표현합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       FILE_OBJECT 구조체                             │
├─────────────────────────────────────────────────────────────────────┤
│  필드                   설명                                         │
├─────────────────────────────────────────────────────────────────────┤
│  DeviceObject           파일이 속한 디바이스                        │
│  Vpb                    Volume Parameter Block                      │
│  FsContext              파일 시스템 컨텍스트 (FCB)                  │
│  FsContext2             파일 시스템 컨텍스트 2 (CCB)                │
│  FileName               파일 경로                                   │
│  CurrentByteOffset      현재 파일 위치                              │
│  Flags                  파일 플래그                                 │
│  ReadAccess             읽기 권한 여부                              │
│  WriteAccess            쓰기 권한 여부                              │
│  DeleteAccess           삭제 권한 여부                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.5.2 FILE_OBJECT 구조체 탐색

```
// FILE_OBJECT 정의
0: kd> dt nt!_FILE_OBJECT
   +0x000 Type             : Int2B
   +0x002 Size             : Int2B
   +0x008 DeviceObject     : Ptr64 _DEVICE_OBJECT
   +0x010 Vpb              : Ptr64 _VPB
   +0x018 FsContext        : Ptr64 Void
   +0x020 FsContext2       : Ptr64 Void
   +0x028 SectionObjectPointer : Ptr64 _SECTION_OBJECT_POINTERS
   +0x030 PrivateCacheMap  : Ptr64 Void
   +0x038 FinalStatus      : Int4B
   +0x040 RelatedFileObject : Ptr64 _FILE_OBJECT
   +0x048 LockOperation    : UChar
   +0x049 DeletePending    : UChar
   +0x04a ReadAccess       : UChar
   +0x04b WriteAccess      : UChar
   +0x04c DeleteAccess     : UChar
   +0x04d SharedRead       : UChar
   +0x04e SharedWrite      : UChar
   +0x04f SharedDelete     : UChar
   +0x050 Flags            : Uint4B
   +0x058 FileName         : _UNICODE_STRING
   +0x068 CurrentByteOffset : _LARGE_INTEGER
   ...
```

### 11.5.3 파일 이름 확인

```
// FILE_OBJECT에서 파일 이름 추출
0: kd> dt nt!_FILE_OBJECT ffff8a0123459000 FileName
   +0x058 FileName : _UNICODE_STRING "\Users\User\Documents\test.txt"

// 또는 직접 UNICODE_STRING 읽기
0: kd> dS ffff8a0123459000+0x58
"\Users\User\Documents\test.txt"

// du 명령어로 읽기 (Buffer 주소 필요)
0: kd> dt nt!_UNICODE_STRING ffff8a0123459000+0x58
   +0x000 Length           : 0x42
   +0x002 MaximumLength    : 0x44
   +0x008 Buffer           : 0xffff8a01`23459100  "\Users\User\Documents\test.txt"

0: kd> du 0xffff8a01`23459100
ffff8a01`23459100  "\Users\User\Documents\test.txt"
```

### 11.5.4 파일 접근 권한 확인

```
// 접근 권한 확인
0: kd> dt nt!_FILE_OBJECT ffff8a0123459000 ReadAccess WriteAccess DeleteAccess
   +0x04a ReadAccess  : 0x1 ''     // 읽기 권한 있음
   +0x04b WriteAccess : 0x1 ''     // 쓰기 권한 있음
   +0x04c DeleteAccess : 0 ''      // 삭제 권한 없음

// 공유 모드 확인
0: kd> dt nt!_FILE_OBJECT ffff8a0123459000 SharedRead SharedWrite SharedDelete
   +0x04d SharedRead   : 0x1 ''    // 읽기 공유 허용
   +0x04e SharedWrite  : 0 ''      // 쓰기 공유 불허
   +0x04f SharedDelete : 0 ''      // 삭제 공유 불허
```

---

## 11.6 Minifilter 구조체: FLT_CALLBACK_DATA

### 11.6.1 FLT_CALLBACK_DATA 개요

FLT_CALLBACK_DATA는 Minifilter 콜백에서 사용되는 핵심 구조체입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   FLT_CALLBACK_DATA 구조체                           │
├─────────────────────────────────────────────────────────────────────┤
│  필드                   설명                                         │
├─────────────────────────────────────────────────────────────────────┤
│  Flags                  콜백 플래그                                 │
│  Thread                 요청 스레드                                 │
│  Iopb                   I/O Parameter Block (핵심!)                 │
│  IoStatus               I/O 완료 상태                               │
│  TagData                재파싱 정보                                 │
│  RequestorMode          요청 모드 (User/Kernel)                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.6.2 FLT_CALLBACK_DATA 분석

```
// FLT_CALLBACK_DATA 정의
0: kd> dt fltmgr!_FLT_CALLBACK_DATA
   +0x000 Flags            : Uint4B
   +0x008 Thread           : Ptr64 _KTHREAD
   +0x010 Iopb             : Ptr64 _FLT_IO_PARAMETER_BLOCK
   +0x018 IoStatus         : _IO_STATUS_BLOCK
   +0x028 TagData          : Ptr64 _FLT_TAG_DATA_BUFFER
   +0x030 QueueLinks       : _LIST_ENTRY
   +0x040 QueueContext     : [2] Ptr64 Void
   +0x050 FilterContext    : [4] Ptr64 Void
   +0x070 RequestorMode    : Char
   ...
```

### 11.6.3 FLT_IO_PARAMETER_BLOCK (Iopb) 분석

Iopb는 실제 I/O 매개변수를 담고 있습니다.

```
// FLT_IO_PARAMETER_BLOCK 정의
0: kd> dt fltmgr!_FLT_IO_PARAMETER_BLOCK
   +0x000 IrpFlags         : Uint4B
   +0x004 MajorFunction    : UChar
   +0x005 MinorFunction    : UChar
   +0x006 OperationFlags   : UChar
   +0x008 TargetFileObject : Ptr64 _FILE_OBJECT
   +0x010 TargetInstance   : Ptr64 _FLT_INSTANCE
   +0x018 Parameters       : _FLT_PARAMETERS

// 실제 분석 예
0: kd> dt fltmgr!_FLT_CALLBACK_DATA ffff8a012345a000 Iopb
   +0x010 Iopb : 0xffff8a01`2345a100 _FLT_IO_PARAMETER_BLOCK

0: kd> dt fltmgr!_FLT_IO_PARAMETER_BLOCK 0xffff8a01`2345a100
   +0x004 MajorFunction    : 0 ''          // IRP_MJ_CREATE
   +0x008 TargetFileObject : 0xffff8a01`23459000 _FILE_OBJECT
   +0x018 Parameters       : _FLT_PARAMETERS
```

### 11.6.4 FLT_PARAMETERS 분석

FLT_PARAMETERS는 유니온으로, MajorFunction에 따라 다른 구조체를 사용합니다.

```
// FLT_PARAMETERS (유니온)
0: kd> dt fltmgr!_FLT_PARAMETERS
   +0x000 Create           : _unnamed_struct_0
   +0x000 Read             : _unnamed_struct_1
   +0x000 Write            : _unnamed_struct_2
   ...

// IRP_MJ_CREATE인 경우 Create 구조 사용
0: kd> dt fltmgr!_FLT_PARAMETERS Create
   +0x000 Create           :
      +0x000 SecurityContext  : Ptr64 _IO_SECURITY_CONTEXT
      +0x008 Options          : Uint4B
      +0x010 FileAttributes   : Uint2B
      +0x012 ShareAccess      : Uint2B
      +0x018 EaLength         : Uint4B

// IRP_MJ_WRITE인 경우 Write 구조 사용
0: kd> dt fltmgr!_FLT_PARAMETERS Write
   +0x000 Write            :
      +0x000 Length           : Uint4B
      +0x008 Key              : Uint4B
      +0x010 ByteOffset       : _LARGE_INTEGER
      +0x018 WriteBuffer      : Ptr64 Void
      +0x020 MdlAddress       : Ptr64 _MDL
```

### 11.6.5 PreCreate 콜백에서 파일 정보 추출

```
// PreCreate 콜백 브레이크포인트에서:
// rcx = Data (FLT_CALLBACK_DATA*)
// rdx = FltObjects (FLT_RELATED_OBJECTS*)

// 1. Data에서 Iopb 가져오기
0: kd> dt fltmgr!_FLT_CALLBACK_DATA @rcx Iopb
   +0x010 Iopb : 0xffff8a01`2345a100

// 2. Iopb에서 TargetFileObject 가져오기
0: kd> dt fltmgr!_FLT_IO_PARAMETER_BLOCK poi(@rcx+0x10) TargetFileObject
   +0x008 TargetFileObject : 0xffff8a01`23459000

// 3. FileObject에서 파일 이름 추출
0: kd> dt nt!_FILE_OBJECT poi(poi(@rcx+0x10)+8) FileName
   +0x058 FileName : _UNICODE_STRING "\Users\User\test.txt"

// 한 줄로 파일 이름 출력
0: kd> dS poi(poi(poi(@rcx+0x10)+8)+0x58)+8
"\Users\User\test.txt"
```

---

## 11.7 컨텍스트 관련 구조체

### 11.7.1 FLT_RELATED_OBJECTS

```
// FLT_RELATED_OBJECTS 정의
0: kd> dt fltmgr!_FLT_RELATED_OBJECTS
   +0x000 Size             : Uint2B
   +0x002 TransactionContext : Uint2B
   +0x008 Filter           : Ptr64 _FLT_FILTER
   +0x010 Volume           : Ptr64 _FLT_VOLUME
   +0x018 Instance         : Ptr64 _FLT_INSTANCE
   +0x020 FileObject       : Ptr64 _FILE_OBJECT
   +0x028 Transaction      : Ptr64 _KTRANSACTION
```

### 11.7.2 FLT_FILTER (필터 정보)

```
// FLT_FILTER 구조
0: kd> dt fltmgr!_FLT_FILTER
   +0x000 Base             : _FLT_OBJECT
   +0x030 Frame            : Ptr64 _FLTP_FRAME
   +0x038 Name             : _UNICODE_STRING         // 필터 이름
   +0x048 DefaultAltitude  : _UNICODE_STRING         // 고도
   +0x058 Flags            : _FLT_FILTER_FLAGS
   +0x060 DriverObject     : Ptr64 _DRIVER_OBJECT
   +0x068 InstanceList     : _FLT_RESOURCE_LIST_HEAD
   +0x0e8 VerifierExtension : Ptr64 _FLT_VERIFIER_EXTENSION
   +0x0f0 VerifiedFiltersLink : _LIST_ENTRY
   +0x100 FilterUnload     : Ptr64     long
   +0x108 InstanceSetup    : Ptr64     long
   ...
```

### 11.7.3 FLT_INSTANCE (인스턴스 정보)

```
// FLT_INSTANCE 구조
0: kd> dt fltmgr!_FLT_INSTANCE
   +0x000 Base             : _FLT_OBJECT
   +0x030 OperationRundownRef : Ptr64 _EX_RUNDOWN_REF_CACHE_AWARE
   +0x038 Volume           : Ptr64 _FLT_VOLUME
   +0x040 Filter           : Ptr64 _FLT_FILTER
   +0x048 Flags            : _FLT_INSTANCE_FLAGS
   +0x050 Altitude         : _UNICODE_STRING
   +0x060 Name             : _UNICODE_STRING
   +0x070 FilterLink       : _LIST_ENTRY
   +0x080 ContextLock      : _EX_PUSH_LOCK
   +0x088 Context          : Ptr64 _CONTEXT_NODE
   +0x090 TransactionContexts : _CONTEXT_LIST_CTRL
   +0x098 TrackCompletionNodes : Ptr64 _TRACK_COMPLETION_NODES
   +0x0a0 CallbackNodes    : [50] Ptr64 _CALLBACK_NODE
```

---

## 11.8 실전 분석: 종합 시나리오

### 시나리오: 파일 쓰기 요청 전체 분석

notepad.exe가 파일을 저장할 때의 전체 흐름을 추적합니다.

```
// 1. Minifilter PreWrite 콜백에 브레이크포인트
bu MyDriver!PreWrite

// 2. Notepad에서 파일 저장 -> 브레이크 히트
0: kd> kb
 # RetAddr           Call Site
00 MyDriver!PreWrite
01 fltmgr!FltpPerformPreCallbacks
02 fltmgr!FltpPassThroughInternal
...

// 3. 콜백 매개변수 분석 (x64 호출 규약)
// rcx = Data, rdx = FltObjects, r8 = CompletionContext

// 4. 요청 프로세스 확인
0: kd> dt fltmgr!_FLT_CALLBACK_DATA @rcx Thread
   +0x008 Thread : 0xffff8a01`23456780 _KTHREAD

0: kd> !thread 0xffff8a01`23456780 0
THREAD ffff8a0123456780  Cid 1234.5678
        ...
Owning Process            ffff8a0123456000       Image:         notepad.exe

// 5. 파일 정보 확인
0: kd> dt fltmgr!_FLT_IO_PARAMETER_BLOCK poi(@rcx+0x10) TargetFileObject
   +0x008 TargetFileObject : 0xffff8a01`23459000

0: kd> dt nt!_FILE_OBJECT 0xffff8a01`23459000 FileName
   +0x058 FileName : "\Users\User\Documents\test.txt"

// 6. 쓰기 매개변수 확인
0: kd> dt fltmgr!_FLT_IO_PARAMETER_BLOCK poi(@rcx+0x10) Parameters.Write
   +0x018 Parameters       :
      +0x000 Write            :
         +0x000 Length           : 0x1000           // 4KB 쓰기
         +0x010 ByteOffset       : _LARGE_INTEGER 0x0  // 파일 시작
         +0x018 WriteBuffer      : 0xffff8a01`2345b000

// 7. 쓰기 데이터 미리보기
0: kd> db 0xffff8a01`2345b000 L40
ffff8a01`2345b000  48 65 6c 6c 6f 2c 20 57-6f 72 6c 64 21 0d 0a 54  Hello, World!..T
ffff8a01`2345b010  68 69 73 20 69 73 20 61-20 74 65 73 74 2e 00 00  his is a test...
...
```

### 분석 결과 정리

```
┌─────────────────────────────────────────────────────────────────────┐
│                      파일 쓰기 요청 분석 결과                        │
├─────────────────────────────────────────────────────────────────────┤
│  항목                   값                                           │
├─────────────────────────────────────────────────────────────────────┤
│  요청 프로세스          notepad.exe (PID: 0x1234)                   │
│  요청 스레드            TID: 0x5678                                 │
│  대상 파일              \Users\User\Documents\test.txt              │
│  작업 유형              IRP_MJ_WRITE                                │
│  쓰기 크기              4,096 바이트 (0x1000)                       │
│  파일 오프셋            0 (파일 시작)                               │
│  데이터 내용            "Hello, World!..."                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 11.9 유용한 분석 스크립트

### 11.9.1 프로세스 정보 출력 스크립트

```
// 현재 프로세스 정보 출력
.printf "=== Process Info ===\n"
.printf "EPROCESS: %p\n", @$proc
.printf "PID: %d\n", poi(@$proc+0x440)
.printf "Image: "
da @$proc+0x5a8
```

### 11.9.2 FLT_CALLBACK_DATA 분석 스크립트

```
// PreOperation 콜백에서 사용
// rcx = Data

.printf "\n=== Callback Data Analysis ===\n"

// MajorFunction
.printf "MajorFunction: 0x%x\n", by(poi(@rcx+0x10)+0x4)

// File Name
.printf "File: "
dS poi(poi(poi(@rcx+0x10)+8)+0x58)+8

// RequestorMode
.printf "RequestorMode: %s\n", by(@rcx+0x70)==0 ? "KernelMode" : "UserMode"
```

---

## 요약

이 챕터에서 학습한 핵심 구조체:

| 구조체 | 용도 | 주요 필드 |
|--------|------|-----------|
| EPROCESS | 프로세스 | UniqueProcessId, ImageFileName |
| ETHREAD | 스레드 | Cid, ThreadsProcess, State |
| DRIVER_OBJECT | 드라이버 | DriverName, MajorFunction[] |
| IRP | I/O 요청 | MajorFunction, IoStatus |
| FILE_OBJECT | 열린 파일 | FileName, ReadAccess, WriteAccess |
| FLT_CALLBACK_DATA | Minifilter 콜백 | Iopb, Thread, RequestorMode |
| FLT_IO_PARAMETER_BLOCK | I/O 매개변수 | MajorFunction, TargetFileObject, Parameters |

다음 챕터에서는 BSOD 크래시 덤프를 분석하는 방법을 학습합니다.

---

## 참고 자료

- [EPROCESS structure](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/eprocess)
- [IRP structure](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/ns-wdm-_irp)
- [FILE_OBJECT structure](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/ns-wdm-_file_object)
- [FLT_CALLBACK_DATA structure](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/fltkernel/ns-fltkernel-_flt_callback_data)
