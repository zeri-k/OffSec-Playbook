---
tags:
  - 환경/windows
  - 서비스/mssql
  - 기능/프로토콜접근
실행환경: ["Linux"]
필요조건: ["유효한 MSSQL 계정"]
결과: ["정보", "데이터", "명령 실행"]
---

# impacket-mssqlclient

## 도구 개요

`impacket-mssqlclient`는 Linux에서 SQL 또는 Windows 인증으로 Microsoft SQL Server에 접속해 query를 실행하는 TDS 클라이언트다. 현재 login과 서버 역할을 확인하고 impersonation, linked server, `xp_cmdshell` 같은 MSSQL 기능을 대화형으로 검증할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: MSSQL/TDS 서비스에 접근 가능한 Linux 호스트
- 필요한 입력: SQL 또는 Windows 인증 정보, 대상 주소/instance와 database
- Windows 인증 조건: 도메인 형식과 `-windows-auth`, Kerberos 사용 시 ticket와 SPN/FQDN을 맞춘다.
- SQL 인증은 `<USER>:<PASSWORD>@<TARGET>`, Windows 인증은 `<DOMAIN>/<USER>@<TARGET>`처럼 구분한다. `<TARGET>`은 TDS endpoint/instance이며 Kerberos는 IP가 아닌 SPN과 일치하는 FQDN 및 앞 단계 ccache를 사용한다.


## 표준 사용법

`<TARGET>`은 TDS listener/instance, `<DOMAIN>/<USER>`·`<PASSWORD>` 또는 `LM:NT`는 SQL/Windows authentication 입력이다. Kerberos block은 FQDN·SPN·ccache를 함께 사용하며 prompt·query success, `xp_cmdshell` state, OS command execution은 별도 출력으로 확인한다.

```bash
impacket-mssqlclient <domain>/<user>:<password>@<target> [options]
```

설치 버전과 옵션은 `impacket-mssqlclient -h`로 확인한다. 현 upstream은 password를 target 문자열에서 생략하면 prompt를 열므로, shell history에 남지 않게 다음처럼 사용할 수 있다.

## 대표 예시

### Windows 인증으로 MSSQL 접속

```bash
impacket-mssqlclient '<DOMAIN>/<USER>@<TARGET>' -windows-auth
```

### 도메인 계정으로 MSSQL 접속

```bash
impacket-mssqlclient <DOMAIN>/<USER>:'<PASSWORD>'@<TARGET> -windows-auth
```


### NTLM hash로 Windows 인증 접속

```bash
impacket-mssqlclient <DOMAIN>/<USER>@<TARGET> -windows-auth -hashes :<NTLM_HASH>
```

### xp_cmdshell 가능성 확인

```text
SQL> SELECT SYSTEM_USER;
SQL> SELECT IS_SRVROLEMEMBER('sysadmin');
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami
SQL> xp_cmdshell whoami /priv
```

`enable_xp_cmdshell`과 `xp_cmdshell <COMMAND>`은 원시 T-SQL이 아니라 `mssqlclient.py`가 해석하는 대화형 셸 명령이다.

- `enable_xp_cmdshell`은 내부에서 `show advanced options = 1`, `RECONFIGURE`, `xp_cmdshell = 1`, `RECONFIGURE`를 순서대로 실행한다.
- `xp_cmdshell whoami /priv`은 Impacket이 `EXEC master..xp_cmdshell 'whoami /priv'` 형태로 변환한다.
- `xp_cmdshell 'whoami /priv';`처럼 T-SQL용 따옴표와 세미콜론을 붙이면 작은따옴표까지 Windows 명령으로 전달되어 SQL 문법 오류가 발생할 수 있다.
- 원시 T-SQL을 직접 실행하려면 `EXEC master..xp_cmdshell 'whoami /priv';`처럼 `EXEC`를 명시한다.
- `disable_xp_cmdshell`은 `xp_cmdshell`과 `show advanced options`를 모두 `0`으로 설정한다. 변경 전 값이 서로 달랐다면 이 단축 명령 대신 원시 T-SQL로 원래 값을 복구한다.


## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-windows-auth` | Windows/NTLM 인증 사용 | 도메인 또는 로컬 Windows 계정으로 접속 |
| `-hashes <LM:NT>` | NTLM hash 인증 | 비밀번호 대신 hash만 확보한 경우 |
| `-k` | Kerberos 인증 사용 | ccache나 티켓 기반 접속 |
| `-no-pass` | 비밀번호 입력 없이 인증 | Kerberos ccache 사용 |
| `-dc-ip <ip>` | DC IP 지정 | Kerberos/도메인 해석이 불안정할 때 |
| `-port <port>` | MSSQL 포트 지정 | 1433이 아닌 포트 또는 터널 사용 |
| `-h`, `--help` | 도움말 확인 | 버전별 지원 옵션 확인 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SQL 프롬프트 또는 쿼리 결과 출력 | MSSQL 로그인 성공 | `SELECT SYSTEM_USER;`, `SELECT IS_SRVROLEMEMBER('sysadmin');` 확인 |
| database/table 조회 가능 | 읽기 권한 존재 | 민감 테이블, linked server, credential 저장 위치 확인 |
| `xp_cmdshell` 출력 | OS 명령 실행 가능 | 실행 계정 권한과 파일 쓰기/셸 획득 가능성 확인 |
| linked server에서 `sysadmin = 1` | 원격 login mapping으로 권한 상승 가능 | [[MSSQL xp_cmdshell 명령 실행]]로 후속 검증 |
| `Login failed` / timeout | credential 또는 네트워크 문제 | 도메인/SQL 계정 형식, 포트, TLS, 터널 여부 확인 |
| 권한 부족 | 일반 DB 사용자 권한 | role, database 권한, linked server 권한 확인 |
| `xp_cmdshell` 실패 | 비활성화 또는 권한 부족 | sysadmin 여부와 advanced options 상태 확인 |
| linked server `untrusted domain` | Windows Integrated Authentication 위임/신뢰 문제 | impersonate 대상, linked server mapping, SQL 인증 매핑 확인 |

## 관련 공격기법

- [[DB 인증과 데이터 열거]]
- [[DB 서버 파일 쓰기 검증]]
- [[MSSQL xp_cmdshell 명령 실행]]
- [[MSSQL 서비스 Hash 캡처]]
- [[MSSQL Impersonation 권한 상승]]
- [[MSSQL Linked Server 내부 이동]]

## 참고 링크

- [Fortra Impacket `mssqlclient.py`](https://github.com/fortra/impacket/blob/master/examples/mssqlclient.py)
