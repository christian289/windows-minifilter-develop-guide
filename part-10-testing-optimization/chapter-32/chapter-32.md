# Chapter 32: 성능 최적화

## 난이도: ★★★★★ (고급)

## 학습 목표
- Minifilter 성능 병목 현상 이해
- 프로파일링 도구 활용 방법
- 메모리 및 CPU 최적화 기법
- 실전 최적화 패턴 적용

## 32.1 성능 병목 현상 이해

### Minifilter가 성능에 미치는 영향

```
+------------------------------------------------------------------+
|                    파일 I/O 경로와 Minifilter                      |
+------------------------------------------------------------------+
|                                                                   |
|  애플리케이션                                                      |
|       |                                                           |
|       | CreateFile("C:\doc.txt")                                  |
|       v                                                           |
|  +------------+                                                   |
|  | I/O 관리자  |                                                   |
|  +-----+------+                                                   |
|        |                                                          |
|        v                                                          |
|  +------------------------+                                       |
|  |    Filter Manager      |                                       |
|  +------------------------+                                       |
|        |                                                          |
|        v                                                          |
|  +------------------------+  ←── 여기서 지연 발생 가능            |
|  | Minifilter (DRM)       |      - PreCreate 처리                 |
|  | - 정책 검사             |      - 파일 이름 조회                  |
|  | - 콘텐츠 스캔           |      - 콘텐츠 분석                     |
|  | - 로깅                  |      - 통신 대기                       |
|  +------------------------+                                       |
|        |                                                          |
|        v                                                          |
|  +------------------------+                                       |
|  | 파일 시스템 (NTFS)      |                                       |
|  +------------------------+                                       |
|        |                                                          |
|        v                                                          |
|  +------------------------+                                       |
|  |      디스크             |                                       |
|  +------------------------+                                       |
|                                                                   |
+------------------------------------------------------------------+
```

### 일반적인 성능 문제

```c
// 일반적인 성능 저하 원인들
// Common causes of performance degradation

// 1. 불필요한 파일 이름 조회
// 1. Unnecessary file name queries
FLT_PREOP_CALLBACK_STATUS
SlowPreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    PFLT_FILE_NAME_INFORMATION nameInfo;

    // 모든 파일에 대해 이름 조회 - 비용이 큼!
    // Query name for all files - expensive!
    FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
        );
    // ... 사용 후 해제
    FltReleaseFileNameInformation(nameInfo);

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

// 2. 동기적 사용자 모드 통신
// 2. Synchronous user mode communication
FLT_PREOP_CALLBACK_STATUS
SlowPreWrite(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    // 모든 쓰기 시 사용자 모드에 동기적 메시지 전송
    // Send synchronous message to user mode on every write
    LARGE_INTEGER timeout = { .QuadPart = -10000000 };  // 1초

    FltSendMessage(
        g_Filter,
        &g_ClientPort,
        &message,
        sizeof(message),
        NULL,
        NULL,
        &timeout  // 1초 대기 - 매 쓰기마다!
    );

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 3. 과도한 메모리 할당
// 3. Excessive memory allocation
FLT_PREOP_CALLBACK_STATUS
SlowPreRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    // 매 콜백마다 메모리 할당
    // Allocate memory on every callback
    PVOID buffer = ExAllocatePool2(
        POOL_FLAG_NON_PAGED,
        4096,
        'fubT'
    );

    // ... 사용

    ExFreePoolWithTag(buffer, 'fubT');

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}

// 4. 잠금 경합
// 4. Lock contention
KSPIN_LOCK g_GlobalLock;  // 전역 스핀락 - 병목!

FLT_PREOP_CALLBACK_STATUS
SlowWithLock(...)
{
    KIRQL oldIrql;

    // 모든 작업에서 동일한 잠금 획득
    // Acquire same lock in all operations
    KeAcquireSpinLock(&g_GlobalLock, &oldIrql);

    // ... 긴 작업 수행

    KeReleaseSpinLock(&g_GlobalLock, oldIrql);

    return FLT_PREOP_SUCCESS_NO_CALLBACK;
}
```

### .NET 개발자를 위한 비교

```csharp
// C# - 비슷한 성능 문제들
// C# - Similar performance issues

public class SlowFileProcessor
{
    private readonly object _globalLock = new();  // 전역 잠금

    // 문제 1: 모든 파일에서 정규식 검사
    // Problem 1: Regex check on all files
    public bool ProcessFile(string path)
    {
        // 매번 새 Regex 인스턴스 생성 - 비효율적
        // Create new Regex instance every time - inefficient
        var regex = new Regex(@"\d{6}-\d{7}");  // 컴파일 비용

        string content = File.ReadAllText(path);  // 전체 로드
        return regex.IsMatch(content);
    }

    // 문제 2: 과도한 동기화
    // Problem 2: Excessive synchronization
    public void LogEvent(string message)
    {
        lock (_globalLock)  // 모든 로깅에 동일한 잠금
        {
            File.AppendAllText("log.txt", message);  // I/O 중 잠금 보유
        }
    }
}

// 커널 드라이버에서는 이러한 문제가 더 심각합니다.
// In kernel drivers, these issues are more severe.
// - CPU 시간이 직접 시스템 성능에 영향
// - CPU time directly affects system performance
// - 잘못된 잠금은 BSOD 유발 가능
// - Wrong locking can cause BSOD
```

## 32.2 성능 측정 도구

### ETW (Event Tracing for Windows)

```c
// ETWTracing.h
// ETW 기반 성능 측정
// ETW-based performance measurement

#pragma once
#include <evntrace.h>

// ETW 프로바이더 GUID
// ETW Provider GUID
// {12345678-1234-1234-1234-123456789ABC}
DEFINE_GUID(DRM_FILTER_PROVIDER_GUID,
    0x12345678, 0x1234, 0x1234, 0x12, 0x34, 0x12, 0x34, 0x56, 0x78, 0x9a, 0xbc);

// 이벤트 ID
// Event IDs
#define EVENT_PRECREATE_START    1
#define EVENT_PRECREATE_END      2
#define EVENT_PREWRITE_START     3
#define EVENT_PREWRITE_END       4
#define EVENT_POLICY_CHECK_START 5
#define EVENT_POLICY_CHECK_END   6
#define EVENT_CONTENT_SCAN_START 7
#define EVENT_CONTENT_SCAN_END   8

// ETW 초기화
// ETW initialization
NTSTATUS
EtwInitialize(
    VOID
    );

// ETW 종료
// ETW shutdown
VOID
EtwShutdown(
    VOID
    );

// 이벤트 기록
// Log event
VOID
EtwLogEvent(
    _In_ ULONG EventId,
    _In_opt_ PCUNICODE_STRING FilePath,
    _In_ ULONG64 AdditionalData
    );
```

