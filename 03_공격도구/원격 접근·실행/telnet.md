---
tags:
  - 서비스/telnet
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows"]
필요조건: ["대상 호스트와 평문 TCP 포트"]
결과: ["배너", "프로토콜 응답", "Telnet 세션"]
---

# telnet

## 도구 개요

`telnet`은 Telnet뿐 아니라 SMTP·POP3 같은 평문 TCP 서비스에 연결해 배너와 텍스트 명령 응답을 직접 확인하는 대화형 클라이언트다. 전용 클라이언트 없이 프로토콜 동작을 빠르게 재현할 때 적합하며, TCP 연결 성공과 서비스 인증 성공은 구분해야 한다.

## 필요한 입력과 실행 환경

- 실행 환경: Telnet client가 설치된 Linux 또는 Windows 호스트
- 입력: 대상 호스트와 평문 TCP 포트
- 프로토콜 입력: 서비스에 맞는 텍스트 명령


## 표준 사용법

```bash
telnet <target> <port>
```

## 대표 예시

### Telnet 서비스 접속

```bash
telnet <TARGET> 23
```

### SMTP 같은 텍스트 기반 서비스 배너 확인

```bash
telnet <TARGET> 25
```

### POP3 USER 응답 확인

```bash
telnet <TARGET> 110
USER john
QUIT
```

사용자 존재 여부에 따라 `+OK`와 `-ERR` 응답이 달라지는지 확인한다.


## 주요 인자와 명령

| 인자·명령 | 의미 | 사용하는 상황 |
|---|---|---|
| `<target> <port>` | 접속할 호스트와 TCP 포트 | Telnet 또는 평문 프로토콜 연결 |
| `Ctrl+]` | telnet command prompt로 전환 | 원격 연결 제어 |
| `status` | 현재 연결 상태 표시 | 대상과 모드 확인 |
| `close` | 현재 연결 종료 | telnet client는 유지 |
| `quit` | telnet client 종료 | 작업 종료 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| login/banner/prompt 출력 | Telnet 또는 평문 TCP 서비스 응답 확인 | 기본 계정, 배너 정보, 수동 프로토콜 대화 확인 |
| 로그인 성공 | 평문 원격 세션 확보 | `id`, `whoami`, 파일 접근, 권한 상승 단서 확인 |
| connection refused/closed | 서비스 비활성화 또는 접근 제한 | 포트 상태와 방화벽, 다른 관리 프로토콜 확인 |
| 입력 후 응답 없음 | 프로토콜 불일치 또는 라인 종료 문제 | `nc`, `curl`, 서비스별 클라이언트로 교차 확인 |

## 관련 공격기법

- [[SMTP 사용자 열거]]
- [[POP3 USER 사용자 열거]]
- [[SMTP 서비스#Open Relay 검증|SMTP Open Relay 검증]]
