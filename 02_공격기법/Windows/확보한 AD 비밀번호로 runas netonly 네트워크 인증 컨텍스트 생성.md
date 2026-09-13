---
tags:
  - 환경/windows
  - 환경/ad
시작조건: ["Windows 셸 또는 GUI 세션 확보", "다른 AD 계정의 사용자명과 평문 비밀번호 확보", "대상 AD 통합 서비스 접근 가능"]
필요권한: ["현재 세션에서 runas 실행", "대상 AD 계정의 원격 서비스 인증 권한"]
필요조건: ["<DOMAIN>\\<USER>", "평문 비밀번호", "인증을 검증할 원격 SMB·LDAP·MSSQL 등 Windows 통합 인증 서비스"]
결과: ["현재 로컬 로그인 계정을 유지하고 원격 인증에 지정 AD 계정을 사용하는 프로세스", "대상 서비스에서 확인한 실제 네트워크 인증 주체와 권한"]
---

# 확보한 AD 비밀번호로 runas netonly 네트워크 인증 컨텍스트 생성

## 한 줄 판단

Windows 셸에서 다른 AD 계정의 사용자명과 평문 비밀번호를 확보했고 SMB·LDAP·MSSQL 같은 원격 Windows 통합 인증 서비스를 사용해야 하면 `runas /netonly`로 네트워크 인증 전용 프로세스를 만들고 대상 서비스에서 실제 인증 주체와 권한을 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 원격 서비스에 연결 가능한 Windows 호스트 | `hostname`, `whoami`, 대상 포트 연결 | 로컬 호스트와 원격 서비스 대상을 구분 |
| 현재 계정 또는 인증 수단 | `<DOMAIN>\<USER>`와 평문 비밀번호 | 도메인명·사용자명 형식 확인 | NT hash·ticket만 있으면 이 기법을 사용하지 않음 |
| 현재 권한 | 현재 세션에서 프로세스 생성 가능, 지정 계정은 원격 서비스 권한 보유 후보 | `runas` 프로세스 생성과 원격 서비스 결과를 분리 확인 | 프로세스 생성 성공만으로 비밀번호 유효성을 판단하지 않음 |
| 공격 대상의 조건 | 대상 서비스가 Windows 통합 인증을 지원 | SMB UNC, LDAP 또는 MSSQL `-E` 등으로 확인 | SQL login·기본 인증처럼 다른 인증 방식이면 해당 서비스 기법 사용 |
| 필요한 파일·목록·주소 | 대상 서비스 FQDN·포트와 실제 인증 주체를 보여 주는 확인 명령 | DNS·SPN·시간과 포트 확인 | IP 대신 FQDN, realm·DNS·시간 정합성 확인 |

## 실행

### 1. 네트워크 인증 전용 CMD 생성

```cmd
runas /netonly /user:<DOMAIN>\<USER> cmd.exe
```

`<DOMAIN>\<USER>`는 원격 서비스에서 사용할 AD 계정 형식이며 `<PASSWORD>`는 prompt에만 입력한다. SMB 예시의 `<DC_FQDN>`은 domain controller FQDN이고, MSSQL 예시의 `<MSSQL_FQDN>`과 `<MSSQL_PORT>`는 SQL Server FQDN과 TCP 포트다. 각 FQDN·포트·SPN은 현재 로컬 `whoami` 결과와 바꾸어 해석하지 않는다.

표시되는 password prompt에 확보한 `<PASSWORD>`를 입력한다. `/netonly`는 이 시점에 원격 서비스로 인증하지 않으므로 프로세스 생성만으로 비밀번호가 유효하다고 판단하지 않는다.

### 2. 새 프로세스에서 로컬 로그인 계정 확인

```cmd
whoami
klist
```

확인할 출력:

- `whoami`는 기존 로컬 로그온 계정을 표시할 수 있다.
- 원격 Kerberos 서비스를 아직 사용하지 않았다면 지정 계정의 ticket이 없을 수 있다.

### 3. 원격 서비스에서 지정 AD 계정 검증

SMB 접근이 목표면 FQDN으로 원격 공유를 요청한다.

```cmd
dir \\<DC_FQDN>\SYSVOL
klist
```

MSSQL Windows 인증이 목표면 같은 CMD에서 실행한다.

```cmd
SQLCMD.EXE -S tcp:<MSSQL_FQDN>,<MSSQL_PORT> -E -Q "SELECT SYSTEM_USER, ORIGINAL_LOGIN(), IS_SRVROLEMEMBER('sysadmin');"
```

확인할 출력:

- SMB는 `SYSVOL` 목록과 접근 오류로 지정 계정의 인증·READ 범위를 판정한다.
- Kerberos를 사용했다면 `klist`에서 지정 계정이 요청한 서비스 ticket을 확인한다.
- MSSQL은 `SYSTEM_USER`와 `ORIGINAL_LOGIN()`이 `<DOMAIN>\<USER>`인지 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 새 CMD가 열리지만 `whoami`는 기존 계정 | `/netonly`의 정상적인 로컬 로그인 계정 유지 | 네트워크 인증 미검증 프로세스 | 원격 서비스 요청으로 지정 계정 검증 |
| 원격 서비스가 `<DOMAIN>\<USER>`로 인증되고 결과를 반환 | 지정 AD 비밀번호와 해당 서비스 권한 확인 | 인증된 AD 서비스 접근 | [[AD Identity 확인 후 도메인 컨텍스트 열거]] 또는 해당 서비스 기법으로 전환 |
| `runas` 프로세스는 열리지만 원격 인증 실패 | 프로세스 생성과 비밀번호 유효성은 별개 | 자격 증명 미검증 또는 실패 | 도메인·사용자·비밀번호, DNS·SPN·시간과 대상 서비스 인증 방식 확인 |
| 원격 인증은 성공하지만 접근 거부 | 지정 AD 계정은 유효하지만 요청한 리소스 권한이 없음 | 인증 성공·권한 부족 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 다른 서비스 권한 확인 |

## 확인할 출력과 권한

- `/netonly` 프로세스의 `whoami`는 지정 AD 계정으로 바뀌지 않는다.
- `Attempting to start`와 새 프로세스 생성은 입력한 비밀번호의 유효성을 증명하지 않는다.
- 실제 인증 주체는 SMB 접근 결과, Kerberos ticket 또는 MSSQL `SYSTEM_USER`처럼 원격 서비스 출력으로 확인한다.
- 원격 인증 성공, READ·WRITE, 원격 명령 실행과 관리자 권한을 각각 구분한다.

## 관련 공격기법

- [[확보한 평문 비밀번호로 runas 사용자 프로세스 실행]]
- [[DB 인증과 데이터 열거]]
- [[Snaffler로 도메인 SMB 공유 민감 파일 탐색]]

## 관련 도구

- [[runas]]
- [[sqlcmd]]

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
