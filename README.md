# 📘 Windows Minifilter 드라이버 개발 가이드

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Live-brightgreen?style=flat-square&logo=github)](https://christian289.github.io/windows-minifilter-develop-guide/)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)
[![Language](https://img.shields.io/badge/Language-한국어-red?style=flat-square)]()

> .NET 개발자를 위한 Windows Minifilter 드라이버 개발 전자책

## 📖 온라인으로 읽기

**[https://christian289.github.io/windows-minifilter-develop-guide/](https://christian289.github.io/windows-minifilter-develop-guide/)**

---

## 🎯 이 책의 대상

- **10년 이상의 .NET/C# 경험**이 있지만 C 언어 경험이 적은 개발자
- Windows 커널 모드 드라이버 개발에 입문하고자 하는 개발자
- 파일 시스템 필터 드라이버(Minifilter) 개발을 배우고자 하는 개발자
- DRM, 안티바이러스, 백업 솔루션 개발에 관심 있는 개발자

## 📚 목차

### Part 1-3: 기초
| Part | 내용 | Chapters |
|------|------|----------|
| **Part 1** | C 언어 기초 | 포인터, 구조체, 전처리기 |
| **Part 2** | Windows 아키텍처 | 커널 모드, IRQL, 메모리 |
| **Part 3** | WDK 환경 | 설치, 빌드, 테스트 서명 |

### Part 4-6: 드라이버 개발 입문
| Part | 내용 | Chapters |
|------|------|----------|
| **Part 4** | WinDbg 마스터 | 디버깅 명령어, 크래시 분석 |
| **Part 5** | 첫 번째 드라이버 | Hello World, 로드/언로드 |
| **Part 6** | Minifilter 기초 | 아키텍처, 콜백, 컨텍스트 |

### Part 7-9: DRM 구현
| Part | 내용 | Chapters |
|------|------|----------|
| **Part 7** | 콘텐츠 분석 | 파일 I/O 인터셉트, 패턴 매칭 |
| **Part 8** | DRM 보호 | 접근 제어, 암호화, DLP |
| **Part 9** | 사용자 모드 연동 | 통신 포트, WPF 콘솔 |

### Part 10-12: 프로덕션
| Part | 내용 | Chapters |
|------|------|----------|
| **Part 10** | 테스트 최적화 | 단위 테스트, 성능 최적화 |
| **Part 11** | 배포 | 코드 서명, WHQL, 업데이트 |
| **Part 12** | 고급 주제 | 가상화, 클라우드, 보안 강화 |

### 부록
- **부록 A**: WDK API 레퍼런스
- **부록 B**: NTSTATUS 코드 레퍼런스
- **부록 C**: IRP 주요 함수 코드
- **부록 D**: 디버깅 체크리스트
- **부록 E**: 프로젝트 템플릿
- **부록 F**: 추가 학습 자료
- **용어집** & **찾아보기**

## ✨ 특징

### 🔄 C# 비유
모든 C/커널 개념을 .NET 개발자가 익숙한 C# 개념으로 비유하여 설명합니다.

```c
// C 커널 코드
PVOID buffer = ExAllocatePool2(POOL_FLAG_NON_PAGED, size, TAG);
ExFreePoolWithTag(buffer, TAG);
```

```csharp
// C# 비유: ArrayPool과 유사
var buffer = ArrayPool<byte>.Shared.Rent(size);
ArrayPool<byte>.Shared.Return(buffer);
```

### 💻 완전한 코드 예제
한글과 영문 주석이 함께 포함된 실행 가능한 코드를 제공합니다.

```c
// SSN 패턴 탐지 함수
// SSN pattern detection function
BOOLEAN ContainsSsnPattern(
    _In_ PUCHAR Buffer,
    _In_ ULONG Length)
{
    // 주민등록번호 패턴: XXXXXX-XXXXXXX
    // Korean SSN pattern: XXXXXX-XXXXXXX
    for (ULONG i = 0; i < Length - 14; i++) {
        if (IsDigit(Buffer[i]) && Buffer[i + 6] == '-') {
            // 패턴 발견
            // Pattern found
            return TRUE;
        }
    }
    return FALSE;
}
```

### 🛡️ 실전 DRM 프로젝트
책의 최종 목표는 SSN(주민등록번호) 패턴을 탐지하는 완전한 DRM Minifilter 드라이버입니다.

## 🛠️ 개발 환경

- Windows 10/11 (64-bit)
- Visual Studio 2022
- Windows Driver Kit (WDK)
- WinDbg Preview
- 가상 머신 (VMware/Hyper-V)

## 📄 라이선스

이 프로젝트는 MIT 라이선스 하에 배포됩니다.

## 🙏 기여

오류 수정, 내용 개선 제안은 [Issues](https://github.com/christian289/windows-minifilter-develop-guide/issues)에 남겨주세요.

---

<p align="center">
  <strong>🤖 Generated with <a href="https://claude.com/claude-code">Claude Code</a></strong>
</p>
