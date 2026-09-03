---
tags:
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows"]
필요조건: ["대상 TLS endpoint, 인증서 또는 암복호화할 파일"]
결과: ["TLS 연결 정보", "인증서", "암호화 파일"]
---

# openssl

## 도구 개요

`OpenSSL`은 TLS 연결과 인증서 체인을 점검하고 key·certificate·암복호화 파일을 다루는 범용 명령줄 도구다. `s_client`와 STARTTLS로 서비스 협상을 수동 확인하거나 `s_server`와 `enc`로 임시 TLS 전송 및 파일 보호 작업을 수행할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux, Windows
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
openssl s_server -quiet -accept 8443 -cert certificate.pem -key key.pem < file.txt
```

### TLS client로 파일 수신

```bash
openssl s_client -connect <TARGET>:8443 -quiet > file.txt
```

### 파일 암호화

```bash
openssl enc -aes-256-cbc -salt -in secret.txt -out secret.enc
```


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

## 관련 공격기법

- [[웹 지문 확인과 공격면 분류]]
- [[25_SMTP#Open Relay 검증|SMTP Open Relay 검증]]
- [[SMTP 사용자 열거]]
- [[IMAP POP3 메일함 수집]]
- [[FTP 익명 접근과 파일 수집]]
