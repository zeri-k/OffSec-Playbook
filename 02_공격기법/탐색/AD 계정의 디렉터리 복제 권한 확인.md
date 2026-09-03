---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["확인할 AD 사용자 또는 그룹 이름 식별", "PowerView를 실행할 수 있는 인증된 Windows PowerShell"]
필요권한: ["현재 PowerShell 세션에서 도메인 객체와 ACL을 읽을 권한"]
필요조건: ["PowerView.ps1", "대상 도메인 DN", "실행 호스트에서 DC LDAP 접근"]
결과: ["확인 대상 SID에 직접 부여된 디렉터리 복제 확장 권한", "DCSync 실행 전제 충족 여부"]
---

# AD 계정의 디렉터리 복제 권한 확인

## 한 줄 판단

확인할 AD 사용자 또는 그룹 이름과 대상 도메인 DN을 알고 PowerView를 실행하는 Windows PowerShell에서 DC LDAP에 접근할 수 있으면, 이름을 SID로 변환하고 도메인 루트 ACL의 복제 확장 권한을 같은 SID로 제한하여 DCSync 실행 전제를 판정한다.

## 사용할 때

- 현재 보유 정보: 새로 확보한 AD 계정의 이름, 또는 ACL·BloodHound에서 복제 권한 후보로 확인한 사용자·그룹 이름이 있다.
- 명령 실행 위치와 접근 대상: PowerView를 불러올 수 있는 Windows PowerShell에서 대상 도메인의 DC LDAP 서비스에 접근할 수 있다.
- 현재 계정과 확인 대상: ACL을 읽는 현재 PowerShell 계정과 복제 권한을 확인할 계정은 서로 달라도 된다. 이 명령은 확인 대상 계정으로 로그인하는 절차가 아니라 그 계정 SID에 설정된 ACE를 읽는 절차다.
- 지금 가능한 행동과 결과: 확인 대상 SID에 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All`이 함께 허용됐는지 확인하여 [[DCSync]] 실행 여부를 결정한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | PowerView를 실행하는 Windows 호스트에서 DC LDAP 접근 가능 | 간단한 `Get-DomainUser` 조회가 응답하는지 확인 | DNS·도메인·DC와 LDAP 경로를 [[AD 도메인 컨텍스트 기본 확인]]에서 재확인 |
| 현재 계정 또는 인증 수단 | 현재 PowerShell 세션이 도메인 객체와 ACL을 읽을 수 있음 | `Get-ObjectAcl` 또는 `Get-DomainObjectACL`이 access denied 없이 반환되는지 확인 | 현재 로그온 계정과 LDAP 인증 상태를 확인 |
| 확인 대상 계정 | 제어 중이거나 권한 후보인 AD 사용자 또는 그룹 이름이 정확함 | `Convert-NameToSid`가 하나의 SID를 반환하는지 확인 | 도메인 접두사·사용자명·그룹명과 대상 도메인을 확인 |
| 공격 대상의 조건 | 복제 권한을 확인할 도메인 루트 DN을 알고 있음 | `DC=<DOMAIN_PART>,DC=<TLD>` 형식과 현재 도메인을 대조 | 다른 도메인이나 하위 객체 DN을 넣지 않았는지 확인 |
| 필요한 파일·함수 | PowerView의 SID 변환 및 ACL 조회 함수 사용 가능 | `Get-Command`로 함수 또는 별칭 확인 | 불러온 PowerView 배포본의 함수명을 확인 |

## 실행

### 1. PowerView 함수 확인

```powershell
Import-Module '<POWERVIEW_PATH>'
Get-Command Convert-NameToSid,Get-ObjectAcl,Get-DomainObjectACL -ErrorAction SilentlyContinue
```

확인할 출력:

- `Convert-NameToSid`와 `Get-ObjectAcl` 또는 `Get-DomainObjectACL` 중 하나.
- 두 ACL 함수가 모두 없으면 다른 배포본의 문법을 섞지 말고 현재 불러온 파일의 함수를 먼저 확인한다.

### 2. 권한을 확인할 계정을 SID로 변환

```powershell
$checkedAccount = '<DOMAIN>\<CONTROLLED_USER_OR_GROUP>'
$sid = Convert-NameToSid $checkedAccount
$sid
```

확인할 출력:

- 권한을 확인하려는 사용자 또는 그룹과 일치하는 하나의 도메인 SID.
- SID가 비어 있으면 ACL 권한 부재로 판단하지 않는다. 이름 형식, 대상 도메인과 DC LDAP 접근을 먼저 확인한다.

### 3. 도메인 루트에서 같은 SID의 복제 권한만 조회

불러온 PowerView 배포본에서 `Get-ObjectAcl` 별칭을 제공하면 다음과 같이 실행한다.

```powershell
$domainDN = '<DOMAIN_DN>'
Get-ObjectAcl $domainDN -ResolveGUIDs |
  Where-Object { $_.ObjectAceType -match '^DS-Replication-Get-Changes' } |
  Where-Object { $_.SecurityIdentifier -eq $sid } |
  Select-Object AceQualifier,ObjectDN,ActiveDirectoryRights,SecurityIdentifier,ObjectAceType,IsInherited |
  Format-List
