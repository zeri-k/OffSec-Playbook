---
tags:
  - 환경/ad
시작조건: ["도메인과 DC 식별", "유효한 AD 계정 자격 증명 또는 도메인 사용자 세션 확보"]
필요권한: ["현재 AD 계정에 허용된 객체·관계 읽기 권한", "선택한 호스트 접촉 수집에 필요한 대상별 권한"]
필요조건: ["Windows에서 SharpHound 또는 Linux에서 bloodhound-python 실행 가능", "DC·DNS 접근", "BloodHound 분석 환경"]
결과: ["BloodHound ZIP·JSON", "사용자·그룹·컴퓨터·ACL·GPO·trust·세션 관계", "직접 검증할 공격 경로 후보"]
---

# AD 관계 그래프 수집과 공격 경로 식별

## 한 줄 판단

유효한 AD 계정과 DC·DNS 경로가 있으면 Windows의 SharpHound 또는 Linux의 bloodhound-python으로 필요한 객체·관계 범위를 수집하고, BloodHound의 경로를 원본 LDAP·SMB·호스트 조회로 재검증할 후보로 사용한다.

## 사용할 때

- 사용자·그룹·컴퓨터 목록을 수동으로 얻었지만 중첩 그룹, ACL, GPO, 세션과 원격 접근 관계를 함께 분석해야 할 때.
- 현재 사용 중인 계정에서 고가치 객체까지 이어지는 관계를 우선순위화할 때.
- 현재 도메인과 신뢰 대상 도메인의 외부 그룹 관계를 한 그래프에서 비교할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 계정 | 계정명이 확인된 비밀번호·NT hash 또는 도메인 사용자 세션 | LDAP 인증과 현재 사용자 확인 | 계정·도메인 형식 재확인 |
| DC·DNS 경로 | 대상 DC FQDN 해석과 LDAP 접근 | DNS·LDAP 응답 | `-ns`, FQDN, route·피벗 확인 |
| 수집 범위 | `DCOnly`, 특정 method 또는 `All` 중 목적에 맞는 범위 | method와 호스트 접촉 여부 확인 | 먼저 DC 중심 수집으로 범위 축소 |
| 결과 분석 | 생성 ZIP·JSON을 BloodHound에 업로드 가능 | 파일 존재·크기와 ingest 결과 | 출력 경로·수집 오류 확인 |

## 실행

### 현재 도메인 대상

#### Linux 공격 호스트에서 실행

기존 수집물과 섞이지 않는 전용 디렉터리를 만들고, 도구가 출력한 ZIP의 exact 경로를 `<CURRENT_DOMAIN_ZIP_PATH>`로 기록한다. 비밀번호는 command line에 넣지 않고 prompt에 입력한다.

```bash
test ! -e '<BLOODHOUND_RUN_DIRECTORY>'
mkdir -m 700 '<BLOODHOUND_RUN_DIRECTORY>'
cd '<BLOODHOUND_RUN_DIRECTORY>'
bloodhound-python -u '<USER>@<DOMAIN>' -ns <DC_IP> -d <DOMAIN> -c All --zip -op <CURRENT_DOMAIN_OUTPUT_PREFIX>
```

#### Windows 공격 호스트에서 실행

```powershell
.\SharpHound.exe -c All --zipfilename <OUTPUT_NAME>
```

전체 수집 전에 도메인 객체 중심 결과만 필요하면 도구 문서의 `DCOnly`와 개별 collection method를 확인한다.

확인할 출력:

- 발견한 domain, user, group, computer, trust 수와 생성된 ZIP·JSON.
- `Enumeration Completed`와 호스트별 접속 오류.
- 일부 호스트 실패를 전체 LDAP 객체 수집 실패와 분리한다.

### 신뢰 대상 도메인 대상

양쪽 도메인을 각각 수집한 뒤 같은 BloodHound 데이터베이스에서 외부 그룹 멤버십과 trust 경로를 확인한다. 다음 명령은 Legacy BloodHound 4.2/4.3 계열 `bloodhound-python` 기준이다. BloodHound CE용 collector는 설치한 `bloodhound-ce-python`의 `--help`와 서버 호환성을 먼저 확인한다.

호스트 접촉이 필요 없는 trust·group·ACL 후보를 먼저 얻기 위해 `DCOnly`로 시작한다. source-domain credential이 target trust domain의 LDAP 조회에 받아들여지는지는 두 번째 명령의 인증 결과로 별도 확인한다. 각 명령이 출력한 ZIP exact 경로를 `<SOURCE_ZIP_PATH>`와 `<TARGET_ZIP_PATH>`로 기록한다.

현재 도메인 절차의 전용 디렉터리를 만들지 않았다면 신뢰 수집 전에도 같은 `test`·`mkdir`·`cd` 세 명령을 먼저 수행한다.

