---
tags:
  - 서비스/smtp
대표포트:
  - "T:25"
  - "T:465"
  - "T:587"
서비스:
  - SMTP
---

# 25_SMTP

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Simple Mail Transfer Protocol(SMTP) 포트에 연결할 수 있고, 메일 계정이나 송신 권한은 아직 확인하지 않은 상태에서 시작한다. `EHLO` 기능, 사용자별 응답 차이, envelope 수신자 수락, relay와 실제 외부 전달을 분리한다.

성공하면 유효 사용자 이름 후보, 암호화·인증 방식 또는 식별 가능한 테스트 메일의 전달 결과를 얻는다. `VRFY`가 막히면 `RCPT TO` 응답 차이를 확인하고, `RCPT TO` 수락 후 외부 메일이 도착하지 않으면 queue·후단 정책·발신자 제한을 확인한다. 배너·인증서의 도메인과 호스트명은 대상명 단서일 뿐이며 포트 오픈이나 envelope 수락만으로 오픈 relay를 단정하지 않는다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | SMTP greeting과 지원 명령 | `nc -nv <TARGET> 25` 후 `EHLO` | `220`, `VRFY`, `EXPN`, `STARTTLS`, 인증 방식과 노출된 도메인을 확인한다. |
| 2 | STARTTLS와 인증서 | `openssl s_client -starttls smtp -connect <TARGET>:587` | TLS 협상, CN/SAN과 Submission 구성을 확인한다. |
| 3 | 사용자별 응답 차이 | `smtp-user-enum -M VRFY -U users.txt -t <TARGET>` | valid/invalid 응답이 재현되는지 확인하고, 차이가 없으면 `RCPT TO` 방식을 검토한다. |
| 4 | 오픈 릴레이 후보 | `nmap --script smtp-open-relay -p25 <TARGET>` | 스크립트 결과를 후보로만 사용하고 실제 외부 수신 여부를 별도 확인한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| `EHLO`/`VRFY`/`EXPN`/`RCPT TO` 응답 차이 | [[SMTP 사용자 열거]] | `smtp-user-enum`, `telnet`, `netcat` | 유효 이메일·사용자명 후보 |
| 릴레이 허용 단서 | 이 문서의 Open Relay 검증 | `nmap`, `swaks`, `telnet` | 식별 가능한 테스트 메일의 실제 외부 전달 여부 |
| STARTTLS 지원과 인증서 이름 | [[SMTP 사용자 열거]] | `openssl` | 암호화 전환 조건과 도메인·호스트명 단서 |
| 사용자 이름·비밀번호 후보 확보 | [[원격 비밀번호 공격]] | `hydra`, `smtp-user-enum` | 잠금·스팸 방지 정책을 반영해 검증한 메일 서비스 계정 |
| 배너·인증서에 내부 도메인 또는 호스트명 노출 | [[DNS 열거와 Zone Transfer]] | `dig` | DNS·웹 vhost·메일 인프라 대상명 확장 |

### Open Relay 검증

자동 스크립트 결과, SMTP envelope 수락과 실제 외부 전달을 서로 다른 단계로 확인한다.

```bash
nmap -p25 -Pn --script smtp-open-relay <TARGET>
swaks --from relay-test@external.test --to receiver@external.test --header 'Subject: Relay Test' --body 'Relay test' --server <TARGET>
```

수동으로 확인할 때는 인증하지 않은 세션에서 서버 도메인이 아닌 발신자와 수신자를 사용한다.

```text
EHLO relay-test.local
MAIL FROM:<relay-test@external.test>
RCPT TO:<receiver@external.test>
QUIT
```

- `RCPT TO`와 `DATA`의 `250` 계열 응답은 SMTP 서버가 요청을 수락한 결과다.
- 테스트 수신함의 실제 메일과 `Received` 헤더까지 확인해야 외부 전달로 확정한다.
- `Relay access denied`, 특정 도메인만 허용, 인증 후에만 허용하는 경우를 각각 구분한다.
- `RCPT TO` 수락 후 메일이 도착하지 않으면 queue, 스팸 필터, 후단 정책과 반송을 확인한다.

## 서비스 고유 주의 사항

- 465는 SMTPS, 587은 Submission/STARTTLS일 수 있으므로 연결 방식을 구분한다.
- `VRFY`가 막혀도 `RCPT TO`의 사용자별 응답은 다를 수 있다.
- 과도한 요청은 계정·스팸 방지 정책에 영향을 주므로 낮은 빈도로 검증한다.
- 오픈 릴레이는 SMTP 수락과 외부 수신을 분리해 확인한다.
