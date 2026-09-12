---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/열거
  - 기능/자격증명수집
실행환경: ["Windows PowerShell"]
필요권한: ["도메인 인증 세션"]
필요조건: ["PowerView.ps1", "도메인 서비스 접근"]
결과: ["AD 객체 및 정책 정보", "그룹과 트러스트 관계", "ACL 공격 경로", "Kerberos TGS hash"]
---

# PowerView

## 도구 개요

PowerView는 PowerShell에서 AD 사용자·그룹·컴퓨터·트러스트·ACL을 폭넓게 열거하고 일부 객체 변경과 Kerberos ticket 요청을 수행하는 스크립트 도구다. 기본 AD 관리 모듈보다 공격 경로 중심의 관계와 권한을 조사할 때 유용하며, 배포 계열에 따라 함수와 매개변수가 달라질 수 있다.

## 필요한 입력과 실행 환경

- 실행 환경: `PowerView.ps1`을 불러올 수 있고 도메인 서비스에 접근 가능한 Windows PowerShell
- 입력: 현재 또는 명시한 도메인, 사용자·그룹·컴퓨터 identity, 필요한 property와 LDAP 조건
- 권한 조건: 대부분의 기본 열거는 도메인 인증 사용자로 가능하며, 반환 범위는 객체별 ACL과 현재 사용 중인 계정의 권한에 따라 달라짐

## 표준 사용법

```powershell
Import-Module .\PowerView.ps1
Get-Domain<OBJECT> [options]
```

## 대표 예시

### 도메인 비밀번호 및 잠금 정책 확인

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

확인할 출력:

- `SystemAccess`의 `MinimumPasswordLength`, `PasswordComplexity`, `LockoutBadCount`
- `ResetLockoutCount`, `LockoutDuration`

### 중첩 그룹을 포함한 구성원 열거

```powershell
Get-DomainGroupMember -Identity "<GROUP>" -Recurse
```

확인할 출력:

- `GroupName`, `MemberName`, `MemberObjectClass`, `MemberSID`
- 중첩 그룹을 통해 고권한 그룹 권한을 상속받는 사용자

### 도메인 사용자와 컴퓨터 객체 조회

```powershell
Get-DomainUser -Identity <USER> -Domain <DOMAIN>
Get-DomainComputer
```

확인할 출력:

- 사용자의 `samaccountname`, `distinguishedname`, `memberof`, `useraccountcontrol`.
- 컴퓨터 객체의 이름, DN, DNS hostname과 운영체제 속성.

### SPN 계정의 TGS를 Hashcat 형식으로 요청

```powershell
Get-DomainUser * -SPN | Select-Object samaccountname
Get-DomainUser * -SPN | Select-Object samaccountname,serviceprincipalname
Get-DomainUser -Identity "<USER>" | Get-DomainSPNTicket -Format Hashcat
```

확인할 출력:

- SPN이 설정된 `samaccountname`과 계정에 연결된 `serviceprincipalname`
- `ServicePrincipalName`과 `$krb5tgs$`로 시작하는 `Hash`
- 특정 `MSSQLSvc/<MSSQL_FQDN>:<PORT>`를 찾는 경우 같은 객체의 `samaccountname`을 TGS 요청 대상으로 사용한다.


## 주요 옵션과 함수

