---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["신뢰 대상 도메인과 DC 식별", "현재 사용할 AD 계정 또는 도메인 사용자 세션 확보"]
필요권한: ["현재 AD 계정으로 대상 도메인의 그룹과 Foreign Security Principal 객체를 읽을 권한"]
필요조건: ["대상 도메인 LDAP와 DNS 접근", "PowerView 실행 가능"]
결과: ["대상 도메인 그룹에 포함된 외부 도메인 계정·그룹", "교차 도메인 권한 관계 후보"]
---

# AD 외부 도메인 그룹 구성원 열거

## 한 줄 판단

신뢰 대상 도메인의 LDAP 객체를 읽을 수 있으면 대상 도메인 그룹에 포함된 외부 도메인 SID를 조회하고 실제 계정·그룹 이름으로 변환하여 교차 도메인 권한 관계 후보를 확인한다.

## 사용할 때

- 도메인 또는 포리스트 trust가 확인됐고 상대 도메인의 그룹에 현재 도메인 계정이 포함됐는지 조사할 때.
- BloodHound의 외부 그룹 관계를 현재 LDAP 조회 결과와 교차 확인할 때.
- 외부 SID가 어느 도메인의 사용자·그룹인지 확인해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 계정과 도메인 | 조회에 사용할 AD 계정과 소속 도메인 확인 | [[AD 도메인 컨텍스트 기본 확인]] | 로컬 계정·머신 계정·사용자 계정을 구분 |
| 대상 도메인 | trust의 상대 도메인 FQDN 확인 | [[AD 도메인 트러스트 열거와 공격 경로 식별]] | Source·Target과 방향을 다시 확인 |
| 디렉터리 경로 | 대상 도메인 DNS·LDAP 접근 | 이름 해석과 LDAP 응답 확인 | DNS, route, LDAP 연결과 인증 실패를 분리 |

## 실행

### Windows 공격 호스트에서 실행

```powershell
Import-Module .\PowerView.ps1
$foreignMembers = Get-DomainForeignGroupMember -Domain <TARGET_FQDN>
$foreignMembers | Select-Object GroupDomain,GroupName,MemberDomain,MemberName,MemberDistinguishedName,MemberObjectClass
$foreignMembers | ForEach-Object { Convert-SidToName $_.MemberName }
```

확인할 출력:

- 대상 그룹의 도메인·이름과 외부 구성원의 SID, 소속 도메인, 객체 종류.
- `Convert-SidToName`으로 변환된 실제 사용자·그룹 이름.
- 빈 결과, SID 변환 실패와 LDAP 조회 거부를 서로 다른 상태로 기록한다.

PowerView를 실행할 수 없는 Linux 경로에서는 [[AD 관계 그래프 수집과 공격 경로 식별]]의 두 도메인 `DCOnly` 수집으로 foreign membership 후보를 찾고, 양쪽 LDAP에서 SID·그룹 객체를 다시 확인한다. 한 collector가 표기한 `MemberDomain`이나 이름만으로 소속 realm을 확정하지 않고 SID의 domain prefix와 해당 도메인의 실제 object를 대조한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 외부 사용자 SID와 대상 그룹이 반환됨 | 다른 도메인 사용자가 대상 도메인 그룹에 직접 포함됨 | 교차 도메인 그룹 멤버십 후보 | 그룹 권한과 적용 대상 서비스·객체를 직접 확인 |
| 외부 그룹 SID가 반환됨 | 중첩 그룹을 통해 권한이 전달될 수 있음 | 교차 도메인 중첩 그룹 후보 | [[AD 고권한 그룹과 중첩 구성원 열거]]에서 중첩 경로 확인 |
| BloodHound 관계와 현재 LDAP 결과가 일치함 | 외부 그룹 관계의 현재성 신뢰도 상승 | 직접 검증할 권한 경로 | [[AD ACL 권한 열거와 공격 경로 식별]] 또는 대상 서비스 접근 확인 |
| 결과가 비어 있음 | 외부 구성원 부재 또는 조회 범위 제한 가능 | 외부 그룹 관계 미확정 | 대상 도메인, 현재 계정 읽기 범위와 LDAP 오류 확인 |

## 확인할 출력과 권한

- 외부 그룹 멤버십은 권한 경로 후보다. 대상 파일·서비스·AD 객체에서 실제 수행 가능한 행동을 별도로 확인한다.
- trust 존재, 상대 도메인 객체 조회, 그룹 멤버십과 원격 서비스 권한을 각각 다른 상태로 기록한다.

## 관련 공격기법

- [[AD 도메인 트러스트 열거와 공격 경로 식별]]
- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]

## 관련 도구

- [[PowerView]]
- [[bloodhound-python]]
- [[BloodHound]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 참고 링크

- [PowerSploit: Get-DomainForeignGroupMember](https://powersploit.readthedocs.io/en/latest/Recon/Get-DomainForeignGroupMember/)
