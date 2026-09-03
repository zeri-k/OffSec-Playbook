---
tags:
  - 서비스/mssql
시작조건: ["MSSQL 인증 세션 확보", "sysadmin 또는 xp_cmdshell 실행 권한 확인"]
필요권한: ["직접 sysadmin 또는 linked server 원격 sysadmin", "xp_cmdshell 활성화/실행 권한"]
필요조건: ["유효한 MSSQL 계정", "xp_cmdshell 사용 가능 또는 활성화 가능", "linked server 경유 시 RPC/RPC Out 또는 원격 query 가능"]
결과: ["SQL Server 서비스 계정의 운영체제 명령 실행", "세션"]
---

# MSSQL xp_cmdshell 명령 실행

## 한 줄 판단

현재 MSSQL 로그인에 `xp_cmdshell` 실행 권한이 있거나 `sysadmin`으로 기능을 활성화할 수 있다면, SQL Server가 실행 중인 호스트에서 서비스 계정 권한으로 운영체제 명령을 실행한다. 데이터베이스 `sysadmin`과 Windows 로컬 관리자는 별도 권한이므로 `hostname`과 `whoami`로 확인한다.

## 사용할 때

- MSSQL 인증에 성공했고 sysadmin 또는 `xp_cmdshell` 실행 권한이 있을 때.
- DB 내부 데이터 수집을 넘어 OS 명령 실행 영향이 필요한 때.
- SQL Server 서비스 계정 권한을 확인하고 reverse shell로 전환할 때.
- [[MSSQL Impersonation 권한 상승]] 또는 [[MSSQL Linked Server 내부 이동]]으로 linked server에서 sysadmin 권한이 확인되었을 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| MSSQL 접속 | mssqlclient/sqlcmd/sqsh | query 실행 가능 |
| sysadmin 여부 | `IS_SRVROLEMEMBER` | `1`이면 활성화 가능성 높음 |
| xp_cmdshell | 직접 실행 또는 설정 확인 | 명령 출력 반환 |
| 실행 위치 | `hostname`, `whoami /all` | 목표 호스트와 실행 계정 확인 |

## 실행

### 방식 선택

| 현재 확인한 상태 | query가 실행되는 SQL Server | 필요한 권한·입력 | 성공 결과 |
|---|---|---|---|
| 현재 서버에서 `xp_cmdshell`이 이미 활성화되고 실행 권한이 있음 | 현재 접속한 MSSQL 서버 | `xp_cmdshell` 실행 권한 | 현재 SQL Server 서비스 계정의 OS 명령 출력 |
| 현재 서버에서 `xp_cmdshell`이 비활성화되고 현재 login이 `sysadmin` | 현재 접속한 MSSQL 서버 | 현재 서버의 `sysadmin`, 변경 전 설정값 | 기능 활성화 후 서비스 계정의 OS 명령 출력 |
| 현재 login 또는 impersonation 컨텍스트가 linked server에서 `sysadmin`으로 매핑됨 | `[<LINKED_SERVER>]`가 가리키는 원격 SQL Server | linked server query·RPC Out 경로와 원격 `sysadmin` 매핑 | 원격 SQL Server 호스트의 서비스 계정으로 OS 명령 실행 |

직접 실행, 기능 활성화와 linked server 실행은 서로 다른 결과다. 앞 행의 조건을 확인하지 않은 상태에서 다음 행의 명령으로 넘어가지 않는다. 로컬 그룹 변경은 [[로컬 관리자 그룹 구성원 추가]]에서 별도로 수행한다.

### 변경 전 상태 기록

```sql
SELECT name, value, value_in_use
FROM sys.configurations
WHERE name IN ('show advanced options', 'xp_cmdshell');
GO
```

확인할 출력:

- `show advanced options`와 `xp_cmdshell`의 `value_in_use`를 기록한다.
- 기존에 활성화된 설정을 작업 후 임의로 비활성화하지 않도록 변경 전 값을 기준으로 복구한다.

