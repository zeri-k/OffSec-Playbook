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

### AD 도메인 SMB Password Spraying

잠금 정책과 공격 대상 사용자 목록을 확인한 뒤 단일 비밀번호를 도메인 계정에 한 번씩 시도한다.

```shell
nxc smb <DC> -d <DOMAIN> -u '<SPRAY_USER_LIST>' -p '<PASSWORD>'
```

NetExec은 기본적으로 첫 유효 로그인에서 중단한다. [[내부 AD Password Spraying]]의 중지 조건을 유지하기 위해 이 절차에서는 `--continue-on-success`를 사용하지 않는다. `--jitter <SECONDS>`는 요청 간격을 조절할 뿐 계정별 잠금 정책·현재 실패 횟수를 확인하거나 잠금을 예방하지 않는다.

### NTLM hash 인증

```shell
nxc smb <TARGET> -u <USER> -H <NTLM_HASH>
```

인증 성공은 SMB 서비스 접근만 뜻한다. Pass the Hash의 대상 조건과 실제 권한 판정은 [[Pass the Hash]]에서 수행한다.

### SMB 공유 열거

```shell
nxc smb <TARGET> -u <USER> -p '<PASSWORD>' --shares
```

공유 이름과 도구가 표시한 권한은 후보이며, 파일 READ·WRITE는 [[SMB 인증 공유 파일 수집]] 또는 [[SMB 쓰기 가능한 공유 검증]]에서 확인한다.

### SMB 공유의 파일명 pattern 검색

```shell
nxc smb <TARGET> -u <USER> -p '<PASSWORD>' --spider '<SHARE>' --pattern '<FILENAME_PATTERN>'
```

현재 공식 `--spider`·`--pattern` 예시는 선택한 공유에서 파일명 pattern을 찾는 절차다. 경로 출력은 본문에 credential이 있거나 파일을 다운로드했다는 증거가 아니다. content 검색과 로컬 loot가 필요하면 [[SMB 공유 자격증명 수집]]에서 별도 도구를 선택한다.

### WinRM 인증 확인

```shell
nxc winrm <TARGET> -u <USER> -p '<PASSWORD>'
```

성공은 WinRM 인증 가능성을 뜻하며 실제 PowerShell 세션은 [[WinRM 원격 PowerShell 세션]]에서 별도로 연다.

### LDAP 인증과 디렉터리 응답 확인

```shell
nxc ldap <DC_FQDN> -u <USER> -p '<PASSWORD>'
```

LDAP 인증 성공은 해당 계정으로 디렉터리 서비스에 bind할 수 있다는 뜻이다. 조회 가능한 객체 범위, 복제 권한과 원격 로그온 권한은 별도 결과다.

### 관리자급 SMB 원격 명령 확인

프로토콜 도움말에서 `-x` 지원을 확인하고, 관리자 표시가 나온 한 호스트에서 영향이 작은 식별 명령으로 실제 실행 경계를 검증한다.

```shell
nxc smb <TARGET> -u <USER> -p '<PASSWORD>' -x whoami
```

도구의 관리자 표시는 실행 후보이고, 대상 계정명이 포함된 명령 출력이 반환돼야 원격 명령 실행 성공이다. 인증 성공 뒤 access denied가 나면 관리자급 실행 조건은 충족하지 않은 것이다.

## 주요 옵션

| 옵션 | 의미 |
| --- | --- |
| `-d <domain>` | 인증할 AD 도메인 지정 |
| `-u <user>` | 사용자 또는 사용자 목록 |
| `-p <password>` | 비밀번호 또는 비밀번호 목록 |
| `-H <hash>` | NTLM 해시 인증 |
| `--local-auth` | 도메인이 아닌 대상 로컬 계정으로 인증 |
| `--continue-on-success` | 성공 후에도 다음 조합 계속 시도 |
| `--jitter <SECONDS>` | 호스트별 인증 요청 사이의 지연. 고정 초 또는 범위 사용 |
| `--shares` | SMB 공유 열거 |
| `--spider`, `--pattern` | 특정 SMB 공유에서 파일명 pattern 후보 선별. content·download 성공과 분리 |
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
| LDAP bind 성공 | 해당 계정의 LDAP 인증 성공 | 디렉터리 조회 결과와 객체별 권한을 별도 확인 |
| `-x whoami`의 대상 계정 출력 | SMB 관리자급 원격 명령 실행 성공 | 명령 실행 계정과 필요한 후속 작업의 영향 확인 |
| share/user/policy 출력 | 열거 성공 | 서비스 문서의 후속 공격 후보와 연결 |
| `Dumping LSA Secrets`와 `<DOMAIN>\<USER>:<PLAINTEXT_PASSWORD>` | LSA 추출과 계정명이 연결된 평문 secret 복구 | 새 계정의 도메인·서비스·객체 권한 확인 |
| `$DCC2$...` | 캐시된 도메인 로그온 hash 추출 | 오프라인 검증 대상으로 분리 |
| lockout/auth/connection 오류 | 계정 정책, credential, 네트워크 문제 | 도메인 형식, 프로토콜, 포트, 시도 빈도 확인 |

## 로컬 workspace database

NetExec은 사용하거나 수집한 credential과 호스트 정보를 선택된 workspace의 protocol database에 자동 저장한다. spraying 전에 `nxcdb`에서 현재 workspace를 확인하고 기존 작업과 분리된 새 workspace를 선택한다.

```text
nxcdb
nxcdb (default) > workspace list
nxcdb (default) > workspace create <WORKSPACE>
```

공식 문서에는 workspace 생성·전환·목록은 있지만 삭제 명령 계약은 제시되어 있지 않다. 공유 `default` database를 통째로 지우지 말고, 작업 전용 workspace에 남은 credential을 민감한 로컬 잔여 자료로 기록한다. 설치 버전의 `nxcdb help`에서 정확한 제거 기능을 확인하지 못했다면 정리 완료라고 표현하지 않는다.

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
- [[SMB 공유 자격증명 수집]]

## 참고 링크

- [NetExec protocol 사용법](https://www.netexec.wiki/getting-started/selecting-and-using-a-protocol)
- [NetExec credential와 Password Spraying 옵션](https://www.netexec.wiki/getting-started/using-credentials)
- [NetExec SMB Password Spraying](https://www.netexec.wiki/smb-protocol/password-spraying)
- [NetExec workspace database](https://www.netexec.wiki/getting-started/database-general-usage)
- [NetExec SMB Spidering Shares](https://www.netexec.wiki/smb-protocol/spidering-shares)
