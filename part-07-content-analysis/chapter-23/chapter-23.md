# Chapter 23: Office 문서 파싱

## 난이도: ⭐⭐⭐⭐⭐ (고급)

## 학습 목표
- Office Open XML(OOXML) 형식의 구조 이해
- 커널에서 ZIP 처리 전략 수립 및 구현
- 텍스트 추출을 위한 아키텍처 설계

---

## 23.1 Office Open XML 형식 이해

### 23.1.1 OOXML의 탄생 배경

Office 2007부터 Microsoft는 기존 바이너리 형식(.doc, .xls, .ppt)에서 XML 기반의 새로운 형식으로 전환했습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Office 파일 형식의 진화                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  이전 형식 (바이너리)              현재 형식 (OOXML)             │
│  Legacy (Binary)                  Current (OOXML)              │
│                                                                 │
│  ┌───────────┐                   ┌───────────┐                 │
│  │  .doc     │                   │  .docx    │                 │
│  │  .xls     │     ────→         │  .xlsx    │                 │
│  │  .ppt     │   Office 2007     │  .pptx    │                 │
│  └───────────┘                   └───────────┘                 │
│                                                                 │
│  특징:                           특징:                          │
│  - 독점적 바이너리                - 공개 표준 (ISO/IEC 29500)    │
│  - 파싱 어려움                    - ZIP 컨테이너 + XML           │
│  - 보안 취약점 많음               - 상대적으로 분석 용이          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 23.1.2 OOXML = ZIP + XML

OOXML 문서는 실제로 ZIP 아카이브입니다. 확장자만 변경하면 일반 압축 도구로 열 수 있습니다.

```
document.docx (실제로는 ZIP 아카이브)
document.docx (actually a ZIP archive)
│
├── [Content_Types].xml       ← 콘텐츠 타입 정의
│                               Content type definitions
├── _rels/
│   └── .rels                 ← 최상위 관계 정의
│                               Top-level relationship definitions
├── word/
│   ├── document.xml          ← ★ 본문 텍스트 (핵심!)
│   │                           ★ Main body text (key!)
│   ├── styles.xml            ← 스타일 정의
│   │                           Style definitions
│   ├── settings.xml          ← 문서 설정
│   │                           Document settings
│   ├── fontTable.xml         ← 폰트 정보
│   │                           Font information
│   ├── webSettings.xml
│   ├── numbering.xml         ← 번호 매기기 스타일
│   │                           Numbering styles
│   ├── media/                ← 삽입된 이미지들
│   │   ├── image1.png          Embedded images
│   │   └── image2.jpg
│   └── _rels/
│       └── document.xml.rels ← 본문 관계 정의
│                               Body relationship definitions
├── docProps/
│   ├── app.xml               ← 애플리케이션 속성
│   │                           Application properties
│   └── core.xml              ← 핵심 메타데이터 (작성자 등)
│                               Core metadata (author, etc.)
└── customXml/                ← 사용자 정의 XML (선택)
                                Custom XML (optional)
```

### 23.1.3 ZIP 파일 구조

ZIP 형식을 이해하면 OOXML 처리 전략을 수립할 수 있습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                       ZIP 파일 구조                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Local File Header #1                                     │  │
│  │ ┌────────────────────────────────────────────────────┐   │  │
│  │ │ Signature: 0x04034B50 ("PK\x03\x04")               │   │  │
│  │ │ Version needed to extract                          │   │  │
│  │ │ General purpose bit flag                           │   │  │
│  │ │ Compression method (0=Store, 8=Deflate)            │   │  │
│  │ │ Last mod file time/date                            │   │  │
│  │ │ CRC-32                                             │   │  │
│  │ │ Compressed size                                    │   │  │
│  │ │ Uncompressed size                                  │   │  │
│  │ │ File name length                                   │   │  │
│  │ │ Extra field length                                 │   │  │
│  │ └────────────────────────────────────────────────────┘   │  │
│  │ File Name                                                │  │
│  │ Extra Field (optional)                                   │  │
│  │ File Data (compressed or stored)                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Local File Header #2                                     │  │
│  │ ...                                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Central Directory                                        │  │
│  │ ┌────────────────────────────────────────────────────┐   │  │
│  │ │ Central Directory File Header #1                   │   │  │
│  │ │ Signature: 0x02014B50 ("PK\x01\x02")               │   │  │
│  │ │ ...                                                │   │  │
│  │ └────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ End of Central Directory Record                          │  │
│  │ Signature: 0x06054B50 ("PK\x05\x06")                     │  │
│  │ Number of entries                                        │  │
│  │ Central directory offset                                 │  │
│  │ Comment                                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 23.1.4 ZIP 시그니처 확인

```c
// ZIP 시그니처 상수
// ZIP signature constants
#define ZIP_LOCAL_FILE_SIGNATURE    0x04034B50  // "PK\x03\x04"
#define ZIP_CENTRAL_DIR_SIGNATURE   0x02014B50  // "PK\x01\x02"
#define ZIP_END_CENTRAL_SIGNATURE   0x06054B50  // "PK\x05\x06"

// ZIP 로컬 파일 헤더 구조체
// ZIP Local File Header structure
#pragma pack(push, 1)
typedef struct _ZIP_LOCAL_FILE_HEADER {
    ULONG Signature;            // 0x04034B50
    USHORT VersionNeeded;       // 압축 해제 필요 버전
                                // Version needed to extract
    USHORT GeneralFlags;        // 일반 플래그
                                // General purpose flags
    USHORT CompressionMethod;   // 압축 방식 (0=Store, 8=Deflate)
                                // Compression method
    USHORT LastModTime;         // 최종 수정 시간
                                // Last modification time
    USHORT LastModDate;         // 최종 수정 날짜
                                // Last modification date
    ULONG Crc32;                // CRC-32 체크섬
                                // CRC-32 checksum
    ULONG CompressedSize;       // 압축된 크기
                                // Compressed size
    ULONG UncompressedSize;     // 원본 크기
                                // Uncompressed size
    USHORT FileNameLength;      // 파일명 길이
                                // File name length
    USHORT ExtraFieldLength;    // 추가 필드 길이
                                // Extra field length
    // 이후 가변 길이 필드들
    // Followed by variable length fields
    // CHAR FileName[FileNameLength];
    // UCHAR ExtraField[ExtraFieldLength];
    // UCHAR FileData[CompressedSize];
} ZIP_LOCAL_FILE_HEADER, *PZIP_LOCAL_FILE_HEADER;
#pragma pack(pop)

// ZIP 파일 여부 확인 (빠른 검사)
// Check if file is ZIP (quick check)
BOOLEAN IsZipFile(
    _In_reads_bytes_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength)
{
    if (BufferLength < sizeof(ULONG)) {
        return FALSE;
    }

    // 시그니처 확인 (리틀 엔디안)
    // Check signature (little-endian)
    ULONG signature = *(PULONG)Buffer;

    return (signature == ZIP_LOCAL_FILE_SIGNATURE);
}

// OOXML 파일 여부 확인 (확장자 + 시그니처)
// Check if file is OOXML (extension + signature)
BOOLEAN IsOoxmlFile(
    _In_ PCUNICODE_STRING FileName,
    _In_reads_bytes_opt_(BufferLength) PVOID Buffer,
    _In_ ULONG BufferLength)
{
    // 확장자 확인
    // Check extension
    UNICODE_STRING docx = RTL_CONSTANT_STRING(L".docx");
    UNICODE_STRING xlsx = RTL_CONSTANT_STRING(L".xlsx");
    UNICODE_STRING pptx = RTL_CONSTANT_STRING(L".pptx");

    BOOLEAN hasOoxmlExtension =
        RtlSuffixUnicodeString(&docx, FileName, TRUE) ||
        RtlSuffixUnicodeString(&xlsx, FileName, TRUE) ||
        RtlSuffixUnicodeString(&pptx, FileName, TRUE);

    if (!hasOoxmlExtension) {
        return FALSE;
    }

    // 버퍼가 있으면 ZIP 시그니처도 확인
    // If buffer provided, also verify ZIP signature
    if (Buffer != NULL && BufferLength >= 4) {
        return IsZipFile(Buffer, BufferLength);
    }

    // 확장자만으로 판단
    // Determine by extension only
    return TRUE;
}
```

