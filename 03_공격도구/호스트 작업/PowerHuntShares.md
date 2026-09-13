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

`<POWERSHARES_PARENT>`는 Windows collector host에서 이번 실행 전에 없던 절대 output directory, `<THREAD_COUNT>`는 관찰한 대상 수·SMB 제한에 맞는 양의 정수다. `<POWERSHARES_RUN_DIR>`은 command가 출력한 정확한 child path를 재사용하며, parent를 추정해 재귀 삭제하지 않는다.

```powershell
if (Test-Path -LiteralPath '<POWERSHARES_PARENT>') { throw 'Output parent already exists' }
New-Item -ItemType Directory -Path '<POWERSHARES_PARENT>'
Import-Module .\PowerHuntShares.psm1
Invoke-HuntSMBShares -Threads <THREAD_COUNT> -OutputDirectory '<POWERSHARES_PARENT>'
```

경로 부재 guard가 통과한 고유 parent만 생성한다. `<THREAD_COUNT>`는 대상 수·SMB 제한·관찰 조건에서 작게 시작하고, 교육 예시의 `100`을 기본값으로 가정하지 않는다. `<POWERSHARES_PARENT>`는 Windows 실행 host의 새 절대 경로다.

## 대표 예시

```powershell
if (Test-Path -LiteralPath '<POWERSHARES_PARENT>') { throw 'Output parent already exists' }
New-Item -ItemType Directory -Path '<POWERSHARES_PARENT>'
Import-Module .\PowerHuntShares.psm1
Invoke-HuntSMBShares -Threads <THREAD_COUNT> -OutputDirectory '<POWERSHARES_PARENT>'
```

실행 출력의 `Output Directory` 행에서 도구가 생성한 `SmbShareHunt-<timestamp>` 절대 경로를 `<POWERSHARES_RUN_DIR>`로 기록한다. parent만 알고 하위 디렉터리를 추정하지 않는다.

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

## 변경 영향과 정리

PowerHuntShares는 원격 SMB·LDAP 조회를 수행하고 `<POWERSHARES_RUN_DIR>`에 HTML·CSV·log를 생성한다. 공유 파일을 수정하지는 않지만 서버 감사 흔적은 로컬 report 삭제로 되돌려지지 않는다. 필요한 결과를 별도 기록한 뒤 출력에서 기록한 정확한 run directory를 먼저 제거하고, 이번 작업이 만든 parent가 빈 후에만 parent를 제거한다.

```powershell
Remove-Item -LiteralPath '<POWERSHARES_RUN_DIR>' -Recurse -Force
Remove-Item -LiteralPath '<POWERSHARES_PARENT>' -Force
Test-Path -LiteralPath '<POWERSHARES_RUN_DIR>'
Test-Path -LiteralPath '<POWERSHARES_PARENT>'
```

두 확인이 `False`여야 로컬 report 정리가 완료된다. parent가 비지 않거나 run directory의 경로·소유자가 실행 출력과 다르면 삭제를 중단하고 남은 파일을 먼저 확인한다.

## 관련 공격기법

- [[SMB 공유 자격증명 수집]]

## 참고 링크

- [NetSPI PowerHuntShares 공식 저장소](https://github.com/NetSPI/PowerHuntShares)
