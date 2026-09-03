---
시작조건: ["DB 서비스 접근 가능", "유효한 DB 자격 증명"]
필요조건: ["DB 서비스 접근 가능", "유효한 DB credential"]
결과: ["데이터", "자격 증명", "권한 단서"]
---

# DB 인증과 데이터 열거

## 한 줄 판단

현재 명령 실행 위치에서 대상 DB 포트에 연결할 수 있고 유효한 데이터베이스 계정이 있다면, 그 로그인으로 볼 수 있는 데이터베이스·테이블·사용자·서버 역할을 열거한다. DB 로그인 성공, DB 관리자 권한, 서버 파일 접근과 운영체제 명령 실행은 각각 별도로 확인한다.

## 사용할 때

- MySQL/MSSQL/Oracle 포트가 열려 있고 credential 후보가 있을 때.
- 웹 설정, SMB/FTP/NFS 파일, 메일에서 DB 접속 문자열을 얻었을 때.
- Kerberoasting으로 `MSSQLSvc` SPN 소유 계정의 평문 비밀번호를 복구했을 때.
- DB 안에 계정/비밀번호/token/업무 데이터가 있을 가능성이 있을 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| DB 접속 정보 | host, port, user, password, DB/SID | 클라이언트 연결 가능 |
| 인증 방식 | SQL/Windows/Oracle SID | 올바른 계정 형식 |
| Kerberoasting 결과 | `SamAccountName`, `MSSQLSvc/<MSSQL_FQDN>:<PORT_OR_INSTANCE>`, 복구한 평문 비밀번호 | SPN 소유 AD 계정과 MSSQL 대상 호스트를 서로 구분할 수 있음 |
| 권한 범위 | `show grants`, role 확인 | 데이터/파일/관리 권한 구분 |

## 실행

### DB와 인증 방식 선택

| 대상·인증 방식 | 현재 보유할 입력 | 연결 명령의 핵심 구분 | 인증 성공 결과 |
|---|---|---|---|
| MySQL 비밀번호 인증 | MySQL `<USER>`와 `<PASSWORD>` | `mysql -h`와 `-u`, 비밀번호 prompt | MySQL prompt와 `SHOW DATABASES` 결과 |
| MSSQL SQL 인증 | MSSQL 내부 SQL login과 비밀번호 | 대상 문자열에 `<USER>:<PASSWORD>` 사용 | SQL prompt와 `SYSTEM_USER` 출력 |
| MSSQL Windows 비밀번호 인증 | 로컬 또는 도메인 Windows 계정과 비밀번호 | `<DOMAIN>/<USER>`와 `-windows-auth` 사용 | Windows login으로 SQL prompt 진입 |
| MSSQL Windows NT hash 인증 | Windows 계정과 그 계정의 NT hash | `-windows-auth -hashes :<NT_HASH>` 사용 | hash를 가진 Windows login으로 SQL prompt 진입 |
| MSSQL Kerberos ticket 인증 | 유효한 ccache, 계정, MSSQL SPN과 일치하는 FQDN | `KRB5CCNAME`, `-k -no-pass` 사용 | ticket의 principal로 SQL prompt 진입 |
| Oracle 비밀번호 인증 | Oracle `<USER>`, `<PASSWORD>`, `<SID_OR_SERVICE>` | 접속 문자열의 SID 또는 service name 일치 | SQL*Plus prompt와 role 조회 결과 |

SQL login, Windows 계정, NT hash와 Kerberos ticket은 서로 다른 인증 입력이다. MSSQL 포트가 열린 사실만으로 어떤 방식이 허용되는지는 알 수 없으므로 현재 보유한 입력과 서버 인증 설정에 맞는 행만 선택한다.

1. SQL 인증, Windows 인증, Kerberos 인증, SID/service name 등 계정 형식을 맞춘다.
2. DB 목록, 테이블 목록, 컬럼을 확인한다.
3. `users`, `settings`, `config`, `tokens`, `password` 계열 테이블을 우선 본다.
4. 권한을 확인해 서버 파일 읽기와 OS 명령 실행 가능성을 분리한다.

### MySQL 접속과 열거

공격 호스트에서 MySQL client를 실행한다.

```bash
mysql -h <TARGET> -u <USER> -p
```

인증에 성공해 열린 MySQL prompt에서 다음 query를 실행한다.

```sql
SHOW DATABASES;
USE <DB>;
SHOW TABLES;
SELECT * FROM users;
SHOW GRANTS;
```