```

`Get-ObjectAcl`이 없고 `Get-DomainObjectACL`이 있는 PowerView에서는 같은 필터를 다음과 같이 적용한다.

```powershell
Get-DomainObjectACL -Identity $domainDN -ResolveGUIDs |
  Where-Object { $_.ObjectAceType -match '^DS-Replication-Get-Changes' } |
  Where-Object { $_.SecurityIdentifier -eq $sid } |
  Select-Object AceQualifier,ObjectDN,ActiveDirectoryRights,SecurityIdentifier,ObjectAceType,IsInherited |
  Format-List
```

확인할 출력:

- `ObjectDN`이 확인하려는 도메인 루트 DN과 일치한다.
- `AceQualifier`가 `AccessAllowed`, `ActiveDirectoryRights`가 `ExtendedRight`다.
- 같은 `SecurityIdentifier`에 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All`이 모두 표시된다.
- `DS-Replication-Get-Changes-In-Filtered-Set`은 추가 권한이며, 이 항목 하나만으로 두 필수 권한을 대신하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 같은 SID에 `Get-Changes`와 `Get-Changes-All`이 모두 `AccessAllowed`로 표시됨 | 확인 대상 계정에 DCSync의 핵심 복제 권한이 직접 부여됨 | DCSync 권한 전제 충족 | 해당 계정의 인증 자료와 DC RPC 경로를 확인한 뒤 [[DCSync]] |
| `Get-Changes-In-Filtered-Set`만 표시됨 | 필터된 속성 집합 권한만 확인됨 | DCSync 권한 전제 미충족 | 다른 복제 ACE와 그룹 SID를 [[AD ACL 권한 열거와 공격 경로 식별]]에서 확인 |
| 두 필수 권한 중 하나만 표시됨 | 복제 권한 조합이 불완전함 | DCSync 권한 전제 미충족 | 같은 계정의 그룹 권한과 다른 ACE를 확인 |
| 복제 권한은 보이지만 `SecurityIdentifier`가 `$sid`와 다름 | 다른 사용자·그룹의 권한임 | 확인 대상 계정의 권한 미확정 | SID 필터와 확인 대상 계정 이름을 재확인 |
| 결과가 비어 있음 | 해당 SID의 직접 복제 ACE를 찾지 못했거나 입력·조회가 잘못됨 | 직접 권한 미확정 | SID·도메인 DN·ACL 읽기 성공을 확인하고 그룹 SID 기반 권한을 [[AD ACL 권한 열거와 공격 경로 식별]]에서 확인 |

## 확인할 출력과 권한

- DCSync 가능성은 도메인 루트, 같은 SID, `AccessAllowed`, `DS-Replication-Get-Changes`, `DS-Replication-Get-Changes-All`을 한 묶음으로 확인한다.
- 이 조회를 실행한 PowerShell 계정이 아니라 `$sid`에 지정한 사용자 또는 그룹의 권한을 판정한다.
- 복제 권한 확인은 Domain Admin 그룹 멤버십, 대상 Windows 호스트의 로컬 관리자 권한 또는 실제 DCSync 성공을 뜻하지 않는다.
- [[DCSync]]에서는 여기서 확인한 사용자 계정의 비밀번호·NT hash·Kerberos ticket과 DC의 DRSUAPI RPC 경로를 별도로 사용한다.

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[DCSync]]

## 관련 도구

- [[PowerView]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[AD 객체 제어권 확보 후 악용 경로 선택]]