| 옵션·함수 | 의미 | 사용하는 상황 |
|---|---|---|
| `Get-DomainPolicy` | 기본 도메인 또는 DC 정책 반환 | Password Spraying 전 잠금 정책 확인 |
| `Get-DomainUser`, `Get-DomainComputer` | 사용자 또는 컴퓨터 객체와 속성 반환 | 인증 후 디렉터리 객체 목록·상세 확인 |
| `-Identity <VALUE>` | 사용자·그룹·객체를 이름 등으로 제한 | 특정 계정·그룹 또는 객체 상세 조회 |
| `-Recurse` | 중첩 그룹 구성원까지 재귀 열거 | 간접 고권한 구성원 확인 |
| `-SPN` | SPN이 설정된 사용자로 제한 | Kerberoasting 대상 계정 식별 |
| `Get-DomainSPNTicket` | 대상 SPN 계정의 TGS 요청 | Kerberoasting용 ticket 추출 |
| `-Format Hashcat` | TGS hash를 Hashcat 형식으로 반환 | 오프라인 크래킹 준비 |
| `Convert-NameToSid` | 사용자·그룹 이름을 SID로 변환 | ACL의 `SecurityIdentifier`와 대조 |
| `Get-DomainGroup -MemberIdentity` | 지정한 사용자·그룹이 속한 유효 그룹을 위로 조회 | 사용자 SID와 그룹 SID에 나뉜 ACE 후보를 함께 확인 |
| `Get-DomainObjectACL` / `Get-ObjectAcl` | 지정 AD 객체의 ACL 반환. 후자는 배포본에 따라 alias로 제공 | 도메인 루트·사용자·그룹의 ACE 조회 |
| `-ResolveGUIDs` | ACL GUID를 사람이 읽을 수 있는 권한명으로 변환 | 활용 가능한 ACE 판단 |
| `Set-DomainUserPassword` | 대상 사용자 비밀번호 재설정 | `ForceChangePassword` 권한 영향 검증 |
| `Add-DomainGroupMember`, `Remove-DomainGroupMember` | 그룹 구성원 추가·제거 | 그룹 쓰기 권한 행사와 원복 |
| `Set-DomainObject -Set/-Clear` | AD 객체 속성 설정·제거 | 임시 SPN 같은 추적 가능한 속성 변경 |
| `Get-NetLocalGroupMember` | 원격 호스트 로컬 그룹 구성원과 `SID`를 조회 | RDP·WinRM 접근 권한 후보 확인. 별도 AD 계정은 사용자·중첩 그룹 SID와 대조 |
| `Get-DomainGPO` | GPO 객체와 속성 조회 | GPO ACL·표시 이름·정책 단서 확인 |
| `Get-DomainTrust`, `Get-DomainTrustMapping` | 직접 trust 또는 연결된 trust 관계 반환 | Source·Target·방향·포리스트 경계 확인 |
| `-Domain <DOMAIN>` | 조회 대상 도메인 지정 | 신뢰 대상 도메인의 사용자·그룹·SPN 조회 |
| `Get-DomainForeignGroupMember` | 대상 도메인 그룹에 포함된 외부 SID 반환 | 다른 도메인 계정의 대상 도메인 권한 후보 확인 |
| `Convert-SidToName` | SID를 도메인 계정·그룹 이름으로 변환 | 외부 그룹 구성원의 실제 계정 식별 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `SystemAccess` | 도메인 비밀번호와 잠금 정책 반환 | spray 횟수와 observation window 결정 |
| `MemberName`과 `MemberSID`, 또는 로컬 그룹 조회의 `SID` | `Get-DomainGroupMember`는 `MemberSID`, `Get-NetLocalGroupMember`는 `SID`를 반환 | 출력 형식을 섞지 말고 실제 권한 상속과 제어 가능한 계정 확인 |
| `$krb5tgs$...` `Hash` | SPN 계정의 TGS를 크래킹 형식으로 확보 | [[오프라인 해시 크래킹]] 수행 |
| `ObjectAceType`과 `ActiveDirectoryRights` | 객체에 대한 구체적인 ACE와 권한 확인 | 비밀번호 변경, 그룹 수정, DCSync 등 실제 상태 전환 검토 |
| `AceQualifier`, 사용자·그룹 `SecurityIdentifier` | allow·deny와 ACE가 적용되는 주체 SID 후보 | 현재 token의 활성 SID·상속·deny를 대조하고 실제 작업으로 최종 확인 |
| `successfully reset` | `Set-DomainUserPassword` 요청 성공 | 새 credential 인증과 복구 상태를 별도 확인 |
| 추가한 계정 또는 그룹의 `MemberName`·`MemberSID` | 그룹 멤버십 변경 확인 | 그룹이 부여하는 실제 권한과 원복 확인 |
| 설정한 `servicePrincipalName` | SPN 속성 변경 확인 | 표적 TGS 요청 후 정확한 값 원복 |
| 빈 결과 | identity 불일치, 권한 부족 또는 해당 객체 없음 | 도메인, identity 형식, 검색 범위와 현재 사용 중인 계정 확인 |
| LDAP/도메인 검색 오류 | DC 접근, DNS, 현재 도메인 컨텍스트 문제 | DC 도달성, DNS, `-Domain` 또는 서버 지정 확인 |
| `SourceName`·`TargetName`·`TrustDirection` | trust 관계와 방향 단서 확인 | [[AD 도메인 트러스트 열거와 공격 경로 식별]]에서 실제 대상 도메인 조회 확인 |
| `GroupDomain`과 외부 `MemberName` | 다른 도메인 계정이 대상 그룹에 포함된 후보 | SID 이름 변환, credential 보유 여부와 대상 서비스 권한 확인 |

## 버전과 환경 차이

- PowerSploit 원본 PowerView는 지원이 종료되었으며 이 문서의 명령은 해당 개발 버전 문법을 사용한다.
- BC-Security의 Empire 계열 PowerView는 별도로 유지되며 추가 함수나 매개변수 차이가 있을 수 있으므로 불러온 파일의 `Get-Command`와 도움말을 기준으로 실행한다.
- [[SharpView]]는 PowerView의 .NET 포트이며 `PowerView.ps1`과 같은 파일이나 실행 방식이 아니다. PowerShell module 문법과 `SharpView.exe <METHOD>` 문법을 혼용하지 않는다.

## 관련 공격기법

- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[SPN 계정 열거]]
- [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]]
- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[AD 계정의 디렉터리 복제 권한 확인]]
- [[Kerberoasting]]
- [[DCSync]]
- [[AD 사용자 비밀번호 강제 재설정]]
- [[AD 그룹 구성원 추가로 권한 확대]]
- [[임시 SPN 설정]]과 [[표적 Kerberoasting]]
- [[AD 원격 접근 권한 열거]]
- [[AD 계정 위험 속성과 Description 열거]]
- [[AD GPO 쓰기 권한과 영향 범위 열거]]
- [[AD 도메인 트러스트 열거와 공격 경로 식별]]

## 참고 링크

- [PowerSploit: PowerView source](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [PowerSploit: PowerView Recon functions](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/README.md)
- [PowerSploit: Get-DomainGroup](https://powersploit.readthedocs.io/en/latest/Recon/Get-DomainGroup/)
- [PowerSploit: Get-NetLocalGroupMember](https://powersploit.readthedocs.io/en/latest/Recon/Get-NetLocalGroupMember/)
- [Microsoft: How AccessCheck Works](https://learn.microsoft.com/en-us/windows/win32/secauthz/how-dacls-control-access-to-an-object)