### 23.1.5 document.xml의 구조

Word 문서의 본문 텍스트는 `word/document.xml`에 있습니다.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<w:document xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main"
            xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships">
  <w:body>
    <w:p>                           <!-- paragraph: 문단 -->
      <w:pPr>                       <!-- paragraph properties: 문단 속성 -->
        <w:jc w:val="center"/>
      </w:pPr>
      <w:r>                         <!-- run: 텍스트 런 -->
        <w:rPr>                     <!-- run properties: 런 속성 -->
          <w:b/>                    <!-- bold: 굵게 -->
        </w:rPr>
        <w:t>제목입니다</w:t>        <!-- text: 실제 텍스트 -->
      </w:r>
    </w:p>
    <w:p>
      <w:r>
        <w:t>본문 내용입니다. 주민등록번호: 880101-1234567</w:t>
      </w:r>
    </w:p>
    <w:sectPr>                      <!-- section properties: 섹션 속성 -->
      <!-- ... -->
    </w:sectPr>
  </w:body>
</w:document>
```

텍스트 추출의 핵심은 모든 `<w:t>` 태그의 내용을 수집하는 것입니다.

---

## 23.2 커널에서 ZIP 처리 전략

### 23.2.1 전략 비교 분석

커널에서 OOXML을 처리하는 세 가지 전략이 있습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    OOXML 처리 전략 비교                          │
├─────────────────┬─────────────────┬─────────────────────────────┤
│      전략        │      장점        │           단점             │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ 1. 커널 내      │ - 빠른 처리     │ - 복잡한 구현 필요           │
│    완전 파싱    │ - 지연 없음     │ - 보안 위험 (파싱 버그)      │
│                 │                 │ - Deflate 구현 필요          │
│                 │                 │ - 메모리 사용량 증가          │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ 2. 사용자 모드  │ - 안전함        │ - 컨텍스트 스위칭 오버헤드   │
│    서비스 연동  │ - 구현 용이     │ - 통신 지연                  │
│                 │ - .NET 라이브러리│ - 서비스 의존성              │
│                 │   활용 가능     │                              │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ 3. 하이브리드   │ - 균형 잡힌     │ - 구현 복잡도 증가           │
│    접근법       │   성능/안전성   │ - 두 가지 코드 경로 관리      │
│                 │ - 유연한 확장   │                              │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

### 23.2.2 권장 접근법: 하이브리드 전략

```
┌─────────────────────────────────────────────────────────────────┐
│                   하이브리드 처리 아키텍처                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Minifilter (커널)                      │  │
│  │  ┌────────────────┐  ┌────────────────────────────────┐  │  │
│  │  │ 1. 확장자 검사  │  │ 4. 추출된 텍스트 SSN 스캔      │  │  │
│  │  │ Extension Check│  │ Scan extracted text for SSN    │  │  │
│  │  └───────┬────────┘  └────────────────▲───────────────┘  │  │
│  │          │                            │                   │  │
│  │          v                            │                   │  │
│  │  ┌────────────────┐                   │                   │  │
│  │  │ 2. ZIP 시그니처│                   │                   │  │
│  │  │ 빠른 검증      │                   │                   │  │
│  │  └───────┬────────┘                   │                   │  │
│  │          │                            │                   │  │
│  └──────────┼────────────────────────────┼───────────────────┘  │
│             │                            │                      │
│             │  FltSendMessage            │  FilterReplyMessage  │
│             │                            │                      │
│  ┌──────────v────────────────────────────┴───────────────────┐  │
│  │                 사용자 모드 서비스                          │  │
│  │  ┌────────────────────────────────────────────────────┐   │  │
│  │  │ 3. ZIP 압축 해제 + XML 파싱 + 텍스트 추출           │   │  │
│  │  │    Decompress ZIP + Parse XML + Extract text       │   │  │
│  │  │    (Open XML SDK / .NET ZipArchive 사용)           │   │  │
│  │  └────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 23.2.3 통신 프로토콜 설계

```c
// ooxml_protocol.h - OOXML 처리 프로토콜 정의
// ooxml_protocol.h - OOXML Processing Protocol Definition

#pragma once

// 메시지 타입
// Message types
typedef enum _OOXML_MESSAGE_TYPE {
    OOXML_MSG_EXTRACT_REQUEST,      // 텍스트 추출 요청
                                    // Text extraction request
    OOXML_MSG_EXTRACT_RESPONSE,     // 추출 응답
                                    // Extraction response
    OOXML_MSG_STATUS_QUERY,         // 서비스 상태 쿼리
                                    // Service status query
    OOXML_MSG_STATUS_RESPONSE       // 상태 응답
                                    // Status response
} OOXML_MESSAGE_TYPE;

// 문서 타입
// Document types
typedef enum _OOXML_DOC_TYPE {
    OOXML_DOC_UNKNOWN = 0,
    OOXML_DOC_WORD,                 // .docx
    OOXML_DOC_EXCEL,                // .xlsx
    OOXML_DOC_POWERPOINT            // .pptx
} OOXML_DOC_TYPE;

// 추출 요청 메시지 (커널 → 사용자 모드)
// Extraction request message (Kernel → User mode)
typedef struct _OOXML_EXTRACT_REQUEST {
    OOXML_MESSAGE_TYPE MessageType;     // = OOXML_MSG_EXTRACT_REQUEST
    ULONG RequestId;                    // 요청 식별자
                                        // Request identifier
    OOXML_DOC_TYPE DocType;             // 문서 타입
                                        // Document type
    ULONG FilePathLength;               // 파일 경로 길이 (바이트)
                                        // File path length (bytes)
    WCHAR FilePath[260];                // 파일 전체 경로
                                        // Full file path
} OOXML_EXTRACT_REQUEST, *POOXML_EXTRACT_REQUEST;

// 추출 응답 메시지 (사용자 모드 → 커널)
// Extraction response message (User mode → Kernel)
typedef struct _OOXML_EXTRACT_RESPONSE {
    OOXML_MESSAGE_TYPE MessageType;     // = OOXML_MSG_EXTRACT_RESPONSE
    ULONG RequestId;                    // 요청 식별자
                                        // Request identifier
    NTSTATUS Status;                    // 처리 결과
                                        // Processing result
    ULONG TextLength;                   // 추출된 텍스트 길이 (바이트)
                                        // Extracted text length (bytes)
    ULONG TextCharCount;                // 추출된 문자 수
                                        // Extracted character count
    WCHAR ExtractedText[1];             // 가변 길이 텍스트
                                        // Variable length text
} OOXML_EXTRACT_RESPONSE, *POOXML_EXTRACT_RESPONSE;

// 응답 최대 크기 (텍스트 포함)
// Maximum response size (including text)
#define MAX_EXTRACT_RESPONSE_SIZE   (1024 * 1024)  // 1MB

// 응답 텍스트 최대 문자 수
// Maximum response text character count
#define MAX_EXTRACT_TEXT_CHARS      ((MAX_EXTRACT_RESPONSE_SIZE - \
                                      sizeof(OOXML_EXTRACT_RESPONSE)) / \
                                      sizeof(WCHAR))
```