```c
// ETWTracing.c
// ETW 구현
// ETW implementation

#include "ETWTracing.h"
#include <wdm.h>

// WPP 대신 수동 ETW 구현
// Manual ETW implementation instead of WPP

REGHANDLE g_EtwRegHandle = 0;

NTSTATUS
EtwInitialize(
    VOID
    )
{
    NTSTATUS status;

    status = EtwRegister(
        &DRM_FILTER_PROVIDER_GUID,
        NULL,
        NULL,
        &g_EtwRegHandle
        );

    if (!NT_SUCCESS(status)) {
        DbgPrint("[DrmFilter] ETW 등록 실패: 0x%08X\n", status);
        // [DrmFilter] ETW registration failed
    }

    return status;
}

VOID
EtwShutdown(
    VOID
    )
{
    if (g_EtwRegHandle != 0) {
        EtwUnregister(g_EtwRegHandle);
        g_EtwRegHandle = 0;
    }
}

VOID
EtwLogEvent(
    _In_ ULONG EventId,
    _In_opt_ PCUNICODE_STRING FilePath,
    _In_ ULONG64 AdditionalData
    )
{
    EVENT_DESCRIPTOR eventDescriptor;
    EVENT_DATA_DESCRIPTOR dataDescriptor[2];
    ULONG dataCount = 0;

    if (g_EtwRegHandle == 0) {
        return;
    }

    // 이벤트 디스크립터 설정
    // Set event descriptor
    RtlZeroMemory(&eventDescriptor, sizeof(eventDescriptor));
    eventDescriptor.Id = (USHORT)EventId;
    eventDescriptor.Version = 0;
    eventDescriptor.Channel = 0;
    eventDescriptor.Level = TRACE_LEVEL_INFORMATION;
    eventDescriptor.Opcode = 0;
    eventDescriptor.Task = 0;
    eventDescriptor.Keyword = 0;

    // 데이터 설정
    // Set data
    if (FilePath != NULL && FilePath->Buffer != NULL) {
        EventDataDescCreate(
            &dataDescriptor[dataCount++],
            FilePath->Buffer,
            FilePath->Length
            );
    }

    EventDataDescCreate(
        &dataDescriptor[dataCount++],
        &AdditionalData,
        sizeof(AdditionalData)
        );

    // 이벤트 기록
    // Write event
    EtwWrite(
        g_EtwRegHandle,
        &eventDescriptor,
        NULL,
        dataCount,
        dataDescriptor
        );
}

// 성능 측정 매크로
// Performance measurement macros
#define MEASURE_START(eventId) \
    LARGE_INTEGER __perfStart; \
    KeQueryPerformanceCounter(&__perfStart); \
    EtwLogEvent(eventId, NULL, 0)

#define MEASURE_END(eventId, filePath) \
    do { \
        LARGE_INTEGER __perfEnd, __perfFreq; \
        __perfEnd = KeQueryPerformanceCounter(&__perfFreq); \
        ULONG64 __elapsed = (__perfEnd.QuadPart - __perfStart.QuadPart) * 1000000 / __perfFreq.QuadPart; \
        EtwLogEvent(eventId, filePath, __elapsed); \
    } while(0)
```

### 성능 카운터

```c
// PerfCounters.h
// 성능 카운터 구현
// Performance counter implementation

#pragma once

// 성능 통계 구조체
// Performance statistics structure
typedef struct _PERF_STATISTICS {
    // 콜백 카운트
    // Callback counts
    volatile LONG64 PreCreateCount;
    volatile LONG64 PostCreateCount;
    volatile LONG64 PreWriteCount;
    volatile LONG64 PreReadCount;
    volatile LONG64 PreSetInfoCount;

    // 차단 카운트
    // Block counts
    volatile LONG64 BlockedWrites;
    volatile LONG64 BlockedDeletes;
    volatile LONG64 BlockedRenames;

    // 타이밍 (마이크로초 단위 누적)
    // Timing (accumulated microseconds)
    volatile LONG64 TotalPreCreateTime;
    volatile LONG64 TotalPolicyCheckTime;
    volatile LONG64 TotalContentScanTime;

    // 최대 지연 시간
    // Maximum latency
    volatile LONG64 MaxPreCreateLatency;
    volatile LONG64 MaxPolicyCheckLatency;

    // 메모리 사용량
    // Memory usage
    volatile LONG64 CurrentPoolUsage;
    volatile LONG64 PeakPoolUsage;
    volatile LONG64 AllocationCount;

} PERF_STATISTICS, *PPERF_STATISTICS;

// 전역 통계
// Global statistics
extern PERF_STATISTICS g_PerfStats;

// 통계 업데이트 함수
// Statistics update functions
VOID PerfIncrementCounter(_Inout_ volatile LONG64* Counter);
VOID PerfAddTiming(_Inout_ volatile LONG64* Total, _In_ LONG64 Value);
VOID PerfUpdateMax(_Inout_ volatile LONG64* Max, _In_ LONG64 Value);

// 통계 조회
// Query statistics
VOID PerfGetStatistics(_Out_ PPERF_STATISTICS Stats);
VOID PerfResetStatistics(VOID);
```

```c
// PerfCounters.c
// 성능 카운터 구현
// Performance counter implementation

#include "PerfCounters.h"

PERF_STATISTICS g_PerfStats = { 0 };

VOID
PerfIncrementCounter(
    _Inout_ volatile LONG64* Counter
    )
{
    InterlockedIncrement64(Counter);
}

VOID
PerfAddTiming(
    _Inout_ volatile LONG64* Total,
    _In_ LONG64 Value
    )
{
    InterlockedAdd64(Total, Value);
}

VOID
PerfUpdateMax(
    _Inout_ volatile LONG64* Max,
    _In_ LONG64 Value
    )
{
    LONG64 currentMax;

    do {
        currentMax = *Max;
        if (Value <= currentMax) {
            break;
        }
    } while (InterlockedCompareExchange64(Max, Value, currentMax) != currentMax);
}

VOID
PerfGetStatistics(
    _Out_ PPERF_STATISTICS Stats
    )
{
    // 원자적 복사
    // Atomic copy
    Stats->PreCreateCount = g_PerfStats.PreCreateCount;
    Stats->PostCreateCount = g_PerfStats.PostCreateCount;
    Stats->PreWriteCount = g_PerfStats.PreWriteCount;
    Stats->PreReadCount = g_PerfStats.PreReadCount;
    Stats->PreSetInfoCount = g_PerfStats.PreSetInfoCount;

    Stats->BlockedWrites = g_PerfStats.BlockedWrites;
    Stats->BlockedDeletes = g_PerfStats.BlockedDeletes;
    Stats->BlockedRenames = g_PerfStats.BlockedRenames;

    Stats->TotalPreCreateTime = g_PerfStats.TotalPreCreateTime;
    Stats->TotalPolicyCheckTime = g_PerfStats.TotalPolicyCheckTime;
    Stats->TotalContentScanTime = g_PerfStats.TotalContentScanTime;

    Stats->MaxPreCreateLatency = g_PerfStats.MaxPreCreateLatency;
    Stats->MaxPolicyCheckLatency = g_PerfStats.MaxPolicyCheckLatency;

    Stats->CurrentPoolUsage = g_PerfStats.CurrentPoolUsage;
    Stats->PeakPoolUsage = g_PerfStats.PeakPoolUsage;
    Stats->AllocationCount = g_PerfStats.AllocationCount;
}

VOID
PerfResetStatistics(
    VOID
    )
{
    RtlZeroMemory(&g_PerfStats, sizeof(g_PerfStats));
}

// 타이밍 측정 헬퍼
// Timing measurement helper
typedef struct _PERF_TIMER {
    LARGE_INTEGER StartTime;
    LARGE_INTEGER Frequency;
} PERF_TIMER, *PPERF_TIMER;

VOID
PerfTimerStart(
    _Out_ PPERF_TIMER Timer
    )
{
    Timer->StartTime = KeQueryPerformanceCounter(&Timer->Frequency);
}

LONG64
PerfTimerStop(
    _In_ PPERF_TIMER Timer
    )
{
    LARGE_INTEGER endTime;
    endTime = KeQueryPerformanceCounter(NULL);

    // 마이크로초 단위로 변환
    // Convert to microseconds
    return (endTime.QuadPart - Timer->StartTime.QuadPart) *
           1000000 / Timer->Frequency.QuadPart;
}
```

