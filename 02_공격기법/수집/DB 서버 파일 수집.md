---
시작조건: ["DB 인증 세션 확보", "DB 서버 파일 읽기 권한 후보 확인"]
필요권한: ["MySQL FILE 또는 MSSQL BULK 파일 읽기 권한", "DB 서비스 계정의 대상 OS 파일 읽기 권한"]
필요조건: ["유효한 DB 계정", "읽을 서버 파일 경로", "MySQL secure_file_priv 또는 MSSQL BULK 접근 조건"]
결과: ["DB 서버의 OS 파일 내용", "설정·계정·네트워크·자격 증명 후보"]
---

# DB 서버 파일 수집

## 한 줄 판단

현재 MySQL 또는 Microsoft SQL Server(MSSQL) 로그인에 서버 파일 읽기 기능을 사용할 권한이 있고 DB 서비스 계정이 대상 운영체제 파일을 읽을 수 있으면, DB query로 파일 내용을 반환받아 설정·계정·네트워크 단서를 수집한다.

## 사용할 때

- DB 로그인 후 MySQL `FILE` 또는 MSSQL BULK 접근 권한 단서가 있을 때.
- DB 서비스 계정이 읽을 수 있는 설정·호스트·애플리케이션 파일을 확인할 때.
- DB query 성공과 실제 운영체제 파일 내용 반환을 구분해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| DB 세션 | MySQL 또는 MSSQL query 실행 가능 | [[DB 인증과 데이터 열거]] | 인증 방식·DB context·현재 login 확인 |
| DB 권한 | MySQL `FILE` 또는 MSSQL BULK 접근 | `SHOW GRANTS`, query 권한 오류 | DB 권한과 서비스 계정 OS 권한을 구분 |
| 파일 경로 | DB 서버 관점의 로컬·UNC 경로 | 절대 경로와 DB 서버 OS 확인 | 클라이언트 경로와 혼동하지 않음 |
| MySQL 경로 제한 | `secure_file_priv` 값 확인 | 허용 경로 또는 빈 값 | `NULL` 원인을 파일 부재와 분리 |

## 실행

### MySQL 서버 파일 읽기

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
SELECT LOAD_FILE('/etc/passwd');
```

확인할 출력:

- query 결과에 실제 파일 내용이 반환되는지.
- `NULL`이면 파일 부재, `secure_file_priv`, 현재 DB 계정의 `FILE` 권한과 MySQL 서비스 계정의 OS 읽기 권한을 각각 확인한다.

### MSSQL 서버 파일 읽기

```sql
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents;
```

확인할 출력:

- query 결과에 실제 파일 내용이 반환되는지.
- 권한·파일 접근 오류이면 현재 login의 BULK 권한, SQL Server 서비스 계정이 보는 로컬·UNC 경로와 OS ACL을 구분한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| OS 파일 내용이 query 결과로 반환 | DB 기능과 서비스 계정 권한으로 해당 파일을 읽음 | 서버 파일 수집 | 파일 안의 설정·계정·내부 주소를 목적별로 검증 |
| MySQL `NULL` | 파일 부재·경로 제한·DB 권한·OS 권한 중 하나 | 파일 읽기 미확정 | `secure_file_priv`, `FILE`, 절대 경로와 OS ACL 확인 |
| MSSQL 권한 오류 | BULK 또는 대상 파일 OS 권한 부족 | 파일 읽기 실패 | 현재 login과 SQL Server 서비스 계정 권한 분리 |
| 자격 증명·key 발견 | 실제 인증 전 후보 | 자격 증명 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| 내부 hostname·IP 발견 | 추가 서비스 조사 후보 | 내부 네트워크 단서 | 현재 명령 실행 위치에서 DNS·포트 도달성 확인 |

## 확인할 출력과 권한

- DB query 실행 성공과 대상 운영체제 파일 내용 반환은 다른 상태다.
- 파일 읽기 성공은 파일 쓰기, Web Shell 또는 운영체제 명령 실행 권한을 뜻하지 않는다.
- 수집한 값은 대상 서비스에서 검증하기 전까지 후보로 유지한다.

## 후속 공격 연결

- 수집한 계정·비밀번호·key: [[원격 비밀번호 공격]], [[확보한 자격 증명으로 원격 접근 경로 선택]]
- 암호화 파일·hash: [[오프라인 해시 크래킹]], [[보호된 파일 및 아카이브 크래킹]]

## 관련 서비스

- [[3306_MySQL]]
- [[1433_MSSQL]]

## 관련 도구

- [[mysql]]
- [[impacket-mssqlclient]]
