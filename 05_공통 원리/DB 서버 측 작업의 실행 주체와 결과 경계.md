# DB 서버 측 작업의 실행 주체와 결과 경계

## 핵심 개념

원격 SQL query 한 줄에는 client, DB 인증 주체, DB 엔진과 운영체제 실행 주체가 함께 등장하지만 같은 권한을 뜻하지 않습니다. DB 로그인은 engine 안에서 query를 제출할 Identity이고, DB role·privilege는 그 query가 특정 기능을 호출할 수 있는지를 결정합니다. 실제 파일·process·네트워크 작업은 DB 서버 호스트의 별도 OS context와 경로에서 일어나며, 그 결과를 웹 서버가 제공하거나 실행하는지는 다시 다른 경계입니다.

| 구성 요소 | 담당하는 판단 | 혼동하면 안 되는 결과 |
|---|---|---|
| DB client와 네트워크 session | 어느 호스트에서 어느 instance로 query를 보냈는지 | client 로컬 파일·process가 자동으로 대상이 됨 |
| DB login·database user·role | query, procedure와 설정 변경을 호출할 DB 권한 | OS 파일 ACL이나 관리자/root 권한 |
| DB engine·기능 | `FILE`, BULK, `UTL_FILE`, OLE, `xp_cmdshell`처럼 서버 측 작업을 수행 | 기능 활성화·호출 성공만으로 실제 파일·명령 결과 성공 |
| OS 실행 Identity | DB service account 또는 기능에 지정된 proxy account의 file·process·network 권한 | DB `sysadmin`과 Windows Administrators·SYSTEM의 동일성 |
| 서버 관점 경로·목적지 | DB 서버가 해석하는 local·UNC path와 outbound address | client가 보는 경로·현재 directory·network route |
| 웹 server와 handler | OS 파일을 URL로 mapping하고 정적 제공 또는 server-side code로 처리 | 파일 존재·HTTP 조회와 command execution의 동일성 |
| 생성 파일·child process·설정 | 실제로 남는 대상 상태 | query session 종료만으로 자동 복구됨 |

## 동작 과정

1. DB client가 network session에서 login하고 DB engine은 login mapping, database context와 role·privilege를 정합니다.
2. query가 서버 파일·process 기능을 호출하면 engine은 해당 기능의 활성 상태와 호출 권한을 먼저 검사합니다.
3. 기능은 DB 서버가 보는 절대 경로·UNC path 또는 command를 해석하고, 실제 OS Identity의 file ACL·logon right·network egress로 작업을 시도합니다. 이 단계의 실패는 유효한 DB login과 모순되지 않습니다.
4. query 결과에 파일 내용, 고유 파일 존재, `hostname`·`whoami` 또는 오류가 반환돼야 해당 서버 측 작업의 결과를 판정할 수 있습니다. query 자체의 완료 메시지만으로 다음 상태를 확대하지 않습니다.
5. 파일이 web root로 예상되더라도 web server의 URL mapping과 static access를 별도로 확인합니다. script extension·handler와 execute policy가 맞고 command 출력이나 callback이 확인돼야 server-side execution입니다.

MSSQL `xp_cmdshell`은 실행 주체가 고정돼 있지 않은 대표 예입니다. `sysadmin`이 호출하면 Windows child process는 SQL Server service account context에서 실행되지만, 실행 권한을 받은 non-sysadmin은 미리 구성된 `##xp_cmdshell_proxy_account##` credential의 Windows account를 사용합니다. 어느 경우든 `whoami`·`hostname`으로 실제 Identity와 host를 확인하며 DB `sysadmin`을 OS 관리자나 SYSTEM으로 치환하지 않습니다.

## 조건이 결과에 미치는 영향

- DB 권한과 OS 권한: MySQL `FILE`, MSSQL BULK·OLE·`xp_cmdshell`, Oracle package·DIRECTORY object 권한은 서버 측 기능의 첫 gate입니다. 이어 service 또는 proxy account의 OS ACL과 network policy가 통과해야 합니다.
- 기능 설정: 비활성 기능을 켤 권한과 그 기능을 실행할 권한, 실제 작업 성공은 서로 다른 상태입니다. 임시로 설정을 변경했다면 이전 값을 따로 보존하고 복구합니다.
- 서버 경로: `C:\...`, `/var/...`와 UNC path는 DB server 관점입니다. client host에 같은 경로가 있어도 대상 파일을 증명하지 않습니다.
- 제품별 제한: MySQL `secure_file_priv`, Oracle DIRECTORY object, SQL Server credential·procedure 권한은 OS ACL보다 앞에서 허용 범위를 좁힙니다.
- 웹 처리: HTTP 200과 고유 원문은 URL mapping·정적 제공을 확인할 수 있지만 handler 실행은 아닙니다. command 출력·지연·callback은 각 증거 수준에 맞춰 별도로 판정합니다.
- 복구: 읽기 query는 보통 생성 자원이 없지만 파일 쓰기, child process, listener와 server configuration 변경은 exact 경로·PID·이전 설정값으로 정리해야 합니다. DB session 종료는 이 자원의 부재를 보장하지 않습니다.

## 실전에서의 해석

| 관찰 | 확정할 수 있는 것 | 다음에 확인할 것 |
|---|---|---|
| DB prompt와 단순 query 성공 | 해당 instance의 DB 인증·query context | 현재 login·role과 필요한 기능 권한 |
| file function이 실제 내용을 반환 | DB 기능과 OS context가 해당 server path를 읽음 | 자료 분류와 후속 사용, 다른 파일·쓰기 권한은 별도 |
| 고유 proof file 존재 | 지정 OS context로 exact server file 생성 | HTTP mapping, static 제공과 handler 실행 |
| URL에서 proof 원문 반환 | web server가 해당 file을 정적으로 제공 | extension handler와 server-side command output |
| `hostname`·`whoami`가 query 결과로 반환 | 표시된 host·OS Identity로 command 실행 | Windows 프로세스 액세스 토큰의 privilege, 관리자/root 여부와 별도 session 획득 |
| callback 또는 새 shell 연결 | 해당 network path와 process 실행 결과 | 원격 Identity·권한, 생성 process·file·listener 정리 |

제품별 query, 성공 출력, 실패 분기와 복구 명령은 [[DB 서버 파일 수집]], [[DB 서버 파일 쓰기 검증]], [[MSSQL xp_cmdshell 명령 실행]], [[Oracle TNS 서비스]]와 [[웹 파일 업로드와 Web Shell]]에 둡니다.

## 참고 링크

- [MySQL 8.4: Privileges Provided by MySQL](https://dev.mysql.com/doc/refman/8.4/en/privileges-provided.html)
- [MySQL 8.4: Making MySQL Secure Against Attackers](https://dev.mysql.com/doc/refman/8.4/en/security-against-attack.html)
- [MySQL 8.0: secure_file_priv](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_secure_file_priv)
- [Microsoft: xp_cmdshell](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql)
- [Microsoft: sp_OACreate](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-oacreate-transact-sql)
- [Oracle Database 19c: UTL_FILE](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/UTL_FILE.html)
- [Microsoft IIS: Handler mappings](https://learn.microsoft.com/iis/configuration/system.webserver/handlers/)
