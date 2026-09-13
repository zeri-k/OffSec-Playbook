---
tags:
  - 환경/ad
시작조건: ["유효한 AD 계정 자격 증명 또는 해당 계정의 Windows 세션 확보", "현재 제어하는 사용자 또는 그룹 식별"]
필요권한: ["현재 인증 주체로 도메인 객체와 ACL을 읽을 권한"]
필요조건: ["PowerView를 실행할 Windows PowerShell", "실행 호스트에서 DC LDAP 접근", "현재 제어하는 사용자 또는 그룹의 SID"]
결과: ["객체별 ACE", "제어 가능한 AD 객체", "권한 상승과 내부 이동 경로 후보"]
---

# AD ACL 권한 열거와 공격 경로 식별

## 한 줄 판단

현재 제어하는 AD 사용자 또는 그룹과 그 SID를 알고 DC LDAP에 닿는 Windows PowerShell 세션이 있으면, 해당 SID에 허용된 ACE와 대상 객체를 조회하여 별도로 검증할 권한 상승·내부 이동 경로 후보를 얻는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | PowerView를 실행할 Windows PowerShell에서 DC LDAP에 도달 | 현재 도메인·DC를 확인하고 LDAP 기반 PowerView 조회가 응답하는지 확인 | DNS 이름 해석, DC 주소, LDAP 포트와 피벗 경로를 먼저 확인 |
| 현재 계정 또는 인증 수단 | 유효한 AD 자격 증명·ticket 또는 해당 사용자 컨텍스트의 세션 | 현재 사용자와 도메인을 확인하고 간단한 도메인 객체 조회 수행 | [[AD 도메인 컨텍스트 기본 확인]]에서 로컬 계정과 AD Identity를 분리 |
| 현재 권한 | 대상 객체와 해당 ACL을 읽을 수 있음 | `Get-DomainObjectACL`이 access denied 없이 ACE를 반환하는지 확인 | 읽기 제한인지 조회 범위·필터 문제인지 작은 대상 객체로 재확인 |
| 공격 대상의 조건 | 제어 관계를 확인할 사용자·그룹·컴퓨터·GPO·도메인 객체가 식별됨 | `ObjectDN`과 객체 종류를 확인 | BloodHound 경로 또는 고가치 객체 목록으로 대상을 먼저 좁힘 |
| 필요한 파일·목록·주소 | PowerView와 제어 중인 사용자·그룹 이름 또는 SID | 모듈 import, `Convert-NameToSid` 결과와 중첩 그룹 확인 | 이름 형식, 도메인 접두사와 그룹 SID를 다시 확인 |

## 실행

### 1. 현재 계정 SID 확인

`<CONTROLLED_USER>`는 ACE를 찾을 현재 제어 주체의 도메인 사용자명(예: `corp.example\\operator`) 또는 그룹명이며, 조회 PowerShell 계정과 다를 수 있다. 아래 `Import-Module` 경로는 Windows 실행 호스트에 있는 PowerView 파일이다.

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid <CONTROLLED_USER>
```

확인할 출력:

- 현재 제어하는 사용자와 일치하는 도메인 SID.
- SID가 반환되지 않으면 사용자 이름 형식, 현재 도메인 컨텍스트와 DC 도달성을 먼저 확인한다. 이 단계 실패는 해당 계정에 ACE가 없다는 뜻이 아니다.

### 2. 해당 SID가 보유한 ACE만 조회

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * |
  Where-Object { $_.SecurityIdentifier -eq $sid }
```

확인할 출력:

- `ObjectDN`, `ActiveDirectoryRights`, `ObjectAceType`, `IsInherited`.
- GUID 대신 `User-Force-Change-Password`처럼 해석된 권한명.
- access denied이면 현재 인증 주체의 ACL 읽기 범위를 확인한다. 결과가 비어 있으면 SID 필터, 상속 ACE와 그룹 SID를 확인한 뒤 권한 부재를 판정한다.

이 조회는 입력한 SID에 직접 연결된 ACE만 보여 준다. 실제 LDAP·서비스 요청의 인가는 요청 인증 컨텍스트에 포함된 사용자·유효 그룹 SID 집합과 allow·deny ACE를 함께 평가하므로, 사용자 SID 결과가 비어 있거나 권한 하나만 보이면 해당 요청자의 유효 그룹 SID를 같은 방식으로 조회한다. Windows 통합 인증을 현재 프로세스에서 수행하는 경우에만 [[Windows 액세스 토큰과 특권 활성화]]의 SID·token 경계를 `whoami /groups`로 대조한다. 여러 allow 권한은 사용자·그룹 SID에 나뉘어 있을 수 있고, 적용되는 deny ACE나 대상 DC·서비스의 인가 결과가 다르면 목록만으로 최종 성공을 확정할 수 없다.

### 3. 그룹 중첩과 다음 제어권 확인

```powershell
Get-DomainGroup -Identity '<GROUP>' | Select-Object memberof
$groupSid = Convert-NameToSid '<GROUP>'
Get-DomainObjectACL -ResolveGUIDs -Identity * |
  Where-Object { $_.SecurityIdentifier -eq $groupSid }
```

확인할 출력:

