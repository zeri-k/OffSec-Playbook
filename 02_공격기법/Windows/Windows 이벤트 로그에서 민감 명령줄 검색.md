---
aliases: ["Event Log Readers 이벤트 로그 자격 증명 검색"]
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트에서 사용자 셸 또는 세션 확보", "현재 계정이 읽을 수 있는 이벤트 로그 확인"]
필요권한: ["대상 로그 채널의 읽기 권한", "Security 로그는 관리자·Event Log Readers 또는 채널 ACL로 부여된 동등한 읽기 권한"]
필요조건: ["대상 호스트에서 wevtutil 또는 PowerShell 실행", "프로세스 생성 감사와 명령줄 포함 정책 또는 PowerShell 로깅이 사전에 활성화됨"]
결과: ["계정과 대상이 연결된 평문 비밀번호·token·명령줄 후보", "이벤트 ID·시각·실행 계정·프로세스가 연결된 자격 증명 단서"]
문서역할: 수동절차
---

# Windows 이벤트 로그에서 민감 명령줄 검색

## 한 줄 판단

현재 Windows 계정이 읽을 수 있는 이벤트 로그에서 프로세스 생성 이벤트 4688과 PowerShell script block 이벤트를 제한적으로 조회하여, 명령 인자로 기록된 계정명·비밀번호·token 후보를 찾고 실제 대상 서비스에서 별도로 검증한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 로그가 생성된 대상 Windows 호스트의 셸 또는 해당 호스트로의 원격 이벤트 로그 경로 | `hostname`, `whoami`와 조회 명령의 대상 확인 | 로컬 로그와 원격 로그를 섞지 말고 RPC·방화벽·인증 조건을 별도 확인 |
| 현재 계정 | 현재 token에 `Event Log Readers` SID 또는 동등한 채널 읽기 ACE가 반영됨 | `whoami /groups`와 실제 채널 조회 결과 확인 | 그룹 디렉터리 상태와 현재 token을 구분하고 새 로그온 필요 여부 확인 |
| 현재 권한 | 선택한 로그 채널의 event read 성공 | `wevtutil gl <LOG>`와 제한된 query의 접근 거부 여부 확인 | 채널별 ACL을 확인하고 다른 채널의 성공을 Security 로그 권한으로 확대하지 않음 |
| 기록 조건 | 4688과 `Process Command Line` 또는 PowerShell Operational 이벤트가 실제로 기록됨 | 최근 이벤트 몇 건의 ID·message·XML 확인 | 이벤트 부재, audit 미설정, retention으로 인한 삭제를 구분 |
| 검색 범위 | 승인된 시간 범위와 최대 이벤트 수를 정함 | 시작 시각과 `/c:<COUNT>` 또는 `-MaxEvents` 기록 | 전체 로그 무제한 dump 대신 최근 범위부터 좁힘 |

Event Log Readers 멤버십은 읽기 후보일 뿐 모든 채널 접근을 보장하지 않는다. 특히 Security 채널은 로컬 정책과 채널 ACL에 따라 별도 권한이 필요할 수 있으므로 실제 query 성공으로 확인한다.

## 실행

### 1. 현재 token과 로그 가시성 확인

대상 Windows CMD에서 현재 token의 그룹과 Security 로그 구성을 확인한다.

```cmd
whoami /groups | findstr /i "S-1-5-32-573 Event Log Readers"
wevtutil gl Security
```

확인할 출력:

- SID `S-1-5-32-573` 또는 현지화된 Event Log Readers 이름이 현재 token에 표시되는지 확인한다.
- `channelAccess`와 log 구성 출력이 반환되면 메타데이터 읽기가 가능하다. 실제 event query 권한은 다음 단계에서 다시 확인한다.
- `Access is denied`이면 그룹 목록만으로 읽기 성공을 기록하지 않는다.

### 2. 4688 프로세스 명령줄을 제한적으로 조회

최근 4688 이벤트를 역순으로 최대 `<MAX_EVENTS>`건 조회하고, 계정·비밀번호를 명령 인자로 받기 쉬운 문자열만 화면에서 좁힌다. 이 명령의 출력에는 실제 secret이 포함될 수 있으므로 파일로 자동 저장하지 않는다.

