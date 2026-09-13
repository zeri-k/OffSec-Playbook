---
tags:
  - 서비스/mysql
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows"]
필요조건: ["MySQL 인증 정보"]
결과: ["정보", "데이터", "파일"]
---

# mysql

## 도구 개요

`mysql`은 MySQL·MariaDB 서버에 접속해 SQL을 대화형 또는 단일 명령으로 실행하는 명령줄 클라이언트다. 현재 사용자와 권한, 데이터베이스·테이블을 빠르게 확인하거나 결과를 셸 작업에 연결할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 MySQL/MariaDB에 접근 가능한 Linux 또는 Windows 호스트
- 필요한 입력: host, port, 사용자명, 비밀번호와 필요하면 database
- 환경별 입력: TLS mode, socket/프로토콜, 기본 schema와 출력 형식
- `<HOST>`와 `<PORT>`는 MySQL listener(예: `database.example.invalid:3306`)이며, CA 파일은 client 실행 호스트의 PEM 경로다. `USER()`와 `CURRENT_USER()`는 접속 주체와 유효 권한 주체를 각각 보여 주므로 같은 값으로 가정하지 않는다.


## 표준 사용법

`<HOST>`·`<PORT>`는 client host에서 도달하는 MySQL/MariaDB listener(예: `database.example.invalid:3306`)이고 `<USER>`·`<PASSWORD>`는 DB authentication 입력이다. `<CA_FILE>`은 client host의 PEM path이며 database/schema 값은 접속 뒤 기본 namespace를 정할 뿐 현재 effective privilege를 보장하지 않는다.

```bash
mysql -h <host> -P <port> -u <user> -p
```

## 대표 예시

### 대화형 MySQL 접속

```bash
mysql -h <TARGET> -u <USER> -p
```

### 접속과 동시에 인증 계정·grant 확인

```bash
mysql -h <TARGET> -u <USER> -p -e 'SELECT USER(), CURRENT_USER(); SHOW GRANTS FOR CURRENT_USER; SHOW DATABASES;'
```

`-p`는 password prompt를 연다. 비밀번호를 `-p<PASSWORD>`나 `--password=<PASSWORD>`로 명령행에 넣지 않는다. `USER()`는 client가 제시한 사용자와 접속 호스트를, `CURRENT_USER()`는 서버가 실제 인증해 grant를 적용한 `user@host`를 반환한다.

### TLS 검증 오류 분리

```bash
mysql --version
mysql -h <TARGET_FQDN> -u <USER> -p --ssl-mode=VERIFY_IDENTITY --ssl-ca=<CA_FILE>
```

`VERIFY_IDENTITY`는 CA와 서버 호스트명을 검증한다. 실패하면 client 버전·CA 파일·접속 FQDN을 먼저 확인한다. `--ssl-mode=DISABLED`는 평문 접속 허용 여부와 TLS 실패 원인을 구분하는 진단에서만 사용한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-h` | DB 서버 주소 |
| `-P` | 포트 지정 |
| `-u` | 사용자명 |
| `-p` | 비밀번호 입력 또는 지정 |
| `-D` | 기본 DB 선택 |
| `-e` | SQL 한 줄 실행 |
| `--ssl-mode` | SSL/TLS 사용 방식 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Welcome to the MySQL monitor` | MySQL/MariaDB 로그인 성공 | `SHOW DATABASES;`, `SELECT user();`, `SHOW GRANTS;` 확인 |
| `USER()`와 `CURRENT_USER()`가 다름 | 제시한 계정과 서버가 grant를 적용한 인증 계정이 다름 | `SHOW GRANTS FOR CURRENT_USER;`로 실제 권한 확인 |
| database/table 조회 가능 | 읽기 권한 존재 | 계정 정보, 설정값, 애플리케이션 secret 검색 |
| `FILE` 권한 또는 `secure_file_priv` 확인 | 파일 읽기/쓰기 가능성 판단 지점 | `LOAD_FILE()`, `INTO OUTFILE` 가능 여부를 조건별로 검토 |
| `Access denied` | credential 오류 또는 권한 부족 | 사용자 호스트 형식, 비밀번호, 권한 범위 확인 |
| DB는 보이나 테이블 접근 실패 | schema/table 권한 부족 | `SHOW GRANTS;`, 접근 가능한 database 재확인 |
| 파일 기능 실패 | `FILE` 권한 없음 또는 `secure_file_priv` 제한 | `SHOW VARIABLES LIKE 'secure_file_priv';` 확인 |
| 연결 실패 | 포트 차단, bind-address, TLS 문제 | 포트 접근성, `--ssl-mode`, 터널 여부 확인 |

## 관련 공격기법

- [[DB 인증과 데이터 열거]]
- [[DB 서버 파일 수집]]
- [[DB 서버 파일 쓰기 검증]]

## 참고 링크

- [MySQL 8.4: mysql Client Options](https://dev.mysql.com/doc/refman/8.4/en/mysql-command-options.html)
- [MySQL 8.4: Information Functions](https://dev.mysql.com/doc/refman/8.4/en/information-functions.html)
- [MySQL 8.4: SHOW GRANTS](https://dev.mysql.com/doc/refman/8.4/en/show-grants.html)
