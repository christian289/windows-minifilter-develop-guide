# Chapter 13: Minifilter 특화 디버깅

## 난이도: ⭐⭐⭐⭐ (중상급)

## 학습 목표
이 챕터를 완료하면 다음을 할 수 있습니다:
- !fltkd 확장 명령어를 사용하여 Minifilter를 분석합니다
- Filter Manager의 내부 구조를 이해합니다
- Minifilter 콜백 흐름을 추적합니다
- 일반적인 Minifilter 문제를 디버깅합니다
- 성능 문제를 진단합니다

## 도입: Minifilter 전용 디버깅 도구

Minifilter 드라이버는 Filter Manager 프레임워크 위에서 동작합니다. 일반적인 커널 디버깅 명령어로도 분석할 수 있지만, **!fltkd** 확장은 Minifilter 구조를 직관적으로 보여주는 전용 도구입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Minifilter 디버깅 도구 계층                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  !fltkd.xxx         ← Minifilter 전용 확장 (가장 편리)              │
│       ↓                                                              │
│  dt fltmgr!_FLT_*   ← Filter Manager 구조체 직접 분석               │
│       ↓                                                              │
│  dt nt!_*           ← 일반 커널 구조체 분석                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 13.1 !fltkd 확장 명령어 개요

### 13.1.1 !fltkd 확장 로드

```
// fltMgr 심볼 로드 확인
0: kd> lm m fltmgr
Browse full module list
start             end                 module name
fffff802`1d000000 fffff802`1d100000   fltmgr     (pdb symbols)

// !fltkd 도움말
0: kd> !fltkd.help
Filter Debugger Extension Commands:

    !fltkd.cbd [address]        - Dump FLT_CALLBACK_DATA
    !fltkd.filter [address]     - Dump FLT_FILTER
    !fltkd.filters              - List all filters
    !fltkd.frame [address]      - Dump frame
    !fltkd.frames               - List all frames
    !fltkd.instance [address]   - Dump FLT_INSTANCE
    !fltkd.irplist [address]    - Dump IRP list
    !fltkd.iopb [address]       - Dump FLT_IO_PARAMETER_BLOCK
    !fltkd.volume [address]     - Dump FLT_VOLUME
    !fltkd.volumes              - List all volumes
    !fltkd.portlist [address]   - Dump communication ports
```

### 13.1.2 주요 !fltkd 명령어 정리

```
┌─────────────────────────────────────────────────────────────────────┐
│                       !fltkd 핵심 명령어                             │
├─────────────────────────────────────────────────────────────────────┤
│  명령어                      설명                                    │
├─────────────────────────────────────────────────────────────────────┤
│  !fltkd.filters              모든 필터 목록                         │
│  !fltkd.filter <주소>        특정 필터 상세 정보                    │
│  !fltkd.volumes              모든 볼륨 목록                         │
│  !fltkd.volume <주소>        특정 볼륨 상세 정보                    │
│  !fltkd.instance <주소>      인스턴스 상세 정보                     │
│  !fltkd.cbd <주소>           FLT_CALLBACK_DATA 분석                 │
│  !fltkd.iopb <주소>          FLT_IO_PARAMETER_BLOCK 분석            │
│  !fltkd.frames               프레임 목록                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 13.2 필터 목록과 스택 분석

### 13.2.1 !fltkd.filters - 모든 필터 나열

```
0: kd> !fltkd.filters

Filter List: ffffb80123456000 "Frame 0"
   FLT_FILTER: ffffb80123460000 "WdFilter" "328010"
      FLT_INSTANCE: ffffb80123470000 "WdFilter Instance" "328010"
         Altitude: 328010, Flags: 0, VolumeLink: ffffb80123480000
      FLT_INSTANCE: ffffb80123471000 "WdFilter Instance" "328010"
         Altitude: 328010, Flags: 0, VolumeLink: ffffb80123481000

   FLT_FILTER: ffffb80123462000 "FileInfo" "45000"
      FLT_INSTANCE: ffffb80123472000 "FileInfo" "45000"
         ...

   FLT_FILTER: ffffb80123464000 "MyDRMFilter" "123456"    ← 우리 필터!
      FLT_INSTANCE: ffffb80123474000 "MyDRMFilter" "123456"
         Altitude: 123456, Flags: 0, VolumeLink: ffffb80123484000
      FLT_INSTANCE: ffffb80123475000 "MyDRMFilter" "123456"
         Altitude: 123456, Flags: 0, VolumeLink: ffffb80123485000

