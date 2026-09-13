---
tags:
  - 환경/ad
  - 기능/열거
실행환경: ["Linux", "Windows"]
필요조건: ["SharpHound 또는 bloodhound-python 수집 결과", "BloodHound 데이터베이스"]
결과: ["AD 객체 관계", "공격 경로 후보", "ACL과 세션 단서"]
---

# BloodHound

## 도구 개요

BloodHound는 수집한 AD 객체·세션·ACL 관계를 그래프로 분석해 권한 상승과 내부 이동 경로 후보를 시각화한다. 여러 관계가 이어지는 경로를 탐색하고 우선순위를 정할 때 유용하지만, 그래프 edge는 수집 시점의 후보이며 실제 행사 가능한 권한을 확정하지 않는다.

## 필요한 입력과 실행 환경

- 실행 환경: BloodHound 클라이언트와 지원 데이터베이스를 실행할 수 있는 Linux 또는 Windows 호스트
- 입력: SharpHound 또는 bloodhound-python이 생성한 ZIP/JSON 수집 결과
- 전제: 수집 시점과 수집에 사용한 AD 계정을 기록해 오래된 세션·호스트 정보를 현재 사실로 오해하지 않는다.

## 표준 사용법

```bash
bloodhound
```

수집 파일을 업로드한 뒤 시작 계정 또는 그룹과 목표 객체를 지정하여 경로를 조회한다.

## 대표 예시

### Domain Admin까지의 최단 경로 확인

Analysis에서 `Find Shortest Paths to Domain Admins`를 실행하고 각 edge의 필요 조건을 확인한다.

확인할 출력:

- 사용자·그룹·컴퓨터 사이의 경로와 `MemberOf`, `AdminTo`, `GenericAll` 같은 edge.

### 현재 사용자의 객체 제어권 확인

시작 사용자를 검색한 뒤 `Node Info > Outbound Object Control`과 `Transitive Object Control`을 확인한다.

확인할 출력:

- 직접 제어하는 객체와 그룹 중첩을 거쳐 최종적으로 제어 가능한 객체.
- 도메인 노드에 대한 `GetChanges`와 `GetChangesAll`이 결합된 `DCSync` edge. `GetChangesInFilteredSet`은 `GetChangesAll`의 대체 권한이 아니다.
- 이 그래프는 수집된 ACL과 그룹 관계의 해석 결과다. 실제 요청자의 현재 그룹·deny 상태와 DRSUAPI 성공은 [[AD 계정의 디렉터리 복제 권한 확인]]과 [[DCSync]]에서 확인한다.

### 원격 접근 edge 확인

현재 선택한 시작 계정 또는 그룹에서 컴퓨터로 향하는 `CanRDP`, `CanPSRemote`, `SQLAdmin` edge를 확인한다.

확인할 출력:

- edge가 가리키는 대상 컴퓨터와 수집 시점.
- 현재 BloodHound의 `CanRDP`는 `Remote Desktop Users` 멤버십과 `SeRemoteInteractiveLogonRight` 관계를 결합한 후보지만, deny 정책·현재 listener·인증 정책과 GUI 세션을 확정하지 않는다.
- `CanPSRemote`는 PowerShell Remoting 세션 후보지만 관리자 실행을 보장하지 않는다.
- 실제 RDP·WinRM·MSSQL 접속 전에는 모두 접근 후보로만 취급한다. Restricted Admin 기반 RDP PtH는 `CanRDP`와 별개로 대상 로컬 Administrators 권한을 확인한다.

### 도메인 트러스트 관계 시각화

Analysis에서 `Map Domain Trusts`를 실행해 현재 데이터에 포함된 도메인과 trust edge를 확인한다.

확인할 출력:

- 현재 도메인과 자식·부모·외부 forest 도메인 노드 사이의 trust 관계.
- 그래프 방향은 관계 단서이며 현재 계정으로 상대 도메인 객체 조회와 서비스 인증이 되는지 별도로 확인한다.


## 주요 기능

| 기능 | 의미 | 사용하는 상황 |
|---|---|---|
| Path Finding | 두 AD 객체 사이의 관계 경로 계산 | 권한 상승·내부 이동 후보 탐색 |
| Node Info | 객체 속성, 세션, 권한과 제어 관계 확인 | 특정 사용자·그룹·컴퓨터 조사 |
| Analysis Queries | 자주 쓰는 조건을 미리 작성한 쿼리 | 빠른 초기 분류 |
| Map Domain Trusts | 수집된 도메인 사이의 trust 관계 표시 | 자식·부모·forest trust 후보 시각화 |
| Users with Foreign Domain Group Membership | 다른 도메인 계정이 들어간 대상 도메인 그룹 표시 | forest 간 원격 접근·관리 권한 후보 확인 |
| Raw Query | Cypher로 사용자 지정 조회 | 환경별 조건과 보고서 검증 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| edge와 경로 표시 | 관계 기반 공격 후보 | edge별 실제 권한·도달성·변경 영향 검증 |
| `Outbound Object Control` | 시작 계정 또는 그룹이 제어 가능한 객체 | [[AD ACL 권한 열거와 공격 경로 식별]]로 교차 확인 |
| `DCSync` edge | 도메인에 대한 `GetChanges`와 `GetChangesAll` 조합 후보 | 현재 사용자·유효 그룹 SID, 수집 시점과 실제 DRSUAPI 요청 확인 |
| 세션·로컬 관리자 edge | 원격 접근 또는 자격 증명 노출 후보 | 호스트 활성 상태와 현재 세션을 재검증 |
| `CanRDP`, `CanPSRemote`, `SQLAdmin` | 서비스별 원격 접근 관계 후보 | [[AD 원격 접근 권한 열거]]와 실제 서비스 세션으로 검증 |
| 경로 없음 | 수집 범위·권한 또는 실제 관계 부족 | 수집 방법, 수집 시점, 필터와 누락 객체 확인 |
| 도메인 노드 사이 trust edge | 수집 시점의 도메인 관계 후보 | [[AD 도메인 트러스트 열거와 공격 경로 식별]]에서 속성·방향·인증 확인 |

## 버전과 환경 차이

- BloodHound 클라이언트와 수집기 스키마는 버전에 따라 달라질 수 있다. 클라이언트가 수집 파일을 거부하면 수집기와 서버의 호환 버전을 먼저 확인한다.
- 수집 시점과 수집 계정의 해석은 필요한 입력과 실행 환경에서 관리한다. 이 절에는 클라이언트·수집기·서버 간 schema 호환성만 기록한다.

## 관련 공격기법

- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[AD 원격 접근 권한 열거]]
- [[AD 도메인 트러스트 열거와 공격 경로 식별]]
- [[AD 보안 구성과 GPO 감사]]

## 참고 링크

- [SpecterOps BloodHound: DCSync edge](https://bloodhound.specterops.io/resources/edges/dc-sync)
- [SpecterOps BloodHound: GetChanges edge](https://bloodhound.specterops.io/resources/edges/get-changes)
- [SpecterOps BloodHound: GetChangesAll edge](https://bloodhound.specterops.io/resources/edges/get-changes-all)
- [SpecterOps BloodHound: CanRDP edge](https://bloodhound.specterops.io/resources/edges/can-rdp)
- [SpecterOps BloodHound: CanPSRemote edge](https://bloodhound.specterops.io/resources/edges/can-ps-remote)