### 권한 확인

```sql
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
```

확인할 출력:

- 현재 SQL login과 sysadmin 여부.

### xp_cmdshell 실행

#### impacket-mssqlclient 대화형 프롬프트

`mssqlclient.py`의 `SQL>` 프롬프트에서 `xp_cmdshell`은 Impacket 셸 명령이다. 뒤에 Windows 명령만 입력한다.

```text
SQL> xp_cmdshell whoami
SQL> xp_cmdshell whoami /priv
```

`xp_cmdshell 'whoami /priv';`처럼 Windows 명령 전체를 작은따옴표와 세미콜론으로 감싸지 않는다. Impacket이 입력을 내부적으로 `EXEC master..xp_cmdshell '<WINDOWS_COMMAND>'` 형태로 변환한다.

#### 원시 T-SQL을 실행하는 클라이언트

`sqlcmd`처럼 원시 T-SQL을 입력하는 클라이언트에서는 `EXEC`와 문자열 따옴표를 직접 사용한다.

```sql
EXEC master..xp_cmdshell 'whoami';
EXEC master..xp_cmdshell 'whoami /priv';
GO
```

확인할 출력:

- SQL Server 서비스 계정 예: `nt service\mssql$sqlexpress`.
- 현재 서비스 계정의 특권. `SeImpersonatePrivilege`는 후속 로컬 권한 상승 후보이며 그 자체로 SYSTEM 실행 증거가 아니다.
- `xp_cmdshell` 비활성 오류이면 다음 활성화 분기로 이동하고, 권한 거부이면 현재 login의 `sysadmin` 또는 명시적 실행 권한을 다시 확인한다.

### xp_cmdshell 활성화

#### impacket-mssqlclient 대화형 프롬프트

```text
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami
```

`enable_xp_cmdshell`은 Impacket 셸 명령이다. 내부에서 `show advanced options`와 `xp_cmdshell`을 각각 `1`로 설정하고 두 번의 `RECONFIGURE`까지 실행하므로 같은 프롬프트에서 `EXECUTE sp_configure`를 별도로 입력할 필요가 없다.

#### 원시 T-SQL을 실행하는 클라이언트

