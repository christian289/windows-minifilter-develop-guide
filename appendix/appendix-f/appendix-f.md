# 부록 F: 추가 학습 자료

## Windows 드라이버 개발 심화 학습 가이드

이 부록은 Minifilter 드라이버 개발 역량을 더욱 발전시키기 위한 추가 학습 자료를 제공합니다. 공식 문서, 도서, 온라인 자료, 샘플 코드 등을 체계적으로 정리했습니다.

---

## F.1 공식 Microsoft 문서

### Windows Driver Kit (WDK) 문서

| 자료 | 설명 | URL |
|------|------|-----|
| WDK 다운로드 | 최신 WDK 설치 | https://learn.microsoft.com/windows-hardware/drivers/download-the-wdk |
| File System Minifilter Drivers | Minifilter 공식 문서 | https://learn.microsoft.com/windows-hardware/drivers/ifs/file-system-minifilter-drivers |
| Filter Manager Concepts | 필터 관리자 개념 | https://learn.microsoft.com/windows-hardware/drivers/ifs/filter-manager-concepts |
| FLT_CALLBACK_DATA | 콜백 데이터 구조체 | https://learn.microsoft.com/windows-hardware/drivers/ddi/fltkernel/ns-fltkernel-_flt_callback_data |

### NTFS 및 파일 시스템

| 자료 | 설명 |
|------|------|
| NTFS Technical Reference | NTFS 파일 시스템 기술 레퍼런스 |
| File System Filter Driver Development | 파일 시스템 필터 드라이버 개발 가이드 |
| IRP Major Function Codes | IRP 주요 함수 코드 설명 |

### 디버깅 도구

| 도구 | 설명 | URL |
|------|------|-----|
| WinDbg Preview | 최신 디버거 | Microsoft Store |
| Driver Verifier | 드라이버 검증 도구 | 내장 |
| Windows Performance Analyzer | 성능 분석 도구 | Windows SDK |

---

## F.2 추천 도서

### 필수 도서

#### 1. Windows Internals (7th Edition)
- **저자**: Pavel Yosifovich, Alex Ionescu, Mark Russinovich, David Solomon
- **출판사**: Microsoft Press
- **내용**: Windows 커널 아키텍처의 바이블
- **추천 장**:
  - Chapter 6: I/O System
  - Chapter 8: Security
  - Chapter 11: File Systems

#### 2. Programming the Windows Driver Model (2nd Edition)
- **저자**: Walter Oney
- **출판사**: Microsoft Press
- **내용**: WDM 드라이버 개발의 고전
- **참고**: 구버전이지만 기본 개념은 여전히 유효

#### 3. Windows NT File System Internals
- **저자**: Rajeev Nagar
- **출판사**: O'Reilly
- **내용**: 파일 시스템 내부 구조 상세 설명
- **참고**: 절판되었으나 PDF로 구할 수 있음

### 보조 도서

#### 4. The Art of Software Security Assessment
- **저자**: Mark Dowd, John McDonald, Justin Schuh
- **내용**: 보안 코드 분석 기법
- **관련성**: DRM 드라이버 보안 강화에 유용

#### 5. Practical Malware Analysis
- **저자**: Michael Sikorski, Andrew Honig
- **내용**: 악성코드 분석 기법
- **관련성**: 보안 드라이버 개발 시 공격 패턴 이해

---

## F.3 온라인 학습 자료

### 공식 교육

| 과정 | 제공자 | 설명 |
|------|--------|------|
| Windows Driver Development | Microsoft Learn | 무료 온라인 학습 경로 |
| Kernel Debugging | Microsoft | 커널 디버깅 워크샵 |

### 커뮤니티 블로그

#### OSR Online
- **URL**: https://www.osr.com/
- **내용**: 드라이버 개발 전문 회사의 기술 블로그
- **추천 글**:
  - "The NT Insider" 뉴스레터
  - Minifilter 관련 기술 문서

