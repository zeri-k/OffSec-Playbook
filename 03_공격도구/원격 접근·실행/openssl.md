---
tags:
  - 기능/프로토콜접근
실행환경: ["Linux"]
필요조건: ["대상 TLS endpoint, 인증서 또는 암복호화할 파일"]
결과: ["TLS 연결 정보", "인증서", "암호화 파일"]
---

# openssl

## 도구 개요

`OpenSSL`은 TLS 연결과 인증서 체인을 점검하고 key·certificate·암복호화 파일을 다루는 범용 명령줄 도구다. `s_client`와 STARTTLS로 서비스 협상을 수동 확인하거나 `s_server`와 `enc`로 임시 TLS 전송 및 파일 보호 작업을 수행할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: 이 문서의 process·파일 보존 명령은 Linux shell 기준이다. Windows OpenSSL은 동일 subcommand를 사용할 수 있지만 process·경로 정리는 PowerShell 기준으로 별도 확인한다.
- 연결 확인 입력: TLS endpoint와 필요한 경우 SNI·STARTTLS 프로토콜
- 임시 TLS 전송 입력: listen 포트, 인증서, private key, 송수신 파일
- 암복호화 입력: 원본 파일, 출력 파일, 암호화 방식과 비밀번호

## 표준 사용법

```bash
openssl <subcommand> [options]
```

## 대표 예시

### TLS 인증서와 연결 정보 확인

```bash
openssl s_client -connect <TARGET>:443 -showcerts
```

### SMTP STARTTLS 확인

```bash
openssl s_client -starttls smtp -connect <TARGET>:587
```

SMTP 명령이 STARTTLS 이후에만 허용되는지, 인증서에서 메일 도메인 단서가 나오는지 확인한다.

### 임시 TLS 서버로 파일 전송

```bash
openssl s_server -quiet -accept <TLS_PORT> -cert '<CERT_PATH>' -key '<KEY_PATH>' < '<TLS_INPUT_FILE>' &
OPENSSL_SERVER_PID=$!
ps -p "$OPENSSL_SERVER_PID" -o pid=,args=
```

### TLS client로 파일 수신

```bash
test ! -e '<TLS_RECEIVED_FILE>'
openssl s_client -connect <TARGET>:<TLS_PORT> -quiet > '<TLS_RECEIVED_FILE>'
sha256sum '<TLS_RECEIVED_FILE>'
```

### 파일 암호화·복호화 왕복 확인

출력 경로가 기존 파일을 덮어쓰지 않는지 먼저 확인한다. passphrase는 명령행 인자로 남기지 않고 prompt에서 입력한다.

```bash
test ! -e '<ENCRYPTED_OUTPUT>' && test ! -e '<ROUNDTRIP_OUTPUT>'
openssl enc -aes-256-cbc -salt -pbkdf2 -in '<INPUT_FILE>' -out '<ENCRYPTED_OUTPUT>'
openssl enc -d -aes-256-cbc -pbkdf2 -in '<ENCRYPTED_OUTPUT>' -out '<ROUNDTRIP_OUTPUT>'
sha256sum '<INPUT_FILE>' '<ROUNDTRIP_OUTPUT>'
```

두 hash가 같아야 같은 passphrase·cipher·KDF 조건으로 원문을 복원한 것이다. 잘못된 passphrase나 손상된 입력은 복호화 오류 또는 hash 불일치로 구분한다. `openssl enc -list`에 선택한 cipher가 없으면 현재 provider·build에서 지원되지 않는 상태다. `enc`는 인증된 암호화 형식을 제공하지 않으므로 장기 저장 형식 선택에는 별도 요구사항을 적용한다.



## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `s_client` | TLS client 연결 | 인증서와 handshake 확인 |
| `-connect <HOST:PORT>` | 접속 endpoint 지정 | TLS 서비스 확인 |
| `-servername <NAME>` | TLS SNI 지정 | 가상 호스트 인증서 확인 |
| `-showcerts` | 서버가 제공한 인증서 chain 출력 | SAN, Issuer, chain 확인 |
| `-starttls <PROTO>` | 평문 연결 후 STARTTLS 협상 | SMTP, IMAP, POP3 등 |
| `s_server -accept <PORT>` | 임시 TLS listener 생성 | 인증서·키 기반 파일 전송 |
| `enc -aes-256-cbc` | AES-256-CBC 파일 암복호화 | 로컬 파일 보호 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 인증서 Subject/SAN/Issuer 출력 | TLS 인증서 기반 호스트명과 조직 단서 확보 | SAN 도메인을 vhost/DNS/웹 열거에 반영 |
| TLS handshake 성공 | TLS 서비스 접근 가능 | 프로토콜, cipher, 인증서 만료/불일치 확인 |
| verify error/self-signed | 신뢰되지 않는 인증서 | 내부 CA, 개발/관리 서비스 가능성 검토 |
| handshake failure | SNI, TLS 버전, 클라이언트 cipher 문제 | `-servername`, `-tls1_2`, 포트/프로토콜 확인 |
| 암호화·복호화 뒤 hash 일치 | 선택한 cipher·KDF·passphrase로 해당 파일 왕복 성공 | 목적에 따라 암호화 출력 보존 또는 정확한 경로 정리 |
| `bad decrypt` 또는 hash 불일치 | passphrase·cipher·KDF 불일치 또는 입력 손상 | 원본과 옵션을 확인하고 복원 성공으로 기록하지 않음 |

테스트 목적으로 만든 `<ENCRYPTED_OUTPUT>`·`<ROUNDTRIP_OUTPUT>`은 경로를 기록하고, 검증이 끝난 뒤 필요한 산출물만 남긴다. 정리할 때는 `rm -- '<ROUNDTRIP_OUTPUT>'`처럼 이번 작업에서 생성한 정확한 경로만 제거하고 기존 파일을 이름 패턴으로 삭제하지 않는다.

임시 TLS 전송에서는 server 시작 전 `<TLS_PORT>`의 기존 listener를 확인하고 `$OPENSSL_SERVER_PID`, 명령행과 수신 전 `<TLS_RECEIVED_FILE>`의 부재를 기록한다. 전송 확인 뒤 server host에서 기록한 PID가 같은 명령행인지 확인한 후 종료한다.

```bash
ps -p <OPENSSL_SERVER_PID> -o pid=,args=
kill <OPENSSL_SERVER_PID>
ps -p <OPENSSL_SERVER_PID> -o pid=,args=
ss -ltnp | grep -E '[:.]<TLS_PORT>[[:space:]]'
```

두 번째 `ps` 출력이 비고 listener가 작업 전 상태로 돌아와야 임시 server 정리가 끝난 것이다. `<TLS_RECEIVED_FILE>`은 의도한 수집 결과이므로 자동 삭제하지 않는다. 테스트용으로만 만든 경우에 한해 기록한 정확한 경로를 제거하며, 기존 인증서·private key·입력 파일은 삭제하지 않는다.

## 관련 공격기법

- [[웹 지문 확인과 공격면 분류]]
- [[SMTP 서비스#Open Relay 검증|SMTP Open Relay 검증]]
- [[SMTP 사용자 열거]]
- [[IMAP POP3 메일함 수집]]
- [[FTP 익명 접근과 파일 수집]]

## 참고 링크

- [OpenSSL `enc` 공식 문서](https://docs.openssl.org/3.6/man1/openssl-enc/)
