---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["현재 사용할 AD 계정과 도메인 확인", "현재 도메인의 LDAP 접근"]
필요권한: ["현재 AD 계정으로 트러스트 객체를 읽을 권한"]
필요조건: ["현재 도메인과 DC", "현재 계정의 비밀번호·hash·ticket 또는 도메인 세션"]
결과: ["트러스트 Source·Target·방향·전이성·포리스트 경계", "신뢰 대상 도메인 조회 후보"]
---

# AD 도메인 트러스트 열거와 공격 경로 식별

## 한 줄 판단

현재 AD 계정으로 도메인의 trust 객체를 읽을 수 있으면 Source·Target, 방향, 전이성, 포리스트 내부 여부와 선택적 인증·SID filtering 단서를 확인하고, 대상 도메인의 SPN·그룹·관계 수집은 각각의 열거 문서로 넘긴다.

## 사용할 때

- 현재 도메인에서 다른 도메인 또는 포리스트를 가리키는 trust가 있는지 확인할 때.
- 자식·부모, forest, external trust를 구분할 때.
- trust 방향 이름과 실제 대상 도메인 인증 가능 범위를 혼동하지 않으려 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 계정과 도메인 | 어느 도메인의 어떤 계정으로 조회하는지 확인 | [[AD 도메인 컨텍스트 기본 확인]] | 로컬 계정과 AD 계정, password·hash·ticket 주체를 구분 |
| 디렉터리 경로 | 현재 DC LDAP와 DNS 도달 | 좁은 객체 조회와 이름 해석 확인 | route·DNS·LDAP 실패를 분리 |
| trust 읽기 권한 | trust 객체가 속성과 함께 반환 | `Get-ADTrust` 또는 `Get-DomainTrust` 결과 | 빈 결과와 조회 실패를 구분 |

## 실행

### Windows 공격 호스트에서 실행

```powershell
Import-Module ActiveDirectory
Get-ADTrust -Filter * | Select-Object Source,Target,Direction,IntraForest,ForestTransitive,SelectiveAuthentication,SIDFilteringForestAware,SIDFilteringQuarantined
```

PowerView 또는 기본 명령을 사용할 수 있으면 같은 속성을 교차 확인한다.

```powershell
Get-DomainTrust
Get-DomainTrustMapping
netdom query /domain:<CURRENT_FQDN> trust
```

확인할 출력:

- `Source`와 `Target`, `Direction`을 한 묶음으로 기록한다.
- `IntraForest`, `ForestTransitive`, 선택적 인증과 SID filtering 속성을 구분한다.
- 한 도구의 빈 결과만으로 trust 부재를 단정하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `IntraForest` 또는 같은 포리스트 관계 | parent-child·tree-root·cross-link 후보 | 포리스트 내부 trust | 부모·자식 관계와 각 도메인 SID를 별도 확인 |
| `ForestTransitive` | forest trust 후보 | 포리스트 간 trust | 대상 도메인 DNS·LDAP·Kerberos 도달성 확인 |
| 포리스트 외부 비전이 trust | external trust 후보 | 직접 연결 trust | 직접 연결된 Source·Target과 SID filtering 확인 |
| 양방향 또는 단방향 표시 | 인증 가능 방향 후보 | 대상 도메인 조회 후보 | 방향 이름만 해석하지 말고 대상 도메인 객체·서비스에서 실제 확인 |
| 대상 도메인 SPN 계정 조사 필요 | trust 열거와 별도 계정·SPN 조회 | 신뢰 대상 서비스 계정 후보 필요 | [[SPN 계정 열거]] |
| 대상 도메인 그룹의 외부 구성원 조사 필요 | trust 열거와 별도 그룹 객체 조회 | 외부 그룹 멤버십 후보 필요 | [[AD 외부 도메인 그룹 구성원 열거]] |
| 양쪽 도메인 객체 관계 조사 필요 | trust 열거와 별도 그래프 수집 | ACL·세션·그룹 관계 후보 필요 | [[AD 관계 그래프 수집과 공격 경로 식별]] |
| 같은 포리스트 자식 도메인 장악 상태 | trust 존재만으로 부모 권한이 확정되지 않음 | 자식→부모 경로 후보 | [[자식 도메인 장악 후 부모 도메인 경로 선택]] |

## 확인할 출력과 권한

- trust 객체 반환은 관계 존재만 뜻하며 현재 계정의 상대 도메인 리소스 접근 성공을 뜻하지 않는다.
- 대상 LDAP 조회, TGS 획득, 원격 로그인과 관리자 권한은 각각 다른 성공 단계다.
- SPN과 외부 그룹 관계를 이 문서에서 다시 수집하지 않고 원자 열거 문서의 결과를 사용한다.

## 관련 공격기법

- [[SPN 계정 열거]]
- [[AD 외부 도메인 그룹 구성원 열거]]
- [[AD 관계 그래프 수집과 공격 경로 식별]]

## 관련 도구

- [[ActiveDirectory PowerShell 모듈]]
- [[PowerView]]
- [[netdom]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[자식 도메인 장악 후 부모 도메인 경로 선택]]
