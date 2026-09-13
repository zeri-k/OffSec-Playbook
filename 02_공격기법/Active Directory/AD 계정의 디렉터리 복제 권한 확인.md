---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["복제 요청자로 확인할 AD 사용자 이름 식별", "PowerView를 실행할 수 있는 인증된 Windows PowerShell"]
필요권한: ["현재 PowerShell 세션에서 도메인 객체와 ACL을 읽을 권한"]
필요조건: ["PowerView.ps1", "대상 도메인 DN", "실행 호스트에서 DC LDAP 접근"]
결과: ["확인 대상의 사용자·유효 그룹 SID에 부여된 디렉터리 복제 확장 권한", "DCSync 실행 전제 후보"]
---

# AD 계정의 디렉터리 복제 권한 확인

## 한 줄 판단

확인할 AD 사용자와 대상 도메인 DN을 알고 PowerView를 실행하는 Windows PowerShell에서 DC LDAP에 접근할 수 있으면, 사용자와 유효 그룹 SID 후보를 도메인 루트 ACL의 복제 확장 권한과 대조하여 DCSync 실행 전제를 판정한다.

## 사용할 때

- 현재 보유 정보: 새로 확보한 AD 계정의 이름, 또는 ACL·BloodHound에서 복제 권한 후보로 확인한 사용자와 그 사용자의 유효 그룹이 있다.
- 명령 실행 위치와 접근 대상: PowerView를 불러올 수 있는 Windows PowerShell에서 대상 도메인의 DC LDAP 서비스에 접근할 수 있다.
- 현재 계정과 확인 대상: ACL을 읽는 현재 PowerShell 계정과 복제 권한을 확인할 계정은 서로 달라도 된다. 이 명령은 확인 대상 계정으로 로그인하는 절차가 아니라 그 계정 SID에 설정된 ACE를 읽는 절차다.
- 지금 가능한 행동과 결과: 확인 대상의 사용자 SID와 유효 그룹 SID 집합에 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All`이 허용됐는지 확인하여 [[DCSync]] 실행 후보 여부를 결정한다. 적용되는 deny ACE와 실제 DRSUAPI 요청 성공은 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | PowerView를 실행하는 Windows 호스트에서 DC LDAP 접근 가능 | 간단한 `Get-DomainUser` 조회가 응답하는지 확인 | DNS·도메인·DC와 LDAP 경로를 [[AD 도메인 컨텍스트 기본 확인]]에서 재확인 |
| 현재 계정 또는 인증 수단 | 현재 PowerShell 세션이 도메인 객체와 ACL을 읽을 수 있음 | `Get-ObjectAcl` 또는 `Get-DomainObjectACL`이 access denied 없이 반환되는지 확인 | 현재 로그온 계정과 LDAP 인증 상태를 확인 |
| 확인 대상 계정 | 제어 중이거나 권한 후보인 AD 사용자 이름과 유효 그룹을 식별함 | `Convert-NameToSid`와 `Get-DomainGroup -MemberIdentity` 결과 확인 | 도메인 접두사·사용자명·그룹 중첩과 대상 도메인을 확인 |
| 공격 대상의 조건 | 복제 권한을 확인할 도메인 루트 DN을 알고 있음 | `DC=<DOMAIN_PART>,DC=<TLD>` 형식과 현재 도메인을 대조 | 다른 도메인이나 하위 객체 DN을 넣지 않았는지 확인 |
| 필요한 파일·함수 | PowerView의 SID 변환 및 ACL 조회 함수 사용 가능 | `Get-Command`로 함수 또는 별칭 확인 | 불러온 PowerView 배포본의 함수명을 확인 |

## 실행

### 1. PowerView 함수 확인

```powershell
Import-Module '<POWERVIEW_PATH>'
Get-Command Convert-NameToSid,Get-DomainGroup,Get-ObjectAcl,Get-DomainObjectACL -ErrorAction SilentlyContinue
```

확인할 출력:

- `Convert-NameToSid`, `Get-DomainGroup`과 `Get-ObjectAcl` 또는 `Get-DomainObjectACL` 중 하나.
- 두 ACL 함수가 모두 없으면 다른 배포본의 문법을 섞지 말고 현재 불러온 파일의 함수를 먼저 확인한다.

### 2. 권한을 확인할 사용자와 유효 그룹을 SID로 변환

```powershell
$checkedAccount = '<DOMAIN>\<CONTROLLED_USER>'
$userSid = Convert-NameToSid $checkedAccount
$groupSids = Get-DomainGroup -MemberIdentity $checkedAccount |
  Select-Object -ExpandProperty objectsid
$effectiveSids = @($userSid) + @($groupSids) |
  Where-Object { $_ } |
  Sort-Object -Unique
$effectiveSids
```

확인할 출력:

- `$userSid`가 권한을 확인하려는 사용자와 일치하고, `$effectiveSids`에 그 SID와 직접·중첩 보안 그룹 SID 후보가 포함된다.
- PowerView의 `Get-DomainGroup -MemberIdentity`는 사용자의 유효 그룹 관계를 위로 조회한다. 불러온 배포본이 이 매개변수를 제공하지 않으면 `Get-Command Get-DomainGroup -Syntax`를 확인하고 [[AD 고권한 그룹과 중첩 구성원 열거]]에서 그룹 목록을 얻어 각 이름을 `Convert-NameToSid`로 변환한다.
- 사용자 SID나 그룹 목록이 비어 있으면 ACL 권한 부재로 판단하지 않는다. 이름 형식, 대상 도메인, DC LDAP 접근과 그룹 조회 범위를 먼저 확인한다.
- 이 목록은 LDAP에서 계산한 후보이며 기존 로컬 로그온 token의 즉시 갱신을 보장하지 않는다. 디렉터리 그룹 관계와 현재 token의 차이는 [[Windows 액세스 토큰과 특권 활성화]]를 따르며, 현재 token을 사용하는 실행이면 `whoami /groups`로 활성·deny-only 상태를 대조한다.

### 3. 도메인 루트에서 요청자 SID 집합의 복제 권한만 조회

불러온 PowerView 배포본에서 `Get-ObjectAcl` 별칭을 제공하면 다음과 같이 실행한다.

```powershell
$domainDN = '<DOMAIN_DN>'
Get-ObjectAcl $domainDN -ResolveGUIDs |
  Where-Object { $_.ObjectAceType -match '^DS-Replication-Get-Changes' } |
  Where-Object { $_.SecurityIdentifier -in $effectiveSids } |
  Select-Object AceQualifier,ObjectDN,ActiveDirectoryRights,SecurityIdentifier,ObjectAceType,IsInherited |
  Format-List