확인할 출력:

- 접근 가능한 DB, 사용자/비밀번호/hash/token 테이블, FILE 권한.
- shell에서 `SHOW DATABASES`를 실행하지 않는다. MySQL prompt가 열리지 않으면 query 권한을 판단하기 전에 포트 도달성, TLS 조건과 사용자명·비밀번호를 확인한다.

### MSSQL 접속과 열거

#### Kerberoasting 결과를 MSSQL 접속 입력으로 변환

Kerberoasting 결과의 각 값을 다음 접속 입력으로 바꾼다.

| Kerberoasting 결과 | MSSQL 접속 입력 | 해석 |
|---|---|---|
| `SamAccountName: <SPN_USER>` | `<SPN_USER>` | 비밀번호를 복구한 AD 계정이다. 전체 SPN 문자열을 사용자명으로 사용하지 않는다. |
| `TargetDomain: <DOMAIN>` 또는 hash의 realm | `<DOMAIN>` | Windows 인증에 사용할 AD 도메인이다. |
| `MSSQLSvc/<MSSQL_FQDN>:<MSSQL_PORT>` | `<MSSQL_FQDN>`, `<MSSQL_PORT>` | MSSQL 서비스가 등록된 대상 호스트와 숫자 포트다. |
| `MSSQLSvc/<MSSQL_FQDN>:<INSTANCE>` | `<MSSQL_FQDN>`, `<INSTANCE>` | suffix가 instance 이름이면 UDP 1434 SQL Browser 또는 서비스 스캔으로 실제 TCP 포트를 먼저 식별한다. |
| Hashcat/John 복구 평문 | `<PASSWORD>` | SPN 소유 AD 계정의 비밀번호 후보다. |

| 명령 실행 위치 | 먼저 사용할 경로 | 대안 |
|---|---|---|
| Windows 공격 호스트 | `runas /netonly`로 복구 계정의 네트워크 로그온 프로세스를 만든 뒤 `sqlcmd -E` | PowerUpSQL `Get-SQLQuery` |
| Linux 공격 호스트 | `impacket-mssqlclient -windows-auth` | SQL login·NT hash·ccache에 맞는 Impacket 인증 방식 |

#### Windows 공격 호스트에서 실행

##### 대상 FQDN과 TDS 포트 확인

```powershell
Resolve-DnsName <MSSQL_FQDN> -Type A
Test-NetConnection <MSSQL_FQDN> -Port <MSSQL_PORT>
```

##### 복구한 AD 계정으로 runas와 sqlcmd 실행

현재 Windows 로그온 계정과 다른 복구 계정을 원격 MSSQL 인증에 사용하려면 [[확보한 AD 비밀번호로 runas netonly 네트워크 인증 컨텍스트 생성]]에 따라 별도 프로세스를 연다. 비밀번호는 prompt에 입력한다.

```cmd
runas /netonly /user:<DOMAIN>\<SPN_USER> cmd.exe
```

새로 열린 `cmd.exe`에서 Windows Integrated Authentication을 명시하고 SQL Server에 접속한다.

```cmd
SQLCMD.EXE -S tcp:<MSSQL_FQDN>,<MSSQL_PORT> -E -W -w 200 -s ","
```

```sql
SELECT CAST(@@SERVERNAME AS varchar(80)) AS server, CAST(SYSTEM_USER AS varchar(80)) AS login, CAST(ORIGINAL_LOGIN() AS varchar(80)) AS original_login, IS_SRVROLEMEMBER('sysadmin') AS sysadmin;
GO
```

확인할 출력:

- `runas /netonly`가 프로세스를 열었다는 메시지는 비밀번호 검증 성공이 아니다. 실제 네트워크 인증은 `sqlcmd`가 MSSQL에 연결할 때 발생한다.
- `/netonly` 프로세스에서 `whoami`는 원래 로컬 계정을 표시할 수 있다. SQL 결과의 `SYSTEM_USER`와 `ORIGINAL_LOGIN()`이 `<DOMAIN>\<SPN_USER>`인지 확인해야 한다.
- `sqlcmd -U/-P`는 SQL login 인증이다. Kerberoasting으로 복구한 AD 계정을 Windows 인증에 사용할 때는 `/netonly` 세션에서 `-E`를 사용한다.

##### 복구한 AD 계정으로 PowerUpSQL 실행

