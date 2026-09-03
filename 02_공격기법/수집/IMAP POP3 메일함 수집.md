---
tags:
  - 서비스/imap
  - 서비스/pop3
시작조건: ["IMAP 또는 POP3 서비스 접근 가능", "유효한 메일 계정 자격 증명"]
필요조건: ["IMAP/POP3 접근 가능", "유효한 메일 계정 credential"]
결과: ["정보", "파일", "자격 증명"]
---

# IMAP POP3 메일함 수집

## 한 줄 판단

현재 명령 실행 위치에서 대상 IMAP 또는 POP3 포트에 연결할 수 있고 유효한 메일 계정이 있다면, 그 계정의 메일함에서 내부 URL·초기 비밀번호·토큰·첨부 파일을 찾는다. 메일 계정 로그인 성공은 다른 서비스의 인증 성공을 의미하지 않는다.

## 사용할 때

- IMAP/POP3 포트가 열려 있고 메일 계정 credential 또는 후보가 있을 때.
- [[SMTP 사용자 열거]], [[POP3 USER 사용자 열거]], 웹 유출, SMB 공유에서 이메일 계정이 확보되었을 때.
- 메일함에 VPN, 웹 포털, 임시 비밀번호, 티켓, 첨부가 있을 가능성이 있을 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 서비스 접근 | `openssl` 또는 `curl` | IMAP/POP3 응답 |
| 계정 credential | 수집/추측/스프레이 결과 | 로그인 성공 |
| 메일 수집 범위 | 폴더/메시지 목록 | 본문 또는 첨부 조회 가능 |

## 실행

### 프로토콜과 전송 방식 선택

| 열린 서비스 | 접속 방식 | 필요한 입력 | 성공 판단 |
|---|---|---|---|
| IMAPS 993/TCP | 처음부터 TLS로 연결 | 메일 `<USER>`와 `<PASSWORD>` | 인증서·IMAP capability 확인 뒤 폴더·메시지 조회 |
| IMAP 143/TCP | 평문 연결 뒤 STARTTLS | 메일 `<USER>`와 `<PASSWORD>` | STARTTLS 협상 뒤 IMAP 응답과 로그인 성공 |
| POP3S 995/TCP | 처음부터 TLS로 연결 | 메일 `<USER>`와 `<PASSWORD>` | 인증서·POP3 capability 확인 뒤 메시지 번호·본문 조회 |
| POP3 110/TCP | 평문 또는 STARTTLS | 메일 `<USER>`와 `<PASSWORD>` | `+OK` 로그인 뒤 `LIST`, `RETR <ID>` 성공 |

IMAP은 폴더와 서버 보관 메시지를 다루고 POP3는 메시지 번호를 중심으로 조회한다. 993·995의 implicit TLS와 143·110의 STARTTLS를 같은 연결 방식으로 취급하지 않는다.

1. TLS/STARTTLS와 capability를 확인한다.
2. credential을 수동 또는 curl로 검증한다.
3. IMAP은 폴더와 메시지, POP3는 메시지 번호를 조회한다.
4. 비밀번호, 토큰, 내부 URL, 파일명을 추출해 후속 서비스로 넘긴다.

### IMAPS 993/TCP

```bash
openssl s_client -connect <TARGET>:993
```

확인할 출력:

- TLS 인증서와 IMAP banner·capability.
- TLS 협상이 실패하면 993번 서비스 식별, SNI·인증서 문제와 서버의 지원 프로토콜을 확인한다.

### IMAP 143/TCP STARTTLS

```bash
openssl s_client -starttls imap -connect <TARGET>:143
```

확인할 출력:

- STARTTLS 협상 뒤 IMAP banner·capability.

### POP3S 995/TCP

```bash
openssl s_client -connect <TARGET>:995
```

확인할 출력:

- TLS 인증서와 `+OK` POP3 banner·capability.
- TLS 연결은 성공했지만 로그인하지 않았다면 서비스 도달성만 확인된 상태다.

