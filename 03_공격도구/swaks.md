---
tags:
  - 서비스/smtp
  - 기능/프로토콜접근
실행환경: ["Linux"]
필요조건: ["SMTP 서버", "envelope 발신자와 수신자", "인증 시험 시 SMTP 계정"]
결과: ["SMTP 단계별 응답", "메일 큐 수락", "수신 여부"]
---

# swaks

## 도구 개요

`swaks`는 SMTP 대화를 단계별로 실행하고 서버 응답, STARTTLS, 인증과 메일 큐 수락 여부를 자세히 보여 주는 테스트 클라이언트다. 발신자·수신자 조합별 릴레이 정책이나 포트별 submission 동작을 재현할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: `swaks`가 설치된 Linux 호스트
- 입력: SMTP 서버, 포트와 envelope 발신자·수신자
- 선택 입력: 제목, 본문, STARTTLS 설정과 SMTP 인증 정보

## 표준 사용법

```bash
swaks --from <FROM> --to <TO> --server <TARGET>
```

발신자, 수신자와 서버를 명시하고 필요하면 제목과 본문을 추가한다. 각 SMTP 단계의 응답과 최종 수신 여부를 함께 확인한다.

## 대표 예시

### 오픈 릴레이 테스트 메일

```bash
swaks --from relay-test@external.test --to receiver@external.test --header 'Subject: Relay Test' --body 'Relay delivery check' --server <TARGET>
```

확인할 출력:

- `MAIL FROM`, `RCPT TO`, `DATA` 단계가 모두 `250` 계열로 수락되는지 확인한다.

### SMTP 포트 지정

```bash
swaks --from relay-test@external.test --to receiver@external.test --server <TARGET> --port 587
```

확인할 출력:

- 25/465/587 중 어떤 포트에서 relay 또는 submission 정책이 다른지 확인한다.

### STARTTLS 확인

```bash
swaks --server <TARGET> --port 587 --tls --from relay-test@external.test --to receiver@external.test
```

확인할 출력:

- STARTTLS 협상 이후 SMTP 명령이 수락되는지 확인한다.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `--server` | 대상 SMTP 서버 | 모든 테스트 |
| `--port` | 대상 포트 지정 | 25/465/587 비교 |
| `--from` | envelope 발신자 | 릴레이/스푸핑 영향 검증 |
| `--to` | envelope 수신자 | 내부/외부 수신자 조합 검증 |
| `--header` | 메일 헤더 추가 | 제목과 표시 발신자 지정 |
| `--body` | 메일 본문 지정 | 수신된 메시지 식별 |
| `--tls` | STARTTLS 사용 | submission 포트 또는 TLS 요구 서버 |
| `--auth`, `--auth-user`, `--auth-password` | SMTP 인증 | 인증 후 relay/submission 검증 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `250 OK` after `MAIL FROM`/`RCPT TO` | envelope 단계 수락 | `DATA` 수락과 실제 수신 여부 확인 |
| `Relay access denied` | 외부 릴레이 제한 | 내부 수신자/인증 후 submission 조건 확인 |
| `STARTTLS` 협상 성공 | TLS 전환 가능 | 인증 방식과 포트별 정책 확인 |
| `Message sent` 또는 `250 OK` after `DATA` | 서버가 메일을 큐에 넣음 | 수신함과 헤더 확인 |
| TLS 오류 | 포트와 STARTTLS/SMTPS 방식 불일치 | `--tls`, `--port`와 SMTPS/STARTTLS 구분 |
| 인증 실패 | 계정 또는 AUTH 방식 불일치 | `EHLO`의 지원 AUTH와 계정 형식 확인 |
| 수락됐지만 수신 안 됨 | 큐잉, 스팸 필터, 후단 차단 | 반송, 수신함 spam, 메일 헤더 확인 |

## 관련 공격기법

- [[25_SMTP#Open Relay 검증|SMTP Open Relay 검증]]

## 관련 서비스

- [[25_SMTP]]