// 출력 해석:
// - Frame 0: 기본 프레임 (보통 하나만 존재)
// - FLT_FILTER: 등록된 필터 드라이버
// - FLT_INSTANCE: 볼륨에 연결된 인스턴스
// - Altitude: 필터 고도 (높을수록 먼저 호출)
```

### 13.2.2 필터 스택 시각화

```
// 고도(Altitude) 순서로 필터 스택 표현

┌──────────────────────────────────────┐
│ 사용자 모드 (User Mode)              │
├──────────────────────────────────────┤
│        ↓ CreateFile() 호출           │
├──────────────────────────────────────┤
│ Filter Manager (fltmgr.sys)          │
│                                      │
│  ┌────────────────────────────────┐  │
│  │ WdFilter    (Altitude: 328010) │  │ ← 가장 먼저 호출
│  ├────────────────────────────────┤  │
│  │ MyDRMFilter (Altitude: 123456) │  │ ← 두 번째
│  ├────────────────────────────────┤  │
│  │ FileInfo    (Altitude: 45000)  │  │ ← 세 번째
│  └────────────────────────────────┘  │
│                                      │
├──────────────────────────────────────┤
│ 파일 시스템 (NTFS, etc.)             │
├──────────────────────────────────────┤
│ 스토리지 드라이버                    │
└──────────────────────────────────────┘
```

### 13.2.3 !fltkd.filter - 특정 필터 상세 정보

```
0: kd> !fltkd.filter ffffb80123464000

FLT_FILTER: ffffb80123464000 "MyDRMFilter" "123456"
   Flags                       : 0x00000002 (FilteringInitiated)
   DriverObject                : ffffb80123450000 \FileSystem\Filters\MyDRMFilter
   InstanceList                : 2 instances
      ffffb80123474000         : Volume ffffb80123500000 "\Device\HarddiskVolume3"
      ffffb80123475000         : Volume ffffb80123501000 "\Device\HarddiskVolume4"
   Operations                  :
      IRP_MJ_CREATE             : ffffb80123466000
      IRP_MJ_READ               : ffffb80123467000
      IRP_MJ_WRITE              : ffffb80123468000
      IRP_MJ_SET_INFORMATION    : ffffb80123469000
      IRP_MJ_CLEANUP            : ffffb8012346a000
   FilterUnload                : ffffb80123465000
   InstanceSetup               : ffffb8012346b000
   InstanceQueryTeardown       : ffffb8012346c000
```

---

## 13.3 볼륨과 인스턴스 분석

### 13.3.1 !fltkd.volumes - 모든 볼륨 나열

```
0: kd> !fltkd.volumes

Volume List:
   FLT_VOLUME: ffffb80123500000 "\Device\HarddiskVolume1"
      Name: "\Device\HarddiskVolume1" (System Reserved)
      FileSystemType: NTFS
      DeviceObject: ffffb80123510000
      Instances: 3

   FLT_VOLUME: ffffb80123501000 "\Device\HarddiskVolume2"
      Name: "\Device\HarddiskVolume2" (C:)
      FileSystemType: NTFS
      DeviceObject: ffffb80123511000
      Instances: 5

   FLT_VOLUME: ffffb80123502000 "\Device\HarddiskVolume3"
      Name: "\Device\HarddiskVolume3" (D:)
      FileSystemType: NTFS
      DeviceObject: ffffb80123512000
      Instances: 4
```

### 13.3.2 !fltkd.volume - 볼륨 상세 정보

```
0: kd> !fltkd.volume ffffb80123501000

