---
tags:
  - 서비스/mssql
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows"]
필요조건: ["SQL 계정 또는 Windows 인증 세션", "SQL Server 이름, 인스턴스 또는 주소"]
결과: ["쿼리 결과", "데이터", "명령 출력"]
---

# sqlcmd

## 도구 개요

`sqlcmd`는 Windows 통합 인증이나 SQL 계정으로 Microsoft SQL Server에 접속해 query와 SQL script를 실행하는 명령줄 클라이언트다. 현재 login과 서버 역할을 확인하고 출력 형식을 조정하거나 반복 가능한 DB 점검을 수행할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: Microsoft ODBC/sqlcmd가 설치된 Linux 또는 Windows 호스트
- 입력: SQL Server 주소·인스턴스와 SQL 계정 또는 Windows 인증 세션
- 다른 AD 계정 입력: `runas /netonly`로 만든 네트워크 로그온 프로세스와 해당 계정의 평문 비밀번호
- 선택 입력: 기본 데이터베이스, 쿼리 또는 SQL 스크립트


## 표준 사용법

```bash
sqlcmd -S <server> -U <user> -P <password>
```

Windows 대상 안에서 현재 로그인한 Windows 계정으로 붙을 때는 서버명만 지정해도 된다.

```cmd
SQLCMD.EXE -S <SERVER>
```

명시적으로 Windows Integrated Authentication을 쓰려면 `-E`를 붙인다.

```cmd
SQLCMD.EXE -S <SERVER> -E
```

## 대표 예시

### Windows 대상 내부에서 로컬 SQL Server 접속

```cmd
SQLCMD.EXE -S <SERVER>
```

`<SERVER>` 호스트에서 현재 Windows 사용자 컨텍스트로 기본 SQL Server 또는 확인된 인스턴스에 접속한다.

### SQL 인증으로 버전 확인

```bash
sqlcmd -S <TARGET> -U sa -P '<PASSWORD>' -Q 'SELECT @@version;'
```

### Windows 인증으로 현재 사용자 확인

```bash
sqlcmd -S <TARGET> -E -Q 'SELECT SYSTEM_USER;'
```

### 복구한 AD 계정으로 원격 MSSQL Windows 인증

현재 Windows 로그온 계정과 다른 AD 계정의 비밀번호를 확보했다면 [[확보한 AD 비밀번호로 runas netonly 네트워크 인증 컨텍스트 생성]]에 따라 별도 네트워크 로그온 프로세스를 연다.

```cmd
runas /netonly /user:<DOMAIN>\<USER> cmd.exe
```

비밀번호를 prompt에 입력한 뒤 새 `cmd.exe`에서 실행한다.

```cmd
SQLCMD.EXE -S tcp:<MSSQL_FQDN>,<MSSQL_PORT> -E -W -w 200 -s ","
```

`runas` 프로세스 생성은 인증 성공 증거가 아니다. `/netonly`에서는 `whoami`가 원래 로컬 계정을 표시할 수 있으므로 다음 SQL 결과로 실제 원격 인증 주체를 확인한다.

```sql
SELECT CAST(@@SERVERNAME AS varchar(80)) AS server, CAST(SYSTEM_USER AS varchar(80)) AS login, CAST(ORIGINAL_LOGIN() AS varchar(80)) AS original_login, IS_SRVROLEMEMBER('sysadmin') AS sysadmin;
GO
```

### 출력 공백 줄이기

```cmd
SQLCMD.EXE -S <SERVER> -E -W -w 200 -s ","
```

`sqlcmd` 기본 출력은 고정 폭 컬럼 때문에 공백이 길게 늘어질 수 있다. `-W`로 trailing space를 줄이고, `-w`로 줄 폭을 늘리며, `-s`로 구분자를 지정한다. 현재 Windows 계정으로 접속할 때 `-E`를 생략해도 되는 환경이 있지만, 인증 방식을 명확히 남기려면 붙이는 편이 좋다.

### 컬럼 폭 줄여 보기

```cmd
SQLCMD.EXE -S <SERVER> -E -y 30 -Y 30 -w 200
```

`-y`는 가변 길이 타입, `-Y`는 고정 길이 타입의 출력 폭을 줄인다. `-W`와 `-y/-Y`는 함께 쓰지 않는 편이 낫다.

### 현재 login과 sysadmin 여부 확인