### 23.2.4 커널 측 구현

```c
// ooxml_kernel.c - 커널 측 OOXML 처리
// ooxml_kernel.c - Kernel-side OOXML processing

#include <fltKernel.h>
#include "ooxml_protocol.h"
#include "ssn_detector.h"

// 전역 변수
// Global variables
PFLT_FILTER g_FilterHandle = NULL;
PFLT_PORT g_ServerPort = NULL;
PFLT_PORT g_ClientPort = NULL;
ULONG g_RequestIdCounter = 0;

// 서비스 연결 콜백
// Service connection callback
NTSTATUS OoxmlConnectNotify(
    _In_ PFLT_PORT ClientPort,
    _In_opt_ PVOID ServerPortCookie,
    _In_reads_bytes_opt_(SizeOfContext) PVOID ConnectionContext,
    _In_ ULONG SizeOfContext,
    _Outptr_result_maybenull_ PVOID* ConnectionPortCookie)
{
    UNREFERENCED_PARAMETER(ServerPortCookie);
    UNREFERENCED_PARAMETER(ConnectionContext);
    UNREFERENCED_PARAMETER(SizeOfContext);

    DbgPrint("[OOXML] 사용자 모드 서비스 연결됨\n");
    // User mode service connected

    g_ClientPort = ClientPort;
    *ConnectionPortCookie = NULL;

    return STATUS_SUCCESS;
}

// 서비스 연결 해제 콜백
// Service disconnect callback
VOID OoxmlDisconnectNotify(_In_opt_ PVOID ConnectionCookie)
{
    UNREFERENCED_PARAMETER(ConnectionCookie);

    DbgPrint("[OOXML] 사용자 모드 서비스 연결 해제됨\n");
    // User mode service disconnected

    FltCloseClientPort(g_FilterHandle, &g_ClientPort);
    g_ClientPort = NULL;
}

// 텍스트 추출 요청
// Request text extraction
NTSTATUS RequestTextExtraction(
    _In_ PCUNICODE_STRING FilePath,
    _In_ OOXML_DOC_TYPE DocType,
    _Out_writes_bytes_(ResponseBufferSize) PVOID ResponseBuffer,
    _In_ ULONG ResponseBufferSize,
    _Out_ PULONG BytesReturned)
{
    NTSTATUS status;
    OOXML_EXTRACT_REQUEST request;
    ULONG replyLength;

    *BytesReturned = 0;

    // 서비스 연결 확인
    // Check service connection
    if (g_ClientPort == NULL) {
        DbgPrint("[OOXML] 서비스가 연결되지 않음\n");
        // Service not connected
        return STATUS_PORT_DISCONNECTED;
    }

    // 요청 구성
    // Build request
    RtlZeroMemory(&request, sizeof(request));
    request.MessageType = OOXML_MSG_EXTRACT_REQUEST;
    request.RequestId = InterlockedIncrement(&g_RequestIdCounter);
    request.DocType = DocType;

    // 파일 경로 복사
    // Copy file path
    if (FilePath->Length > sizeof(request.FilePath) - sizeof(WCHAR)) {
        return STATUS_BUFFER_TOO_SMALL;
    }

    RtlCopyMemory(request.FilePath, FilePath->Buffer, FilePath->Length);
    request.FilePathLength = FilePath->Length;

    // 메시지 전송 및 응답 대기
    // Send message and wait for response
    replyLength = ResponseBufferSize;

    status = FltSendMessage(
        g_FilterHandle,
        &g_ClientPort,
        &request,
        sizeof(request),
        ResponseBuffer,
        &replyLength,
        NULL            // 타임아웃 없음 (무한 대기)
                        // No timeout (infinite wait)
    );

    if (NT_SUCCESS(status)) {
        *BytesReturned = replyLength;
    }

    return status;
}

// OOXML 파일 처리
// Process OOXML file
NTSTATUS ProcessOoxmlFile(
    _In_ PFLT_CALLBACK_DATA Data,
    _In_ PCFLT_RELATED_OBJECTS FltObjects,
    _In_ PCUNICODE_STRING FilePath)
{
    NTSTATUS status;
    POOXML_EXTRACT_RESPONSE response = NULL;
    ULONG responseSize = MAX_EXTRACT_RESPONSE_SIZE;
    ULONG bytesReturned;
    OOXML_DOC_TYPE docType;
    SSN_SCAN_RESULT scanResult;

    UNREFERENCED_PARAMETER(Data);
    UNREFERENCED_PARAMETER(FltObjects);

    // 문서 타입 결정
    // Determine document type
    UNICODE_STRING docx = RTL_CONSTANT_STRING(L".docx");
    UNICODE_STRING xlsx = RTL_CONSTANT_STRING(L".xlsx");
    UNICODE_STRING pptx = RTL_CONSTANT_STRING(L".pptx");

    if (RtlSuffixUnicodeString(&docx, FilePath, TRUE)) {
        docType = OOXML_DOC_WORD;
    } else if (RtlSuffixUnicodeString(&xlsx, FilePath, TRUE)) {
        docType = OOXML_DOC_EXCEL;
    } else if (RtlSuffixUnicodeString(&pptx, FilePath, TRUE)) {
        docType = OOXML_DOC_POWERPOINT;
    } else {
        return STATUS_NOT_SUPPORTED;
    }

    // 응답 버퍼 할당
    // Allocate response buffer
    response = ExAllocatePool2(
        POOL_FLAG_PAGED,
        responseSize,
        'pxoO'
    );

    if (response == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    __try {
        // 텍스트 추출 요청
        // Request text extraction
        status = RequestTextExtraction(
            FilePath,
            docType,
            response,
            responseSize,
            &bytesReturned
        );

        if (!NT_SUCCESS(status)) {
            DbgPrint("[OOXML] 텍스트 추출 요청 실패: 0x%X\n", status);
            // Text extraction request failed
            __leave;
        }

        // 응답 검증
        // Validate response
        if (bytesReturned < sizeof(OOXML_EXTRACT_RESPONSE)) {
            status = STATUS_INVALID_BUFFER_SIZE;
            __leave;
        }

        if (!NT_SUCCESS(response->Status)) {
            DbgPrint("[OOXML] 서비스 처리 실패: 0x%X\n", response->Status);
            // Service processing failed
            status = response->Status;
            __leave;
        }

        // 추출된 텍스트로 SSN 스캔
        // Scan extracted text for SSN
        if (response->TextCharCount > 0) {
            status = SsnScanUnicodeBuffer(
                &g_SsnDetector,
                response->ExtractedText,
                response->TextCharCount,
                &scanResult
            );

            if (NT_SUCCESS(status) && scanResult.ValidatedCount > 0) {
                DbgPrint("[OOXML] SSN 감지! 파일: %wZ, 개수: %lu\n",
                         FilePath, scanResult.ValidatedCount);
                // SSN detected! File, Count

                // 여기서 차단 또는 로깅 로직 수행
                // Perform blocking or logging logic here
            }
        }

        status = STATUS_SUCCESS;
    }
    __finally {
        if (response != NULL) {
            ExFreePoolWithTag(response, 'pxoO');
        }
    }

    return status;
}

// 통신 포트 초기화
// Initialize communication port
NTSTATUS InitializeOoxmlCommunication(_In_ PFLT_FILTER Filter)
{
    NTSTATUS status;
    PSECURITY_DESCRIPTOR sd;
    OBJECT_ATTRIBUTES oa;
    UNICODE_STRING portName = RTL_CONSTANT_STRING(L"\\OoxmlExtractorPort");

    g_FilterHandle = Filter;

    // 보안 설명자 생성
    // Create security descriptor
    status = FltBuildDefaultSecurityDescriptor(&sd, FLT_PORT_ALL_ACCESS);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    InitializeObjectAttributes(
        &oa,
        &portName,
        OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
        NULL,
        sd
    );

    // 통신 포트 생성
    // Create communication port
    status = FltCreateCommunicationPort(
        Filter,
        &g_ServerPort,
        &oa,
        NULL,
        OoxmlConnectNotify,
        OoxmlDisconnectNotify,
        NULL,               // MessageNotify (동기 통신이므로 불필요)
                            // MessageNotify (not needed for sync)
        1                   // 최대 연결 수
                            // Max connections
    );

    FltFreeSecurityDescriptor(sd);

    if (NT_SUCCESS(status)) {
        DbgPrint("[OOXML] 통신 포트 생성됨: %wZ\n", &portName);
        // Communication port created
    }

    return status;
}

// 통신 포트 정리
// Cleanup communication port
VOID CleanupOoxmlCommunication(VOID)
{
    if (g_ServerPort != NULL) {
        FltCloseCommunicationPort(g_ServerPort);
        g_ServerPort = NULL;
    }

    DbgPrint("[OOXML] 통신 포트 정리됨\n");
    // Communication port cleaned up
}
```