```bash
bloodhound-python -d <SOURCE_DOMAIN> -dc <SOURCE_DC_FQDN> -ns <SOURCE_DNS_IP> -c DCOnly --zip -op <SOURCE_OUTPUT_PREFIX> -u '<USER>@<SOURCE_DOMAIN>'
bloodhound-python -d <TARGET_TRUST_DOMAIN> -dc <TARGET_DC_FQDN> -ns <TARGET_DNS_IP> -c DCOnly --zip -op <TARGET_OUTPUT_PREFIX> -u '<USER>@<SOURCE_DOMAIN>'
```

확인할 출력:

- 두 실행의 domain·user·group·trust 수와 서로 다른 ZIP 경로.
- 첫 수집 성공은 source LDAP 읽기만, 둘째 수집 성공은 target LDAP 읽기까지 확인한 상태다.
- collector ZIP을 같은 BloodHound에 ingest한 뒤 표시되는 trust·membership edge는 실행 후보이며 현재 권한 행사를 증명하지 않는다.

## 변경 영향과 복구

수집이 만든 로컬 ZIP·JSON만 정리한다. BloodHound 서버에 업로드했다면 로컬 파일 삭제와 서버 데이터 제거는 별개이며, 공유 분석 서버의 기존 데이터를 일괄 삭제하지 않는다.

| 생성 항목 | 작업 전 확인과 식별 | 정리 명령 | 완료 확인 |
|---|---|---|---|
| Linux 전용 수집 디렉터리 | `test ! -e '<BLOODHOUND_RUN_DIRECTORY>'`; 실행 뒤 도구가 출력한 exact ZIP 경로 기록 | ingest·분석이 끝난 뒤 `rm -- '<CURRENT_DOMAIN_ZIP_PATH>'` 또는 신뢰 수집이면 `rm -- '<SOURCE_ZIP_PATH>' '<TARGET_ZIP_PATH>'`; 생성된 JSON이 남았다면 출력에서 확인한 exact 경로만 `rm -- '<EXACT_JSON_PATH>'`; 이어서 `cd ..`와 `rmdir '<BLOODHOUND_RUN_DIRECTORY>'` | `test ! -e '<BLOODHOUND_RUN_DIRECTORY>'` |
| Windows SharpHound ZIP | 실행 전 `<SHARPHOUND_ZIP_PATH>` 부재 확인, 명령이 생성한 exact 경로 기록 | BloodHound ingest 뒤 `Remove-Item -LiteralPath '<SHARPHOUND_ZIP_PATH>'` | `Test-Path -LiteralPath '<SHARPHOUND_ZIP_PATH>'`가 `False` |

수집 시 발생한 LDAP·SMB·RPC 접근 기록과 이미 업로드한 분석 서버 데이터는 이 파일 정리로 되돌아가지 않는다. ZIP 경로가 불명확하면 이름 pattern으로 삭제하지 말고 collector 출력과 생성 시각을 먼저 대조한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 사용자·그룹·컴퓨터 JSON 생성 | LDAP 객체 수집 성공 | 그래프 객체 자료 | 그룹 경로는 [[AD 고권한 그룹과 중첩 구성원 열거]]로 직접 확인 |
| `GenericAll`·`GenericWrite`·`WriteDACL`·복제 edge | 객체 제어권 후보 | ACL 검증 대상 | [[AD ACL 권한 열거와 공격 경로 식별]] |
| `AdminTo`·`CanRDP`·`CanPSRemote`·세션 edge | 호스트 접근·세션 후보 | 원격 접근 검증 대상 | [[AD 원격 접근 권한 열거]], [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]] |
| trust·외부 그룹 관계 | 교차 도메인 권한 후보 | trust 경로 후보 | [[AD 도메인 트러스트 열거와 공격 경로 식별]] |
| 호스트 접촉 오류 다수 | 방화벽·오프라인·권한 또는 수집 범위 문제 | 부분 수집 | DC 중심 결과와 호스트별 SMB·관리 서비스 도달성 분리 확인 |

## 확인할 출력과 권한

- BloodHound의 shortest path와 edge는 조사 순서다. 현재 권한 행사나 공격 성공을 입증하지 않는다.
- 수집 시점이 오래되면 세션·로컬 관리자·컴퓨터 상태가 달라질 수 있다. 중요한 edge는 원본 조회로 다시 확인한다.

## 관련 도구

- [[bloodhound-python]]
- [[SharpHound]]
- [[BloodHound]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- 객체 제어권 확인 후: [[AD 객체 제어권 확보 후 악용 경로 선택]]

## 참고 링크

- [BloodHound.py 공식 저장소와 Legacy·CE collector 구분](https://github.com/dirkjanm/BloodHound.py)
- [BloodHound.py CLI 구현](https://github.com/dirkjanm/BloodHound.py/blob/master/bloodhound/__init__.py)
