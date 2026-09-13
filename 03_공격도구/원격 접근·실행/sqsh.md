---
tags:
  - 서비스/mssql
  - 기능/프로토콜접근
실행환경: ["Linux"]
필요조건: ["MSSQL 또는 Sybase 인증 정보"]
결과: ["쿼리 결과", "데이터"]
---

# sqsh

## 도구 개요

`sqsh`는 Tabular Data Stream(TDS)을 사용하는 MSSQL 또는 Sybase 서버에 접속해 query를 실행하는 Linux 명령줄 클라이언트다. 가벼운 대화형 프롬프트에서 현재 login과 DB 권한을 확인하는 작업에 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: `sqsh`가 설치된 Linux 호스트
- 입력: MSSQL 또는 Sybase 서버와 계정
- 선택 입력: 기본 데이터베이스와 interfaces 파일
- `<SERVER>`는 interfaces 별칭 또는 MSSQL/Sybase listener 주소(예: `database.example.invalid`)다. 사용자·비밀번호는 DB 인증 입력이며, Linux의 interfaces 파일 경로가 설정된 경우 해당 별칭의 실제 host·port를 먼저 확인한다.


## 표준 사용법

`<SERVER>`는 interfaces alias 또는 MSSQL/Sybase TDS listener, `<USER>`·`<PASSWORD>`는 DB authentication input이며 `<DATABASE>`는 optional default namespace다. query output은 현재 DB login의 query permission을 보여 줄 뿐 host OS command execution을 뜻하지 않는다.

```bash
sqsh -S <server> -U <user> -P <password>
```

## 대표 예시

### MSSQL/Sybase 대화형 쿼리 실행

```bash
sqsh -S <TARGET> -U sa -P '<PASSWORD>'
1> select @@version
2> go
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-S` | 서버 이름 또는 주소 |
| `-U`, `-P` | 사용자명과 비밀번호 |
| `-D` | 기본 DB 지정 |
| `-I` | interfaces 파일 지정 |
| `-h` | 헤더 출력 간격 조정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SQL 프롬프트 또는 쿼리 결과 출력 | MSSQL 로그인 성공 | `SELECT SYSTEM_USER;`, `SELECT IS_SRVROLEMEMBER('sysadmin');` 확인 |
| database/table 조회 가능 | 읽기 권한 존재 | 민감 테이블, linked server, credential 저장 위치 확인 |
| `xp_cmdshell` 출력 | OS 명령 실행 가능 | 실행 계정 권한과 파일 쓰기/셸 획득 가능성 확인 |
| `Login failed` | 계정 또는 인증 방식 불일치 | SQL 계정 형식과 서버 인증 정책 확인 |
| 권한 오류 | 일반 DB 사용자 권한 | role, database와 linked server 권한 확인 |
| `xp_cmdshell` 오류 | 기능 비활성화 또는 권한 부족 | sysadmin 여부와 advanced options 상태 확인 |
| timeout 또는 연결 실패 | 포트 차단, TLS 또는 instance 문제 | 포트, instance와 터널 여부 확인 |

## 관련 공격기법

- [[DB 인증과 데이터 열거]]
- [[DB 서버 파일 쓰기 검증]]
- [[MSSQL Impersonation 권한 상승]]
- [[MSSQL Linked Server 내부 이동]]
- [[MSSQL xp_cmdshell 명령 실행]]