---

## 23.3 텍스트 추출 구현

### 23.3.1 사용자 모드 서비스 (.NET)

```csharp
// OoxmlExtractorService.cs - OOXML 텍스트 추출 서비스
// OoxmlExtractorService.cs - OOXML Text Extraction Service

using System;
using System.IO;
using System.IO.Compression;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading;
using System.Xml.Linq;
using DocumentFormat.OpenXml.Packaging;

namespace OoxmlExtractorService
{
    /// <summary>
    /// OOXML 메시지 타입
    /// OOXML Message Types
    /// </summary>
    public enum OoxmlMessageType : uint
    {
        ExtractRequest = 0,
        ExtractResponse = 1,
        StatusQuery = 2,
        StatusResponse = 3
    }

    /// <summary>
    /// OOXML 문서 타입
    /// OOXML Document Types
    /// </summary>
    public enum OoxmlDocType : uint
    {
        Unknown = 0,
        Word = 1,
        Excel = 2,
        PowerPoint = 3
    }

    /// <summary>
    /// 추출 요청 구조체 (커널에서 수신)
    /// Extraction Request Structure (received from kernel)
    /// </summary>
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    public struct ExtractRequest
    {
        public OoxmlMessageType MessageType;
        public uint RequestId;
        public OoxmlDocType DocType;
        public uint FilePathLength;
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 260)]
        public string FilePath;
    }

    /// <summary>
    /// 텍스트 추출기 클래스
    /// Text Extractor Class
    /// </summary>
    public sealed class TextExtractor : IDisposable
    {
        private const int MaxTextLength = 512 * 1024;  // 최대 512KB 텍스트
                                                        // Maximum 512KB text

        /// <summary>
        /// 파일에서 텍스트 추출
        /// Extract text from file
        /// </summary>
        public string ExtractText(string filePath, OoxmlDocType docType)
        {
            if (!File.Exists(filePath))
            {
                throw new FileNotFoundException(
                    "파일을 찾을 수 없습니다.",
                    // "File not found."
                    filePath);
            }

            return docType switch
            {
                OoxmlDocType.Word => ExtractFromDocx(filePath),
                OoxmlDocType.Excel => ExtractFromXlsx(filePath),
                OoxmlDocType.PowerPoint => ExtractFromPptx(filePath),
                _ => throw new NotSupportedException(
                    $"지원하지 않는 문서 타입: {docType}")
                    // $"Unsupported document type: {docType}"
            };
        }

        /// <summary>
        /// Word 문서에서 텍스트 추출
        /// Extract text from Word document
        /// </summary>
        private string ExtractFromDocx(string filePath)
        {
            var sb = new StringBuilder();

            using var doc = WordprocessingDocument.Open(filePath, false);

            var body = doc.MainDocumentPart?.Document?.Body;
            if (body is null)
            {
                return string.Empty;
            }

            // 모든 텍스트 요소 수집
            // Collect all text elements
            foreach (var text in body.Descendants<DocumentFormat.OpenXml.Wordprocessing.Text>())
            {
                sb.Append(text.Text);
                sb.Append(' ');

                if (sb.Length > MaxTextLength)
                {
                    break;
                }
            }

            return sb.ToString();
        }

        /// <summary>
        /// Excel 문서에서 텍스트 추출
        /// Extract text from Excel document
        /// </summary>
        private string ExtractFromXlsx(string filePath)
        {
            var sb = new StringBuilder();

            using var doc = SpreadsheetDocument.Open(filePath, false);

            var workbookPart = doc.WorkbookPart;
            if (workbookPart is null)
            {
                return string.Empty;
            }

            // SharedStringTable에서 문자열 가져오기
            // Get strings from SharedStringTable
            var sharedStrings = workbookPart.SharedStringTablePart?
                .SharedStringTable?
                .Elements<DocumentFormat.OpenXml.Spreadsheet.SharedStringItem>()
                .ToList();

            // 각 워크시트 처리
            // Process each worksheet
            foreach (var worksheetPart in workbookPart.WorksheetParts)
            {
                var sheetData = worksheetPart.Worksheet?
                    .GetFirstChild<DocumentFormat.OpenXml.Spreadsheet.SheetData>();

                if (sheetData is null) continue;

                foreach (var row in sheetData.Elements<DocumentFormat.OpenXml.Spreadsheet.Row>())
                {
                    foreach (var cell in row.Elements<DocumentFormat.OpenXml.Spreadsheet.Cell>())
                    {
                        var cellValue = GetCellValue(cell, sharedStrings);
                        if (!string.IsNullOrEmpty(cellValue))
                        {
                            sb.Append(cellValue);
                            sb.Append(' ');
                        }

                        if (sb.Length > MaxTextLength)
                        {
                            break;
                        }
                    }
                }
            }

            return sb.ToString();
        }

        /// <summary>
        /// 셀 값 가져오기
        /// Get cell value
        /// </summary>
        private static string GetCellValue(
            DocumentFormat.OpenXml.Spreadsheet.Cell cell,
            List<DocumentFormat.OpenXml.Spreadsheet.SharedStringItem>? sharedStrings)
        {
            var value = cell.CellValue?.Text;
            if (value is null)
            {
                return string.Empty;
            }

            // SharedString 참조인 경우
            // If it's a SharedString reference
            if (cell.DataType?.Value ==
                DocumentFormat.OpenXml.Spreadsheet.CellValues.SharedString)
            {
                if (sharedStrings is not null &&
                    int.TryParse(value, out var index) &&
                    index >= 0 && index < sharedStrings.Count)
                {
                    return sharedStrings[index].InnerText;
                }
            }

            return value;
        }

        /// <summary>
        /// PowerPoint 문서에서 텍스트 추출
        /// Extract text from PowerPoint document
        /// </summary>
        private string ExtractFromPptx(string filePath)
        {
            var sb = new StringBuilder();

            using var doc = PresentationDocument.Open(filePath, false);

            var presentationPart = doc.PresentationPart;
            if (presentationPart is null)
            {
                return string.Empty;
            }

            // 각 슬라이드 처리
            // Process each slide
            foreach (var slidePart in presentationPart.SlideParts)
            {
                // 모든 텍스트 요소 수집
                // Collect all text elements
                foreach (var text in slidePart.Slide.Descendants<DocumentFormat.OpenXml.Drawing.Text>())
                {
                    sb.Append(text.Text);
                    sb.Append(' ');

                    if (sb.Length > MaxTextLength)
                    {
                        break;
                    }
                }
            }

            return sb.ToString();
        }

        public void Dispose()
        {
            // 현재 정리할 리소스 없음
            // No resources to clean up currently
        }
    }

    /// <summary>
    /// 필터 드라이버 통신 클래스
    /// Filter Driver Communication Class
    /// </summary>
    public sealed class FilterCommunication : IDisposable
    {
        private readonly IntPtr _portHandle;
        private readonly TextExtractor _extractor;
        private bool _disposed;
        private CancellationTokenSource? _cts;
        private Thread? _messageThread;

        // P/Invoke 선언
        // P/Invoke declarations
        [DllImport("fltlib.dll", CharSet = CharSet.Unicode)]
        private static extern uint FilterConnectCommunicationPort(
            string portName,
            uint options,
            IntPtr context,
            uint contextSize,
            IntPtr securityAttributes,
            out IntPtr portHandle);

        [DllImport("fltlib.dll")]
        private static extern uint FilterGetMessage(
            IntPtr portHandle,
            IntPtr messageBuffer,
            uint messageBufferSize,
            IntPtr overlapped);

        [DllImport("fltlib.dll")]
        private static extern uint FilterReplyMessage(
            IntPtr portHandle,
            IntPtr replyBuffer,
            uint replyBufferSize);

        [DllImport("kernel32.dll")]
        private static extern bool CloseHandle(IntPtr handle);

        public FilterCommunication(string portName)
        {
            _extractor = new TextExtractor();

            var result = FilterConnectCommunicationPort(
                portName,
                0,
                IntPtr.Zero,
                0,
                IntPtr.Zero,
                out _portHandle);

            if (result != 0)
            {
                throw new InvalidOperationException(
                    $"필터 포트 연결 실패: 0x{result:X8}");
                    // $"Failed to connect filter port: 0x{result:X8}"
            }

            Console.WriteLine($"필터 드라이버 연결됨: {portName}");
            // $"Connected to filter driver: {portName}"
        }

        /// <summary>
        /// 메시지 처리 시작
        /// Start message processing
        /// </summary>
        public void StartMessageLoop()
        {
            _cts = new CancellationTokenSource();
            _messageThread = new Thread(MessageLoop)
            {
                IsBackground = true,
                Name = "FilterMessageThread"
            };
            _messageThread.Start();
        }

        /// <summary>
        /// 메시지 처리 중지
        /// Stop message processing
        /// </summary>
        public void StopMessageLoop()
        {
            _cts?.Cancel();
            _messageThread?.Join(TimeSpan.FromSeconds(5));
        }

        /// <summary>
        /// 메시지 루프
        /// Message loop
        /// </summary>
        private void MessageLoop()
        {
            const int bufferSize = 4096;
            var messageBuffer = Marshal.AllocHGlobal(bufferSize);

            try
            {
                while (_cts is not null && !_cts.Token.IsCancellationRequested)
                {
                    // 메시지 수신
                    // Receive message
                    var result = FilterGetMessage(
                        _portHandle,
                        messageBuffer,
                        (uint)bufferSize,
                        IntPtr.Zero);

                    if (result != 0)
                    {
                        if (result == 0x800704C7)  // ERROR_OPERATION_ABORTED
                        {
                            break;
                        }
                        Console.WriteLine($"메시지 수신 오류: 0x{result:X8}");
                        // $"Message receive error: 0x{result:X8}"
                        Thread.Sleep(100);
                        continue;
                    }

                    // 메시지 처리
                    // Process message
                    ProcessMessage(messageBuffer);
                }
            }
            finally
            {
                Marshal.FreeHGlobal(messageBuffer);
            }
        }

        /// <summary>
        /// 메시지 처리
        /// Process message
        /// </summary>
        private void ProcessMessage(IntPtr messageBuffer)
        {
            // FILTER_MESSAGE_HEADER (16 bytes) 스킵 후 요청 읽기
            // Skip FILTER_MESSAGE_HEADER (16 bytes) then read request
            var requestPtr = IntPtr.Add(messageBuffer, 16);
            var request = Marshal.PtrToStructure<ExtractRequest>(requestPtr);

            if (request.MessageType != OoxmlMessageType.ExtractRequest)
            {
                return;
            }

            Console.WriteLine($"추출 요청 수신: {request.FilePath}");
            // $"Extract request received: {request.FilePath}"

            // 텍스트 추출
            // Extract text
            string extractedText;
            uint status;

            try
            {
                extractedText = _extractor.ExtractText(
                    request.FilePath,
                    request.DocType);
                status = 0;  // STATUS_SUCCESS
            }
            catch (Exception ex)
            {
                Console.WriteLine($"추출 오류: {ex.Message}");
                // $"Extraction error: {ex.Message}"
                extractedText = string.Empty;
                status = 0xC0000001;  // STATUS_UNSUCCESSFUL
            }

            // 응답 전송
            // Send response
            SendResponse(request.RequestId, status, extractedText);
        }

        /// <summary>
        /// 응답 전송
        /// Send response
        /// </summary>
        private void SendResponse(uint requestId, uint status, string text)
        {
            // 응답 크기 계산
            // Calculate response size
            var textBytes = Encoding.Unicode.GetBytes(text);
            var responseSize = 24 + textBytes.Length + 2;  // 헤더 + 텍스트 + null
                                                           // header + text + null

            var responseBuffer = Marshal.AllocHGlobal(responseSize);

            try
            {
                // FILTER_REPLY_HEADER (16 bytes): Status(4) + MessageId(8) + Reserved(4)
                Marshal.WriteInt32(responseBuffer, 0, (int)status);
                Marshal.WriteInt64(responseBuffer, 4, requestId);

                // 응답 데이터 (16바이트 오프셋부터)
                // Response data (from 16-byte offset)
                var dataOffset = 16;
                Marshal.WriteInt32(responseBuffer, dataOffset, (int)OoxmlMessageType.ExtractResponse);
                Marshal.WriteInt32(responseBuffer, dataOffset + 4, (int)requestId);
                Marshal.WriteInt32(responseBuffer, dataOffset + 8, (int)status);
                Marshal.WriteInt32(responseBuffer, dataOffset + 12, textBytes.Length);
                Marshal.WriteInt32(responseBuffer, dataOffset + 16, text.Length);

                // 텍스트 복사
                // Copy text
                if (textBytes.Length > 0)
                {
                    Marshal.Copy(textBytes, 0, IntPtr.Add(responseBuffer, dataOffset + 20), textBytes.Length);
                }

                var result = FilterReplyMessage(
                    _portHandle,
                    responseBuffer,
                    (uint)responseSize);

                if (result != 0)
                {
                    Console.WriteLine($"응답 전송 오류: 0x{result:X8}");
                    // $"Response send error: 0x{result:X8}"
                }
            }
            finally
            {
                Marshal.FreeHGlobal(responseBuffer);
            }
        }

        public void Dispose()
        {
            if (_disposed) return;
            _disposed = true;

            StopMessageLoop();

            if (_portHandle != IntPtr.Zero)
            {
                CloseHandle(_portHandle);
            }

            _extractor.Dispose();
            _cts?.Dispose();
        }
    }

    /// <summary>
    /// 서비스 진입점
    /// Service entry point
    /// </summary>
    internal static class Program
    {
        private static void Main(string[] args)
        {
            const string portName = "\\OoxmlExtractorPort";

            Console.WriteLine("OOXML 추출 서비스 시작...");
            // "OOXML Extraction Service starting..."

            using var comm = new FilterCommunication(portName);

            comm.StartMessageLoop();

            Console.WriteLine("서비스 실행 중. 종료하려면 Enter를 누르세요.");
            // "Service running. Press Enter to exit."
            Console.ReadLine();

            Console.WriteLine("서비스 종료 중...");
            // "Service stopping..."
        }
    }
}
```

