---
tags:
  - 서비스/mysql
  - 서비스/mssql
시작조건: ["DB 인증 세션 확보", "DB 서버 파일 쓰기 권한 후보 확인"]
필요권한: ["MySQL FILE 또는 MSSQL OLE Automation procedure 실행 권한", "DB 서비스 계정의 대상 디렉터리 쓰기 권한"]
필요조건: ["DB 서버 관점의 고유 파일 경로", "기존 파일 부재 확인", "이번에 만든 파일을 정확히 제거할 복구 경로"]
결과: ["DB 서버의 고유 파일 생성", "웹 제공 경로 또는 후속 파일 작업 후보"]
---

# DB 서버 파일 쓰기 검증

## 한 줄 판단

현재 MySQL 또는 Microsoft SQL Server(MSSQL) 세션에 서버 측 파일 생성 기능을 사용할 권한이 있고 DB 서비스 계정이 대상 디렉터리에 쓸 수 있다면, 기존 파일을 덮어쓰지 않는 고유 파일 하나를 생성해 파일 쓰기와 웹 노출·실행을 각각 분리해 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치 | DB에 query를 보낼 수 있는 클라이언트 호스트와 SQL prompt | [[DB 인증과 데이터 열거]] | 대상·인스턴스·인증 방식과 현재 DB login 확인 |
| MySQL 권한·경로 | `FILE`, `secure_file_priv`, mysqld의 디렉터리 쓰기 권한 | `SHOW GRANTS FOR CURRENT_USER;`, `SHOW VARIABLES LIKE 'secure_file_priv';` | `NULL`·허용 디렉터리·빈 값과 OS ACL을 구분 |
| MSSQL 권한·설정 | OLE procedure 실행 권한, 비활성 설정을 바꾸려면 `ALTER SETTINGS` | `IS_SRVROLEMEMBER`, `HAS_PERMS_BY_NAME`, `sys.configurations` | 설정 변경 권한과 각 procedure의 `EXECUTE` 권한을 구분 |
| 대상 파일 | DB 서버 관점의 절대 경로와 고유 이름, 기존 파일 부재 | MySQL은 `INTO OUTFILE`의 비덮어쓰기 결과, MSSQL은 `FileExists` | 파일이 있으면 중단하고 새 고유 이름 선택 |
| 복구 경로 | 생성한 exact 경로를 DB 기능 또는 서버 관리 셸로 제거 가능 | 아래 제품별 정리 명령을 작업 전에 선택 | 제거 경로가 없으면 쓰기 검증을 수행하지 않음 |

MySQL의 `SELECT ... INTO OUTFILE`은 클라이언트가 아니라 서버 호스트에 파일을 만들며 `FILE` 권한을 요구한다. MSSQL의 OLE Automation은 SQL Server 프로세스에서 등록된 OLE object를 호출하므로, 설정 활성화 가능성과 `Scripting.FileSystemObject` 생성·파일 쓰기 성공은 별도 단계다. DB 권한→OS Identity·path→웹 mapping·handler의 공통 경계는 [[DB 서버 측 작업의 실행 주체와 결과 경계]]를 따른다.

## 실행

아래 `<UNIQUE_PROOF>`는 DB 서버에서 새로 만들 basename(예: `db-check-42f1`)이고, 경로는 DB 서비스 계정이 보는 서버의 절대 경로다. MySQL·MSSQL 명령은 각각 DB query를 실행하는 클라이언트의 해당 SQL prompt에서 실행하며, 뒤의 HTTP 확인에서 `<TARGET>`은 파일을 제공할 수 있는 서버 주소다. 같은 proof 이름은 생성·조회·삭제 단계에서 재사용한다.

### MySQL에서 고유 proof 파일 생성

MySQL prompt에서 현재 grant와 경로 제한을 먼저 확인한다.

```sql
SHOW GRANTS FOR CURRENT_USER;
SHOW VARIABLES LIKE 'secure_file_priv';
```

