---
tags:
  - 서비스/mysql
대표포트:
  - "T:3306"
서비스:
  - MySQL
---

# MySQL 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:3306`의 MySQL에 연결할 수 있고, 아직 유효한 데이터베이스(DB) 계정이나 DB·서버 파일 권한은 확인하지 않은 상태에서 시작한다. 직접 DB 인증은 웹 입력을 통한 Structured Query Language(SQL) Injection과 구분하고, `user@host` 인증 조건, DB·테이블 읽기, 관리 권한, `FILE` 권한과 `secure_file_priv` 경로를 단계별로 판단한다.

성공하면 해당 `user@host`가 읽을 수 있는 DB·테이블과 grant를 얻는다. 연결 실패 시 bind address·방화벽·TLS를, `Access denied` 시 사용자·비밀번호·source host 조건을, 파일 작업 실패 시 `FILE` 권한·`secure_file_priv`·서버 파일 시스템 권한을 다시 확인한다. DB 권한이나 서버 파일 쓰기 가능성은 운영체제 명령 실행 권한과 같지 않다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | MySQL 정보와 빈 비밀번호 | `nmap -sV -sC -p3306 --script mysql-info,mysql-empty-password <TARGET>` | 제품·버전·인증 정보와 빈 비밀번호 후보를 확인한다. |
| 2 | 원격 DB 인증 | `mysql -h <TARGET> -u <USER> -p` | password prompt 후 MySQL prompt가 열리는지 확인하고, `Access denied for user '<USER>'@'<SOURCE_HOST>'`를 소스 호스트 조건과 함께 판독한다. |
| 3 | 제시한 계정과 실제 인증 계정 | `SELECT USER(), CURRENT_USER();`, `SHOW GRANTS FOR CURRENT_USER;` | `USER()`의 client 입력·소스와 `CURRENT_USER()`의 grant 적용 `user@host`를 구분한다. |
| 4 | TLS 오류 분리 | `mysql --version`, `mysql -h <TARGET> -u <USER> -p --ssl-mode=VERIFY_IDENTITY --ssl-ca=<CA_FILE>` | 현 client가 지원하는 옵션과 CA·호스트명 검증 실패를 인증 실패와 분리한다. 암호화 비활성화는 승인된 진단에서만 사용한다. |
| 5 | DB 목록 | `SHOW DATABASES;` | 현재 인증된 계정으로 접근 가능한 DB를 확인한다. |
| 6 | 권한과 파일 경로 | `SHOW VARIABLES LIKE 'secure_file_priv';`, `SHOW GRANTS FOR CURRENT_USER;` | `FILE` 등 실제 grant와 import/export 제한 경로를 연결한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| 빈 비밀번호·기본 계정 또는 확보한 사용자 이름·비밀번호 | [[DB 인증과 데이터 열거]] | `mysql`, `dbeaver` | 해당 `user@host`의 DB 로그인 |
| DB 접근 성공 | [[DB 인증과 데이터 열거]] | `mysql`, `dbeaver` | 접근 가능한 DB·테이블·민감 레코드와 사용자 권한 |
| FILE 권한과 `secure_file_priv` 경로 | [[DB 서버 파일 수집]] | `mysql` | 허용 경로의 서버 파일 내용 |
| FILE 권한과 쓰기 가능한 서버 경로 | 이 문서의 선택적 파일 쓰기 proof | `mysql`, `curl` | DB 서비스 계정의 파일 생성과 별도 HTTP 노출 여부 |
| 플러그인 또는 User-Defined Function(UDF) 조건 | 수동 확인: plugin directory, 파일 생성 권한과 UDF 호출 권한을 각각 확인 | `mysql` | DB·파일·플러그인 조건이 모두 맞는 OS 영향 후보 |

### 선택적 파일 쓰기 proof

`FILE` 권한, `secure_file_priv` 허용 경로와 MySQL 서비스 계정의 운영체제 쓰기 권한이 모두 확인된 경우에만 고유한 텍스트 파일을 만든다.

```sql
SELECT 'db-file-write-proof' INTO OUTFILE '/var/www/html/<UNIQUE_PROOF>.txt';
```

```bash
curl "http://<TARGET>/<UNIQUE_PROOF>.txt"
```

- `Query OK`는 DB 서비스 계정의 파일 쓰기만 의미한다. HTTP에서 고유 문자열이 일치해야 웹에서 제공되는 경로임을 확인할 수 있다.
- `File exists`이면 덮어쓰지 말고 새 파일명을 사용한다.
- 서버 관리 경로로 이번에 만든 파일 하나를 제거할 수 없으면 쓰기 proof를 수행하지 않는다.
- 정적 파일 조회는 서버 측 실행이 아니다. 실행 handler 조건은 [[웹 파일 업로드와 Web Shell]]에서 별도로 확인한다.

## 서비스 고유 주의 사항

- `-p`만 사용해 prompt에 비밀번호를 입력한다. `-p<PASSWORD>` 형식은 공백 없이 동작하지만 shell history와 process 인자에 자격 증명을 남길 수 있어 기본 절차로 사용하지 않는다.
- MySQL 계정은 `user@host` 조건에 따라 원격 접속이 제한될 수 있다.
- TLS 오류와 인증 실패를 구분한다.
- FILE 권한, `secure_file_priv`, 웹 루트 경로와 OS 권한은 각각 별도 조건이다.

## 참고 링크

- [MySQL 8.4: Connection Verification](https://dev.mysql.com/doc/refman/8.4/en/connection-access.html)
- [MySQL 8.4: Information Functions](https://dev.mysql.com/doc/refman/8.4/en/information-functions.html)
- [MySQL 8.4: Encrypted Connections](https://dev.mysql.com/doc/refman/8.4/en/using-encrypted-connections.html)