### 23.3.2 프로젝트 파일 구성

```xml
<!-- OoxmlExtractorService.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  </PropertyGroup>

  <ItemGroup>
    <!-- Open XML SDK -->
    <PackageReference Include="DocumentFormat.OpenXml" Version="3.0.2" />
  </ItemGroup>

</Project>
```

### 23.3.3 간단한 ZIP 파서 (커널용, 선택적)

특수한 경우 커널에서 직접 ZIP을 파싱해야 할 때 사용할 수 있는 최소한의 구현입니다.

```c
// zip_mini_parser.c - 최소 ZIP 파서 (비압축 파일만 지원)
// zip_mini_parser.c - Minimal ZIP parser (uncompressed files only)

#include <ntifs.h>

// ZIP 파일 목록 조회 (비압축 파일만)
// List ZIP files (uncompressed only)
typedef struct _ZIP_FILE_ENTRY {
    ULONG Offset;               // 파일 데이터 오프셋
                                // File data offset
    ULONG CompressedSize;       // 압축 크기
                                // Compressed size
    ULONG UncompressedSize;     // 원본 크기
                                // Uncompressed size
    USHORT CompressionMethod;   // 압축 방식
                                // Compression method
    CHAR FileName[256];         // 파일명
                                // File name
} ZIP_FILE_ENTRY, *PZIP_FILE_ENTRY;

// ZIP 아카이브에서 파일 찾기
// Find file in ZIP archive
NTSTATUS ZipFindFile(
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _In_ PCSTR FileName,
    _Out_ PZIP_FILE_ENTRY Entry)
{
    ULONG offset = 0;
    ULONG fileNameLen = (ULONG)strlen(FileName);

    while (offset + sizeof(ZIP_LOCAL_FILE_HEADER) <= BufferLength) {
        PZIP_LOCAL_FILE_HEADER header = (PZIP_LOCAL_FILE_HEADER)(Buffer + offset);

        // 시그니처 확인
        // Check signature
        if (header->Signature != ZIP_LOCAL_FILE_SIGNATURE) {
            break;  // 파일 목록 끝
                    // End of file list
        }

        // 파일명 비교
        // Compare file name
        PCHAR currentFileName = (PCHAR)(Buffer + offset + sizeof(ZIP_LOCAL_FILE_HEADER));

        if (header->FileNameLength == fileNameLen &&
            RtlCompareMemory(currentFileName, FileName, fileNameLen) == fileNameLen) {
            // 파일 찾음!
            // File found!
            Entry->Offset = offset + sizeof(ZIP_LOCAL_FILE_HEADER) +
                           header->FileNameLength + header->ExtraFieldLength;
            Entry->CompressedSize = header->CompressedSize;
            Entry->UncompressedSize = header->UncompressedSize;
            Entry->CompressionMethod = header->CompressionMethod;

            RtlCopyMemory(Entry->FileName, currentFileName,
                          min(header->FileNameLength, sizeof(Entry->FileName) - 1));
            Entry->FileName[min(header->FileNameLength, sizeof(Entry->FileName) - 1)] = '\0';

            return STATUS_SUCCESS;
        }

        // 다음 파일로 이동
        // Move to next file
        offset += sizeof(ZIP_LOCAL_FILE_HEADER) +
                  header->FileNameLength +
                  header->ExtraFieldLength +
                  header->CompressedSize;
    }

    return STATUS_NOT_FOUND;
}

// 비압축 파일 데이터 추출
// Extract uncompressed file data
NTSTATUS ZipExtractStoredFile(
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _In_ PZIP_FILE_ENTRY Entry,
    _Out_writes_bytes_(OutputBufferSize) PUCHAR OutputBuffer,
    _In_ ULONG OutputBufferSize,
    _Out_ PULONG BytesExtracted)
{
    *BytesExtracted = 0;

    // 비압축(Stored) 파일만 지원
    // Only support uncompressed (Stored) files
    if (Entry->CompressionMethod != 0) {
        DbgPrint("[ZIP] Deflate 압축은 지원하지 않음 (방식: %d)\n",
                 Entry->CompressionMethod);
        // Deflate compression not supported (method: %d)
        return STATUS_NOT_SUPPORTED;
    }

    // 범위 검증
    // Validate range
    if (Entry->Offset + Entry->UncompressedSize > BufferLength) {
        return STATUS_BUFFER_OVERFLOW;
    }

    if (Entry->UncompressedSize > OutputBufferSize) {
        return STATUS_BUFFER_TOO_SMALL;
    }

    // 데이터 복사
    // Copy data
    RtlCopyMemory(OutputBuffer, Buffer + Entry->Offset, Entry->UncompressedSize);
    *BytesExtracted = Entry->UncompressedSize;

    return STATUS_SUCCESS;
}

// DOCX에서 document.xml 찾아 텍스트 추출 (간단 버전)
// Find document.xml in DOCX and extract text (simple version)
NTSTATUS ExtractDocxTextSimple(
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength,
    _Out_writes_bytes_(TextBufferSize) PWCHAR TextBuffer,
    _In_ ULONG TextBufferSize,
    _Out_ PULONG TextLength)
{
    NTSTATUS status;
    ZIP_FILE_ENTRY entry;
    PUCHAR xmlBuffer = NULL;
    ULONG bytesExtracted;

    *TextLength = 0;

    // document.xml 찾기
    // Find document.xml
    status = ZipFindFile(Buffer, BufferLength, "word/document.xml", &entry);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    // 비압축 파일만 처리
    // Only handle uncompressed files
    if (entry.CompressionMethod != 0) {
        DbgPrint("[DOCX] document.xml이 압축됨 - 사용자 모드 서비스 필요\n");
        // document.xml is compressed - user mode service required
        return STATUS_NOT_SUPPORTED;
    }

    // XML 버퍼 할당
    // Allocate XML buffer
    xmlBuffer = ExAllocatePool2(
        POOL_FLAG_PAGED,
        entry.UncompressedSize + 1,
        'lmxD'
    );

    if (xmlBuffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    __try {
        // XML 추출
        // Extract XML
        status = ZipExtractStoredFile(
            Buffer,
            BufferLength,
            &entry,
            xmlBuffer,
            entry.UncompressedSize,
            &bytesExtracted
        );

        if (!NT_SUCCESS(status)) {
            __leave;
        }

        xmlBuffer[bytesExtracted] = '\0';

        // 간단한 <w:t> 태그 추출
        // Simple <w:t> tag extraction
        // 주의: 실제 프로덕션에서는 적절한 XML 파서 사용 필요
        // Note: In production, use proper XML parser
        PCHAR p = (PCHAR)xmlBuffer;
        ULONG textIndex = 0;
        ULONG maxChars = (TextBufferSize / sizeof(WCHAR)) - 1;

        while (*p && textIndex < maxChars) {
            // <w:t> 또는 <w:t ...> 찾기
            // Find <w:t> or <w:t ...>
            PCHAR tagStart = strstr(p, "<w:t");
            if (tagStart == NULL) {
                break;
            }

            // > 찾기
            // Find >
            PCHAR contentStart = strchr(tagStart, '>');
            if (contentStart == NULL) {
                break;
            }
            contentStart++;  // > 다음 문자
                             // Character after >

            // </w:t> 찾기
            // Find </w:t>
            PCHAR contentEnd = strstr(contentStart, "</w:t>");
            if (contentEnd == NULL) {
                break;
            }

            // 텍스트 복사 (ASCII -> Unicode)
            // Copy text (ASCII -> Unicode)
            while (contentStart < contentEnd && textIndex < maxChars) {
                TextBuffer[textIndex++] = (WCHAR)*contentStart++;
            }

            // 공백 추가
            // Add space
            if (textIndex < maxChars) {
                TextBuffer[textIndex++] = L' ';
            }

            p = contentEnd + 6;  // </w:t> 다음
                                 // After </w:t>
        }

        TextBuffer[textIndex] = L'\0';
        *TextLength = textIndex;

        status = STATUS_SUCCESS;
    }
    __finally {
        if (xmlBuffer != NULL) {
            ExFreePoolWithTag(xmlBuffer, 'lmxD');
        }
    }

    return status;
}
```

