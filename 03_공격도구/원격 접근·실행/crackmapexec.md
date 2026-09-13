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

CrackMapExec은 여러 Windows 서비스의 인증·열거·원격 작업을 자동화한 도구다. 현재 흐름에서는 [[netexec]]을 우선하며, 이 문서는 구형 환경이나 기존 `crackmapexec smb` 명령을 재현할 때 필요한 SMB 인증·열거·관리자급 작업의 대표 경계만 다룬다. 비-SMB 프로토콜은 설치된 버전의 도움말을 확인하고 NetExec 문서로 전환한다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 SMB에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 호스트/대역, 도메인·사용자·비밀번호 또는 NTLM hash
- spraying 입력: 사용자·비밀번호 목록과 사전에 확인한 계정 잠금 정책
- `<TARGET>`은 SMB listener의 IP·CIDR 또는 Linux 실행 host의 한 줄 한 대상 목록 파일이고, `<DC>`는 domain controller 역할을 가진 SMB host다. `<DOMAIN>`은 AD domain, `<USER>`/`<PASSWORD>` 또는 목록 파일은 같은 account namespace의 credential 입력이며, 인증 성공은 command execution·LSA 접근을 뜻하지 않는다.
- 기본 선택은 [[netexec]]이며, 이 문서는 CrackMapExec 문법을 그대로 재현해야 할 때만 사용한다.

## 표준 사용법

`<target>`은 SMB listener의 IP·CIDR 또는 Linux 실행 host의 대상 목록 파일이고, `<user_or_list>`·`<password_or_list>`는 같은 account namespace의 한 사용자/비밀번호 또는 한 줄 목록이다. 예시의 `<DC>`는 domain controller SMB host, `<DOMAIN>`은 AD DNS/NetBIOS domain이며 blank credential은 anonymous session 확인에만 쓴다.

```bash
crackmapexec smb <target> -u <user_or_list> -p <password_or_list>
```

## 대표 예시

### AD 도메인 SMB Password Spraying

```bash
crackmapexec smb <DC> -d <DOMAIN> -u '<SPRAY_USER_LIST>' -p '<PASSWORD>'
```

이 예시는 도메인 계정에 단일 비밀번호를 시도한다. `--local-auth`는 대상 호스트의 로컬 계정 데이터베이스로 인증할 때만 사용하며, 로컬 관리자 비밀번호 재사용 검사는 [[원격 비밀번호 공격]]의 별도 분기다. AD Password Spraying에 `--local-auth`를 붙이면 의도한 도메인 계정이 아니라 로컬 계정을 검사하게 된다.

### SMB share 권한 확인

`<TARGET>`은 앞 단계와 같은 SMB host 또는 목록 항목이다. 빈 `-u`·`-p`는 anonymous session 시도이며 share 표시가 file READ·WRITE 또는 command execution을 뜻하지 않는다.

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

### 관리자급 원격 명령과 LSA 수집 경계

SMB 인증 결과에 관리자 표시가 나온 단일 대상에서만 영향이 작은 식별 명령을 먼저 실행한다.

```bash
crackmapexec smb <TARGET> -u '<USER>' -p '<PASSWORD>' -x whoami
crackmapexec smb <TARGET> -u '<USER>' -p '<PASSWORD>' --lsa
```

- `-x whoami`의 대상 계정 출력이 있어야 원격 명령 실행 성공이다. 인증 성공이나 관리자 표시만으로 명령 실행을 확정하지 않는다.
- `--lsa`는 관리자급 원격 작업과 registry hive 접근이 가능한 경우에만 진행한다. `Dumping LSA Secrets` 뒤 실제 secret 유형을 확인하고, 출력은 Vault가 아닌 민감 자료 저장 위치에서 다룬다.


## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `smb` | 이 문서에서 재현하는 프로토콜. 다른 프로토콜은 현재 설치 버전의 `--help`와 [[netexec]] 확인 |
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
| `-x whoami`의 대상 계정 출력 | SMB를 통한 원격 명령 실행 성공 | 실행 계정과 후속 작업 영향을 별도 확인 |
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

## 참고 링크

- [CrackMapExec 공식 Wiki — SMB domain·local authentication](https://github.com/byt3bl33d3r/CrackMapExec/wiki/SMB-Command-Reference)