### WPA (Windows Performance Analyzer) 활용

```powershell
# CapturePerformanceTrace.ps1
# WPR로 성능 트레이스 캡처
# Capture performance trace with WPR

param(
    [int]$DurationSeconds = 30,
    [string]$OutputPath = ".\DrmFilterTrace.etl"
)

Write-Host "성능 트레이스 캡처 시작" -ForegroundColor Green
# Starting performance trace capture

# WPR 프로필 생성 (XML)
# Create WPR profile (XML)
$wprProfile = @"
<?xml version="1.0" encoding="utf-8"?>
<WindowsPerformanceRecorder Version="1.0">
  <Profiles>
    <SystemCollector Id="SystemCollector" Name="NT Kernel Logger">
      <BufferSize Value="256"/>
      <Buffers Value="64"/>
    </SystemCollector>

    <EventCollector Id="EventCollector" Name="Event Collector">
      <BufferSize Value="256"/>
      <Buffers Value="64"/>
    </EventCollector>

    <SystemProvider Id="SystemProvider">
      <Keywords>
        <Keyword Value="CSwitch"/>
        <Keyword Value="DiskIO"/>
        <Keyword Value="FileIO"/>
        <Keyword Value="Loader"/>
        <Keyword Value="ProcessThread"/>
      </Keywords>
    </SystemProvider>

    <EventProvider Id="DrmFilterProvider" Name="12345678-1234-1234-1234-123456789ABC">
      <Keywords>
        <Keyword Value="0xFFFFFFFF"/>
      </Keywords>
    </EventProvider>

    <Profile Id="DrmFilterProfile" Name="DRM Filter Performance" Description="DRM Filter Analysis" LoggingMode="File" DetailLevel="Verbose">
      <Collectors>
        <SystemCollectorId Value="SystemCollector">
          <SystemProviderId Value="SystemProvider"/>
        </SystemCollectorId>
        <EventCollectorId Value="EventCollector">
          <EventProviderId Value="DrmFilterProvider"/>
        </EventCollectorId>
      </Collectors>
    </Profile>
  </Profiles>
</WindowsPerformanceRecorder>
"@

$profilePath = ".\DrmFilterProfile.wprp"
$wprProfile | Out-File -FilePath $profilePath -Encoding UTF8

# 트레이스 시작
# Start trace
Write-Host "트레이스 시작 ($DurationSeconds 초)..." -ForegroundColor Yellow
# Starting trace (seconds)
wpr -start $profilePath -filemode

# 대기
# Wait
Start-Sleep -Seconds $DurationSeconds

# 트레이스 중지 및 저장
# Stop trace and save
Write-Host "트레이스 중지 및 저장..." -ForegroundColor Yellow
# Stopping trace and saving
wpr -stop $OutputPath

Write-Host "완료: $OutputPath" -ForegroundColor Green
# Complete
Write-Host "WPA (Windows Performance Analyzer)로 분석하세요."
# Analyze with WPA
```

## 32.3 메모리 최적화

### Lookaside List 사용

```c
// LookasideOptimization.c
// Lookaside List를 사용한 메모리 풀 최적화
// Memory pool optimization using Lookaside List

#include <fltKernel.h>

// 자주 할당되는 구조체
// Frequently allocated structure
typedef struct _OPERATION_CONTEXT {
    PFLT_CALLBACK_DATA Data;
    PCFLT_RELATED_OBJECTS FltObjects;
    UNICODE_STRING FileName;
    WCHAR FileNameBuffer[260];
    LARGE_INTEGER StartTime;
    ULONG Flags;
} OPERATION_CONTEXT, *POPERATION_CONTEXT;

// Lookaside List (NPAGED)
// Lookaside List (Non-paged)
NPAGED_LOOKASIDE_LIST g_OperationContextLookaside;

// 초기화
// Initialize
NTSTATUS
InitializeLookasideLists(
    VOID
    )
{
    // Lookaside List 초기화
    // Initialize Lookaside List
    ExInitializeNPagedLookasideList(
        &g_OperationContextLookaside,
        NULL,                           // 사용자 정의 할당 함수 (NULL = 기본)
        NULL,                           // 사용자 정의 해제 함수 (NULL = 기본)
        POOL_NX_ALLOCATION,            // Non-executable
        sizeof(OPERATION_CONTEXT),      // 항목 크기
        'xLtO',                         // 태그
        0                               // Depth (0 = 시스템 결정)
    );

    DbgPrint("[DrmFilter] Lookaside List 초기화 완료\n");
    // [DrmFilter] Lookaside List initialized

    return STATUS_SUCCESS;
}

// 종료
// Cleanup
VOID
CleanupLookasideLists(
    VOID
    )
{
    ExDeleteNPagedLookasideList(&g_OperationContextLookaside);

    DbgPrint("[DrmFilter] Lookaside List 정리 완료\n");
    // [DrmFilter] Lookaside List cleaned up
}

// 컨텍스트 할당 (최적화됨)
// Allocate context (optimized)
POPERATION_CONTEXT
AllocateOperationContext(
    VOID
    )
{
    POPERATION_CONTEXT context;

    // Lookaside에서 할당 - 매우 빠름
    // Allocate from Lookaside - very fast
    context = ExAllocateFromNPagedLookasideList(&g_OperationContextLookaside);

    if (context != NULL) {
        RtlZeroMemory(context, sizeof(OPERATION_CONTEXT));

        // 파일 이름 버퍼 초기화
        // Initialize file name buffer
        context->FileName.Buffer = context->FileNameBuffer;
        context->FileName.MaximumLength = sizeof(context->FileNameBuffer);
        context->FileName.Length = 0;

        PerfIncrementCounter(&g_PerfStats.AllocationCount);
    }

    return context;
}

// 컨텍스트 해제 (최적화됨)
// Free context (optimized)
VOID
FreeOperationContext(
    _In_ POPERATION_CONTEXT Context
    )
{
    if (Context != NULL) {
        // Lookaside로 반환 - 매우 빠름
        // Return to Lookaside - very fast
        ExFreeToNPagedLookasideList(&g_OperationContextLookaside, Context);
    }
}

// 사용 예
// Usage example
FLT_PREOP_CALLBACK_STATUS
OptimizedPreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    POPERATION_CONTEXT context;

    // 빠른 할당
    // Fast allocation
    context = AllocateOperationContext();
    if (context == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 컨텍스트 설정
    // Set context
    context->Data = Data;
    context->FltObjects = FltObjects;
    KeQueryPerformanceCounter(&context->StartTime);

    *CompletionContext = context;

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

FLT_POSTOP_CALLBACK_STATUS
OptimizedPostCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags
    )
{
    POPERATION_CONTEXT context = (POPERATION_CONTEXT)CompletionContext;

    // ... 작업 수행

    // 빠른 해제
    // Fast free
    FreeOperationContext(context);

    return FLT_POSTOP_FINISHED_PROCESSING;
}
```