---

## 23.4 성능 및 보안 고려사항

### 23.4.1 성능 최적화

```
┌─────────────────────────────────────────────────────────────────┐
│                    OOXML 처리 성능 최적화                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 확장자 기반 조기 필터링                                      │
│     Early filtering based on extension                          │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ if (!IsOoxmlExtension(fileName)) {                  │    │
│     │     return FLT_PREOP_SUCCESS_NO_CALLBACK;           │    │
│     │ }                                                   │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  2. 파일 크기 제한                                               │
│     File size limits                                            │
│     - 너무 작은 파일: 유효한 OOXML 아님                           │
│       Too small: not valid OOXML                                │
│     - 너무 큰 파일: 성능 영향, 스킵 또는 비동기 처리               │
│       Too large: performance impact, skip or async              │
│                                                                 │
│  3. 캐싱                                                        │
│     Caching                                                     │
│     - 동일 파일 반복 스캔 방지                                    │
│       Prevent repeated scans of same file                       │
│     - 파일 해시 기반 캐시                                        │
│       Hash-based file cache                                     │
│                                                                 │
│  4. 비동기 처리                                                  │
│     Async processing                                            │
│     - 시스템 워커 스레드 활용                                     │
│       Use system worker threads                                 │
│     - 블로킹 최소화                                              │
│       Minimize blocking                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 23.4.2 파일 크기 필터링

```c
// 파일 크기 기반 필터링
// File size-based filtering
typedef struct _OOXML_SIZE_CONFIG {
    ULONG MinValidSize;         // 최소 유효 크기 (바이트)
                                // Minimum valid size (bytes)
    ULONG MaxSyncScanSize;      // 동기 스캔 최대 크기
                                // Maximum size for sync scan
    ULONG MaxAsyncScanSize;     // 비동기 스캔 최대 크기
                                // Maximum size for async scan
} OOXML_SIZE_CONFIG;

