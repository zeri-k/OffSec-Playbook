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
- `<SMTP_HOST>`는 SMTP listener의 IP/FQDN, `<TEST_ID>`는 제목·header·body에 같은 가상 식별자로 넣는 값이다. envelope `--from`/`--to`와 message header는 별도이며 `250` queue 수락은 mailbox 수신을 단독으로 보장하지 않는다.

## 표준 사용법

```bash
swaks --from <FROM> --to <TO> --server <TARGET>
```

발신자, 수신자와 서버를 명시하고 필요하면 제목과 본문을 추가한다. 각 SMTP 단계의 응답과 최종 수신 여부를 함께 확인한다.

## 대표 예시

### 오픈 릴레이 envelope 후보 확인

```bash
swaks --from relay-test@external.test --to receiver@external.test --server <TARGET> --quit-after RCPT
```

확인할 출력:

- `MAIL FROM`, `RCPT TO` 단계가 `250` 계열로 수락되는지 확인한다.
- `--quit-after RCPT`는 DATA를 보내지 않으므로 실제 queue 수락·외부 전달은 아직 미확인이다.

### 전체 전달 확인

통제하는 외부 수신함에 같은 `<TEST_ID>`가 없음을 확인한 뒤 실행한다.

```bash
swaks --from relay-test@external.test --to receiver@external.test --header 'Subject: Relay Test <TEST_ID>' --header 'X-Assessment-ID: <TEST_ID>' --body 'Relay delivery check <TEST_ID>' --server <TARGET>
```

확인할 출력:

- DATA 전 `354`, 본문 전송 뒤 최종 `250`은 서버의 queue 수락을 뜻한다.
- 수신함의 `<TEST_ID>` 메시지와 `Received` header를 확인해야 외부 전달 성공이다.

### SMTP 포트 지정

```bash
swaks --from relay-test@external.test --to receiver@external.test --server <SMTP_HOST> --port 587 --tls --quit-after RCPT
```

확인할 출력:

- 25/465/587 중 어떤 포트에서 relay 또는 submission 정책이 다른지 확인한다.

### STARTTLS 확인

```bash
swaks --server <SMTP_HOST> --port 587 --tls --from relay-test@external.test --to receiver@external.test --quit-after RCPT
```

확인할 출력:

- STARTTLS 협상 이후 SMTP 명령이 수락되는지 확인한다.

465/TCP처럼 연결 직후 TLS를 협상하는 endpoint는 `--tls`가 아니라 `--tls-on-connect`를 사용한다.

```bash
swaks --server <SMTP_HOST> --port 465 --tls-on-connect --from relay-test@external.test --to receiver@external.test --quit-after RCPT
```

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
| `--tls-on-connect` | 연결 직후 TLS 협상 | 465/TCP 같은 implicit TLS endpoint |
| `--quit-after RCPT` | RCPT 응답 뒤 DATA 없이 종료 | 실제 메시지를 생성하기 전 relay envelope 후보 확인 |
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

전체 전달을 수행하면 수신함 메시지, SMTP queue 처리와 서버·보안 로그가 생긴다. 제목과 `X-Assessment-ID`로 이번 작업 메시지를 식별하고, 권한이 있는 수신함에서 정확히 일치하는 메시지만 삭제한다. queue·전달 로그와 탐지 이벤트는 외부 client에서 되돌릴 수 없으므로 메시지 삭제와 원상복구를 같은 의미로 기록하지 않는다. 인증 password는 명령행 옵션에 직접 넣지 말고 현재 Swaks 버전의 prompt·보호된 입력 방식을 확인한다.

## 관련 공격기법

- [[SMTP 서비스#Open Relay 검증|SMTP Open Relay 검증]]

## 관련 서비스

- [[SMTP 서비스]]

## 참고 링크

- [Swaks 공식 저장소와 내장 문서](https://github.com/jetmore/swaks)
- [RFC 5321 — SMTP envelope와 DATA 응답](https://www.rfc-editor.org/rfc/rfc5321.html)
- [RFC 8314 — SMTP Submission의 STARTTLS와 Implicit TLS](https://www.rfc-editor.org/rfc/rfc8314.html)
