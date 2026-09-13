---
tags:
  - 환경/windows
시작조건: ["Windows 사용자 셸 확보", "현재 사용자 프로필에서 다른 계정의 저장된 대화형 자격 증명 확인"]
필요권한: ["현재 사용자 세션에서 runas 실행", "저장된 계정에 로컬 대화형 로그온 권한"]
필요조건: ["cmdkey 목록의 Domain:interactive 대상과 사용자", "실행할 프로그램"]
결과: ["저장된 계정으로 실행된 새 Windows 프로세스", "새 프로세스의 사용자·그룹·privilege·무결성 수준"]
---

# 저장된 자격 증명으로 runas 프로세스 실행

## 한 줄 판단

현재 Windows 사용자 프로필의 `cmdkey /list`에 다른 계정의 `Domain:interactive` 자격 증명이 있으면 `runas /savecred`로 그 계정의 새 프로세스를 만들고 새 창에서 사용자·그룹·privilege와 무결성 수준을 확인한다.

## 사용할 때

- `cmdkey /list`에 현재 사용자와 다른 로컬 또는 도메인 계정이 표시될 때.
- 저장된 비밀번호를 직접 복호화하지 않고 해당 계정으로 프로그램을 실행하려 할 때.
- 저장된 계정이 현재 계정보다 더 높은 로컬·도메인 권한을 가졌는지 확인해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 자격 증명이 저장된 사용자 프로필의 Windows 세션 | `hostname`, `whoami`, `%USERPROFILE%` | 다른 사용자 프로필의 저장 항목과 혼동하지 않음 |
| 저장 항목 | `Domain:interactive=<DOMAIN_OR_HOST>\<USER>`와 같은 대화형 대상 | `cmdkey /list`의 `Target`, `Type`, `User` | Generic·네트워크 대상과 대화형 자격 증명을 구분 |
| 실행 대상 계정 | 저장 항목의 사용자 이름과 `/user` 값이 일치 | 계정 형식을 그대로 대조 | 로컬 계정은 `<HOST>\<USER>`, 도메인 계정은 `<DOMAIN>\<USER>` 사용 |
| 로그온 조건 | 대상 계정으로 로컬 대화형 프로세스 생성 가능 | 새 프로세스 생성 또는 `RUNAS ERROR` 확인 | 계정 상태와 대화형 로그온 정책 확인 |

## 실행

### 저장된 계정과 현재 실행 계정 확인

```cmd
hostname
whoami
cmdkey /list
```

확인할 출력:

- 현재 호스트·사용자와 저장 항목의 `Target`, `Type`, `User`.
- 현재 사용자와 `runas`로 실행할 저장 계정을 서로 다른 계정으로 기록한다.

### 저장된 자격 증명으로 새 CMD 실행

다른 기존 `cmd.exe`와 구분할 `<RUNAS_WINDOW_TITLE>`을 정하고 같은 제목의 창이 없는지 먼저 확인한다.

```cmd
tasklist /v /fi "WINDOWTITLE eq <RUNAS_WINDOW_TITLE>"
runas /savecred /user:<DOMAIN_OR_HOST>\<USER> "cmd.exe /k title <RUNAS_WINDOW_TITLE>"
```

새 CMD 창에서 실행 계정과 현재 권한을 다시 확인한다.

```cmd
hostname
whoami
whoami /groups
whoami /priv
```

원래 창에서 제목으로 새 `cmd.exe`를 조회하고 exact PID를 `<RUNAS_PROCESS_PID>`로 기록한다.

```cmd
tasklist /v /fi "WINDOWTITLE eq <RUNAS_WINDOW_TITLE>" /fo list
```

확인할 출력:

- `Attempting to start` 뒤 새 CMD 창이 실제로 열리는지.
- 새 창의 `whoami`가 저장 항목의 사용자와 일치하는지.
- 그룹 목록, privilege와 무결성 수준. 새 프로세스 생성과 관리자·도메인 고권한을 구분한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 새 프로세스의 `whoami`가 저장된 계정과 일치 | 저장 자격 증명으로 대화형 프로세스 생성 성공 | 다른 Windows 계정의 프로세스 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 호스트·계정·권한 재평가 |
| 새 프로세스가 로컬 Administrators 그룹을 가지며 상승된 token으로 실행됨 | 대상 호스트의 고권한 프로세스 확인 | 로컬 관리자 세션 | [[고권한 세션 확보 후 후속 판단]] |
| 새 프로세스는 열렸지만 일반 사용자 그룹만 보임 | 계정 전환은 성공했으나 현재 목표 권한은 없음 | 다른 일반 사용자 프로세스 | 해당 사용자 프로필·파일·서비스 접근을 다시 열거 |
| `RUNAS ERROR: 2` | 실행 프로그램이나 경로를 찾지 못함 | 프로세스 미생성 | `cmd.exe` 절대 경로와 인용부호 확인 |
| `RUNAS ERROR: 1326` 또는 로그온 실패 | 저장 항목을 사용할 수 없거나 계정 형식·로그온 조건 불일치 | 프로세스 미생성 | `cmdkey` 대상·사용자, 계정 상태와 로그온 정책 확인 |
| `Attempting to start`만 보이고 새 창이 없음 | 프로세스 생성 결과 미확정 | 시작 상태 유지 | 현재 desktop session, 프로그램 경로와 이벤트 오류 확인 |

## 확인할 출력과 권한

- `cmdkey /list`는 저장 항목 존재만 보여 주며 비밀번호나 계정 유효성을 입증하지 않는다.
- `Attempting to start` 문구가 아니라 새 프로세스의 `whoami /all`로 실행 계정을 확정한다.
- 다른 사용자 프로세스, 로컬 관리자 token, 도메인 그룹과 원격 서비스 권한을 각각 다른 상태로 기록한다.

## 변경 영향과 복구

이 기법은 저장된 credential 항목을 새로 만들거나 수정하지 않지만, 저장된 계정의 `cmd.exe`와 그 하위 process를 만든다. 필요한 원격·파일 정리를 새 창에서 먼저 끝낸 다음 이번 실행에서 기록한 PID만 종료한다.

```cmd
tasklist /fi "PID eq <RUNAS_PROCESS_PID>"
taskkill /PID <RUNAS_PROCESS_PID> /T
tasklist /fi "PID eq <RUNAS_PROCESS_PID>"
```

첫 조회의 image·PID가 기록과 일치할 때만 종료한다. `Access is denied`이면 새 창에서 `exit`로 정상 종료한 뒤 원래 창에서 PID 부재를 다시 확인한다. 마지막 조회에 해당 PID가 없어야 process 정리가 확인된다. 이미 창이 닫혔다면 PID가 재사용됐을 수 있으므로 다른 process를 종료하지 말고, 실행 중 생성한 파일·원격 세션은 각 절차의 exact 식별값으로 별도 확인한다. `cmdkey`의 기존 항목은 이번 기법이 만든 자원이 아니므로 삭제하지 않는다.

## 관련 공격기법

- [[Windows 저장 자격증명 수집]]
- [[확보한 평문 비밀번호로 runas 사용자 프로세스 실행]]
- [[확보한 AD 비밀번호로 runas netonly 네트워크 인증 컨텍스트 생성]]

## 관련 도구

- [[runas]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Microsoft Learn: cmdkey](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/cmdkey)
- [Microsoft Learn: tasklist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tasklist)
- [Microsoft Learn: taskkill](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/taskkill)