#### ReactOS Wiki
- **URL**: https://reactos.org/wiki/
- **내용**: Windows 호환 OS 개발 프로젝트
- **유용성**: Windows 내부 구조 이해에 도움

#### Nynaeve
- **URL**: https://www.yourkit.com/docs/java/help/
- **내용**: 커널 내부 분석 블로그

### YouTube 채널

| 채널 | 내용 |
|------|------|
| LiveOverflow | 보안 및 리버스 엔지니어링 |
| GynvaelEN | 보안, 디버깅, 프로그래밍 |
| hasherezade | 악성코드 분석 및 리버싱 |

---

## F.4 샘플 코드 및 오픈소스

### Microsoft 공식 샘플

#### Windows Driver Samples (GitHub)
- **URL**: https://github.com/microsoft/Windows-driver-samples
- **Minifilter 샘플**:

```
Windows-driver-samples/
├── filesys/
│   ├── miniFilter/
│   │   ├── avscan/        # 안티바이러스 스캐너
│   │   ├── cdo/           # Control Device Object
│   │   ├── ctx/           # 컨텍스트 사용 예제
│   │   ├── delete/        # 삭제 감지
│   │   ├── metadata/      # 메타데이터 관리
│   │   ├── minispy/       # 파일 활동 모니터
│   │   ├── nullFilter/    # 최소 필터
│   │   ├── passthrough/   # 패스스루 필터
│   │   ├── scanner/       # 콘텐츠 스캐너
│   │   └── swapBuffer/    # 버퍼 스왑
```

#### 주요 샘플 설명

| 샘플 | 설명 | 학습 포인트 |
|------|------|------------|
| minispy | 파일 I/O 모니터링 | IRP 추적, 로깅 |
| scanner | 콘텐츠 스캔 | 파일 읽기, 패턴 매칭 |
| ctx | 컨텍스트 사용 | 컨텍스트 생성, 관리 |
| delete | 삭제 감지 | SetInformation 처리 |
| nullFilter | 최소 구현 | 기본 구조 이해 |

### 오픈소스 프로젝트

#### 1. Process Monitor (소스 비공개, 기능 참고)
- **제작**: Sysinternals/Microsoft
- **용도**: 파일/레지스트리/프로세스 모니터링
- **학습**: 모니터링 기능 구현 참고

#### 2. OSR Community 샘플
- **URL**: https://www.osr.com/downloads/
- **내용**: 다양한 드라이버 샘플과 유틸리티

#### 3. ReactOS 소스 코드
- **URL**: https://github.com/reactos/reactos
- **유용한 부분**:
  - `ntoskrnl/` - 커널 구현
  - `drivers/filesystems/` - 파일 시스템 드라이버

---

## F.5 개발 도구

### 필수 도구

| 도구 | 용도 | 설치 방법 |
|------|------|----------|
| Visual Studio 2022 | IDE | visualstudio.microsoft.com |
| WDK | 드라이버 개발 키트 | learn.microsoft.com |
| WinDbg Preview | 디버거 | Microsoft Store |
| Sysinternals Suite | 시스템 유틸리티 | learn.microsoft.com/sysinternals |

### Sysinternals 도구

| 도구 | 용도 |
|------|------|
| Process Monitor | 파일/레지스트리/프로세스 활동 모니터 |
| Process Explorer | 고급 작업 관리자 |
| WinObj | 커널 객체 뷰어 |
| DebugView | 디버그 출력 뷰어 |
| Autoruns | 자동 시작 프로그램 관리 |
| AccessChk | 보안 권한 확인 |

### 보조 도구

| 도구 | 용도 | 비고 |
|------|------|------|
| IDA Pro | 디스어셈블러 | 유료 |
| Ghidra | 리버스 엔지니어링 | 무료 (NSA) |
| x64dbg | 사용자 모드 디버거 | 무료 |
| HxD | 헥스 에디터 | 무료 |
| Dependency Walker | DLL 의존성 확인 | 무료 |

### 빌드 및 배포 도구

