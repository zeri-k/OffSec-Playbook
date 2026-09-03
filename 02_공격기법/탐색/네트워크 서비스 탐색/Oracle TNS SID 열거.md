---
tags:
  - 서비스/oracle
시작조건: ["Oracle TNS 서비스 식별"]
필요조건: ["명령 실행 호스트에서 Oracle TNS 1521/TCP 접근", "SID 또는 service name 후보를 시험할 도구"]
결과: ["유효한 Oracle SID 또는 service name 후보"]
---

# Oracle TNS SID 열거

## 한 줄 판단

명령 실행 호스트에서 대상 Oracle TNS 1521/TCP에 도달할 수 있으면 listener가 받아들이는 System Identifier(SID) 또는 service name 후보를 확인하고, 계정 추측과 인증 후 데이터 조회는 별도 기법으로 넘긴다.

## 사용할 때

- Oracle listener는 응답하지만 접속에 사용할 SID 또는 service name을 모를 때.
- `ORA-12505`, `ORA-12514`처럼 접속 식별자 오류와 계정 인증 오류를 분리해야 할 때.
- 계정 후보를 반복 검증하기 전에 정확한 DB 인스턴스 이름을 좁힐 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 네트워크 경로 | 명령 실행 호스트에서 1521/TCP 도달 | listener 응답 또는 Oracle 오류 확인 | 대상 주소·포트·방화벽·피벗 경로 확인 |
| 대상 서비스 | Oracle TNS listener | 배너와 Oracle 오류 형식 확인 | 일반 TCP listener와 구분 |
| 입력 후보 | 기본 SID·환경에서 얻은 service name 또는 도구 wordlist | 후보 출처와 대소문자 확인 | 설정 파일·웹·배포 단서에서 후보 보강 |

## 실행

Oracle 계정을 입력하지 않고 listener가 받아들이는 SID 또는 service name 후보를 확인한다.

```bash
nmap -p1521 -sV --script oracle-sid-brute <TARGET>
```

확인할 출력:

- 유효하다고 반환된 SID 또는 service name 후보.
- listener 응답과 SID 발견을 구분한다. 이 결과는 유효한 Oracle 계정이나 schema 접근을 증명하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| SID 또는 service name이 반환됨 | listener가 해당 접속 식별자를 인식 | Oracle 접속 식별자 | 계정 후보가 있으면 [[원격 비밀번호 공격]], 이미 유효한 계정이 있으면 [[DB 인증과 데이터 열거]] |
| listener는 응답하지만 후보가 없음 | wordlist 부적합 또는 listener 제한 가능 | 접속 식별자 미확정 | `tnsnames.ora`, 웹·배포·백업 파일 단서 확인 |
| `ORA-12505`·`ORA-12514` | 요청한 SID 또는 service name이 listener 등록과 맞지 않음 | 접속 식별자 오류 | SID와 service name 형식을 바꿔 재확인 |
| timeout·연결 실패 | SID 판정 전 네트워크 단계 실패 | Oracle TNS 상태 미확정 | 주소·포트·route·방화벽 확인 |

## 확인할 출력과 권한

- SID 또는 service name은 로그인 대상만 확정한다.
- 계정·비밀번호 검증, DB role, hash·데이터·파일 접근은 각각 별도 성공 단계다.

## 후속 공격 연결

- Oracle 계정 후보 검증: [[원격 비밀번호 공격]]
- 유효한 Oracle 계정의 데이터·role 확인: [[DB 인증과 데이터 열거]]

## 관련 서비스

- [[1521_Oracle_TNS]]

## 관련 도구

- [[nmap]]
- [[odat]]
