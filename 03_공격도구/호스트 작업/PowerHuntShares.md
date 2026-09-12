---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/자격증명수집
실행환경: ["Windows PowerShell"]
필요권한: ["현재 Windows 계정으로 네트워크 공유를 열거하고 파일을 읽을 권한"]
결과: ["접근 가능한 SMB 공유와 파일 후보", "HTML 보고서와 로그"]
---

# PowerHuntShares

## 도구 개요

PowerHuntShares는 Windows PowerShell에서 여러 SMB 공유를 탐색하고 접근 가능한 파일과 민감 정보 후보를 HTML 보고서로 정리하는 도구다. 공유가 많아 수동 탐색 결과를 우선순위화해야 할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: PowerHuntShares 모듈을 불러올 수 있고 대상 SMB 공유에 도달할 수 있는 Windows 호스트
- 인증 주체: 현재 PowerShell 프로세스를 실행하는 Windows 계정
- 필요한 입력: 결과 저장 디렉터리와 병렬 작업 수

## 표준 문법

```powershell
Import-Module .\PowerHuntShares.psm1
Invoke-HuntSMBShares -Threads <THREAD_COUNT> -OutputDirectory <OUTPUT_DIRECTORY>
```

## 대표 예시

```powershell
Import-Module .\PowerHuntShares.psm1
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public\ShareHunt
```

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `-Threads` | 동시에 수행할 공유 탐색 작업 수 |
| `-OutputDirectory` | HTML 보고서와 로그를 저장할 경로 |

## 도구 고유 출력

| 출력·파일 | 의미 | 다음 확인 |
|---|---|---|
| 탐색 호스트·공유 진행 상태 | 현재 계정으로 공유 열거 수행 | 연결 실패와 접근 거부를 구분 |
| HTML 보고서 | 접근 가능한 공유와 파일 후보 정리 | 실제 파일 ACL과 내용 직접 확인 |
| 다수의 일치 항목 | 자동 분류 후보이며 오탐 포함 가능 | 경로, 파일 유형과 내용을 수동 검토 |

## 관련 공격기법

- [[SMB 공유 자격증명 수집]]
