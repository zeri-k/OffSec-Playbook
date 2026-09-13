---
tags:
  - 환경/ad
  - 환경/windows
시작조건: ["유효한 AD 계정 자격 증명 또는 해당 계정의 Windows 세션 확보", "대상 Windows 호스트 식별"]
필요권한: ["현재 인증 주체로 대상 로컬 그룹 또는 수집된 AD 관계 정보를 읽을 권한"]
필요조건: ["PowerView 실행 호스트에서 대상 Windows 관리 경로 접근 또는 최신 BloodHound 수집 결과", "현재성 확인이 가능한 대상 호스트 목록"]
결과: ["RDP 접근 후보", "WinRM 접근 후보", "MSSQL 관리 권한 후보"]
---

# AD 원격 접근 권한 열거

## 한 줄 판단

유효한 AD Identity와 대상 Windows 호스트 목록이 있으면, 대상 관리 경로에 닿는 PowerView 실행 호스트 또는 최신 BloodHound 자료에서 로컬 그룹·관계 edge를 조회해 RDP·WinRM·MSSQL 관리 후보를 좁히고 실제 서비스 세션으로 최종 확정한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | PowerView 실행 호스트에서 대상 Windows 관리 경로에 도달하거나 BloodHound 자료 사용 가능 | 대상 DNS·SMB/RPC 응답 또는 수집 데이터의 호스트·시각 확인 | 방화벽·피벗 경로를 확인하거나 최신 수집으로 전환 |
| 현재 계정 또는 인증 수단 | 유효한 AD 자격 증명·ticket 또는 해당 사용자 컨텍스트의 세션 | 현재 사용자·도메인과 대상 조회에 사용된 주체 확인 | 로컬 계정과 AD Identity를 분리하고 인증 수단 갱신 |
| 현재 권한 | 대상 로컬 그룹 또는 AD 관계 정보를 읽을 수 있음 | `Get-NetLocalGroupMember` 반환 또는 BloodHound edge 확인 | access denied와 네트워크 오류를 구분하고 다른 수집 경로 사용 |
| 공격 대상의 조건 | 현재 활성 여부를 확인할 Windows 호스트와 RDP·WinRM·MSSQL 후보 서비스 | DNS, SMB·WinRM 응답과 BloodHound 수집 시점 확인 | 오래된 호스트·세션 정보를 제외하고 서비스 상태 재확인 |
| 필요한 파일·목록·주소 | 대상 호스트 목록, PowerView 또는 최신 BloodHound 데이터 | 호스트명·FQDN과 수집 데이터 포함 여부 확인 | [[AD 컴퓨터 객체 열거]]와 [[AD 관계 그래프 수집과 공격 경로 식별]]에서 대상 보강 |

## 실행

### 1. 확인할 AD 계정과 그룹 SID 집합 생성

원격 로컬 그룹의 도메인 계정·그룹 SID와 대조할 확인 대상을 먼저 고정한다. 이 명령을 실행하는 현재 PowerShell 계정과 `<CHECKED_ACCOUNT>`는 서로 다를 수 있다.

`<DOMAIN>\\<CHECKED_ACCOUNT>`는 권한 후보를 확인할 AD 계정(예: `CORP\\operator`)이고, 대상 `<HOST>`는 FQDN 또는 NetBIOS 이름을 문서 안에서 하나로 통일한다. 조회 계정·checked account·대상 서비스의 실제 로그온 주체는 별도로 판정한다.

```powershell
Import-Module .\PowerView.ps1
$checkedAccount = '<DOMAIN>\<CHECKED_ACCOUNT>'
$userSid = Convert-NameToSid $checkedAccount
$groupSids = Get-DomainGroup -MemberIdentity $checkedAccount |
  Select-Object -ExpandProperty objectsid
$effectiveSidCandidates = @($userSid) + @($groupSids) |
  Where-Object { $_ } |
  Sort-Object -Unique
$effectiveSidCandidates
```

확인할 출력:

- 사용자 SID와 직접·중첩 도메인 보안 그룹 SID 후보.
- 빈 그룹 결과를 권한 부재로 단정하지 말고 계정 표기·도메인·LDAP 조회 범위를 확인한다.

### 2. RDP 허용 그룹 확인

대상 호스트의 `Remote Desktop Users` 구성원 중 현재 AD Identity 또는 그 중첩 그룹이 포함되는지 확인한다.

```powershell
$rdpMembers = Get-NetLocalGroupMember -ComputerName '<HOST>' -GroupName 'Remote Desktop Users'
$rdpMembers | Select-Object ComputerName,GroupName,MemberName,SID,IsGroup,IsDomain
$rdpMembers | Where-Object { $_.SID -in $effectiveSidCandidates }
```