`secure_file_priv`가 디렉터리이면 그 안의 경로만 사용한다. `NULL`이면 이 파일 기능이 비활성화된 상태이며, 빈 값이어도 mysqld의 운영체제 쓰기 권한은 별도로 필요하다. 고유 이름을 사용해 기존 파일 충돌을 피할 수 있을 때 파일을 생성한다.

`<UNIQUE_PROOF>`는 DB 서버에서 새로 만들 basename이며 가상 예시는 `db-check-42f1`이다. 이 블록은 MySQL prompt에서 실행하고 `/var/www/html/`은 DB 서버의 절대 경로 예시다.

```sql
SELECT 'db-file-write-proof' INTO OUTFILE '/var/www/html/<UNIQUE_PROOF>.txt';
```

확인할 출력:

- `Query OK`와 새 파일은 mysqld 계정의 해당 경로 파일 생성만 입증한다.
- `File exists`이면 MySQL의 비덮어쓰기 보호가 동작한 것이다. 기존 파일을 건드리지 말고 새 고유 이름을 선택한다.
- 권한 오류나 `secure-file-priv` 오류이면 `FILE`, 허용 디렉터리와 mysqld OS ACL을 차례로 확인한다.

웹 제공 경로라고 예상한 경우 정적 응답을 별도로 확인한다.

```bash
curl "http://<TARGET>/<UNIQUE_PROOF>.txt"
```

HTTP 응답의 고유 문자열은 파일이 해당 URL에서 제공된다는 뜻이다. 정적 파일 조회는 서버 측 코드 실행이 아니며, handler를 통한 실행은 [[웹 파일 업로드와 Web Shell]]에서 별도로 검증한다.

### MSSQL에서 OLE 설정과 권한 기준선 확인

원시 T-SQL을 받는 `sqlcmd`, `sqsh` 또는 `impacket-mssqlclient`의 `SQL>` prompt에서 실행한다. 현재 login, 설정의 구성값·실행값과 설정 변경 권한을 기록한다.

```sql
SELECT SYSTEM_USER AS current_login,
       IS_SRVROLEMEMBER('sysadmin') AS is_sysadmin,
       HAS_PERMS_BY_NAME(NULL, NULL, 'ALTER SETTINGS') AS can_alter_settings;
SELECT name, value, value_in_use
FROM sys.configurations
WHERE name IN ('show advanced options', 'Ole Automation Procedures');
```

OLE Automation이 이미 활성화돼 있고 현재 login에 필요한 `sp_OA*` procedure 실행 권한이 직접 부여된 경우에는 설정을 바꾸지 않는다. 비활성 상태를 바꾸려면 `ALTER SETTINGS`가 필요하며, `sysadmin`과 `serveradmin`은 이를 암시적으로 가진다.

```sql
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'Ole Automation Procedures', 1;
RECONFIGURE;
```

설정 변경 성공은 OLE object 생성이나 파일 쓰기 성공이 아니다. 변경 전 두 설정의 `value`와 `value_in_use`를 따로 기록하고, 이번 작업에서 바꾼 값만 복구한다.

### MSSQL에서 기존 파일을 덮어쓰지 않는 proof 생성

아래 batch는 `FileExists`로 exact 경로를 확인하고, `CreateTextFile`의 `overwrite` 인수를 `0`으로 지정한다. `<UNIQUE_PROOF>`는 이번 작업에서만 사용하는 이름으로 바꾼다.