### 스트림 컨텍스트 캐싱

```c
// StreamContextCache.c
// 스트림 컨텍스트로 정보 캐싱
// Cache information with stream context

#include <fltKernel.h>

// 캐시된 파일 정보
// Cached file information
typedef struct _STREAM_CONTEXT {
    // 기본 정보
    // Basic information
    ULONG Size;
    ULONG Flags;

    // 캐시된 보호 상태 - 매번 검사 불필요
    // Cached protection state - no need to check every time
    BOOLEAN ProtectionChecked;
    BOOLEAN IsProtected;
    ULONG ProtectionFlags;

    // 캐시된 파일 정보
    // Cached file information
    BOOLEAN FileInfoCached;
    UNICODE_STRING CachedFileName;
    WCHAR FileNameBuffer[260];
    UNICODE_STRING CachedExtension;
    WCHAR ExtensionBuffer[32];

    // 캐시 타임스탬프
    // Cache timestamp
    LARGE_INTEGER CacheTime;

    // 접근 통계
    // Access statistics
    volatile LONG AccessCount;
    volatile LONG WriteCount;

} STREAM_CONTEXT, *PSTREAM_CONTEXT;

#define STREAM_CONTEXT_TAG 'xCtS'
#define CACHE_VALIDITY_SECONDS 60  // 캐시 유효 시간

// 컨텍스트 할당 콜백
// Context allocation callback
NTSTATUS
StreamContextAllocate(
    _In_ PFLT_FILTER Filter,
    _Outptr_ PSTREAM_CONTEXT* StreamContext
    )
{
    NTSTATUS status;
    PSTREAM_CONTEXT context;

    status = FltAllocateContext(
        Filter,
        FLT_STREAM_CONTEXT,
        sizeof(STREAM_CONTEXT),
        NonPagedPool,
        (PFLT_CONTEXT*)&context
    );

    if (NT_SUCCESS(status)) {
        RtlZeroMemory(context, sizeof(STREAM_CONTEXT));
        context->Size = sizeof(STREAM_CONTEXT);

        // 버퍼 초기화
        // Initialize buffers
        context->CachedFileName.Buffer = context->FileNameBuffer;
        context->CachedFileName.MaximumLength = sizeof(context->FileNameBuffer);
        context->CachedExtension.Buffer = context->ExtensionBuffer;
        context->CachedExtension.MaximumLength = sizeof(context->ExtensionBuffer);

        *StreamContext = context;
    }

    return status;
}

// 캐시 유효성 검사
// Check cache validity
BOOLEAN
IsCacheValid(
    _In_ PSTREAM_CONTEXT Context
    )
{
    LARGE_INTEGER currentTime;
    LONGLONG elapsedSeconds;

    if (!Context->FileInfoCached) {
        return FALSE;
    }

    KeQuerySystemTime(&currentTime);
    elapsedSeconds = (currentTime.QuadPart - Context->CacheTime.QuadPart) / 10000000;

    return (elapsedSeconds < CACHE_VALIDITY_SECONDS);
}

// 스트림 컨텍스트 가져오기 또는 생성
// Get or create stream context
NTSTATUS
GetOrCreateStreamContext(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ BOOLEAN CreateIfNotExist,
    _Outptr_ PSTREAM_CONTEXT* StreamContext,
    _Out_opt_ PBOOLEAN ContextCreated
    )
{
    NTSTATUS status;
    PSTREAM_CONTEXT context = NULL;
    PSTREAM_CONTEXT oldContext;
    BOOLEAN created = FALSE;

    // 기존 컨텍스트 조회
    // Get existing context
    status = FltGetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        (PFLT_CONTEXT*)&context
    );

    if (NT_SUCCESS(status)) {
        *StreamContext = context;
        if (ContextCreated) *ContextCreated = FALSE;
        return STATUS_SUCCESS;
    }

    if (!CreateIfNotExist) {
        return status;
    }

    // 새 컨텍스트 생성
    // Create new context
    status = StreamContextAllocate(FltObjects->Filter, &context);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 컨텍스트 설정
    // Set context
    status = FltSetStreamContext(
        FltObjects->Instance,
        FltObjects->FileObject,
        FLT_SET_CONTEXT_KEEP_IF_EXISTS,
        context,
        (PFLT_CONTEXT*)&oldContext
    );

    if (status == STATUS_FLT_CONTEXT_ALREADY_DEFINED) {
        // 다른 스레드가 먼저 설정함
        // Another thread set it first
        FltReleaseContext(context);
        context = oldContext;
        status = STATUS_SUCCESS;
    }
    else if (NT_SUCCESS(status)) {
        created = TRUE;
    }
    else {
        FltReleaseContext(context);
        return status;
    }

    *StreamContext = context;
    if (ContextCreated) *ContextCreated = created;

    return STATUS_SUCCESS;
}

// 최적화된 보호 검사 (캐시 활용)
// Optimized protection check (using cache)
BOOLEAN
IsFileProtectedCached(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Out_ PULONG ProtectionFlags
    )
{
    PSTREAM_CONTEXT context;
    NTSTATUS status;
    BOOLEAN isProtected = FALSE;

    *ProtectionFlags = 0;

    // 컨텍스트 가져오기
    // Get context
    status = GetOrCreateStreamContext(
        Data,
        FltObjects,
        TRUE,
        &context,
        NULL
    );

    if (!NT_SUCCESS(status)) {
        // 캐시 없이 검사
        // Check without cache
        return CheckProtectionDirect(Data, FltObjects, ProtectionFlags);
    }

    // 캐시된 결과 확인
    // Check cached result
    if (context->ProtectionChecked && IsCacheValid(context)) {
        // 캐시 히트!
        // Cache hit!
        isProtected = context->IsProtected;
        *ProtectionFlags = context->ProtectionFlags;

        InterlockedIncrement(&context->AccessCount);
        FltReleaseContext(context);

        return isProtected;
    }

    // 캐시 미스 - 실제 검사 수행
    // Cache miss - perform actual check
    isProtected = CheckProtectionDirect(Data, FltObjects, ProtectionFlags);

    // 결과 캐싱
    // Cache result
    context->ProtectionChecked = TRUE;
    context->IsProtected = isProtected;
    context->ProtectionFlags = *ProtectionFlags;
    KeQuerySystemTime(&context->CacheTime);

    FltReleaseContext(context);

    return isProtected;
}
```

