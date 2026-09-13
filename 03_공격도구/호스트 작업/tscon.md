---
tags:
  - 환경/windows
  - 서비스/rdp
  - 기능/세션관리
실행환경: ["Windows"]
필요권한: ["대상 세션의 Full Control 또는 Connect 권한"]
필요조건: ["대상 session ID", "연결할 destination session name", "다른 사용자 소유 세션은 문서화된 절차에서 소유자 암호"]
결과: ["지정한 Windows Terminal Services session 연결"]
---

# tscon

## 도구 개요

`tscon.exe`는 Windows Terminal Services session을 다른 session에 연결하는 기본 명령이다. `query user`로 확인한 기존 RDP session을 현재 RDP desktop에 연결할 때 사용한다. Microsoft의 현재 문서화된 계약은 대상 세션의 Full Control 또는 Connect 권한과, 다른 사용자 소유 세션이면 그 사용자 암호를 요구한다.

## 필요한 입력과 실행 환경

- 실행 환경: 대상 RDP session이 존재하는 Windows 호스트
- 입력: 대상 session ID와 현재 destination session name
- 권한: 해당 session의 Full Control 또는 Connect 권한
- 인증: 다른 사용자 소유 session은 `/password:*`로 소유자 암호를 prompt에 입력하는 것이 문서화된 기본이다. SYSTEM 무암호 연결은 [[RDP 세션 하이재킹]]에서 버전·구성 의존 기법으로 별도 판정한다.
- `<TARGET_SESSION_ID>`는 `query user` 출력의 numeric ID, `<CURRENT_SESSION_NAME>`은 현재 연결 대상 session name이다. session 존재는 Connect/Full Control 권한이나 대상 사용자 인증을 뜻하지 않는다.

## 표준 문법

`<TARGET_SESSION_ID>`는 target Windows host에서 `query user`가 반환한 numeric session ID, `<CURRENT_SESSION_NAME>`은 현재 destination session name이다. `/password:*`는 대상 session 소유자 credential을 interactive prompt로 받으며 Connect/Full Control 권한과 session 존재·연결 성공·desktop access 결과를 구분한다.

```cmd
tscon <TARGET_SESSION_ID> /dest:<CURRENT_SESSION_NAME> /password:* /v
```

## 대표 예시

예시의 session ID와 name은 직전 `query user` output에서 같은 host 기준으로 얻어 재사용한다. 연결 뒤 기존 session state를 바꾸거나 새 process·파일을 만들지 않는 이 도구의 cleanup은 새 session 생성 여부가 아니라 original destination/session mapping을 다시 확인하는 것으로 한정한다.

```cmd
query user
tscon <TARGET_SESSION_ID> /dest:<CURRENT_SESSION_NAME> /password:* /v
```

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `<TARGET_SESSION_ID>` | 연결할 사용자 session의 숫자 ID |
| `/dest:<SESSION_NAME>` | 대상 session을 연결할 현재 destination session 이름 |
| `/password:*` | 소유자가 다른 session의 암호를 prompt에서 입력 |
| `/v` | session 연결 작업의 세부 정보 표시 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| 현재 desktop이 다른 사용자 session으로 전환 | session 연결 성공 | 새 desktop에서 `whoami /all` 확인 |
| `Access is denied` 또는 암호 요구 | Connect·Full Control 권한, 대상 소유자 암호 또는 구현 조건 미충족 | 세션 소유자·권한·현재 Windows·RDS 구성 확인 |
| session을 찾지 못함 | ID가 종료·변경됐거나 다른 호스트의 ID | `query user` 재실행 |
| 명령 뒤 변화 없음 | destination 이름 또는 OS 지원 조건 불일치 | 현재 session name과 운영체제 확인 |

## 관련 공격기법

- [[RDP 세션 하이재킹]]

## 참고 링크

- [Microsoft: tscon](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tscon)