FLT_VOLUME: ffffb80123501000 "\Device\HarddiskVolume2"
   Flags                  : 0x00000164 (SetupComplete, MountNotified, ...)
   FileSystemType         : FLT_FSTYPE_NTFS
   DeviceObject           : ffffb80123511000
   DiskDeviceObject       : ffffb80123520000
   FrameZeroVolume        : ffffb80123501000
   VolumeInNextFrame      : 0000000000000000

   Instance List (top -> bottom):
      FLT_INSTANCE: ffffb80123530000 "WdFilter" Altitude: 328010
      FLT_INSTANCE: ffffb80123531000 "MyDRMFilter" Altitude: 123456
      FLT_INSTANCE: ffffb80123532000 "FileInfo" Altitude: 45000

   Callbacks registered:
      IRP_MJ_CREATE            : 3 filters
      IRP_MJ_READ              : 2 filters
      IRP_MJ_WRITE             : 2 filters
      ...
```

### 13.3.3 !fltkd.instance - 인스턴스 상세 정보

```
0: kd> !fltkd.instance ffffb80123531000

FLT_INSTANCE: ffffb80123531000 "MyDRMFilter"
   Filter                 : ffffb80123464000 "MyDRMFilter"
   Volume                 : ffffb80123501000 "\Device\HarddiskVolume2"
   Altitude               : "123456"
   Flags                  : 0x00000004 (SetupComplete)
   VolumeLink             : Flink=ffffb80123530018 Blink=ffffb80123532018
   FilterLink             : Flink=ffffb80123474018 Blink=ffffb80123475018
   Context                : ffffb80123540000    ← 인스턴스 컨텍스트!
   CallbackNodes          :
      IRP_MJ_CREATE        : PreCallback=ffffb80123466100, PostCallback=ffffb80123466200
      IRP_MJ_WRITE         : PreCallback=ffffb80123467100, PostCallback=0 (없음)
```

---

## 13.4 콜백 데이터 분석

### 13.4.1 !fltkd.cbd - FLT_CALLBACK_DATA 분석

PreOperation 또는 PostOperation 콜백에서 브레이크할 때 사용합니다.

```
// PreOperation 브레이크포인트에서:
// rcx = FLT_CALLBACK_DATA*

0: kd> !fltkd.cbd @rcx

FLT_CALLBACK_DATA: ffffb80123600000
   Flags                 : 0x00000001 (IRP-based)
   Thread                : ffffb80123610000 (TID: 1234)
   RequestorMode         : UserMode
   IoStatus.Status       : 0x00000000 (STATUS_SUCCESS)
   IoStatus.Information  : 0

   FLT_IO_PARAMETER_BLOCK: ffffb80123600030
      IrpFlags              : 0x00060000 (IRP_CREATE_OPERATION | ...)
      MajorFunction         : 0 (IRP_MJ_CREATE)
      MinorFunction         : 0
      OperationFlags        : 0
      TargetFileObject      : ffffb80123620000
      TargetInstance        : ffffb80123531000 "MyDRMFilter"

      Parameters.Create:
         SecurityContext      : ffffb80123630000
         Options              : 0x01200000 (FILE_OPEN, FILE_SYNCHRONOUS_IO_NONALERT)
         FileAttributes       : 0x00000080 (FILE_ATTRIBUTE_NORMAL)
         ShareAccess          : 0x00000007 (READ | WRITE | DELETE)
         EaLength             : 0

   TargetFileObject Details:
      FileName: "\Users\TestUser\Documents\sensitive.docx"
      Vpb: ffffb80123640000
      DeviceObject: ffffb80123511000
```

### 13.4.2 !fltkd.iopb - I/O 매개변수 분석

```
0: kd> !fltkd.iopb ffffb80123600030

FLT_IO_PARAMETER_BLOCK: ffffb80123600030
   IrpFlags           : 0x00060000
   MajorFunction      : IRP_MJ_CREATE (0)
   MinorFunction      : 0
   OperationFlags     : 0x00000000
   Reserved           : 0
   TargetFileObject   : ffffb80123620000 "\Users\TestUser\Documents\sensitive.docx"
   TargetInstance     : ffffb80123531000

   Parameters (union based on MajorFunction):
      Create.SecurityContext    : ffffb80123630000
      Create.Options            : 0x01200000
      Create.FileAttributes     : FILE_ATTRIBUTE_NORMAL
      Create.ShareAccess        : FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE
      Create.EaLength           : 0