### POP3 110/TCP STARTTLS 확인

```bash
openssl s_client -starttls pop3 -connect <TARGET>:110
```

확인할 출력:

- STARTTLS 협상 뒤 `+OK` POP3 응답. 서버가 STARTTLS를 제공하지 않으면 평문 수동 접근의 노출 위험과 정책을 별도로 기록한다.

### IMAP 접근

```bash
curl -k 'imaps://<TARGET>' --user '<USER>:<PASSWORD>' -v
curl -k 'imaps://<TARGET>/INBOX' --user '<USER>:<PASSWORD>'
```

확인할 출력:

- 로그인 성공, 폴더 목록, 메시지 목록.

### POP3 수동 접근

실행 위치: 대상 110/TCP에 연결할 수 있는 명령 실행 호스트. 이 명령은 평문 POP3이므로 통제된 실습·검증 범위에서만 사용한다.

```bash
telnet <TARGET> 110
USER <USER>
PASS <PASSWORD>
LIST
RETR <ID>
```

확인할 출력:

- `+OK`, 메시지 수, 본문/첨부 단서.
- `-ERR`가 `PASS` 뒤에 나오면 서비스 연결과 계정 인증 실패를 구분하고 사용자명 형식, 비밀번호와 계정 잠금을 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| TLS 연결과 IMAP·POP3 capability가 보임 | 메일 서비스와 암호화 방식을 식별함 | 메일 인증 경로 | 올바른 계정 형식으로 로그인 |
| IMAP 로그인 성공 또는 POP3 `+OK` 후 목록이 나옴 | 유효한 메일 계정으로 접근함 | 메일함 접근 | 필요한 폴더와 메시지만 조회 |
| 본문·첨부를 내려받고 크기·hash가 확인됨 | 메일 데이터를 온전히 수집함 | 메시지 또는 첨부 파일 | 내부 URL·토큰·자격 증명 단서 분석 |
| 다른 서비스에서 수집한 credential이 실제 인증됨 | 메일 단서가 후속 접근으로 이어짐 | 재사용 가능한 자격 증명 | 해당 서비스 노트에서 권한 확인 |
| 로그인 실패 | username 형식, 도메인 또는 비밀번호가 맞지 않음 | 서비스 접근만 유지 | 전체 이메일 주소와 짧은 username 형식 비교 |
| TLS handshake가 실패함 | 포트·STARTTLS·인증서 처리 방식이 다름 | 암호화 경로 미확정 | `openssl`로 protocol과 포트 확인 |
| 폴더·목록은 보이지만 본문이 거부됨 | mailbox ACL 또는 protocol 제한 | 메타데이터만 접근 | 다른 폴더와 IMAP·POP3 권한 차이 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| TLS·capability | 배너, protocol capability, 인증서 이름 | 서비스 식별과 계정 인증을 구분 |
| 인증 | IMAP 성공 응답 또는 POP3 `+OK` | 유효한 메일 identity 확인 |
| 메일 권한 | 폴더 목록, message list, `RETR`·fetch 결과 | 목록 권한과 본문·첨부 READ를 구분 |
| 첨부 무결성 | 파일 크기, MIME 정보, `sha256sum` | 손상된 디코딩과 정상 수집을 구분 |
| 후속 credential | 별도 서비스의 실제 인증 결과 | 메일에 문자열이 있다는 사실과 credential 유효성을 구분 |

## 후속 공격 연결

- 새 credential: [[원격 비밀번호 공격]]
- 첨부 암호화 파일: [[보호된 파일 및 아카이브 크래킹]]
- 내부 URL: [[웹 정찰과 경로 열거]]
- POP3 사용자 후보: [[POP3 USER 사용자 열거]]

## 관련 서비스

- [[110_143_993_995_IMAP_POP3]]

## 관련 상태 라우터

- 메일함에서 재사용 가능한 credential을 확인했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[curl]]
- [[openssl]]
- [[telnet]]
- [[hydra]]