## 32.4 CPU 최적화

### 조기 반환 패턴

```c
// EarlyExit.c
// 빠른 조기 반환 패턴
// Fast early exit pattern

FLT_PREOP_CALLBACK_STATUS
OptimizedPreCreate(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    // 1. 가장 빠른 검사부터 수행
    // 1. Perform fastest checks first

    // 커널 모드 요청 무시 (비용: 매우 낮음)
    // Ignore kernel mode requests (cost: very low)
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 디렉터리 무시 (비용: 낮음)
    // Ignore directories (cost: low)
    if (FlagOn(Data->Iopb->Parameters.Create.Options, FILE_DIRECTORY_FILE)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 네트워크 요청 무시 (비용: 낮음)
    // Ignore network requests (cost: low)
    if (FlagOn(Data->Iopb->OperationFlags, SL_OPEN_TARGET_DIRECTORY)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 2. 필터링 대상 여부 빠른 검사
    // 2. Quick check if filtering is needed

    // 읽기 전용 접근 무시 (정책에 따라)
    // Ignore read-only access (depending on policy)
    ACCESS_MASK desiredAccess = Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess;

    if (!(desiredAccess & (FILE_WRITE_DATA | FILE_APPEND_DATA |
                           DELETE | FILE_WRITE_ATTRIBUTES))) {
        // 쓰기/삭제/속성변경 없음 - 무시
        // No write/delete/attribute change - ignore
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 3. 이제 비용이 큰 검사 수행
    // 3. Now perform expensive checks

    // 파일 이름 조회 (비용: 높음)
    // File name query (cost: high)
    PFLT_FILE_NAME_INFORMATION nameInfo;
    NTSTATUS status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (!NT_SUCCESS(status)) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    FltParseFileNameInformation(nameInfo);

    // 4. 확장자 기반 빠른 필터링
    // 4. Quick filtering by extension

    static const UNICODE_STRING protectedExtensions[] = {
        RTL_CONSTANT_STRING(L".docx"),
        RTL_CONSTANT_STRING(L".xlsx"),
        RTL_CONSTANT_STRING(L".pdf"),
        { 0, 0, NULL }  // 종료 표시
    };

    BOOLEAN needsProtection = FALSE;

    for (int i = 0; protectedExtensions[i].Buffer != NULL; i++) {
        if (RtlEqualUnicodeString(
                &nameInfo->Extension,
                &protectedExtensions[i],
                TRUE)) {
            needsProtection = TRUE;
            break;
        }
    }

    if (!needsProtection) {
        FltReleaseFileNameInformation(nameInfo);
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 5. 실제 보호 로직 수행
    // 5. Perform actual protection logic
    // ...

    FltReleaseFileNameInformation(nameInfo);
    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

### 비동기 처리

```c
// AsyncProcessing.c
// 비동기 처리로 콜백 지연 최소화
// Minimize callback latency with async processing

#include <fltKernel.h>

// 작업 항목 구조체
// Work item structure
typedef struct _ASYNC_WORK_ITEM {
    PFLT_DEFERRED_IO_WORKITEM WorkItem;
    PFLT_CALLBACK_DATA Data;
    PCFLT_RELATED_OBJECTS FltObjects;
    UNICODE_STRING FilePath;
    WCHAR FilePathBuffer[260];
    ULONG Operation;
} ASYNC_WORK_ITEM, *PASYNC_WORK_ITEM;

// 비동기 작업자 함수
// Async worker function
VOID
AsyncWorkItemRoutine(
    _In_ PFLT_DEFERRED_IO_WORKITEM WorkItem,
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PVOID Context
    )
{
    PASYNC_WORK_ITEM asyncItem = (PASYNC_WORK_ITEM)Context;

    // 여기서 비용이 큰 작업 수행
    // Perform expensive operations here
    // - 사용자 모드 통신
    // - User mode communication
    // - 콘텐츠 스캔
    // - Content scanning
    // - 데이터베이스 조회
    // - Database lookup

    // 예: 로깅을 위한 사용자 모드 메시지
    // Example: User mode message for logging
    SendMessageToUserMode(
        asyncItem->Operation,
        &asyncItem->FilePath,
        Data->IoStatus.Status
    );

    // 작업 항목 해제
    // Free work item
    FltFreeDeferredIoWorkItem(WorkItem);
    ExFreePoolWithTag(asyncItem, 'ysAW');
}

