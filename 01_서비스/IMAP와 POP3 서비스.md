---
tags:
  - 서비스/imap
  - 서비스/pop3
대표포트:
  - "T:110"
  - "T:143"
  - "T:993"
  - "T:995"
서비스:
  - IMAP
  - POP3
---

# IMAP와 POP3 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Internet Message Access Protocol(IMAP) 또는 Post Office Protocol version 3(POP3) 포트에 연결할 수 있고, 아직 유효한 메일 계정이나 메일 읽기 권한은 확인하지 않은 상태에서 시작한다. 평문·Transport Layer Security(TLS) 연결 방식과 인증 방법을 먼저 맞춘 뒤 사용자 존재 단서, 로그인 성공, 메일 목록 조회와 본문 읽기를 단계별로 구분한다.

성공하면 해당 계정이 읽을 수 있는 메일과 도메인·내부 URL·비밀번호·토큰 후보를 얻는다. TLS 협상 실패는 포트·STARTTLS 방식을, 인증 실패는 사용자 형식·인증 방식·잠금 정책을, 로그인 후 조회 실패는 메일함 권한을 각각 다시 확인한다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | IMAPS와 POP3S TLS 응답 | `openssl s_client -connect <TARGET>:imaps`, `openssl s_client -connect <TARGET>:pop3s` | 인증서, 배너와 TLS 포트별 프로토콜을 확인한다. |
| 2 | IMAP 인증과 capability | `curl 'imaps://<MAIL_HOST>' --user '<USER>' -v` | 인증서 검증 뒤 password prompt로 로그인하고 메일함 응답을 확인한다. 인증서 오류와 인증 실패를 분리한다. |
| 3 | POP3 명령과 사용자 응답 | `telnet <TARGET> 110` 후 `USER`, `PASS`, `LIST` | `+OK`와 사용자별 응답 차이, 로그인·목록 조회를 구분한다. |
| 4 | 메일함 읽기 범위 | IMAP `LIST`, `EXAMINE`, `UID FETCH <UID> (FLAGS BODY.PEEK[])`; POP3 `LIST`, `RETR <ID>` | 폴더·메시지 목록과 실제 본문 읽기 가능성을 확인한다. IMAP은 기존 `\Seen`을 기록하고 읽기 전용 명령을 사용한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| CAPABILITY 또는 STARTTLS 응답 | [[IMAP POP3 메일함 수집]] | `telnet`, `curl`, `openssl` | 사용할 연결·인증 방식 확정 |
| POP3 `USER` 응답 차이 | [[POP3 USER 사용자 열거]] | `telnet`, `openssl` | 계정 존재 여부 단서 |
| 계정 후보 | [[원격 비밀번호 공격]] | `curl`, `hydra` | 유효 메일 계정 또는 정책에 따른 거부 |
| 로그인과 `LIST` 성공 | [[IMAP POP3 메일함 수집]] | `curl`, 메일 클라이언트 | 해당 계정의 폴더·메시지 목록 접근 |
| `FETCH` 또는 `RETR` 성공, 메일 내 비밀정보 | [[IMAP POP3 메일함 수집]] | `curl`, 검색 도구 | 비밀번호·토큰·내부 URL·파일 경로 등 본문 정보 확보 |

## 서비스 고유 주의 사항

- 993/995는 TLS 포트이므로 일반 telnet 대신 `openssl` 등 TLS 가능한 클라이언트를 사용한다.
- IMAP 명령에는 태그가 필요하므로 `1 LOGIN`, `1 LIST "" *`처럼 보낸다.
- IMAP 143의 프로토콜 명령은 `STARTTLS`, POP3 110의 명령은 `STLS`다. `openssl s_client -starttls imap|pop3`는 이 차이를 client 옵션으로 처리한다.
- POP3는 IMAP보다 메일함 구조가 제한적이며 인증 실패를 계정 부재로 단정하지 않는다.
- `-ERR` 또는 IMAP `NO`/`BAD`는 계정 부재, 잘못된 비밀번호, 비허용 인증 방식과 명령 문법을 구분해 해석한다.
- 기본 확인은 목록과 본문 읽기까지만 한다. IMAP `FETCH BODY[]`는 `\Seen`을 설정할 수 있으므로 `EXAMINE`과 `BODY.PEEK[]`를 우선한다. POP3 `DELE`는 사용하지 않으며, 같은 세션에서 잘못 표시했다면 `QUIT` 전에 `RSET`으로 삭제 표시를 취소한다.

## 참고 링크

- [RFC 9051 — IMAP4rev2의 EXAMINE, FLAGS와 BODY.PEEK](https://www.rfc-editor.org/rfc/rfc9051.html)
- [RFC 1939 — POP3의 RETR, DELE, RSET과 UPDATE 상태](https://www.rfc-editor.org/rfc/rfc1939.html)
- [RFC 8314 — IMAP·POP3의 Implicit TLS와 STARTTLS](https://www.rfc-editor.org/rfc/rfc8314.html)
