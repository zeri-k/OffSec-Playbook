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

```bash
bloodhound-python -u '<USER>' -p '<PASSWORD>' -ns <DC_IP> -d <DOMAIN> -c all
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

양쪽 도메인을 각각 수집한 뒤 같은 BloodHound 데이터베이스에서 외부 그룹 멤버십과 trust 경로를 확인한다.

```bash
bloodhound-python -d <SOURCE_DOMAIN> -dc <SOURCE_DC_FQDN> -c All -u <USER> -p '<PASSWORD>'
bloodhound-python -d <TARGET_TRUST_DOMAIN> -dc <TARGET_DC_FQDN> -c All -u '<USER>@<SOURCE_DOMAIN>' -p '<PASSWORD>'
zip -r trusted-forest-bloodhound.zip *.json
```

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