static OOXML_SIZE_CONFIG g_SizeConfig = {
    .MinValidSize = 1024,               // 1KB 미만은 유효한 OOXML 아님
                                        // Less than 1KB not valid OOXML
    .MaxSyncScanSize = 5 * 1024 * 1024, // 5MB까지 동기 스캔
                                        // Up to 5MB for sync scan
    .MaxAsyncScanSize = 50 * 1024 * 1024 // 50MB까지 비동기 스캔
                                         // Up to 50MB for async scan
};

typedef enum _OOXML_SCAN_DECISION {
    OOXML_SCAN_SKIP,        // 스캔 안 함
                            // Don't scan
    OOXML_SCAN_SYNC,        // 동기 스캔
                            // Synchronous scan
    OOXML_SCAN_ASYNC        // 비동기 스캔
                            // Asynchronous scan
} OOXML_SCAN_DECISION;

OOXML_SCAN_DECISION DecideScanMethod(ULONGLONG FileSize)
{
    if (FileSize < g_SizeConfig.MinValidSize) {
        return OOXML_SCAN_SKIP;
    }

    if (FileSize <= g_SizeConfig.MaxSyncScanSize) {
        return OOXML_SCAN_SYNC;
    }

    if (FileSize <= g_SizeConfig.MaxAsyncScanSize) {
        return OOXML_SCAN_ASYNC;
    }

    // 너무 큰 파일은 스킵
    // Skip files that are too large
    DbgPrint("[OOXML] 파일 크기 초과, 스캔 스킵: %llu bytes\n", FileSize);
    // File size exceeded, skipping scan
    return OOXML_SCAN_SKIP;
}
```

### 23.4.3 보안 고려사항

```c
// 보안 검증 함수들
// Security validation functions

// ZIP 폭탄 감지 (압축률 검사)
// ZIP bomb detection (compression ratio check)
BOOLEAN IsZipBomb(PZIP_LOCAL_FILE_HEADER Header)
{
    // 압축률이 100:1 이상이면 의심
    // Suspicious if compression ratio exceeds 100:1
    const ULONG MAX_COMPRESSION_RATIO = 100;

    if (Header->CompressedSize == 0) {
        return FALSE;  // Stored 파일
                       // Stored file
    }

    ULONG ratio = Header->UncompressedSize / Header->CompressedSize;

    if (ratio > MAX_COMPRESSION_RATIO) {
        DbgPrint("[보안] ZIP 폭탄 의심: 압축률 %lu:1\n", ratio);
        // [Security] Suspected ZIP bomb: compression ratio
        return TRUE;
    }

    return FALSE;
}

// 경로 순회 공격 감지 (../ 등)
// Path traversal attack detection (../ etc.)
BOOLEAN HasPathTraversal(PCSTR FileName)
{
    // ".." 시퀀스 검사
    // Check for ".." sequence
    if (strstr(FileName, "..") != NULL) {
        DbgPrint("[보안] 경로 순회 시도 감지: %s\n", FileName);
        // [Security] Path traversal attempt detected
        return TRUE;
    }

    // 절대 경로 검사 (Windows와 Unix 스타일)
    // Check absolute paths (Windows and Unix style)
    if (FileName[0] == '/' || FileName[0] == '\\' ||
        (strlen(FileName) > 2 && FileName[1] == ':')) {
        DbgPrint("[보안] 절대 경로 감지: %s\n", FileName);
        // [Security] Absolute path detected
        return TRUE;
    }

    return FALSE;
}

