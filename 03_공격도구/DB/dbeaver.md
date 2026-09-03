---
tags:
  - 서비스/mysql
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows"]
필요조건: ["대상 DB 인증 정보"]
결과: ["정보", "데이터"]
---

# dbeaver

## 도구 개요

DBeaver는 JDBC driver를 통해 MySQL, MSSQL 등 여러 DB에 접속하고 schema, 테이블, 데이터를 GUI에서 탐색하는 클라이언트다. 명령줄보다 구조를 시각적으로 훑고 여러 종류의 DB 연결을 한곳에서 관리하기 좋은 도구다.

## 필요한 입력과 실행 환경

- 실행 위치: GUI를 사용할 수 있는 Linux 또는 Windows 호스트와 DB별 JDBC driver
- 필요한 입력: DB 종류, host, port, database/SID/service, 사용자명과 비밀번호
- 환경별 입력: SSL/TLS와 DB별 driver property; 연결 전 대상 DB에 대한 네트워크 접근이 필요하다.


## 표준 사용법

1. 새 Database Connection을 만든다.
2. DB 종류를 선택한다.
3. Host, Port, Database/SID, Username, Password를 입력한다.
4. Test Connection으로 인증과 네트워크 연결을 확인한다.

## 대표 예시

### MySQL 계정으로 GUI 연결 검증

```text
Driver: MySQL
Host: <TARGET>
Port: 3306
Database: app
Username: <USER>
Password: <password>
```

### MSSQL 계정으로 테이블 탐색

```text
Driver: SQL Server
Host: <TARGET>
Port: 1433
Username: <USER>
Password: <password>
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| Host/Port | DB 서버 주소와 포트 |
| Database/SID/Service | 접속할 DB 이름 또는 Oracle 식별자 |
| Username/Password | 인증 정보 |
| SSL/TLS | 암호화 연결 여부 |
| Driver Properties | DB별 세부 JDBC 옵션 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 연결 성공 및 schema/tree 표시 | credential로 DB 접근 가능 | DB/테이블 목록, 권한, 민감 데이터 위치 확인 |
| 테이블 조회 가능 | 데이터 읽기 권한 존재 | 계정, 토큰, 설정값, 개인정보 등 영향 범위 정리 |
| 권한 오류 | 접속은 됐지만 조회/수정 권한 제한 | 현재 사용자 권한과 접근 가능한 schema 확인 |
| 연결 실패 | 호스트, 포트, DB 종류, SSL, credential 문제 | CLI 클라이언트로 같은 값 재검증 |

## 관련 공격기법

- [[DB 인증과 데이터 열거]]
