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

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 네트워크 경로 | 명령 실행 호스트에서 1521/TCP 도달 | listener 응답 또는 Oracle 오류 확인 | 대상 주소·포트·방화벽·피벗 경로 확인 |
| 대상 서비스 | Oracle TNS listener | 배너와 Oracle 오류 형식 확인 | 일반 TCP listener와 구분 |
| 입력 후보 | 기본 SID·환경에서 얻은 service name 또는 도구 wordlist | 후보 출처와 대소문자 확인 | 설정 파일·웹·배포 단서에서 후보 보강 |

## 실행

`<TARGET>`은 Oracle listener 주소(가상 예시 `192.0.2.152`)이고, `<SID_WORDLIST>`·`<SERVICE_WORDLIST>`는 공격 호스트에서 준비한 후보 목록 경로다. `<SID>`·`<SERVICE>`는 아래 열거 출력에서 확인한 서로 다른 접속 식별자이며, SID와 service name을 같은 값으로 가정하지 않는다. 명령은 listener에 도달하는 공격 호스트에서 실행한다.

Oracle 계정을 입력하지 않고 listener가 받아들이는 SID 또는 service name 후보를 확인한다. Nmap 절차는 SID를, ODAT의 두 module은 SID와 service name을 각각 확인하므로 현재 필요한 식별자에 맞는 경로를 선택한다.

### Nmap으로 SID 후보 확인

```bash
nmap -p1521 -sV --script oracle-sid-brute <TARGET>
```

확인할 출력:

- 유효하다고 반환된 SID 또는 service name 후보.
- listener 응답과 SID 발견을 구분한다. 이 결과는 유효한 Oracle 계정이나 schema 접근을 증명하지 않는다.

### ODAT로 SID와 service name 분리 확인

설치된 버전의 module·옵션을 먼저 확인한다.

```bash
odat sidguesser --help
odat snguesser --help
odat sidguesser -s <TARGET> -p 1521 --sids-file <SID_WORDLIST>
odat snguesser -s <TARGET> -p 1521 --service-name-file <SERVICE_NAME_WORDLIST>
```

확인할 출력:

- `sidguesser`의 valid SID와 `snguesser`의 valid Service Name을 따로 기록한다.
- `ORA-12519`나 연결 포화 단서가 보이면 더 많은 재시도를 즉시 늘리지 말고 listener 상태와 현재 시도 속도를 확인한다.
- 반환된 식별자는 인증 대상 후보일 뿐이다. 유효한 DB 계정·role·데이터 접근은 다음 절차에서 확인한다.

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

- [[Oracle TNS 서비스]]

## 관련 도구

- [[nmap]]
- [[odat]]

## 참고 링크

- [Nmap `oracle-sid-brute`](https://nmap.org/nsedoc/scripts/oracle-sid-brute.html)
- [ODAT 공식 저장소](https://github.com/quentinhardy/odat)