```

### 13.4.3 작업별 매개변수 해석

```c
// IRP_MJ_CREATE 매개변수
typedef struct _FLT_PARAMETERS_CREATE {
    PIO_SECURITY_CONTEXT SecurityContext;
    ULONG Options;           // CreateDisposition (상위 8비트) + CreateOptions (하위 24비트)
    USHORT FileAttributes;
    USHORT ShareAccess;
    ULONG EaLength;
} FLT_PARAMETERS_CREATE;

// Options 해석:
// 상위 8비트: CreateDisposition
//   FILE_SUPERSEDE (0x00)
//   FILE_OPEN (0x01)
//   FILE_CREATE (0x02)
//   FILE_OPEN_IF (0x03)
//   FILE_OVERWRITE (0x04)
//   FILE_OVERWRITE_IF (0x05)

// 하위 24비트: CreateOptions
//   FILE_DIRECTORY_FILE (0x001)
//   FILE_NON_DIRECTORY_FILE (0x040)
//   FILE_DELETE_ON_CLOSE (0x1000)
//   ...
```

```
// WinDbg에서 Options 해석
0: kd> ? 0x01200000 >> 18
Evaluate expression: 4 = 00000000`00000004    // CreateDisposition

0: kd> ? 0x01200000 & 0xFFFFFF
Evaluate expression: 2097152 = 00000000`00200000  // CreateOptions
```

---

## 13.5 콜백 흐름 추적

### 13.5.1 콜백 브레이크포인트 설정

```
// 특정 필터의 PreCreate 콜백에 브레이크포인트
0: kd> bp MyDRMFilter!PreCreate

// 모든 Pre 콜백에 브레이크포인트 (패턴 매칭)
0: kd> bm MyDRMFilter!Pre*

// 조건부 브레이크포인트 (특정 파일 확장자만)
0: kd> bp MyDRMFilter!PreCreate "$$>a<c:\\scripts\\checkext.txt"
```

### 13.5.2 콜백 진입 시 정보 수집

```
// PreOperation 콜백 진입 시:
// rcx = FLT_CALLBACK_DATA*
// rdx = FLT_RELATED_OBJECTS*
// r8 = CompletionContext**

0: kd> bp MyDRMFilter!PreCreate "!fltkd.cbd @rcx; g"

// 또는 수동 분석:
0: kd> bp MyDRMFilter!PreCreate
0: kd> g

Breakpoint 0 hit
MyDRMFilter!PreCreate:
fffff800`11112000 mov     rbp,rsp

// 파일 이름 확인
0: kd> dt fltmgr!_FLT_CALLBACK_DATA @rcx Iopb
   +0x010 Iopb : 0xffffb801`23600030

0: kd> dt fltmgr!_FLT_IO_PARAMETER_BLOCK poi(@rcx+0x10) TargetFileObject
   +0x008 TargetFileObject : 0xffffb801`23620000

0: kd> dt nt!_FILE_OBJECT poi(poi(@rcx+0x10)+8) FileName
   +0x058 FileName : _UNICODE_STRING "\Users\TestUser\sensitive.docx"
```

### 13.5.3 콜백 체인 추적

```
// Filter Manager 내부 콜백 호출 흐름 추적

// PreCallbacks 실행
0: kd> bp fltmgr!FltpPerformPreCallbacks
0: kd> g

// 콜백 호출 확인
0: kd> kb
 # RetAddr           Call Site
00 fltmgr!FltpPerformPreCallbacks
01 fltmgr!FltpPassThroughInternal
02 fltmgr!FltpCreate
03 nt!IopParseDevice
...

// 어느 필터가 호출되는지 확인
0: kd> dt fltmgr!_CALLBACK_NODE poi(@rsp+0x28)
   +0x000 CallbackLinks : ...
   +0x010 Instance      : 0xffffb801`23531000    // 인스턴스 주소

0: kd> !fltkd.instance 0xffffb801`23531000
FLT_INSTANCE: ... "MyDRMFilter"
```

