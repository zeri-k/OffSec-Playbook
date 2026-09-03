---
tags:
  - 환경/windows
  - 서비스/rdp
  - 기능/세션관리
실행환경: ["Windows"]
필요권한: ["다른 사용자 세션 연결은 SYSTEM 수준의 session 제어 권한"]
필요조건: ["대상 session ID", "연결할 destination session name"]
결과: ["지정한 Windows Terminal Services session 연결"]
---

# tscon

## 도구 개요

`tscon.exe`는 Windows Terminal Services session을 다른 session에 연결하는 기본 명령이다. `query user`로 확인한 기존 RDP session을 현재 RDP desktop에 연결할 때 사용하며, 다른 사용자 session 연결에는 SYSTEM 수준의 권한이 필요하다.

## 필요한 입력과 실행 환경

- 실행 환경: 대상 RDP session이 존재하는 Windows 호스트
- 입력: 대상 session ID와 현재 destination session name
- 권한: SYSTEM 또는 해당 session을 제어할 수 있는 권한

## 표준 문법

```cmd
tscon <TARGET_SESSION_ID> /dest:<CURRENT_SESSION_NAME>
```

## 대표 예시

```cmd
query user
tscon 2 /dest:rdp-tcp#13
```

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `<TARGET_SESSION_ID>` | 연결할 사용자 session의 숫자 ID |
| `/dest:<SESSION_NAME>` | 대상 session을 연결할 현재 destination session 이름 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| 현재 desktop이 다른 사용자 session으로 전환 | session 연결 성공 | 새 desktop에서 `whoami /all` 확인 |
| `Access is denied` | 실행 권한 부족 | SYSTEM 실행 여부 확인 |
| session을 찾지 못함 | ID가 종료·변경됐거나 다른 호스트의 ID | `query user` 재실행 |
| 명령 뒤 변화 없음 | destination 이름 또는 OS 지원 조건 불일치 | 현재 session name과 운영체제 확인 |

## 관련 공격기법

- [[RDP 세션 하이재킹]]
