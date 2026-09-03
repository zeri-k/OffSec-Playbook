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


## 표준 사용법

```bash
mysql -h <host> -P <port> -u <user> -p
```

## 대표 예시

### 대화형 MySQL 접속

```bash
mysql -h <TARGET> -u <USER> -p
```

### 접속과 동시에 DB 목록 확인

```bash
mysql -h <TARGET> -u <USER> -p'<PASSWORD>' -e 'SHOW DATABASES;'
```

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
| database/table 조회 가능 | 읽기 권한 존재 | 계정 정보, 설정값, 애플리케이션 secret 검색 |
| `FILE` 권한 또는 `secure_file_priv` 확인 | 파일 읽기/쓰기 가능성 판단 지점 | `LOAD_FILE()`, `INTO OUTFILE` 가능 여부를 조건별로 검토 |
| `Access denied` | credential 오류 또는 권한 부족 | 사용자 호스트 형식, 비밀번호, 권한 범위 확인 |
| DB는 보이나 테이블 접근 실패 | schema/table 권한 부족 | `SHOW GRANTS;`, 접근 가능한 database 재확인 |
| 파일 기능 실패 | `FILE` 권한 없음 또는 `secure_file_priv` 제한 | `SHOW VARIABLES LIKE 'secure_file_priv';` 확인 |
| 연결 실패 | 포트 차단, bind-address, TLS 문제 | 포트 접근성, `--ssl-mode`, 터널 여부 확인 |

## 관련 공격기법

- [[DB 인증과 데이터 열거]]
- [[DB 서버 파일 수집]]