```powershell
# 필수 도구 확인 스크립트
# Essential tools verification script

function Test-DriverDevEnv {
    $tools = @{
        'Visual Studio' = Test-Path "${env:ProgramFiles}\Microsoft Visual Studio\2022"
        'WDK' = Test-Path "${env:ProgramFiles(x86)}\Windows Kits\10\Include"
        'WinDbg' = Get-Command windbg.exe -ErrorAction SilentlyContinue
        'SignTool' = Get-Command signtool.exe -ErrorAction SilentlyContinue
    }

    foreach ($tool in $tools.GetEnumerator()) {
        $status = if ($tool.Value) { "OK" } else { "Missing" }
        Write-Host "$($tool.Key): $status"
    }
}

Test-DriverDevEnv
```

---

## F.6 인증 및 자격증

### Microsoft 인증

| 인증 | 대상 | 내용 |
|------|------|------|
| Azure Developer | 클라우드 개발자 | Azure 서비스 활용 |
| Security, Compliance, and Identity | 보안 전문가 | 보안 아키텍처 |

### 관련 자격증

| 자격증 | 발급 기관 | 관련성 |
|--------|----------|--------|
| OSCP | Offensive Security | 보안 테스트 |
| OSCE | Offensive Security | 익스플로잇 개발 |
| GREM | GIAC | 리버스 엔지니어링 |

---

## F.7 컨퍼런스 및 이벤트

### 주요 컨퍼런스

| 컨퍼런스 | 시기 | 내용 |
|----------|------|------|
| Microsoft Build | 5월 | Microsoft 기술 발표 |
| DEF CON | 8월 | 해킹/보안 컨퍼런스 |
| Black Hat | 8월 | 보안 연구 발표 |
| BlueHat | 다양 | Microsoft 보안 컨퍼런스 |
| Windows Internals Conf | 다양 | 커널 개발 집중 |

### 온라인 이벤트

| 이벤트 | 특징 |
|--------|------|
| Microsoft Reactor | 온라인 기술 세미나 |
| MSDN Webinars | 기술 웨비나 |

---

## F.8 커뮤니티 및 포럼

### 공식 포럼

| 포럼 | URL | 용도 |
|------|-----|------|
| Microsoft Q&A | learn.microsoft.com/answers | 기술 질문 |
| MSDN Forums | social.msdn.microsoft.com | 레거시 포럼 |

### 비공식 커뮤니티

| 커뮤니티 | 플랫폼 | 특징 |
|----------|--------|------|
| OSR Community | 자체 사이트 | 드라이버 전문 |
| r/ReverseEngineering | Reddit | 리버싱 커뮤니티 |
| StackOverflow | 웹 | 일반 프로그래밍 Q&A |
| Rootkit.com (Archive) | Archive.org | 역사적 자료 |

### 메일링 리스트

| 리스트 | 특징 |
|--------|------|
| ntdev | Microsoft 드라이버 개발 |
| ntfsd | 파일 시스템 드라이버 |

---

## F.9 연습 문제 및 프로젝트

### 기초 연습

#### 연습 1: 파일 모니터링
**목표**: 모든 파일 열기를 로깅하는 Minifilter 작성
**학습 포인트**:
- IRP_MJ_CREATE 처리
- 파일 이름 가져오기
- DbgPrint 사용

```c
// 힌트: PreCreate 콜백에서
FltGetFileNameInformation(Data, FLT_FILE_NAME_NORMALIZED, &nameInfo);
DbgPrint("File opened: %wZ\n", &nameInfo->Name);
```

#### 연습 2: 확장자 필터링
**목표**: 특정 확장자(.txt) 파일만 모니터링
**학습 포인트**:
- 파일 이름 파싱
- 확장자 비교

#### 연습 3: 통신 포트 구현
**목표**: 사용자 모드 앱과 통신하는 포트 추가
**학습 포인트**:
- FltCreateCommunicationPort
- FltSendMessage
- 사용자 모드 연결

