---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/자격증명수집
  - 기능/인증검증
실행환경: ["Linux"]
필요조건: ["인증 가능한 PFX와 비밀번호"]
결과: ["ccache", "ticket"]
---

# gettgtpkinit

## 도구 개요

`gettgtpkinit.py`는 PFX 파일이나 Base64 certificate로 Public Key Cryptography for Initial Authentication(PKINIT)을 수행해 AD 계정의 Ticket-Granting Ticket(TGT) ccache를 만든다. 인증서 기반 AD 인증을 Kerberos ticket으로 전환할 때 사용하는 PKINITtools 도구다.

## 필요한 입력과 실행 환경

- 실행 위치: PKINITtools가 설치되고 DC의 Kerberos에 접근 가능한 Linux 호스트
- 필요한 입력: PFX 파일과 비밀번호, `<DOMAIN>/<USER>`, DC 주소 또는 FQDN, 출력 ccache 경로
- 환경 조건: 공격 호스트와 DC의 시간을 동기화하고 이후 사용할 `KRB5CCNAME` 경로를 정한다.


## 표준 사용법

```bash
python3 gettgtpkinit.py -cert-pfx <ACCOUNT>.pfx -dc-ip <DC_IP> '<DOMAIN>/<ACCOUNT>' /tmp/account.ccache
```

## 대표 예시

### PFX로 TGT ccache 생성

```bash
python3 gettgtpkinit.py -cert-pfx <PFX_FILE> -pfx-pass '<PFX_PASS>' -dc-ip <TARGET> <DOMAIN>/<USER> <CCACHE_FILE>
export KRB5CCNAME=<CCACHE_FILE>
klist
```

### base64 certificate로 TGT ccache 생성

```bash
python3 gettgtpkinit.py '<DOMAIN>/<ACCOUNT>' -pfx-base64 '<BASE64_CERTIFICATE>' <CCACHE_FILE>
export KRB5CCNAME=<CCACHE_FILE>
klist
```

확인할 출력:

- `AS-REP encryption key`, `Saved TGT to file`.
- `klist`의 `Default principal`에 표시된 계정이 certificate 주체와 일치한다.

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `-cert-pfx` | 인증에 사용할 PFX 파일 |
| `-pfx-pass` | PFX password |
| `-pfx-base64` | base64 인코딩 certificate를 직접 입력 |
| `-dc-ip` | 도메인 컨트롤러 IP 지정 |
| 출력 경로 | 생성할 ccache 파일 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Saved TGT to file` | ccache 생성 성공 | `KRB5CCNAME` 지정 후 `klist` 확인 |
| `AS-REP encryption key` | PKINIT 응답 복호화 key 출력 | PKINITtools의 후속 U2U hash 복구에 사용할 때만 보호하여 전달 |
| `Default principal`에 티켓 계정 표시 | certificate 주체로 인증 성공 | SMB/LDAP/WinRM 접근 검증 |
| PKINIT 오류 | DC/certificate 조건 문제 | DC 인증서, EKU, realm, 시간 확인 |
| KDC/DNS 오류 | 도메인/이름 해석 문제 | `-dc-ip`, FQDN, `/etc/hosts`, clock skew 확인 |
| PFX password 오류 | 잘못된 password 또는 파일 | pywhisker/certipy 출력 재확인 |
| ccache 사용 실패 | 환경 변수 또는 realm 문제 | `KRB5CCNAME`, `klist`, FQDN 사용 |

## 관련 공격기법

- [[AD CS ESC8 NTLM Relay]]
- [[Shadow Credentials]]