---

## 13.6 컨텍스트 디버깅

### 13.6.1 컨텍스트 확인

Minifilter는 다양한 객체에 컨텍스트를 연결할 수 있습니다.

```
// 인스턴스 컨텍스트 확인
0: kd> !fltkd.instance ffffb80123531000
   ...
   Context : ffffb80123540000

// 컨텍스트 내용 확인 (사용자 정의 구조체)
0: kd> dt MyDRMFilter!_MY_INSTANCE_CONTEXT ffffb80123540000
   +0x000 VolumeName : _UNICODE_STRING "C:"
   +0x010 IsProtected : 0x1 ''
   +0x018 FileCount : 0x42
```

### 13.6.2 스트림/스트림 핸들 컨텍스트

```
// FILE_OBJECT에서 스트림 컨텍스트 찾기
0: kd> dt nt!_FILE_OBJECT ffffb80123620000 FsContext FsContext2
   +0x018 FsContext  : 0xffffb801`23700000  // SCB (Stream Control Block)
   +0x020 FsContext2 : 0xffffb801`23710000  // CCB (Context Control Block)

// FltGetStreamContext로 설정한 컨텍스트는 다른 위치에 저장됨
// Filter Manager 내부 구조를 통해 찾아야 함

// 일반적인 방법: 콜백에서 FltGetStreamContext 호출 전후로 브레이크
0: kd> bp fltmgr!FltGetStreamContext
0: kd> g

// 반환값(rax)이 컨텍스트 포인터
0: kd> gu
0: kd> r rax
rax=ffffb80123720000

0: kd> dt MyDRMFilter!_MY_STREAM_CONTEXT ffffb80123720000
   +0x000 FileId : 0x12345678
   +0x008 IsConfidential : 0x1 ''
   +0x010 PatternFound : 0x0 ''
```

---

## 13.7 일반적인 Minifilter 문제 디버깅

### 13.7.1 드라이버 로드 실패

```
// 드라이버 로드 시도
sc start MyDRMFilter

// 실패 시 WinDbg에서:
0: kd> bu MyDRMFilter!DriverEntry
0: kd> g

// DriverEntry 진입 전 크래시하면:
// - 드라이버 서명 문제
// - 종속성 문제
// - 드라이버 초기화 순서 문제

// DriverEntry 내부에서 실패하면:
0: kd> gu  // DriverEntry 반환까지 실행
0: kd> r rax
rax=00000000c0000034    // STATUS_OBJECT_NAME_NOT_FOUND

// 상태 코드 해석
0: kd> !error c0000034
Error code: (NTSTATUS) 0xc0000034 - Object Name not found.
```

### 13.7.2 FltRegisterFilter 실패

```
0: kd> bp fltmgr!FltRegisterFilter
0: kd> g

Breakpoint hit
0: kd> gu    // 함수 완료까지 실행
0: kd> r rax
rax=00000000c000000d    // STATUS_INVALID_PARAMETER

// 흔한 원인:
// 1. FLT_REGISTRATION 구조체 오류
// 2. 콜백 함수 포인터 오류
// 3. 컨텍스트 등록 오류

// FLT_REGISTRATION 확인
0: kd> dt MyDRMFilter!FilterRegistration
   +0x000 Size : 0x140
   +0x002 Version : 0x0203
   +0x004 Flags : 0
   +0x008 ContextRegistration : 0xfffff800`11110000
   +0x010 OperationRegistration : 0xfffff800`11111000
   ...
```

### 13.7.3 인스턴스 연결 실패

```
// InstanceSetup 콜백 디버깅
0: kd> bp MyDRMFilter!InstanceSetup
0: kd> g

// 매개변수 확인
// rcx = FltObjects
// rdx = Flags
// r8 = VolumeDeviceType
// r9 = VolumeFilesystemType

0: kd> dt fltmgr!_FLT_RELATED_OBJECTS @rcx
   +0x008 Filter  : 0xffffb801`23464000
   +0x010 Volume  : 0xffffb801`23501000
   +0x018 Instance : 0x00000000`00000000  // 아직 생성 안됨

