---
tags:
  - 서비스/oracle
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows", "Unix"]
필요권한: ["로컬 SYSDBA 접속 시 OSDBA 권한"]
필요조건: ["Oracle 계정 또는 로컬 Oracle 세션", "원격 접속 시 Service Name 또는 SID"]
결과: ["쿼리 결과", "데이터"]
---

# sqlplus

## 도구 개요

`sqlplus`는 Oracle Database에 원격 계정 또는 로컬 SYSDBA 컨텍스트로 접속해 SQL과 script를 실행하는 명령줄 클라이언트다. 현재 사용자와 역할, 조회 가능한 객체·데이터를 직접 확인하는 Oracle 작업에 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: Oracle SQL*Plus client가 설치된 Linux, Windows 또는 Unix 호스트
- 원격 입력: Oracle 서버, 포트, Service Name 또는 SID와 계정
- 로컬 입력: Oracle 세션과 OSDBA 권한


## 표준 사용법

```bash
sqlplus <user>/<password>@<host>:<port>/<service>
```

## 대표 예시

### Oracle service name으로 원격 접속

```bash
sqlplus scott/tiger@<TARGET>:1521/XE
```

### 로컬 SYSDBA 접속

```bash
sqlplus / as sysdba
```

## 주요 옵션

| 옵션/형식 | 설명 |
| --- | --- |
| `user/pass@host:port/service` | 원격 Oracle 접속 문자열 |
| `-L` | 로그인 실패 시 재시도 없이 종료 |
| `/ as sysdba` | 로컬 SYSDBA 접속 |
| `@script.sql` | SQL 스크립트 실행 |
| `SET` | 출력 형식과 SQLPlus 환경 조정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Connected` | Oracle 계정으로 접속 성공 | `SELECT * FROM v$version;`, 현재 사용자/권한 확인 |
| schema/table 조회 가능 | 읽기 권한 존재 | 민감 테이블, credential, 설정값 검색 |
| `ORA-01031` | 권한 부족 | 현재 role, DBA 권한, 실행 가능한 package 확인 |
| `ORA-01017` / `ORA-12514` | 인증 실패 또는 service name/SID 문제 | 계정, 비밀번호, SID/service name, listener 상태 재확인 |
| 기능 실행 실패 | package 비활성화 또는 권한 없음 | `UTL_FILE`, Java, external table 등 기능별 권한 확인 |

## 관련 공격기법

- [[DB 인증과 데이터 열거]]
- [[1521_Oracle_TNS#선택적 UTL_FILE 텍스트 쓰기 proof|Oracle UTL_FILE 텍스트 쓰기 검증]]