- `Get-NetLocalGroupMember`의 SID 필드는 `MemberSID`가 아니라 `SID`다.
- 일치 행은 `<CHECKED_ACCOUNT>` 또는 그 도메인 그룹이 로컬 `Remote Desktop Users`에 들어간 후보다. 중첩 로컬 그룹, `Allow log on through Remote Desktop Services`, 적용되는 deny 권한은 이 필터만으로 완전히 판정하지 못한다.
- 실패하면 호스트 이름 해석, SMB/RPC·방화벽, 원격 그룹 조회 권한과 로컬 그룹 이름을 순서대로 확인한다.

### 3. WinRM 허용 그룹 확인

같은 대상에서 `Remote Management Users` 구성원과 현재 AD Identity의 관계를 확인한다.

```powershell
$winrmMembers = Get-NetLocalGroupMember -ComputerName '<HOST>' -GroupName 'Remote Management Users'
$winrmMembers | Select-Object ComputerName,GroupName,MemberName,SID,IsGroup,IsDomain
$winrmMembers | Where-Object { $_.SID -in $effectiveSidCandidates }
```

확인할 출력:

- `MemberName`, `SID`, `IsGroup`, `IsDomain`.
- 현재 사용 중인 계정의 직접 멤버십뿐 아니라 중첩된 도메인 그룹 관계.
- `Remote Management Users` 일치는 WinRM 후보이며 endpoint ACL·인증 방식·listener까지 확정하지 않는다.
- 결과가 비어 있거나 조회가 거부되면 그룹에 구성원이 없다는 판정과 조회 실패를 구분하고 BloodHound 또는 실제 WinRM 연결로 교차 검증한다.

### 4. BloodHound 관계 교차 검증

- 현재 사용 중인 계정에서 나가는 `CanRDP`, `CanPSRemote`, `SQLAdmin` edge를 확인한다.
- 호스트와 세션 정보의 수집 시점을 확인하고, 각 서비스의 실제 연결로 최종 판정한다.
- edge가 없으면 현재 수집 범위와 권한을 확인한다. edge 존재 여부만으로 서비스가 현재 열려 있거나 세션 생성이 허용된다고 판정하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `<CHECKED_ACCOUNT>` SID 집합과 `Remote Desktop Users` 구성원 SID가 일치 | RDP 그룹 조건 후보. 사용자 권한·deny는 미확인 | RDP 접근 후보 | [[RDP 로그인과 GUI 세션]]에서 실제 GUI 생성 확인 |
| `Remote Management Users` 직접·중첩 멤버십 | WinRM 로그인 가능성 | WinRM 접근 후보 | [[WinRM 원격 PowerShell 세션]]에서 실제 prompt 확인 |
| `SQLAdmin` edge 또는 SQL 관리 계정 단서 | MSSQL 고권한 가능성 | MSSQL 관리 후보 | [[DB 인증과 데이터 열거]]에서 SQL login과 `sysadmin` 확인 |
| 그룹·edge는 있으나 접속 실패 | 정책, endpoint, 방화벽 또는 오래된 수집 가능 | 미확정 원격 접근 | 서비스 상태·NLA·WinRM endpoint·GPO를 구분 |
| 원격 그룹 조회 실패 | 조회 권한 또는 RPC 경로 문제 | 접근 여부 미판정 | 실패를 로그인 불가로 해석하지 말고 BloodHound·직접 접속으로 교차 검증 |

## 확인할 출력과 권한

- 그룹 멤버십과 BloodHound edge는 후보이며 실제 RDP GUI, WinRM prompt, MSSQL query 성공이 최종 증거다.
- 열거 결과는 특정 Identity와 호스트 사이의 로컬 그룹 또는 수집된 관계를 확정한다. 대상 서비스 도달성, deny logon 정책, 사용 가능한 인증 방식과 실제 원격 권한은 별도 접속 전까지 확정하지 않는다.
- `Domain Users`가 표시되어도 deny logon 정책, NLA, 방화벽 또는 endpoint 제한으로 세션이 실패할 수 있다.

## 참고 링크

- [PowerSploit: Get-NetLocalGroupMember](https://powersploit.readthedocs.io/en/latest/Recon/Get-NetLocalGroupMember/)
- [PowerSploit: Get-DomainGroup](https://powersploit.readthedocs.io/en/latest/Recon/Get-DomainGroup/)
- [SpecterOps BloodHound: CanRDP](https://bloodhound.specterops.io/resources/edges/can-rdp)
- [SpecterOps BloodHound: CanPSRemote](https://bloodhound.specterops.io/resources/edges/can-ps-remote)
- [Microsoft: RDS access denied and user authorization troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/troubleshooting-access-denied-and-user-not-authorized-rds-issues)

## 관련 공격기법

- [[AD 컴퓨터 객체 열거]]
- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]]
- [[RDP 로그인과 GUI 세션]]
- [[WinRM 원격 PowerShell 세션]]
- [[DB 인증과 데이터 열거]]

## 관련 도구

- [[PowerView]]
- [[BloodHound]]
- [[PowerUpSQL]]

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]