// 반환값 확인
0: kd> gu
0: kd> r rax
rax=00000000c00000bb    // STATUS_NOT_SUPPORTED

// 특정 볼륨 유형 제외 시 의도적으로 반환
// 예: CD-ROM, 네트워크 드라이브 제외
```

### 13.7.4 콜백에서 데드락

```
// 데드락 증상: 시스템 멈춤, 특정 파일 접근 시 무한 대기

// 스레드 스택 확인
0: kd> !process 0 17 MyDRMFilter.sys

// 대기 중인 스레드 찾기
THREAD ffffb80123800000  Cid 1234.5678  Teb: ... WAIT: (WrResource)
    ffffb80123810000  SynchronizationEvent

0: kd> !thread ffffb80123800000
...
Wait Start TickCount    123456          Ticks: 99999 (long wait!)
...

// 콜스택
0: kd> .thread ffffb80123800000
0: kd> kbn
 # RetAddr           Call Site
00 nt!KiSwapContext
01 nt!KiSwapThread
02 nt!KiCommitThreadWait
03 nt!KeWaitForSingleObject
04 nt!ExAcquireResourceExclusiveLite
05 MyDRMFilter!AcquireLock           ← 락 획득 시도
06 MyDRMFilter!PreWrite
07 fltmgr!FltpPerformPreCallbacks
...

// 누가 락을 들고 있는지 확인
0: kd> !locks
Resource @ ffffb80123820000    Exclusively owned
    Contention Count = 1
    Threads: ffffb80123830000-01<*>    ← 이 스레드가 보유 중

0: kd> .thread ffffb80123830000
0: kd> kbn
// 이 스레드도 다른 락을 기다리고 있으면 = 데드락!
```

### 13.7.5 성능 문제 진단

```
// 콜백 실행 시간 측정
0: kd> bp MyDRMFilter!PreWrite ".printf \"Enter: %I64d\\n\", @$prcb->CurrentThread->CycleTime; g"
0: kd> bp MyDRMFilter!PreWrite+0x150 ".printf \"Exit: %I64d\\n\", @$prcb->CurrentThread->CycleTime; g"

// 또는 ETW 사용 (더 정확)
// logman start mytrace -p Microsoft-Windows-FilterManager -ets

// 콜백 호출 빈도 확인
0: kd> bp MyDRMFilter!PreWrite "r $t0 = @$t0 + 1; .printf \"Call count: %d\\n\", @$t0; g"
```

---

## 13.8 Filter Manager 내부 구조 분석

### 13.8.1 프레임 구조

```
// Filter Manager 프레임 확인
0: kd> !fltkd.frames

Frame List:
   FLTP_FRAME: ffffb80123456000 "Frame 0"
      Flags: 0x0001 (Primary)
      AttachedVolumes: 5
      RegisteredFilters: 8

// 프레임 상세 정보
0: kd> dt fltmgr!_FLTP_FRAME ffffb80123456000
   +0x000 Type             : _FLT_TYPE
   +0x008 Links            : _LIST_ENTRY
   +0x018 FrameID          : 0
   +0x020 AltitudeIntervalLow : _UNICODE_STRING "0"
   +0x030 AltitudeIntervalHigh : _UNICODE_STRING "999999"
   +0x040 LargeIrpCtrlStackSize : 0x6 ''
   +0x041 SmallIrpCtrlStackSize : 0x2 ''
   +0x048 RegisteredFilters : _FLT_RESOURCE_LIST_HEAD
   +0x0c8 AttachedVolumes  : _FLT_RESOURCE_LIST_HEAD
   ...
```

### 13.8.2 콜백 노드 구조

```
// 콜백 노드 분석
0: kd> dt fltmgr!_CALLBACK_NODE
   +0x000 CallbackLinks    : _LIST_ENTRY
   +0x010 Instance         : Ptr64 _FLT_INSTANCE
   +0x018 PreOperation     : Ptr64     enum _FLT_PREOP_CALLBACK_STATUS
   +0x020 PostOperation    : Ptr64     enum _FLT_POSTOP_CALLBACK_STATUS
   +0x028 Flags            : _CALLBACK_NODE_FLAGS