// PostCreate에서 비동기 로깅
// Async logging in PostCreate
FLT_POSTOP_CALLBACK_STATUS
PostCreateWithAsyncLogging(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_opt_ PVOID CompletionContext,
    _In_ FLT_POST_OPERATION_FLAGS Flags
    )
{
    PASYNC_WORK_ITEM asyncItem;
    PFLT_DEFERRED_IO_WORKITEM workItem;
    NTSTATUS status;

    // 성공한 작업만 로깅
    // Only log successful operations
    if (!NT_SUCCESS(Data->IoStatus.Status)) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 비동기 작업 항목 할당
    // Allocate async work item
    asyncItem = ExAllocatePool2(
        POOL_FLAG_NON_PAGED,
        sizeof(ASYNC_WORK_ITEM),
        'ysAW'
    );

    if (asyncItem == NULL) {
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 지연 작업 항목 할당
    // Allocate deferred work item
    workItem = FltAllocateDeferredIoWorkItem();
    if (workItem == NULL) {
        ExFreePoolWithTag(asyncItem, 'ysAW');
        return FLT_POSTOP_FINISHED_PROCESSING;
    }

    // 작업 정보 복사
    // Copy work info
    asyncItem->WorkItem = workItem;
    asyncItem->Data = Data;
    asyncItem->FltObjects = FltObjects;
    asyncItem->Operation = IRP_MJ_CREATE;

    // 파일 경로 복사 (지금 복사해야 함)
    // Copy file path (must copy now)
    asyncItem->FilePath.Buffer = asyncItem->FilePathBuffer;
    asyncItem->FilePath.MaximumLength = sizeof(asyncItem->FilePathBuffer);

    if (CompletionContext != NULL) {
        POPERATION_CONTEXT ctx = (POPERATION_CONTEXT)CompletionContext;
        RtlCopyUnicodeString(&asyncItem->FilePath, &ctx->FileName);
    }

    // 비동기 작업 큐잉
    // Queue async work
    status = FltQueueDeferredIoWorkItem(
        workItem,
        Data,
        AsyncWorkItemRoutine,
        DelayedWorkQueue,  // 낮은 우선순위
        asyncItem
    );

    if (!NT_SUCCESS(status)) {
        FltFreeDeferredIoWorkItem(workItem);
        ExFreePoolWithTag(asyncItem, 'ysAW');
    }

    return FLT_POSTOP_FINISHED_PROCESSING;
}

// 시스템 스레드에서 무거운 작업 수행
// Perform heavy work in system thread
VOID
HeavyWorkSystemThread(
    _In_ PVOID Context
    )
{
    PWORK_QUEUE_ITEM workItem;

    while (TRUE) {
        // 작업 대기
        // Wait for work
        workItem = DequeueWorkItem();

        if (workItem == NULL) {
            // 종료 신호
            // Shutdown signal
            break;
        }

        // 무거운 작업 수행 (콜백 외부)
        // Perform heavy work (outside callback)
        PerformContentScan(workItem);
        UpdateDatabase(workItem);
        SendNotification(workItem);

        FreeWorkItem(workItem);
    }

    PsTerminateSystemThread(STATUS_SUCCESS);
}
```

### 잠금 최적화

```c
// LockOptimization.c
// 잠금 전략 최적화
// Lock strategy optimization

#include <fltKernel.h>

// 잘못된 예: 전역 스핀락
// Bad example: Global spinlock
KSPIN_LOCK g_BadGlobalLock;

// 좋은 예: 세분화된 잠금
// Good example: Fine-grained locking

// 해시 테이블 기반 잠금
// Hash table based locking
#define LOCK_TABLE_SIZE 64

typedef struct _POLICY_CACHE {
    LIST_ENTRY Entries;
    EX_PUSH_LOCK Lock;  // 푸시 잠금 사용
} POLICY_CACHE, *PPOLICY_CACHE;

POLICY_CACHE g_PolicyCache[LOCK_TABLE_SIZE];

// 해시 함수
// Hash function
ULONG
GetCacheBucket(
    _In_ PCUNICODE_STRING Path
    )
{
    ULONG hash = 0;

    for (USHORT i = 0; i < Path->Length / sizeof(WCHAR); i++) {
        hash = hash * 31 + towlower(Path->Buffer[i]);
    }

    return hash % LOCK_TABLE_SIZE;
}

// 초기화
// Initialize
VOID
InitializePolicyCache(
    VOID
    )
{
    for (int i = 0; i < LOCK_TABLE_SIZE; i++) {
        InitializeListHead(&g_PolicyCache[i].Entries);
        FltInitializePushLock(&g_PolicyCache[i].Lock);
    }
}

// 캐시 조회 (읽기 잠금)
// Cache lookup (read lock)
PPOLICY_ENTRY
LookupPolicyCache(
    _In_ PCUNICODE_STRING Path
    )
{
    ULONG bucket = GetCacheBucket(Path);
    PPOLICY_CACHE cache = &g_PolicyCache[bucket];
    PPOLICY_ENTRY result = NULL;

    // 공유(읽기) 잠금 획득
    // Acquire shared (read) lock
    FltAcquirePushLockShared(&cache->Lock);

    for (PLIST_ENTRY entry = cache->Entries.Flink;
         entry != &cache->Entries;
         entry = entry->Flink) {

        PPOLICY_ENTRY policyEntry = CONTAINING_RECORD(entry, POLICY_ENTRY, ListEntry);

        if (RtlEqualUnicodeString(&policyEntry->Path, Path, TRUE)) {
            result = policyEntry;
            break;
        }
    }

    FltReleasePushLock(&cache->Lock);

    return result;
}

// 캐시 추가 (쓰기 잠금)
// Cache add (write lock)
VOID
AddToPolicyCache(
    _In_ PPOLICY_ENTRY Entry
    )
{
    ULONG bucket = GetCacheBucket(&Entry->Path);
    PPOLICY_CACHE cache = &g_PolicyCache[bucket];

    // 배타적(쓰기) 잠금 획득
    // Acquire exclusive (write) lock
    FltAcquirePushLockExclusive(&cache->Lock);

    InsertTailList(&cache->Entries, &Entry->ListEntry);

    FltReleasePushLock(&cache->Lock);
}

// 락프리 카운터 사용
// Use lock-free counters
typedef struct _LOCK_FREE_STATS {
    volatile LONG64 ReadCount;
    volatile LONG64 WriteCount;
    volatile LONG64 BlockCount;
} LOCK_FREE_STATS, *PLOCK_FREE_STATS;

LOCK_FREE_STATS g_Stats = { 0 };

// 잠금 없이 통계 업데이트
// Update statistics without locking
VOID
IncrementReadCount(VOID)
{
    InterlockedIncrement64(&g_Stats.ReadCount);
}

VOID
IncrementWriteCount(VOID)
{
    InterlockedIncrement64(&g_Stats.WriteCount);
}

// Read-Copy-Update (RCU) 패턴
// Read-Copy-Update (RCU) pattern
typedef struct _CONFIG_DATA {
    volatile LONG RefCount;
    ULONG Flags;
    UNICODE_STRING ProtectedPath;
    // ... 기타 설정
} CONFIG_DATA, *PCONFIG_DATA;

PCONFIG_DATA volatile g_CurrentConfig = NULL;

// 설정 읽기 (잠금 없음)
// Read config (no lock)
PCONFIG_DATA
AcquireConfigRef(VOID)
{
    PCONFIG_DATA config;

    do {
        config = g_CurrentConfig;
        if (config == NULL) {
            return NULL;
        }
    } while (InterlockedIncrement(&config->RefCount) <= 1);

    // 다른 스레드가 해제 중일 수 있음
    // Another thread might be freeing

    return config;
}

VOID
ReleaseConfigRef(
    _In_ PCONFIG_DATA Config
    )
{
    if (InterlockedDecrement(&Config->RefCount) == 0) {
        // 마지막 참조 - 메모리 해제
        // Last reference - free memory
        ExFreePoolWithTag(Config, 'gfnC');
    }
}

// 설정 업데이트 (새 복사본)
// Update config (new copy)
VOID
UpdateConfig(
    _In_ PCONFIG_DATA NewConfig
    )
{
    PCONFIG_DATA oldConfig;

    // 새 설정에 참조 카운트 1로 시작
    // Start new config with ref count 1
    NewConfig->RefCount = 1;

    // 원자적 교체
    // Atomic swap
    oldConfig = InterlockedExchangePointer((PVOID*)&g_CurrentConfig, NewConfig);

    // 이전 설정 참조 해제
    // Release old config
    if (oldConfig != NULL) {
        ReleaseConfigRef(oldConfig);
    }
}
```

## 32.5 I/O 최적화

### 파일 이름 캐싱

```c
// FileNameCaching.c
// 파일 이름 조회 최적화
// File name query optimization

// FltGetFileNameInformation은 비용이 큼
// FltGetFileNameInformation is expensive
// - 파일 시스템 쿼리 필요
// - File system query required
// - 문자열 정규화 처리
// - String normalization
// - 메모리 할당
// - Memory allocation

// 최적화 전략:
// Optimization strategies:

// 1. 필요할 때만 조회
// 1. Query only when needed
FLT_PREOP_CALLBACK_STATUS
OptimizedNameQuery(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    // 먼저 간단한 검사로 조기 반환
    // First, early return with simple checks
    if (Data->RequestorMode == KernelMode) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    ACCESS_MASK access = Data->Iopb->Parameters.Create.SecurityContext->DesiredAccess;
    if (!(access & (FILE_WRITE_DATA | DELETE))) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 이제 파일 이름이 필요 - 조회
    // Now file name is needed - query
    PFLT_FILE_NAME_INFORMATION nameInfo;
    NTSTATUS status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    // ...

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

// 2. 적절한 플래그 사용
// 2. Use appropriate flags

// 느림: FLT_FILE_NAME_NORMALIZED (정규화 필요)
// Slow: FLT_FILE_NAME_NORMALIZED (needs normalization)

// 빠름: FLT_FILE_NAME_OPENED (이미 열린 이름 사용)
// Fast: FLT_FILE_NAME_OPENED (use already opened name)

// 3. 캐시 플래그 활용
// 3. Use cache flags

// 캐시에서 조회 시도
// Try to query from cache
status = FltGetFileNameInformation(
    Data,
    FLT_FILE_NAME_NORMALIZED |
    FLT_FILE_NAME_QUERY_CACHE_ONLY,  // 캐시만 확인
    &nameInfo
);

if (status == STATUS_FLT_NAME_CACHE_MISS) {
    // 캐시 미스 - 필요하면 전체 조회
    // Cache miss - full query if needed
    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED |
        FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );
}

// 4. 스트림 컨텍스트에 이름 캐싱
// 4. Cache name in stream context

NTSTATUS
CacheFileName(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PSTREAM_CONTEXT Context
    )
{
    PFLT_FILE_NAME_INFORMATION nameInfo;
    NTSTATUS status;

    // 이미 캐시되어 있으면 건너뜀
    // Skip if already cached
    if (Context->FileInfoCached) {
        return STATUS_SUCCESS;
    }

    status = FltGetFileNameInformation(
        Data,
        FLT_FILE_NAME_NORMALIZED | FLT_FILE_NAME_QUERY_DEFAULT,
        &nameInfo
    );

    if (NT_SUCCESS(status)) {
        FltParseFileNameInformation(nameInfo);

        // 스트림 컨텍스트에 복사
        // Copy to stream context
        RtlCopyUnicodeString(&Context->CachedFileName, &nameInfo->Name);
        RtlCopyUnicodeString(&Context->CachedExtension, &nameInfo->Extension);
        KeQuerySystemTime(&Context->CacheTime);
        Context->FileInfoCached = TRUE;

        FltReleaseFileNameInformation(nameInfo);
    }

    return status;
}
```

### 버퍼 처리 최적화

```c
// BufferOptimization.c
// 버퍼 처리 최적화
// Buffer processing optimization

// 대용량 버퍼 처리 시 최적화
// Optimization for large buffer processing

FLT_PREOP_CALLBACK_STATUS
OptimizedPreRead(
    _Inout_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _Flt_CompletionContext_Outptr_ PVOID *CompletionContext
    )
{
    // 작은 읽기는 무시
    // Ignore small reads
    if (Data->Iopb->Parameters.Read.Length < 4096) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // MDL 사용 시 시스템 버퍼 매핑 최소화
    // Minimize system buffer mapping when using MDL
    PVOID buffer = NULL;

    if (Data->Iopb->Parameters.Read.MdlAddress != NULL) {
        // MDL에서 주소 가져오기 (매핑 불필요할 수 있음)
        // Get address from MDL (mapping may not be needed)
        buffer = MmGetSystemAddressForMdlSafe(
            Data->Iopb->Parameters.Read.MdlAddress,
            NormalPagePriority | MdlMappingNoExecute
        );
    }
    else if (FlagOn(Data->Flags, FLTFL_CALLBACK_DATA_SYSTEM_BUFFER)) {
        // 시스템 버퍼 직접 사용
        // Use system buffer directly
        buffer = Data->Iopb->Parameters.Read.ReadBuffer;
    }

    if (buffer == NULL) {
        return FLT_PREOP_SUCCESS_NO_CALLBACK;
    }

    // 버퍼 처리...
    // Process buffer...

    return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}

// 콘텐츠 스캔 최적화
// Content scan optimization
BOOLEAN
OptimizedContentScan(
    _In_ PVOID Buffer,
    _In_ ULONG Length,
    _Out_ PULONG MatchOffset
    )
{
    // 1. 작은 버퍼 최적화
    // 1. Small buffer optimization
    if (Length < 64) {
        return SimpleScan(Buffer, Length, MatchOffset);
    }

    // 2. SIMD 활용 (가능한 경우)
    // 2. Use SIMD (if available)
    // x64에서는 SSE/AVX 사용 가능
    // SSE/AVX available on x64

    // 3. 청크 단위 처리
    // 3. Process in chunks
    const ULONG CHUNK_SIZE = 4096;
    PUCHAR current = (PUCHAR)Buffer;
    ULONG remaining = Length;

    while (remaining > 0) {
        ULONG scanSize = min(remaining, CHUNK_SIZE);

        if (ScanChunk(current, scanSize, MatchOffset)) {
            *MatchOffset += (ULONG)(current - (PUCHAR)Buffer);
            return TRUE;
        }

        current += scanSize;
        remaining -= scanSize;
    }

    return FALSE;
}
```

## 32.6 성능 모니터링 대시보드

### WPF 성능 모니터 UI

```csharp
// PerformanceMonitorView.xaml.cs
// 성능 모니터링 UI
// Performance monitoring UI

namespace DrmFilterManager.Views;

public partial class PerformanceMonitorView : UserControl
{
    private readonly DispatcherTimer _refreshTimer;
    private readonly IDriverService _driverService;

    public PerformanceMonitorView(IDriverService driverService)
    {
        InitializeComponent();
        _driverService = driverService;

        // 1초마다 갱신
        // Refresh every second
        _refreshTimer = new DispatcherTimer
        {
            Interval = TimeSpan.FromSeconds(1)
        };
        _refreshTimer.Tick += RefreshTimer_Tick;
    }

    private async void RefreshTimer_Tick(object? sender, EventArgs e)
    {
        var stats = await _driverService.GetPerformanceStatsAsync();

        // UI 업데이트
        // Update UI
        TotalEventsText.Text = $"{stats.TotalEvents:N0}";
        EventsPerSecText.Text = $"{stats.EventsPerSecond:N0}/sec";
        AvgLatencyText.Text = $"{stats.AverageLatencyUs:F2} μs";
        MaxLatencyText.Text = $"{stats.MaxLatencyUs:F2} μs";

        BlockedCountText.Text = $"{stats.BlockedOperations:N0}";
        BlockRateText.Text = $"{stats.BlockRatePercent:F1}%";

        MemoryUsageText.Text = $"{stats.MemoryUsageKB:N0} KB";
        PoolAllocationsText.Text = $"{stats.PoolAllocations:N0}";

        // 그래프 업데이트
        // Update graph
        UpdateLatencyGraph(stats.LatencyHistogram);
        UpdateThroughputGraph(stats.EventsPerSecond);
    }

    public void StartMonitoring()
    {
        _refreshTimer.Start();
    }

    public void StopMonitoring()
    {
        _refreshTimer.Stop();
    }

    private void UpdateLatencyGraph(int[] histogram)
    {
        // LiveCharts 사용
        // Using LiveCharts
        LatencySeries.Values = new ChartValues<int>(histogram);
    }

    private void UpdateThroughputGraph(double eventsPerSec)
    {
        // 시계열 데이터 추가
        // Add time series data
        ThroughputSeries.Values.Add(eventsPerSec);

        // 최근 60개만 유지
        // Keep only recent 60 points
        if (ThroughputSeries.Values.Count > 60)
        {
            ThroughputSeries.Values.RemoveAt(0);
        }
    }
}
```

```xml
<!-- PerformanceMonitorView.xaml -->
<UserControl x:Class="DrmFilterManager.Views.PerformanceMonitorView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:lvc="clr-namespace:LiveChartsCore.SkiaSharpView.WPF;assembly=LiveChartsCore.SkiaSharpView.WPF">

    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- 통계 카드 -->
        <!-- Statistics Cards -->
        <UniformGrid Grid.Row="0" Columns="4" Margin="0,0,0,10">
            <Border Background="#E3F2FD" CornerRadius="5" Padding="15" Margin="5">
                <StackPanel>
                    <TextBlock Text="총 이벤트" FontSize="12" Foreground="Gray"/>
                    <TextBlock x:Name="TotalEventsText" FontSize="24" FontWeight="Bold"/>
                    <TextBlock x:Name="EventsPerSecText" FontSize="12" Foreground="Green"/>
                </StackPanel>
            </Border>

            <Border Background="#FFF3E0" CornerRadius="5" Padding="15" Margin="5">
                <StackPanel>
                    <TextBlock Text="평균 지연" FontSize="12" Foreground="Gray"/>
                    <TextBlock x:Name="AvgLatencyText" FontSize="24" FontWeight="Bold"/>
                    <TextBlock x:Name="MaxLatencyText" FontSize="12" Foreground="Orange"/>
                </StackPanel>
            </Border>

            <Border Background="#FFEBEE" CornerRadius="5" Padding="15" Margin="5">
                <StackPanel>
                    <TextBlock Text="차단됨" FontSize="12" Foreground="Gray"/>
                    <TextBlock x:Name="BlockedCountText" FontSize="24" FontWeight="Bold" Foreground="#D32F2F"/>
                    <TextBlock x:Name="BlockRateText" FontSize="12" Foreground="Red"/>
                </StackPanel>
            </Border>

            <Border Background="#E8F5E9" CornerRadius="5" Padding="15" Margin="5">
                <StackPanel>
                    <TextBlock Text="메모리" FontSize="12" Foreground="Gray"/>
                    <TextBlock x:Name="MemoryUsageText" FontSize="24" FontWeight="Bold"/>
                    <TextBlock x:Name="PoolAllocationsText" FontSize="12" Foreground="Green"/>
                </StackPanel>
            </Border>
        </UniformGrid>

        <!-- 처리량 그래프 -->
        <!-- Throughput Graph -->
        <Border Grid.Row="1" Background="White" CornerRadius="5" Padding="10" Margin="5">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>

                <TextBlock Text="처리량 (이벤트/초)" FontWeight="SemiBold" Margin="0,0,0,10"/>

                <lvc:CartesianChart Grid.Row="1" x:Name="ThroughputChart">
                    <lvc:CartesianChart.Series>
                        <lvc:LineSeries x:Name="ThroughputSeries"
                                        Stroke="Blue"
                                        Fill="LightBlue"
                                        GeometrySize="0"/>
                    </lvc:CartesianChart.Series>
                </lvc:CartesianChart>
            </Grid>
        </Border>

        <!-- 지연 시간 히스토그램 -->
        <!-- Latency Histogram -->
        <Border Grid.Row="2" Background="White" CornerRadius="5" Padding="10" Margin="5">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>

                <TextBlock Text="지연 시간 분포" FontWeight="SemiBold" Margin="0,0,0,10"/>

                <lvc:CartesianChart Grid.Row="1" x:Name="LatencyChart">
                    <lvc:CartesianChart.Series>
                        <lvc:ColumnSeries x:Name="LatencySeries"
                                          Fill="Orange"/>
                    </lvc:CartesianChart.Series>
                    <lvc:CartesianChart.XAxes>
                        <lvc:Axis Labels="&lt;1μs,1-10μs,10-100μs,100μs-1ms,&gt;1ms"/>
                    </lvc:CartesianChart.XAxes>
                </lvc:CartesianChart>
            </Grid>
        </Border>
    </Grid>
</UserControl>
```

## 32.7 정리

이 챕터에서 학습한 내용:

1. **성능 병목 이해**
   - 파일 이름 조회 비용
   - 동기적 통신 지연
   - 잠금 경합 문제

2. **측정 도구**
   - ETW 트레이싱
   - 성능 카운터
   - WPA 분석

3. **메모리 최적화**
   - Lookaside List 활용
   - 스트림 컨텍스트 캐싱
   - 풀 사용량 최소화

4. **CPU 최적화**
   - 조기 반환 패턴
   - 비동기 처리
   - 세분화된 잠금

5. **I/O 최적화**
   - 파일 이름 캐싱
   - 버퍼 처리 효율화
   - MDL 활용

6. **모니터링**
   - 실시간 성능 대시보드
   - 통계 수집 및 분석

Part 10이 완료되었습니다. 다음 Part 11에서는 **배포**를 다룹니다.

---

**다음 챕터 예고**: Chapter 33에서는 드라이버 서명, 인증서 관리, Windows 보안 정책을 학습합니다.