```

`Get-ObjectAcl`이 없고 `Get-DomainObjectACL`이 있는 PowerView에서는 같은 필터를 다음과 같이 적용한다.

```powershell
Get-DomainObjectACL -Identity $domainDN -ResolveGUIDs |
  Where-Object { $_.ObjectAceType -match '^DS-Replication-Get-Changes' } |
  Where-Object { $_.SecurityIdentifier -in $effectiveSids } |
  Select-Object AceQualifier,ObjectDN,ActiveDirectoryRights,SecurityIdentifier,ObjectAceType,IsInherited |
  Format-List
```

확인할 출력:

- `ObjectDN`이 확인하려는 도메인 루트 DN과 일치한다.
- `AceQualifier`가 `AccessAllowed`, `ActiveDirectoryRights`가 `ExtendedRight`다.
- 요청자 사용자 또는 유효 그룹 `SecurityIdentifier` 전체에서 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All`이 모두 허용된다. 두 권한은 서로 다른 적용 SID의 allow ACE로 충족될 수 있다.
- 같은 SID 집합에 적용되는 `AccessDenied` ACE와 현재 token의 disabled·deny-only 그룹이 없는지 별도로 확인한다.
- `DS-Replication-Get-Changes-In-Filtered-Set`은 추가 권한이며, 이 항목 하나만으로 두 필수 권한을 대신하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 요청자의 사용자·유효 그룹 SID 집합에 `Get-Changes`와 `Get-Changes-All`이 모두 `AccessAllowed`로 표시되고 적용 deny가 없음 | DCSync 핵심 복제 권한의 ACL 후보 충족 | DCSync 실행 후보 | 해당 계정의 인증 자료와 DC RPC 경로를 확인한 뒤 최소 범위 [[DCSync]] 요청으로 최종 확인 |
| `Get-Changes-In-Filtered-Set`만 표시됨 | 필터된 속성 집합 권한만 확인됨 | DCSync 권한 전제 미충족 | 다른 복제 ACE와 그룹 SID를 [[AD ACL 권한 열거와 공격 경로 식별]]에서 확인 |
| 두 필수 권한 중 하나만 표시됨 | 복제 권한 조합이 불완전함 | DCSync 권한 전제 미충족 | 같은 계정의 그룹 권한과 다른 ACE를 확인 |
| 복제 권한은 보이지만 `SecurityIdentifier`가 `$effectiveSids`에 없음 | 요청자에게 적용되는 것으로 확인되지 않은 다른 주체의 권한 | 확인 대상 계정의 권한 미확정 | 사용자·그룹 SID 집합과 확인 대상 계정 이름을 재확인 |
| 결과가 비어 있음 | 요청자의 SID 집합에 적용되는 복제 allow ACE를 찾지 못했거나 입력·조회가 잘못됨 | 권한 미확정 | 사용자·그룹 SID, 도메인 DN과 ACL 읽기 성공을 재확인 |

## 확인할 출력과 권한

- DCSync 가능성은 도메인 루트, 요청자의 사용자·유효 그룹 SID 집합, `AccessAllowed`, `DS-Replication-Get-Changes`, `DS-Replication-Get-Changes-All`과 적용 deny 여부를 한 묶음으로 확인한다.
- 이 조회를 실행한 PowerShell 계정이 아니라 `$checkedAccount`와 그 유효 그룹의 권한 후보를 판정한다.
- 복제 권한 확인은 Domain Admin 그룹 멤버십, 대상 Windows 호스트의 로컬 관리자 권한 또는 실제 DCSync 성공을 뜻하지 않는다.
- [[DCSync]]에서는 여기서 확인한 사용자 계정의 비밀번호·NT hash·Kerberos ticket과 DC의 DRSUAPI RPC 경로를 별도로 사용한다.

## 참고 링크

- [Microsoft: DS-Replication-Get-Changes extended right](https://learn.microsoft.com/en-us/windows/win32/adschema/r-ds-replication-get-changes)
- [Microsoft: DS-Replication-Get-Changes-All extended right](https://learn.microsoft.com/en-us/windows/win32/adschema/r-ds-replication-get-changes-all)
- [Microsoft: DS-Replication-Get-Changes-In-Filtered-Set extended right](https://learn.microsoft.com/en-us/windows/win32/adschema/r-ds-replication-get-changes-in-filtered-set)
- [Microsoft: How AccessCheck Works](https://learn.microsoft.com/en-us/windows/win32/secauthz/how-dacls-control-access-to-an-object)
- [PowerSploit: Get-DomainGroup](https://powersploit.readthedocs.io/en/latest/Recon/Get-DomainGroup/)
- [SpecterOps BloodHound: DCSync edge](https://bloodhound.specterops.io/resources/edges/dc-sync)

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[DCSync]]

## 관련 도구

- [[PowerView]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[AD 객체 제어권 확보 후 악용 경로 선택]]
