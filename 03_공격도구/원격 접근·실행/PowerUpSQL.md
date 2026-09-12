---
tags:
  - 환경/ad
  - 서비스/mssql
  - 기능/열거
  - 기능/인증검증
실행환경: ["Windows PowerShell"]
필요조건: ["PowerUpSQL.ps1", "AD 또는 MSSQL 접근", "기능에 따라 SQL credential"]
결과: ["MSSQL 인스턴스", "SPN과 서비스 계정", "SQL query 결과", "인증 결과"]
---

# PowerUpSQL

## 도구 개요

PowerUpSQL은 AD에 등록된 MSSQL SPN·인스턴스를 찾고 지정한 SQL Server에서 query와 인증 검증을 수행하는 PowerShell 도구 모음이다. 도메인의 MSSQL 후보를 수집한 뒤 실제 DB 로그인과 서버 역할을 구분해 확인하는 작업에 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: `PowerUpSQL.ps1`을 불러올 수 있고 AD·MSSQL에 접근 가능한 Windows PowerShell
- 도메인 열거 입력: 현재 도메인 세션 또는 LDAP에 접근 가능한 AD 계정
- query 입력: `<HOST>,<PORT>`, Windows 또는 SQL credential, 실행할 query
- 출력 해석: 인스턴스 발견, SQL 인증 성공, `sysadmin`, OS 명령 실행 권한을 서로 구분한다.

## 표준 사용법

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
Get-SQLQuery -Instance '<HOST>,<PORT>' -Username '<DOMAIN>\<USER>' -Password '<PASSWORD>' -Query '<QUERY>'
```

## 대표 예시

### 도메인 MSSQL 인스턴스와 SPN 열거

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
```

확인할 출력:

- `ComputerName`, `Instance`, `DomainAccount`, `Service`, `Spn`.
- 결과는 AD의 서비스 단서이며 현재 계정의 SQL 로그인 성공을 뜻하지 않는다.

### 지정 credential로 SQL 연결과 버전 확인

```powershell
Get-SQLQuery -Verbose -Instance '<HOST>,1433' -Username '<DOMAIN>\<USER>' -Password '<PASSWORD>' -Query 'SELECT @@VERSION'
```

확인할 출력:

- `<HOST>,1433 : Connection Success.`.
- SQL Server 버전이 담긴 query 결과.

### Kerberoasting으로 복구한 MSSQLSvc 계정 검증

`MSSQLSvc/<MSSQL_FQDN>:<MSSQL_PORT>`의 소유 `SamAccountName`과 복구 비밀번호를 명시한다.

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLQuery -Verbose -Instance '<MSSQL_FQDN>,<MSSQL_PORT>' -Username '<DOMAIN>\<SPN_USER>' -Password '<PASSWORD>' -Query "SELECT CAST(@@SERVERNAME AS varchar(80)) AS server, CAST(SYSTEM_USER AS varchar(80)) AS login, IS_SRVROLEMEMBER('sysadmin') AS sysadmin"
```

`Connection Success.`와 함께 `login`이 `<DOMAIN>\<SPN_USER>`로 반환되는지 확인한다. 버전 또는 대상 인증 설정에 따라 명시적 credential 연결이 실패하면 `runas /netonly`로 해당 AD 계정의 네트워크 로그온 프로세스를 만들고 [[sqlcmd]]의 `-E` 경로를 사용한다.

### 현재 SQL login과 sysadmin 여부 확인

```powershell
Get-SQLQuery -Verbose -Instance '<HOST>,1433' -Username '<DOMAIN>\<USER>' -Password '<PASSWORD>' -Query "SELECT SYSTEM_USER AS LoginName, IS_SRVROLEMEMBER('sysadmin') AS IsSysadmin"
```

확인할 출력:

- 실제 SQL login.
- `IsSysadmin`이 `1`인지 여부.

## 주요 옵션과 함수

| 옵션·함수 | 의미 | 사용하는 상황 |
|---|---|---|
| `Get-SQLInstanceDomain` | AD의 MSSQL SPN과 인스턴스 탐색 | 서비스 계정·대상 DB 식별 |
| `Get-SQLQuery` | 지정 인스턴스에서 query 실행 | 인증과 DB 권한 검증 |
| `-Instance` | `<HOST>,<PORT>` 또는 인스턴스 지정 | 비표준 포트와 명명 인스턴스 |
| `-Username`, `-Password` | 명시적 credential 전달 | 현재 세션과 다른 계정 검증 |
| `-Query` | 실행할 SQL 문 | 버전, login, role 확인 |
| `-Verbose` | 연결 성공·실패 정보 표시 | query 결과와 transport 상태 구분 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `ComputerName`, `Instance`, `Spn` | 도메인에 등록된 MSSQL 대상 후보 | 호스트 활성·포트·실제 SQL 연결 확인 |
| `Connection Success.` | MSSQL 인증과 query transport 성공 | `SYSTEM_USER`, `IS_SRVROLEMEMBER` 확인 |
| 연결은 성공했지만 `SYSTEM_USER`가 기대 계정과 다름 | 의도한 SPN 소유 계정의 인증으로 확정할 수 없음 | 현재 프로세스 token·도구 버전·연결 인증 방식 확인 |
| query 결과 반환 | 현재 login에 해당 query 권한 존재 | DB role과 데이터 접근 범위 확인 |
| 인스턴스는 발견되나 연결 실패 | 오래된 SPN, 방화벽 또는 credential 문제 | DNS·포트·계정 형식·SQL 인증 모드 확인 |
| `IsSysadmin = 1` | 현재 SQL login이 sysadmin | [[MSSQL xp_cmdshell 명령 실행]]의 변경 전 상태 확인 |

## 관련 공격기법

- [[AD 원격 접근 권한 열거]]
- [[DB 인증과 데이터 열거]]
- [[MSSQL xp_cmdshell 명령 실행]]
- [[MSSQL Impersonation 권한 상승]]