```sql
DECLARE @fso int, @stream int, @hr int, @exists bit;

EXEC @hr = sp_OACreate 'Scripting.FileSystemObject', @fso OUT;
IF @hr <> 0 BEGIN EXEC sp_OAGetErrorInfo @fso; RETURN; END;

EXEC @hr = sp_OAMethod @fso, 'FileExists', @exists OUT,
  'C:\\inetpub\\wwwroot\\<UNIQUE_PROOF>.txt';
IF @hr <> 0 BEGIN EXEC sp_OAGetErrorInfo @fso; EXEC sp_OADestroy @fso; RETURN; END;
IF @exists = 1 BEGIN PRINT 'ABORT: target file already exists'; EXEC sp_OADestroy @fso; RETURN; END;

EXEC @hr = sp_OAMethod @fso, 'CreateTextFile', @stream OUT,
  'C:\\inetpub\\wwwroot\\<UNIQUE_PROOF>.txt', 0, 0;
IF @hr <> 0 BEGIN EXEC sp_OAGetErrorInfo @fso; EXEC sp_OADestroy @fso; RETURN; END;

EXEC @hr = sp_OAMethod @stream, 'WriteLine', NULL, 'db-file-write-proof';
IF @hr <> 0 EXEC sp_OAGetErrorInfo @stream;
EXEC sp_OAMethod @stream, 'Close';
EXEC sp_OADestroy @stream;
EXEC sp_OADestroy @fso;
```

확인할 출력:

- 모든 OLE 호출의 반환 코드 `0`은 해당 호출의 성공이다. 비zero HRESULT가 나오면 바로 다음 호출로 덮기 전에 `sp_OAGetErrorInfo`로 원인을 확인한다.
- `Invalid class string`이면 해당 호스트에 ProgID가 등록되지 않았고, access denied·path not found이면 procedure 권한과 SQL Server 서비스 계정의 경로·ACL을 확인한다.
- DB query 완료만으로 파일 생성을 단정하지 않는다. 아래 존재 확인 또는 서버 관리 경로에서 exact 파일과 고유 문자열을 확인한다.

```sql
DECLARE @fso int, @exists bit;
EXEC sp_OACreate 'Scripting.FileSystemObject', @fso OUT;
EXEC sp_OAMethod @fso, 'FileExists', @exists OUT,
  'C:\\inetpub\\wwwroot\\<UNIQUE_PROOF>.txt';
SELECT @exists AS proof_file_exists;
EXEC sp_OADestroy @fso;
```

`proof_file_exists = 1`은 SQL Server 서비스 계정이 볼 수 있는 경로에 파일이 있다는 뜻이다. 웹 응답·handler 실행·Windows 고권한은 각각 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 고유 proof 파일과 문자열 확인 | DB 기능으로 서버 파일 생성 성공 | 서버 파일 쓰기 | 웹 경로이면 [[웹 파일 업로드와 Web Shell]], 아니면 목적에 맞는 파일 작업 판단 |
| SQL 성공, HTTP 404 또는 다른 내용 | 파일 생성과 웹 URL mapping이 일치하지 않음 | 서버 파일 쓰기만 확인 | document root·virtual directory·URL mapping 재확인 |
| 정적 proof 응답만 확인 | 웹 제공 가능, 코드 실행 미확정 | 웹 파일 접근 | handler 조건을 별도 검증 |
| MySQL `File exists` 또는 MSSQL 사전 존재 확인 | 기존 파일 충돌 | 변경 없음 | 새 고유 이름 선택 |
| DB 권한 오류 | DB 파일 기능 또는 설정 변경 권한 부족 | 기존 DB 접근 유지 | 현재 grant·role·procedure permission 확인 |
| OS path 오류·access denied | DB 서비스 계정의 경로 또는 ACL 미충족 | 파일 쓰기 실패 | 서버 관점 절대 경로·서비스 계정·ACL 확인 |

## 변경 영향과 복구

이번 작업에서 기록한 exact 경로만 제거한다. 파일명 pattern이나 디렉터리 전체를 대상으로 정리하지 않는다.

### MySQL proof 정리

`SELECT ... INTO OUTFILE`에는 생성 파일 삭제 기능이 없다. 작업 전에 선택한 서버 관리 셸에서 exact 파일의 내용·경로를 확인한 뒤 제거한다.

```bash
test "$(cat '/var/www/html/<UNIQUE_PROOF>.txt')" = 'db-file-write-proof' && rm -- '/var/www/html/<UNIQUE_PROOF>.txt'
test ! -e '/var/www/html/<UNIQUE_PROOF>.txt'
```