```sql
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

확인할 출력:

- 설정 변경 성공 후 명령 실행 가능.
- 설정은 바뀌었지만 명령 출력이 없으면 `xp_cmdshell`을 다시 호출해 확인한다. 설정 변경 성공만으로 OS 명령 실행 성공으로 기록하지 않는다.
- `enable_xp_cmdshell`과 원시 T-SQL 절차를 연달아 실행하지 않는다. 둘은 같은 설정 변경을 수행하는 대체 경로다.

### linked server에서 xp_cmdshell 실행

```sql
EXECUTE('EXEC sp_configure ''show advanced options'', 1; RECONFIGURE; EXEC sp_configure ''xp_cmdshell'', 1; RECONFIGURE; EXEC xp_cmdshell ''hostname & whoami /all'';') AT [<LINKED_SERVER>];
GO
```

확인할 출력:

- linked server 쪽 SQL 컨텍스트가 sysadmin일 때 `xp_cmdshell`을 활성화하고 실행한다.
- `hostname`이 목표 서버인지 확인한다.
- `whoami`가 `nt authority\system`이면 해당 호스트에서 SYSTEM 권한으로 명령이 실행된 것이다.
- login timeout·untrusted domain 오류이면 원격 OS 권한 문제가 아니라 linked server 연결·login mapping 문제다. RPC Out 또는 원격 query가 거부되면 linked server 옵션과 현재 impersonation 컨텍스트를 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `whoami`와 `hostname`이 DB query 결과로 반환된다. | SQL Server 서비스 계정으로 OS 명령 실행 확인 | Windows 명령 실행 | 원복 후 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] |
| reverse shell 또는 파일 전송 명령이 동작한다. | 대상에서 외부 연결 또는 파일 쓰기 가능 | Windows 세션 또는 파일 전송 | [[Reverse Shell 획득]] 후 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] |
| `hostname`과 `whoami`에서 목표 호스트의 `nt authority\system` 실행이 확인된다. | SYSTEM 컨텍스트 확인 | SYSTEM 명령 실행 | 원복 후 [[고권한 세션 확보 후 후속 판단]] |
| xp_cmdshell disabled | 기본 비활성 | 시작 상태 유지 | sysadmin 여부, sp_configure 가능 여부 |
| permission denied | sysadmin 아님 | 시작 상태 유지 | [[MSSQL Impersonation 권한 상승]] |
| 명령 실행되나 연결 없음 | egress 차단 | 시작 상태 유지 | 파일 쓰기, SMB hash capture, 다른 포트 |
| `SeImpersonatePrivilege`가 `Enabled` | 서비스 계정 token에 impersonation 특권 존재 | 로컬 권한 상승 후보 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]에서 SYSTEM 명령 실행 검증 |
| linked server에서 실행 위치 혼동 | linked server가 다른 서버 또는 동일 호스트 별칭을 가리킴 | 시작 상태 유지 | `hostname & whoami`, `@@SERVERNAME` 확인 |

## 확인할 출력과 권한

- 판정 기준: `IS_SRVROLEMEMBER`, `hostname`과 `whoami /all`을 확인해 DB 권한과 OS 명령 실행 권한을 구분한다.

## 변경 영향과 복구

`xp_cmdshell`을 이번 검증에서 활성화했고 변경 전 `value_in_use`가 `0`이었을 때만 비활성화한다.

`mssqlclient.py`에서는 다음 셸 명령이 `xp_cmdshell`과 `show advanced options`를 모두 `0`으로 설정하고 각각 `RECONFIGURE`한다.

```text
SQL> disable_xp_cmdshell
```

두 설정이 변경 전에 모두 `0`이었을 때만 이 단축 명령을 그대로 사용한다. `show advanced options`가 원래 `1`이었다면 아래 원시 T-SQL로 각 값을 따로 복구한다.

```sql
EXECUTE sp_configure 'xp_cmdshell', 0;
RECONFIGURE;
GO
```

`show advanced options`도 변경 전 `value_in_use`가 `0`이었을 때 원래 값으로 되돌린다.

```sql
EXECUTE sp_configure 'show advanced options', 0;
RECONFIGURE;
GO
```

linked server를 변경했다면 같은 복구 query를 해당 서버에서 실행한다.

```sql
EXECUTE('EXEC sp_configure ''xp_cmdshell'', 0; RECONFIGURE; EXEC sp_configure ''show advanced options'', 0; RECONFIGURE;') AT [<LINKED_SERVER>];
GO
```

마지막으로 설정값을 다시 조회해 변경 전 `value_in_use`와 일치하는지 확인한다.

## 후속 공격 연결

- [[Reverse Shell 획득]]
- [[상황별 파일 전송]]
- [[MSSQL 서비스 Hash 캡처]]
- [[Certutil로 Windows HTTP 파일 반입]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]
- [[Windows 권한 상승 열거]]
- OS 명령 실행 주체가 로컬 그룹을 변경할 권한이 있고 새 관리자 로그온이 필요함: [[로컬 관리자 그룹 구성원 추가]]

## 관련 상태 라우터

- Windows 명령 실행 또는 일반 사용자 세션: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- MSSQL 인증·query·xp_cmdshell 상태 재평가: [[MSSQL 인증 세션 확보 후 권한과 실행 경로 선택]]
- 로컬 관리자 또는 SYSTEM 실행 확인: [[고권한 세션 확보 후 후속 판단]]

## 관련 서비스

- [[1433_MSSQL]]

## 관련 도구

- [[impacket-mssqlclient]]
- [[sqlcmd]]
- [[sqsh]]