```cmd
wevtutil qe Security /q:"*[System[(EventID=4688)]]" /rd:true /f:text /c:<MAX_EVENTS> | findstr /i /c:"/user:" /c:"password" /c:"passwd" /c:"token"
```

PowerShell을 사용할 수 있으면 provider 측에서 ID를 먼저 제한하고, 반환 message에서 후보 문자열을 찾는다.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688} -MaxEvents <MAX_EVENTS> |
  Where-Object Message -Match '(?i)(/user:|password|passwd|token)' |
  Select-Object TimeCreated,Id,ProviderName,Message
```

확인할 출력:

- `TimeCreated`, event ID 4688, Creator/Subject 계정, 새 process와 `Process Command Line`을 함께 기록한다.
- `Process Command Line`이 비어 있으면 명령줄 포함 정책이 적용되지 않았거나 해당 event version·기록 조건이 다를 수 있다.
- 키워드 hit는 secret 후보일 뿐이다. 마스킹된 값, option 이름, 만료된 token과 실제 평문을 구분한다.

### 3. PowerShell Operational 로그 확인

Script Block Logging이 켜진 호스트에서는 event ID 4104에 script 내용이 남을 수 있다. 현재 계정이 해당 채널을 읽을 수 있을 때만 최근 범위를 확인한다.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; Id=4104} -MaxEvents <MAX_EVENTS> |
  Where-Object Message -Match '(?i)(password|passwd|credential|token|secret)' |
  Select-Object TimeCreated,Id,Message
```

확인할 출력:

- 실제 script block text와 생성 시각. event 존재는 script가 성공했거나 값이 현재 유효하다는 증거가 아니다.
- 채널 미존재, event 없음과 접근 거부를 별도 상태로 기록한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 4688에 실행 계정·대상과 함께 평문 비밀번호가 보임 | 명령줄 감사가 secret을 평문으로 기록함 | 계정명이 연결된 자격 증명 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 대상 서비스·유효성·권한을 최소 시도로 검증 |
| 4104에 token·secret 후보가 보임 | PowerShell script text에 민감 값이 기록됨 | token 또는 자격 증명 후보 | token 형식·issuer·audience·만료와 대상 서비스를 확인 |
| 4688은 있으나 `Process Command Line`이 비어 있음 | 프로세스 생성은 기록됐지만 명령줄 포함 조건 미충족 | 명령줄 자료 미확인 | audit 설정과 event version을 확인하되 정책을 임의 변경하지 않음 |
| Security는 거부되고 다른 채널만 읽힘 | 채널별 ACL 범위가 다름 | 부분 로그 가시성 | 허용된 채널만 조사하고 Security 접근 성공으로 확대하지 않음 |
| 키워드 hit가 option 이름·마스킹 값뿐임 | 민감 값 미확보 | 자격 증명 미확인 | [[Windows 파일 자격증명 검색]]과 [[Windows 저장 자격증명 수집]]을 별도 수행 |
| 결과 없음 | 선택한 시간·ID·retention 범위에서 후보 없음 | 로그 기반 자격 증명 미확인 | 시간 범위와 audit 설정을 확인하고 다른 저장소로 전환 |

## 확인할 출력과 권한

- 그룹 구성원 목록, 현재 process token의 그룹 SID, 특정 채널 event read 성공을 서로 분리한다.
- event에 기록된 문자열은 과거 명령 인자다. 현재 유효한 비밀번호·token인지, 어느 계정과 서비스에 속하는지는 별도 검증이 필요하다.
- 조회만 수행하며 로그를 clear·export하거나 audit 정책을 변경하지 않는다. `wevtutil cl`은 증적을 지우므로 이 절차에서 사용하지 않는다.

## 후속 공격 연결

- 평문 비밀번호·token 후보: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- 다른 로컬 저장소 확인: [[Windows 파일 자격증명 검색]], [[Windows 저장 자격증명 수집]]

## 관련 도구

- [[wevtutil]]
- [[powershell]]

## 관련 상태 라우터

- [[Windows 위임 운영 그룹 확인 후 권한 경로 선택]]

## 참고 링크

- [Microsoft Learn: Event Log Readers](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#event-log-readers)
- [Microsoft Learn: Command line process auditing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/component-updates/command-line-process-auditing)
- [Microsoft Learn: Event 4688](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4688)
- [Microsoft Learn: Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
