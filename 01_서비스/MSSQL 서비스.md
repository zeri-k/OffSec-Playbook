---
tags:
  - 환경/windows
  - 서비스/mssql
대표포트:
  - "T:1433"
  - "U:1434"
서비스:
  - MSSQL
  - SQL Server
---

# MSSQL 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Microsoft SQL Server(MSSQL) TCP 포트 또는 UDP 1434 SQL Browser에 도달할 수 있고, 아직 유효한 SQL·Windows 계정이나 데이터베이스(DB) 권한은 확인하지 않은 상태에서 시작한다. 인스턴스와 SQL·Windows 인증 방식을 맞춘 뒤 DB 사용자 권한, `sysadmin`, `IMPERSONATE`, linked server mapping, `xp_cmdshell`의 실제 운영체제 실행 계정을 분리한다.

성공하면 로그인 주체가 읽을 수 있는 DB·테이블, 서버 역할과 연결 서버 경로를 얻는다. 연결 실패 시 인스턴스명·동적 TCP 포트·TLS를, `Login failed` 시 인증 방식·도메인/로컬 사용자 형식을, query 거부 시 현재 DB 주체와 역할을 다시 확인하며 DB 로그인이나 `sysadmin`만으로 호스트 관리자 권한을 단정하지 않는다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | SQL Browser의 named instance·동적 포트 | UDP 1434 응답과 `nmap --script ms-sql-info -sU -p1434 <TARGET>` | named instance가 반환하는 동적 TCP 포트와 인스턴스명을 확인한다. |
| 2 | 인스턴스, 빈 비밀번호, NTLM 정보 | `nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-ntlm-info -p<SQL_TCP_PORT> <TARGET>` | 서버명·버전·인스턴스와 인증 후보를 확인한다. |
| 3 | SQL 인증과 Windows 인증 | `impacket-mssqlclient <USER>:<PASSWORD>@<TARGET>`, `impacket-mssqlclient <USER>:<PASSWORD>@<TARGET> -windows-auth` | 두 인증 방식의 성공·실패를 구분한다. |
| 4 | Windows 내부 접속 | `SQLCMD.EXE -S <SERVER>`, `SQLCMD.EXE -S <SERVER> -E -W -w 200 -s ","` | 현재 Windows 계정의 SQL prompt 접근과 가독성 있는 query 결과를 확인한다. |
| 5 | DB 목록·서버 역할·linked server 목록 | `select name from sys.databases;`, `select is_srvrolemember('sysadmin');`, `select name, product, data_source from sys.servers;` | 접근 가능한 DB, `sysadmin = 1`과 linked server 후보를 확인한다. |
| 6 | impersonation과 linked server mapping | `sys.server_permissions`의 `IMPERSONATE`와 `sys.servers` 결과를 확인 | 어떤 login이 누구를 가장하고 어느 서버로 query를 보낼 수 있는지 확인한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| SQL/Windows 인증 가능 또는 DB 접근 성공 | [[DB 인증과 데이터 열거]] | `impacket-mssqlclient`, `sqlcmd`, `sqsh` | DB 목록·테이블과 로그인 사용자의 DB 권한 |
| 사용자 이름·비밀번호 후보 | [[DB 인증과 데이터 열거]] | `impacket-mssqlclient`, `sqlcmd` | SQL/Windows 인증 방식별 DB 로그인과 로그인 주체 |
| BULK 파일 읽기 권한과 서버 파일 경로 | [[DB 서버 파일 수집]] | `impacket-mssqlclient` | SQL Server 서비스 계정이 읽을 수 있는 파일 내용 |
| OLE Automation 설정 단서 | 이 문서의 선택적 OLE Automation 설정 확인 | `impacket-mssqlclient` | 설정 변경 가능성, 파일·명령 실행은 미확정 |
| `sysadmin = 1` 또는 `xp_cmdshell` 후보 | [[MSSQL xp_cmdshell 명령 실행]] | `impacket-mssqlclient` | `whoami`, `hostname`으로 확인한 OS 명령 실행 계정·호스트 |
| `xp_cmdshell`의 `whoami /priv`에서 `SeImpersonatePrivilege`가 `Enabled` | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]] | `PrintSpoofer` | SQL Server 호스트의 SYSTEM 명령 실행 |
| DB query 실행과 outbound SMB 가능 | [[MSSQL 서비스 Hash 캡처]] | `impacket-mssqlclient`, `impacket-smbserver` | SQL Server 서비스 계정의 인증 시도 단서 |
| Windows 인증 relay 대상 | [[NTLM Relay 조건 검토]] | `impacket-ntlmrelayx` | 보호 설정과 relay 계정 권한이 반영된 대상 후보 |
| `IMPERSONATE` 권한 | [[MSSQL Impersonation 권한 상승]] | `impacket-mssqlclient` | 가장한 login의 DB 역할과 실제 권한 |
| linked server 존재 | [[MSSQL Linked Server 내부 이동]] | `impacket-mssqlclient` | 원격 또는 동일 호스트 SQL Server query 권한 |
| impersonation 후 linked server에서 `sysadmin = 1` | [[MSSQL xp_cmdshell 명령 실행]] | `sqlcmd`, `impacket-mssqlclient` | linked server mapping을 통한 DB 관리자·OS 명령 실행 후보 |

### 선택적 OLE Automation 설정 확인

현재 설정값과 현재 login의 서버 설정 변경 권한을 확인한다. 설정 활성화는 파일 쓰기나 운영체제 명령 실행 성공이 아니다.

```sql
SELECT value_in_use FROM sys.configurations WHERE name = 'Ole Automation Procedures';
EXEC sp_configure 'Ole Automation Procedures', 1;
RECONFIGURE;
```

- 변경 전 `value_in_use`를 기록하고, 이번 확인에서 `0`을 `1`로 바꾼 경우에만 원복한다.
- 권한 거부이면 현재 login의 서버 설정 변경 권한을 확인한다.
- 실제 파일 작업이나 명령 실행은 별도 OLE 객체 호출 또는 [[MSSQL xp_cmdshell 명령 실행]]으로 검증한다.

```sql
EXEC sp_configure 'Ole Automation Procedures', 0;
RECONFIGURE;
```

## 서비스 고유 주의 사항

- SQL 인증과 Windows 인증, TLS/self-signed certificate 오류를 구분한다.
- `xp_cmdshell`은 비활성화될 수 있고, 활성화 가능 여부와 실행 계정의 OS 권한은 DB 권한과 별도다.
- linked server의 `untrusted domain` 오류는 SQL impersonation만으로 Windows Integrated Authentication 위임 문제가 해결되지 않을 때 발생할 수 있다.
- linked server가 동일 호스트를 가리킬 수 있으므로 `@@SERVERNAME`, `hostname`, `whoami`로 위치와 계정을 확인한다.
- DB 경유로 로컬 Administrators에 사용자를 추가해도 기존 RDP 토큰은 즉시 바뀌지 않으므로 새 로그온과 UAC elevation을 별도로 확인한다.

## 참고 링크

- [Microsoft: sys.databases](https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-databases-transact-sql?view=sql-server-ver16)
- [Microsoft: SYSTEM_USER](https://learn.microsoft.com/en-us/sql/t-sql/functions/system-user-transact-sql?view=sql-server-ver16)
- [Microsoft: IS_SRVROLEMEMBER](https://learn.microsoft.com/en-us/sql/t-sql/functions/is-srvrolemember-transact-sql?view=sql-server-ver16)
- [Fortra Impacket `mssqlclient.py`](https://github.com/fortra/impacket/blob/master/examples/mssqlclient.py)