- 제어 가능한 그룹이 상위 그룹에 중첩되는지 여부.
- 그룹 권한으로 새롭게 제어할 수 있는 사용자·그룹·컴퓨터·도메인 객체.
- `memberof`가 비어 있어도 다른 그룹의 ACL이나 현재 토큰에 적용된 권한까지 부정하는 것은 아니므로 BloodHound와 개별 SID 조회로 교차 확인한다.

### 4. BloodHound로 경로 교차 검증

- 시작 사용자의 `Outbound Object Control`과 `Transitive Object Control`을 확인한다.
- edge 도움말의 필요 권한과 변경 영향을 확인하고, 실제 수행은 대응하는 세부 기법으로 넘긴다.
- 수집 시점이 오래됐거나 PowerView 결과와 다르면 현재 LDAP ACL과 현재 그룹 멤버십을 기준으로 다시 수집한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `ForceChangePassword` | 대상 사용자의 기존 비밀번호 없이 재설정 가능 | 사용자 객체 변경 후보 | 복구 계획을 세운 뒤 [[AD 사용자 비밀번호 강제 재설정]] |
| 그룹에 대한 `GenericWrite`·`AddSelf` | 그룹 구성원 변경 가능 | 새 그룹 권한 후보 | 영향 그룹을 확인한 뒤 [[AD 그룹 구성원 추가로 권한 확대]] |
| 사용자에 대한 `servicePrincipalName` 쓰기 | SPN을 임시 등록할 수 있음 | 표적 Kerberoasting 후보 | 기존 SPN 기준값을 기록하고 [[임시 SPN 설정]] 뒤 [[Kerberoasting]] |
| 사용자·컴퓨터에 대한 `GenericAll`·`GenericWrite` | 대상 객체의 속성 변경 가능 | Shadow Credentials 등 후보 | 대상 객체와 변경 목적에 따라 [[Shadow Credentials]] 등 별도 기법 선택 |
| 도메인 객체에 복제 관련 권한 | DCSync 가능성 | 도메인 자격 증명 복제 후보 | [[AD 계정의 디렉터리 복제 권한 확인]]에서 요청자의 사용자·유효 그룹 SID 집합에 두 필수 권한이 적용되는지 확인하고, 충족하면 [[DCSync]] |
| GPO 객체에 `WriteProperty`·`WriteDacl` | 정책 변경 가능성 | 쓰기 가능한 GPO 후보 | [[AD GPO 쓰기 권한과 영향 범위 열거]]로 표시 이름과 적용 범위 확인 |
| BloodHound 경로와 PowerView 결과가 일치 | 제어 관계 신뢰도 상승 | 개별 공격기법 선택 가능 | 각 edge를 상태 변경 기법으로 하나씩 검증 |
| ACL 결과가 지나치게 많음 | 전체 덤프는 판단 비용이 큼 | 우선순위 미확정 | 제어 중인 사용자·그룹 SID와 고가치 객체로 범위 축소 |
| GUID가 해석되지 않음 | 권한 종류 미확정 | ACE 후보 | `-ResolveGUIDs` 또는 Extended Rights 역조회로 이름 확인 |

## 확인할 출력과 권한

- 이 절차가 공격 경로로 읽는 것은 객체 접근을 허용·거부하는 DACL의 ACE다. SACL은 어떤 접근을 감사할지 정하며 객체 변경 권한을 부여하지 않으므로, SACL 항목을 발견했다는 이유로 후속 기법을 선택하지 않는다.
- `AccessAllowed`, 대상 `ObjectDN`, 현재 계정의 사용자 SID 또는 적용되는 그룹 `SecurityIdentifier`와 권한 이름이 함께 확인되어야 한다.
- 이 조합은 해당 ACE가 디렉터리에 존재한다는 사실과 후속 검증 대상을 확정한다. 현재 로그온 토큰에 권한이 유효하게 적용됐거나 대상 변경이 성공한다는 사실은 확정하지 않는다.
- ACL 읽기 성공은 객체 수정 성공을 뜻하지 않는다. 상속, deny ACE, 보호 객체와 현재 토큰을 별도로 확인한다.
- 이 문서는 조회 전용이다. 비밀번호 재설정, 그룹 변경, SPN 변경, ACL 변경은 복구 절차가 있는 별도 기법으로 수행한다.

## 참고 링크

- [Microsoft: How AccessCheck Works](https://learn.microsoft.com/en-us/windows/win32/secauthz/how-dacls-control-access-to-an-object)
- [Microsoft: SID Attributes in an Access Token](https://learn.microsoft.com/en-us/windows/win32/secauthz/sid-attributes-in-an-access-token)
- [PowerSploit: PowerView Recon functions](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/README.md)
- [SpecterOps BloodHound: DCSync edge](https://bloodhound.specterops.io/resources/edges/dc-sync)

## 관련 공격기법

- [[AD 계정의 디렉터리 복제 권한 확인]]
- [[DCSync]]
- [[Shadow Credentials]]
- [[Kerberoasting]]
- [[AD 사용자 비밀번호 강제 재설정]]
- [[AD 그룹 구성원 추가로 권한 확대]]
- [[임시 SPN 설정]]
- [[AD GPO 쓰기 권한과 영향 범위 열거]]

## 관련 도구

- [[PowerView]]
- [[BloodHound]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[AD 객체 제어권 확보 후 악용 경로 선택]]