// 악성 XML 엔티티 검사 (XXE 방지)
// Malicious XML entity check (XXE prevention)
BOOLEAN HasXxePattern(PUCHAR XmlBuffer, ULONG Length)
{
    // DOCTYPE 선언에서 ENTITY 검사
    // Check for ENTITY in DOCTYPE declaration
    static const CHAR* dangerousPatterns[] = {
        "<!ENTITY",
        "<!DOCTYPE",
        "SYSTEM",
        "PUBLIC",
        NULL
    };

    for (int i = 0; dangerousPatterns[i] != NULL; i++) {
        if (FindBytePattern(
            XmlBuffer, Length,
            (PUCHAR)dangerousPatterns[i],
            (ULONG)strlen(dangerousPatterns[i]),
            NULL)) {
            DbgPrint("[보안] XXE 패턴 감지: %s\n", dangerousPatterns[i]);
            // [Security] XXE pattern detected
            return TRUE;
        }
    }

    return FALSE;
}

// 안전한 OOXML 처리
// Safe OOXML processing
NTSTATUS SafeProcessOoxml(
    _In_reads_bytes_(BufferLength) PUCHAR Buffer,
    _In_ ULONG BufferLength)
{
    NTSTATUS status = STATUS_SUCCESS;
    ULONG offset = 0;
    ULONG fileCount = 0;
    const ULONG MAX_FILES = 1000;  // 최대 파일 수 제한
                                   // Maximum file count limit

    while (offset + sizeof(ZIP_LOCAL_FILE_HEADER) <= BufferLength) {
        PZIP_LOCAL_FILE_HEADER header = (PZIP_LOCAL_FILE_HEADER)(Buffer + offset);

        if (header->Signature != ZIP_LOCAL_FILE_SIGNATURE) {
            break;
        }

        // 파일 수 제한
        // File count limit
        if (++fileCount > MAX_FILES) {
            DbgPrint("[보안] 파일 수 초과: %lu\n", fileCount);
            // [Security] File count exceeded
            return STATUS_QUOTA_EXCEEDED;
        }

        // ZIP 폭탄 검사
        // ZIP bomb check
        if (IsZipBomb(header)) {
            return STATUS_MALICIOUS_FILE;
        }

        // 파일명 검사
        // File name check
        PCHAR fileName = (PCHAR)(Buffer + offset + sizeof(ZIP_LOCAL_FILE_HEADER));
        CHAR fileNameBuffer[256];

        ULONG nameLen = min(header->FileNameLength, sizeof(fileNameBuffer) - 1);
        RtlCopyMemory(fileNameBuffer, fileName, nameLen);
        fileNameBuffer[nameLen] = '\0';

        if (HasPathTraversal(fileNameBuffer)) {
            return STATUS_MALICIOUS_FILE;
        }

        offset += sizeof(ZIP_LOCAL_FILE_HEADER) +
                  header->FileNameLength +
                  header->ExtraFieldLength +
                  header->CompressedSize;
    }

    return status;
}
```

---

## 23.5 Excel과 PowerPoint 처리

### 23.5.1 Excel (xlsx) 구조

```
spreadsheet.xlsx
├── [Content_Types].xml
├── _rels/
│   └── .rels
├── xl/
│   ├── workbook.xml           ← 워크북 정의
│   │                            Workbook definition
│   ├── sharedStrings.xml      ← ★ 공유 문자열 (핵심!)
│   │                            ★ Shared strings (key!)
│   ├── styles.xml
│   ├── worksheets/
│   │   ├── sheet1.xml         ← 시트 데이터
│   │   │                        Sheet data
│   │   └── sheet2.xml
│   └── _rels/
│       └── workbook.xml.rels
└── docProps/
    ├── app.xml
    └── core.xml
```

**sharedStrings.xml 예시:**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main"
     count="5" uniqueCount="4">
  <si><t>이름</t></si>
  <si><t>주민등록번호</t></si>
  <si><t>홍길동</t></si>
  <si><t>880101-1234567</t></si>
</sst>
```

### 23.5.2 PowerPoint (pptx) 구조

```
presentation.pptx
├── [Content_Types].xml
├── _rels/
│   └── .rels
├── ppt/
│   ├── presentation.xml        ← 프레젠테이션 정의
│   │                            Presentation definition
│   ├── slides/
│   │   ├── slide1.xml          ← ★ 슬라이드 내용 (핵심!)
│   │   │                         ★ Slide content (key!)
│   │   └── slide2.xml
│   ├── slideMasters/
│   │   └── slideMaster1.xml
│   ├── slideLayouts/
│   │   └── slideLayout1.xml
│   └── _rels/
│       └── presentation.xml.rels
└── docProps/
    ├── app.xml
    └── core.xml
```

**slide1.xml 예시:**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<p:sld xmlns:p="http://schemas.openxmlformats.org/presentationml/2006/main"
       xmlns:a="http://schemas.openxmlformats.org/drawingml/2006/main">
  <p:cSld>
    <p:spTree>
      <p:sp>
        <p:txBody>
          <a:p>
            <a:r>
              <a:t>제목 슬라이드</a:t>
            </a:r>
          </a:p>
          <a:p>
            <a:r>
              <a:t>고객 주민번호: 880101-1234567</a:t>
            </a:r>
          </a:p>
        </p:txBody>
      </p:sp>
    </p:spTree>
  </p:cSld>
</p:sld>
```

---

## 23.6 요약

### 핵심 내용 정리

| 항목 | 내용 |
|------|------|
| **OOXML 형식** | ZIP 아카이브 + XML 문서의 조합 |
| **핵심 파일** | Word: document.xml, Excel: sharedStrings.xml, PPT: slide*.xml |
| **권장 전략** | 하이브리드 접근법 (커널 필터링 + 사용자 모드 파싱) |
| **통신** | FltSendMessage/FilterReplyMessage 기반 동기 통신 |
| **보안** | ZIP 폭탄, 경로 순회, XXE 공격 방지 필수 |

### .NET 개발자 참고 사항

```
┌─────────────────────────────────────────────────────────────────┐
│                C# vs C OOXML 처리 비교                          │
├─────────────────────────────────────────────────────────────────┤
│ C# (사용자 모드)                │ C (커널 모드)                   │
├─────────────────────────────────┼─────────────────────────────────┤
│ System.IO.Compression           │ 직접 ZIP 헤더 파싱             │
│ DocumentFormat.OpenXml SDK      │ 문자열 패턴 매칭               │
│ System.Xml.Linq                 │ 기본 태그 추출                  │
│ 풍부한 라이브러리               │ 최소한의 구현만 가능            │
│ GC 자동 메모리 관리             │ 수동 ExAllocate/ExFree          │
│ 예외 처리                       │ NTSTATUS + __try/__except       │
└─────────────────────────────────┴─────────────────────────────────┘

★ 권장: 커널에서는 파일 식별만, 실제 파싱은 .NET 서비스에서 수행
★ Recommended: Kernel handles file identification only,
                actual parsing done in .NET service
```

### 다음 챕터 예고

Chapter 24에서는 **보호 정책 설계**를 다룹니다. 지금까지 구현한 파일 감지와 패턴 매칭을 기반으로, DRM 보호 정책을 어떻게 설계하고 구현할지 학습합니다.

---

## 참고 자료

- [ECMA-376: Office Open XML File Formats](https://www.ecma-international.org/publications-and-standards/standards/ecma-376/)
- [Open XML SDK Documentation](https://docs.microsoft.com/en-us/office/open-xml/open-xml-sdk)
- [ZIP File Format Specification](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)
- [Microsoft Filter Communications](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/communication-between-user-mode-and-kernel-mode)