// 실제 콜백 주소 확인
0: kd> !fltkd.instance ffffb80123531000
   ...
   CallbackNodes:
      [0] IRP_MJ_CREATE: Pre=ffffb80123466100 Post=ffffb80123466200

0: kd> u ffffb80123466100
MyDRMFilter!PreCreate:
fffff800`11112100 mov     rbp,rsp
...
```

---

## 13.9 실전 시나리오: 파일 보호 문제 디버깅

### 시나리오: 특정 파일이 보호되지 않음

```
// 1. 해당 파일에 접근하는 콜백에 브레이크포인트 설정
0: kd> bp MyDRMFilter!PreWrite

// 2. 파일에 쓰기 시도 (다른 머신이나 프로세스에서)

// 3. 브레이크 히트 확인
Breakpoint 0 hit
MyDRMFilter!PreWrite:

// 4. 파일 이름 확인
0: kd> !fltkd.cbd @rcx
   ...
   FileName: "\Users\TestUser\Documents\important.xlsx"

// 5. 보호 로직 확인
0: kd> p    // Step
// IsProtectedFile 함수 호출 확인
0: kd> p
MyDRMFilter!IsProtectedFile:

// 6. 반환값 확인
0: kd> gu
0: kd> r rax
rax=0000000000000000    // FALSE! 보호 대상 아님으로 판단

// 7. 왜 FALSE인지 조사
// IsProtectedFile 함수 내부 로직 분석
0: kd> bp MyDRMFilter!IsProtectedFile
0: kd> g

// 8. 내부 조건 확인
0: kd> dv /V
           Local  @rbp-0x10  fileExt = L".xlsx"
           Local  @rbp-0x20  isProtected = 0

// 9. 보호 확장자 리스트 확인
0: kd> dt MyDRMFilter!g_ProtectedExtensions
   [0] = L".docx"
   [1] = L".doc"
   [2] = L".pdf"
   // .xlsx가 없음! → 설정 문제 발견
```

---

## 13.10 디버깅 스크립트 모음

### 13.10.1 파일 정보 출력 스크립트

```
// PreOperation에서 파일 정보 출력
// 사용: $><c:\scripts\fileinfo.txt

// fileinfo.txt 내용:
.printf "=== File Operation ===\n"
.printf "Major: %d\n", by(poi(@rcx+0x10)+4)
.printf "File: "
dS poi(poi(poi(@rcx+0x10)+8)+0x58)+8
.printf "Thread: %p\n", poi(@rcx+8)
.printf "Process: "
!process poi(@rcx+8) 0
```

### 13.10.2 콜백 통계 스크립트

```
// 콜백 호출 횟수 통계
// 초기화
r $t0 = 0; r $t1 = 0; r $t2 = 0

// PreCreate 카운터
bp MyDRMFilter!PreCreate "r $t0 = @$t0 + 1; g"

// PreRead 카운터
bp MyDRMFilter!PreRead "r $t1 = @$t1 + 1; g"

// PreWrite 카운터
bp MyDRMFilter!PreWrite "r $t2 = @$t2 + 1; g"

// 통계 출력 (수동)
.printf "Create: %d, Read: %d, Write: %d\n", @$t0, @$t1, @$t2
```

---

## 요약

이 챕터에서 학습한 내용:

1. **!fltkd 확장**: Minifilter 전용 디버깅 도구 활용
2. **필터 스택**: !fltkd.filters로 필터 계층 확인
3. **볼륨/인스턴스**: 연결 상태 및 컨텍스트 분석
4. **콜백 데이터**: !fltkd.cbd, !fltkd.iopb로 I/O 정보 분석
5. **문제 진단**: 로드 실패, 데드락, 성능 문제 디버깅

다음 챕터에서는 조건부 브레이크포인트, 스크립팅 등 고급 WinDbg 기법을 학습합니다.

---

## 참고 자료

- [Filter Manager Debugging](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/filter-manager-debugging)
- [!fltkd Extension](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-fltkd)
- [Minifilter Altitude](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/allocated-altitudes)