### 중급 연습

#### 연습 4: 콘텐츠 스캐너
**목표**: 파일 읽기 시 특정 패턴 검색
**학습 포인트**:
- PostRead 콜백
- 버퍼 접근
- 패턴 매칭

#### 연습 5: 쓰기 차단
**목표**: 보호된 폴더에 쓰기 차단
**학습 포인트**:
- PreWrite 콜백
- 경로 비교
- STATUS_ACCESS_DENIED 반환

#### 연습 6: 삭제 방지
**목표**: 특정 파일 삭제 방지
**학습 포인트**:
- FileDispositionInformation 감지
- PreSetInformation 처리

### 고급 연습

#### 연습 7: 실시간 암호화
**목표**: 파일 읽기/쓰기 시 투명 암호화
**학습 포인트**:
- 버퍼 수정
- 암호화 알고리즘 적용
- 파일 메타데이터 관리

#### 연습 8: 버전 관리
**목표**: 파일 수정 시 이전 버전 자동 백업
**학습 포인트**:
- 파일 복사
- 버전 명명 규칙
- 스토리지 관리

#### 연습 9: 감사 로그
**목표**: 모든 파일 접근을 데이터베이스에 기록
**학습 포인트**:
- 비동기 처리
- 사용자 모드 서비스 연동
- 대용량 데이터 처리

### 프로젝트 아이디어

| 프로젝트 | 난이도 | 설명 |
|----------|--------|------|
| 파일 접근 로거 | 초급 | 모든 파일 접근 기록 |
| 폴더 보호기 | 초급 | 특정 폴더 쓰기 방지 |
| 확장자 차단기 | 중급 | .exe 파일 실행 차단 |
| 랜섬웨어 방지 | 중급 | 대량 파일 수정 감지/차단 |
| 투명 암호화 | 고급 | 실시간 파일 암호화 |
| DLP 솔루션 | 고급 | 데이터 유출 방지 |

---

## F.10 학습 로드맵

### 초급 (1-3개월)

```
1주차: C 언어 복습
  └── 포인터, 구조체, 메모리 관리

2-3주차: Windows 아키텍처
  └── 커널 모드 vs 사용자 모드
  └── I/O 관리자, 파일 시스템

4-5주차: 개발 환경 설정
  └── Visual Studio + WDK 설치
  └── 가상 머신 디버깅 설정

6-8주차: 첫 드라이버
  └── Hello World 드라이버
  └── WinDbg 기초

9-12주차: 기본 Minifilter
  └── 파일 모니터링
  └── 로깅 구현
```

### 중급 (3-6개월)

```
1-2개월: 콜백 심화
  └── 모든 IRP 유형 이해
  └── Pre/Post 처리 패턴

2-3개월: 컨텍스트 관리
  └── 스트림/파일 컨텍스트
  └── 인스턴스 컨텍스트

3-4개월: 통신
  └── 커널-사용자 통신
  └── 사용자 모드 서비스

4-6개월: 최적화
  └── 성능 측정
  └── 메모리 최적화
```

### 고급 (6-12개월)

```
6-8개월: 보안 기능
  └── 암호화 구현
  └── 접근 제어

8-10개월: 배포
  └── 드라이버 서명
  └── WHQL 인증

10-12개월: 프로덕션
  └── 에러 처리
  └── 업데이트 메커니즘
  └── 원격 관리
```

---

## 요약

Windows Minifilter 드라이버 개발은 지속적인 학습이 필요한 분야입니다:

1. **공식 문서**: Microsoft 문서와 WDK 샘플로 시작
2. **도서**: Windows Internals는 필독서
3. **실습**: 직접 코드를 작성하고 디버깅
4. **커뮤니티**: OSR, 포럼에서 질문하고 공유
5. **프로젝트**: 실제 문제를 해결하는 프로젝트 수행

꾸준한 학습과 실습을 통해 전문 드라이버 개발자로 성장할 수 있습니다.