사용 중인 PowerUpSQL 버전이 명시적 credential parameter를 지원한다면 다음처럼 대상 instance와 계정을 함께 지정한다.

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLQuery -Verbose -Instance '<MSSQL_FQDN>,<MSSQL_PORT>' -Username '<DOMAIN>\<SPN_USER>' -Password '<PASSWORD>' -Query "SELECT CAST(@@SERVERNAME AS varchar(80)) AS server, CAST(SYSTEM_USER AS varchar(80)) AS login, IS_SRVROLEMEMBER('sysadmin') AS sysadmin"
```

`Connection Success.`만 보지 말고 반환된 `login`이 `<DOMAIN>\<SPN_USER>`인지 확인한다. PowerUpSQL 버전이나 대상 인증 설정에서 명시적 credential 경로가 실패하면 `runas /netonly`와 `sqlcmd -E`로 Windows Integrated Authentication을 다시 검증한다.

#### Linux 공격 호스트에서 실행

##### 대상 FQDN과 TDS 포트 확인

```bash
getent hosts <MSSQL_FQDN>
nc -vz <MSSQL_FQDN> <MSSQL_PORT>
```

##### Kerberoasting으로 복구한 AD 계정과 비밀번호

비밀번호에 `@`, `:`, `/` 같은 문자가 있어 target 문자열 해석이 깨지는 것을 피하려면 비밀번호를 생략하고 prompt에 입력한다.

```bash
impacket-mssqlclient '<DOMAIN>/<SPN_USER>@<MSSQL_FQDN>' -windows-auth -port <MSSQL_PORT>
```

확인할 출력:

- `SQL>` prompt가 열리면 해당 AD 계정으로 MSSQL Windows 인증과 SQL login 매핑이 모두 성공한 상태다.
- `Login failed for user`는 비밀번호 오류뿐 아니라 SQL Server에 해당 Windows login이 없거나 `CONNECT SQL`이 거부된 경우에도 나타날 수 있다. 동일 계정의 AD 인증 유효성과 MSSQL login 권한을 분리해서 확인한다.
- timeout 또는 연결 거부이면 credential보다 SPN의 FQDN·포트·instance, DNS, 피벗 경로와 현재 listener를 먼저 확인한다. 오래되거나 중복된 SPN은 실제 서비스 위치와 다를 수 있다.

##### SQL login과 비밀번호

```bash
impacket-mssqlclient '<USER>:<PASSWORD>@<TARGET>'
```

##### Windows 계정과 비밀번호

```bash
impacket-mssqlclient '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>' -windows-auth
```

##### Windows 계정과 NT hash

```bash
impacket-mssqlclient '<DOMAIN>/<USER>@<TARGET>' -windows-auth -hashes :<NT_HASH>
```

##### Kerberos ccache

```bash
export KRB5CCNAME=<CCACHE_FILE>
impacket-mssqlclient -k -no-pass -dc-ip <DC_IP> '<DOMAIN>/<USER>@<MSSQL_FQDN>'
```

#### 인증 성공 후 공통 MSSQL 열거

인증에 성공한 SQL prompt에서 다음 열거 query를 실행한다.

```sql
SELECT name FROM master.dbo.sysdatabases;
USE <DB>;
SELECT table_name FROM <DB>.INFORMATION_SCHEMA.TABLES;
SELECT SYSTEM_USER;
SELECT ORIGINAL_LOGIN();
SELECT @@SERVERNAME;
SELECT IS_SRVROLEMEMBER('sysadmin');
```

확인할 출력:

- DB 목록, 테이블, 현재 사용자, 원래 login, 실제 SQL Server와 sysadmin 여부.
- `Login failed`이면 MSSQL 서비스 도달성과 인증 실패를 구분하고 SQL login인지 Windows login인지, `-windows-auth` 사용 여부와 계정 형식을 확인한다.
- Kerberos 실패이면 ccache 만료, `KRB5CCNAME`, MSSQL SPN과 `<MSSQL_FQDN>`, DNS·시간을 확인한다. `-no-pass`는 무인증이 아니라 ccache 인증에서 비밀번호 prompt를 생략하는 옵션이다.

### Oracle 접속과 열거

공격 호스트에서 SQL*Plus를 실행한다.

```bash
sqlplus <USER>/<PASSWORD>@<TARGET>/<SID>
```

인증에 성공해 열린 SQL*Plus prompt에서 다음 query를 실행한다.

```sql
select table_name from all_tables;
select * from user_role_privs;
```

확인할 출력:

- 테이블 목록, role, DBA/sysdba 가능성.
- SQL*Plus prompt가 열리지 않으면 SID·service name, 계정·비밀번호와 Oracle listener 도달성을 확인한다. 로그인 성공만으로 DBA 또는 SYSDBA 권한을 획득했다고 판단하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| DB client가 로그인하고 현재 user·server가 확인됨 | DB 인증 성공 | DB 세션 | DB·schema·table과 role 열거 |
| Kerberoasting 복구 계정으로 `SQL>` prompt가 열림 | SPN 소유 AD 계정이 대상 SQL Server의 Windows login으로 허용됨 | MSSQL Windows 인증 세션 | `SYSTEM_USER`, `ORIGINAL_LOGIN()`, `@@SERVERNAME`, `sysadmin`을 각각 확인 |
| `/netonly` 세션의 `sqlcmd -E`에서 기대한 `<DOMAIN>\<SPN_USER>`가 `SYSTEM_USER`로 반환됨 | 복구한 AD 계정의 Windows Integrated Authentication과 SQL login mapping 성공 | Windows 공격 호스트에서 MSSQL 세션 | DB role·데이터·impersonation·linked server 권한 확인 |
| AD 인증은 유효하지만 MSSQL에서 `Login failed for user`가 반환됨 | 서비스 계정 비밀번호 확보와 SQL login 허용은 별개임 | AD credential 유효·DB 세션 미확보 | SPN 최신성, 대상 instance와 Windows login 등록·`CONNECT SQL` 권한 확인 |
| DB·table 목록은 보이지만 일부 객체가 거부됨 | 제한된 데이터 READ 권한이 있음 | 제한된 DB 접근 | 읽기 가능한 schema 안에서만 수집 |
| 민감 table에서 credential, hash, token 또는 내부 URL이 나옴 | 후속 접근 후보를 수집함 | 민감 데이터 또는 자격 증명 후보 | 평문·hash·token을 구분해 별도 검증 |
| FILE·sysadmin·DBA 같은 권한이 확인됨 | 데이터 조회를 넘어 파일·명령 실행 후보가 있음 | 고권한 DB 기능 후보 | 전용 파일 쓰기·명령 실행 기법으로 이동 |
| 다른 서비스에서 DB credential이 실제 인증됨 | 계정 재사용이 확인됨 | 재사용 가능한 자격 증명 | 새 서비스의 identity와 권한 확인 |
| 로그인 실패 | 인증 방식, 계정 형식, DB·SID 또는 TLS가 맞지 않음 | DB 세션 미확보 | SQL·Windows 인증과 SID·service name 분리 확인 |
| hash만 수집됨 | 평문 credential이 아님 | 오프라인 검증 대상 | [[오프라인 해시 크래킹]]으로 이동 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 인증 identity | DB server, 현재 user, 최초 login | OS 계정과 DB login을 구분 |
| 데이터 접근 | DB·schema·table·row 조회 성공 | 연결 성공과 객체 READ 권한을 구분 |
| 관리자 기능 | `SHOW GRANTS`, server role, Oracle role | FILE, sysadmin, DBA와 일반 SELECT 권한을 분리 |
| 민감 데이터 | column 의미와 row 문맥 | 문자열을 credential로 단정하지 않고 유형 확인 |
| 후속 인증 | 다른 서비스의 실제 로그인 결과 | DB 내부 후보와 재사용 성공을 구분 |

## 후속 공격 연결

- [[DB 서버 파일 수집]]
- [[MSSQL xp_cmdshell 명령 실행]]
- [[MSSQL Impersonation 권한 상승]]
- [[MSSQL Linked Server 내부 이동]]
- Oracle `UTL_FILE` 텍스트 쓰기 확인: [[1521_Oracle_TNS]]
- [[원격 비밀번호 공격]]

## 관련 상태 라우터

- DB에서 재사용 가능한 비밀번호·hash·token을 확인했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- DB 인증 성공만 확인한 상태라면 자격 증명 상태 라우터에서 서비스 권한을 재평가한다.

## 관련 서비스

- [[3306_MySQL]]
- [[1433_MSSQL]]
- [[1521_Oracle_TNS]]

## 관련 도구

- [[mysql]]
- [[PowerUpSQL]]
- [[impacket-mssqlclient]]
- [[sqlcmd]]
- [[sqsh]]
- [[sqlplus]]
- [[dbeaver]]