서버 관리 경로가 끊겨 삭제를 확인하지 못하면 복구 완료가 아니라 `원격 정리 미확인`으로 기록한다. MySQL 설정은 이 절차에서 변경하지 않는다.

### MSSQL proof와 설정 복구

OLE 연결이 살아 있을 때 exact proof 파일을 제거하고 부재를 확인한다. `DeleteFile`의 `force`는 `0`으로 두며 wildcard를 사용하지 않는다.

```sql
DECLARE @fso int, @exists bit, @hr int;
EXEC @hr = sp_OACreate 'Scripting.FileSystemObject', @fso OUT;
IF @hr <> 0 BEGIN EXEC sp_OAGetErrorInfo @fso; RETURN; END;
EXEC @hr = sp_OAMethod @fso, 'DeleteFile', NULL,
  'C:\\inetpub\\wwwroot\\<UNIQUE_PROOF>.txt', 0;
IF @hr <> 0 BEGIN EXEC sp_OAGetErrorInfo @fso; EXEC sp_OADestroy @fso; RETURN; END;
EXEC sp_OAMethod @fso, 'FileExists', @exists OUT,
  'C:\\inetpub\\wwwroot\\<UNIQUE_PROOF>.txt';
SELECT @exists AS proof_file_exists_after_cleanup;
EXEC sp_OADestroy @fso;
```

`proof_file_exists_after_cleanup = 0`을 확인한 뒤, OLE 설정을 이번 작업에서 활성화했고 변경 전 실행값이 `0`이었을 때만 비활성화한다.

```sql
EXECUTE sp_configure 'Ole Automation Procedures', 0;
RECONFIGURE;
```

`show advanced options`도 이번 작업에서 `0`에서 `1`로 바꾼 경우에만 마지막에 복구한다.

```sql
EXECUTE sp_configure 'show advanced options', 0;
RECONFIGURE;
SELECT name, value, value_in_use
FROM sys.configurations
WHERE name IN ('show advanced options', 'Ole Automation Procedures');
```

마지막 출력이 작업 전 기록과 일치해야 설정 복구 완료다. 파일은 지웠지만 설정값이 다르면 생성 자원 정리만 완료된 것이며 원래 설정 복구는 미완료다. 연결이 먼저 끊겨 파일·설정 상태를 확인하지 못하면 둘 다 완료로 표현하지 않는다.

## 관련 서비스

- [[MySQL 서비스]]
- [[MSSQL 서비스]]

## 관련 도구

- [[mysql]]
- [[sqlcmd]]
- [[sqsh]]
- [[impacket-mssqlclient]]

## 참고 링크

- [MySQL 8.0: SELECT ... INTO Statement](https://dev.mysql.com/doc/refman/8.0/en/select-into.html)
- [MySQL 8.4: Making MySQL Secure Against Attackers](https://dev.mysql.com/doc/refman/8.4/en/security-against-attack.html)
- [Microsoft: Ole Automation Procedures server configuration](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/ole-automation-procedures-server-configuration-option?view=sql-server-ver17)
- [Microsoft: sp_configure](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-configure-transact-sql?view=sql-server-ver17)
- [Microsoft: sp_OACreate](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oacreate-transact-sql?view=sql-server-ver17)
- [Microsoft: sp_OAMethod](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oamethod-transact-sql?view=sql-server-ver17)
- [Microsoft: sp_OAGetErrorInfo](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oageterrorinfo-transact-sql?view=sql-server-ver17)
- [Microsoft: sp_OADestroy](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oadestroy-transact-sql?view=sql-server-ver17)
- [Microsoft: FileSystemObject](https://learn.microsoft.com/en-us/office/vba/language/reference/user-interface-help/filesystemobject-object)
- [Microsoft: CreateTextFile](https://learn.microsoft.com/en-us/office/vba/language/reference/user-interface-help/createtextfile-method)
- [Microsoft: DeleteFile](https://learn.microsoft.com/en-us/office/vba/language/reference/user-interface-help/deletefile-method)