```sql
SELECT CAST(@@SERVERNAME AS varchar(40)) AS server, CAST(SYSTEM_USER AS varchar(40)) AS login, IS_SRVROLEMEMBER('sysadmin') AS sysadmin;
GO
```

공백이 길면 쿼리에서 `CAST(... AS varchar(N))`로 컬럼 폭을 줄인다.

### impersonation 후 linked server 권한 확인

```sql
USE master;
EXECUTE AS LOGIN = '<IMPERSONATE_TARGET>';
SELECT SYSTEM_USER;
SELECT ORIGINAL_LOGIN();
EXECUTE('SELECT CAST(@@SERVERNAME AS varchar(40)) AS server, CAST(SYSTEM_USER AS varchar(40)) AS login, IS_SRVROLEMEMBER(''sysadmin'') AS sysadmin') AT [<LINKED_SERVER>];
REVERT;
GO
```

원래 login은 낮은 권한이어도 impersonated login이 linked server에서 고권한 login으로 매핑될 수 있다. linked server에서 OS 명령 실행으로 이어가는 상세 흐름은 [[MSSQL xp_cmdshell 명령 실행]]에서 확인한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-S` | 서버와 인스턴스 지정 |
| `-U`, `-P` | SQL 인증 사용자명과 비밀번호 |
| `-E` | Windows Integrated Authentication 사용 |
| `-d` | 기본 DB 지정 |
| `-Q` | 쿼리 실행 후 종료 |
| `-i`, `-o` | 입력 스크립트/출력 파일 지정 |
| `-W` | 컬럼 뒤쪽 공백 제거 |
| `-w <width>` | 출력 줄 폭 지정 |
| `-s <separator>` | 컬럼 구분자 지정 |
| `-y <width>` | varchar/nvarchar 등 가변 길이 타입 출력 폭 지정 |
| `-Y <width>` | char/nchar 등 고정 길이 타입 출력 폭 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SQL 프롬프트 또는 쿼리 결과 출력 | MSSQL 로그인 성공 | `SELECT SYSTEM_USER;`, `SELECT IS_SRVROLEMEMBER('sysadmin');` 확인 |
| `/netonly`에서 프로세스만 생성됨 | 자격 증명은 아직 원격 서비스에서 검증되지 않음 | `sqlcmd -E`의 연결과 `SYSTEM_USER` 확인 |
| `SYSTEM_USER`가 지정한 `<DOMAIN>\<USER>`로 반환됨 | 해당 AD 계정으로 Windows Integrated Authentication 성공 | DB role과 객체 권한 확인 |
| database/table 조회 가능 | 읽기 권한 존재 | 민감 테이블, linked server, credential 저장 위치 확인 |
| `xp_cmdshell` 출력 | OS 명령 실행 가능 | 실행 계정 권한과 파일 쓰기/셸 획득 가능성 확인 |
| `SYSTEM_USER` 변경 | SQL impersonation 성공 | `ORIGINAL_LOGIN()`과 비교해 최초 접속 계정 구분 |
| linked server에서 `sysadmin = 1` | 원격 login mapping으로 권한 상승 가능 | `xp_cmdshell`을 `EXECUTE ... AT`로 실행 |
| `Login failed` | SQL 인증과 Windows 인증 방식 또는 계정 불일치 | `DOMAIN\\user`, SQL 계정과 Kerberos/NTLM 방식 구분 |
| 권한 오류 | 일반 DB 사용자 권한 | role, database와 linked server 권한 확인 |
| `xp_cmdshell` 오류 | 기능 비활성화 또는 권한 부족 | sysadmin 여부와 advanced options 상태 확인 |
| timeout 또는 연결 실패 | 포트 차단, TLS 또는 named instance 문제 | 포트, instance, 암호화 옵션과 터널 확인 |
| `untrusted domain` | linked server가 현재 Windows 보안 컨텍스트로 Integrated Authentication 시도 | `EXECUTE AS LOGIN`, linked server login mapping, SQL credential 저장 여부 확인 |
| 출력 공백이 너무 많음 | sqlcmd 고정 폭 출력 | `-W -w 200 -s ","`, 또는 쿼리에서 `CAST(... AS varchar(N))` 사용 |

## 관련 공격기법

- [[DB 인증과 데이터 열거]]
- [[MSSQL Impersonation 권한 상승]]
- [[MSSQL Linked Server 내부 이동]]
- [[MSSQL xp_cmdshell 명령 실행]]
