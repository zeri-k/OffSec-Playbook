---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["유효한 AD 계정 자격 증명 또는 해당 계정의 Windows 세션 확보", "제어 중인 사용자 또는 그룹 식별"]
필요권한: ["현재 인증 주체로 GPO 객체와 ACL을 읽을 권한"]
필요조건: ["DC LDAP에 닿는 Windows PowerShell", "제어 중인 사용자 또는 그룹 SID", "PowerView", "선택적으로 GroupPolicy PowerShell 모듈"]
결과: ["쓰기 가능한 GPO 후보", "영향받는 사용자·컴퓨터 범위 후보"]
---

# AD GPO 쓰기 권한과 영향 범위 열거

## 한 줄 판단

현재 제어하는 AD 사용자 또는 그룹의 SID와 GPO·ACL 읽기 권한이 있으면, DC LDAP에 닿는 Windows PowerShell에서 쓰기 ACE를 찾고 GPO GUID를 표시 이름과 연결하여 변경 가능성과 잠재 영향 범위를 별도로 검증할 후보를 얻는다.

## 사용할 때

- 현재 보유 정보: [[AD ACL 권한 열거와 공격 경로 식별]] 또는 BloodHound에서 GPO 제어 edge와 제어 중인 사용자·그룹을 확인했다.
- 명령 실행 위치와 도달 대상: PowerView를 불러올 수 있는 Windows PowerShell에서 대상 도메인의 DC LDAP에 접근할 수 있다.
- 현재 계정과 권한: 조회에 사용하는 AD Identity가 GPO 객체와 ACL을 읽을 수 있다. 조회 계정과 ACE의 `SecurityIdentifier`가 가리키는 제어 주체가 다를 수 있으므로 각각 확인한다.
- 지금 가능한 행동과 결과: GPO를 변경하기 전에 쓰기 ACE의 대상 GPO, 표시 이름과 잠재 적용 OU·사용자·컴퓨터를 좁힐 수 있으며, 이 문서에서는 정책을 수정하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | PowerView를 실행할 Windows PowerShell에서 DC LDAP에 도달 | 현재 도메인·DC를 확인하고 `Get-DomainGPO` 응답 확인 | DNS, DC, LDAP 포트와 피벗 경로를 먼저 확인 |
| 현재 계정 또는 인증 수단 | 유효한 AD 자격 증명·ticket 또는 해당 사용자 컨텍스트의 세션 | 현재 사용자·도메인과 기본 GPO 조회 확인 | [[AD 도메인 컨텍스트 기본 확인]]에서 로컬 계정과 AD Identity 구분 |
| 현재 권한 | GPO 객체와 ACL 읽기 | `Get-DomainGPO`, `Get-ObjectAcl`이 access denied 없이 반환 | 객체 읽기와 ACL 읽기 중 거부되는 단계를 작은 범위로 재확인 |
| 공격 대상의 조건 | 쓰기 ACE가 연결된 GPO와 잠재 적용 대상이 식별 가능 | `ObjectDN`, GUID, 표시 이름과 GPO 링크 확인 | 현재 ACL과 OU·site·domain 링크를 다시 수집 |
| 필요한 파일·목록·주소 | 제어 중인 사용자·그룹 SID, PowerView, 필요하면 GroupPolicy module | `Convert-NameToSid`와 모듈 import 결과 확인 | 이름 형식·그룹 중첩을 확인하고 RSAT가 없으면 PowerView 결과만 사용 |

## 실행

### 도구 경로 선택

| 실행 환경 | 사용할 경로 | 확인하는 결과 | 제한과 실패 시 확인 |
|---|---|---|---|
| PowerView를 불러올 수 있는 Windows PowerShell | `Get-DomainGPO`, `Get-ObjectAcl`, `Convert-NameToSid` | GPO 목록과 현재 제어 주체 SID에 연결된 ACE | 모듈 import, DC LDAP 도달성과 SID 변환을 확인 |
| RSAT의 GroupPolicy module도 설치됨 | `Get-GPO` | GUID·표시 이름·소유자·상태를 교차 확인 | cmdlet 부재는 GPO 부재가 아니므로 PowerView 결과를 사용 |
| BloodHound edge만 보유 | 아래 명령으로 현재 LDAP 상태를 직접 재검증 | 현재 ACE와 GPO 메타데이터 | 그래프 edge만으로 쓰기 가능성이나 영향 범위를 확정하지 않음 |

PowerView 경로가 기본 조회와 ACL 판정을 담당한다. GroupPolicy module은 사용할 수 있을 때 GPO 메타데이터를 교차 확인하는 보조 경로이며, `Get-GPO` 성공만으로 현재 계정의 쓰기 ACE가 확인되는 것은 아니다.

### 1. GPO 목록과 목적 확인

```powershell
Import-Module .\PowerView.ps1
Get-DomainGPO | Select-Object displayname,name,gpcfilesyspath
```

기본 GroupPolicy module을 사용할 수 있다면 교차 검증한다.

```powershell
Get-GPO -All | Select-Object DisplayName,Id,GpoStatus
```

확인할 출력:

- 각 GPO의 `displayname`, GUID인 `name`, `gpcfilesyspath`가 대응하는지 확인한다.
- 결과가 없거나 access denied이면 도메인 컨텍스트, DC 도달성, 객체 읽기 권한을 확인한다. `Get-GPO`만 실패하면 GroupPolicy module과 RSAT 설치 여부를 별도로 확인한다.

### 2. 제어 중인 SID의 GPO ACE만 조회

```powershell
$sid = Convert-NameToSid '<CONTROLLED_USER_OR_GROUP>'
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
```
> ex) `$sid=Convert-NameToSid "Domain Users"`

확인할 출력:

- `ObjectDN`, `SecurityIdentifier`, `ActiveDirectoryRights`가 같은 ACE에 표시된다.
- `WriteProperty`, `WriteDacl`, `WriteOwner`, `GenericAll` 같은 변경 권한.
- 결과가 비어 있으면 즉시 쓰기 권한 부재로 단정하지 않고 사용자 SID, 중첩 그룹 SID, 상속 ACE와 조회 범위를 확인한다.

### 3. GUID를 사람이 읽을 수 있는 이름으로 변환

PowerView만 사용할 수 있으면 다음 명령으로 ACE의 GUID와 GPO 이름을 연결한다.

```powershell
Get-DomainGPO -Identity '<GPO_GUID>' | Select-Object displayname,name,gpcfilesyspath
```

GroupPolicy module을 사용할 수 있으면 같은 GUID를 교차 검증한다.

```powershell
Get-GPO -Guid '<GPO_GUID>'
```

확인할 출력:

- `DisplayName`, `DomainName`, `Owner`, `GpoStatus`.
- GPO 링크와 적용 범위는 OU·site·domain 연결 및 security filtering을 추가 확인한다.
- 한 도구에서 GUID 조회가 실패하면 cmdlet·module 가용성, 중괄호 포함 형식, 대상 도메인과 수집 시점을 확인한다. 표시 이름 확인 실패는 ACE 자체가 없다는 뜻이 아니다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 제어 중인 SID에 GPO 쓰기 ACE | GPO 변경 가능성 | 쓰기 가능한 GPO 후보 | 링크·필터·영향 호스트를 확인할 때까지 변경하지 않음 |
| GUID와 표시 이름 연결 | 정책 목적 식별 | 우선순위가 정해진 GPO 후보 | OU 연결과 적용 대상 직접 확인 |
| 적용 OU에 고가치 호스트 포함 | 대규모 영향 가능 | 권한 상승·내부 이동 후보 | 별도 변경 기법과 복구 절차가 없으면 실행하지 않음 |
| ACE는 있으나 적용 대상 없음 | 변경해도 endpoint 영향이 없을 수 있음 | GPO 제어권만 확인 | 링크·상속·security filtering 재검증 |
| BloodHound edge와 ACL 불일치 | 오래된 수집 또는 SID·권한 해석 문제 | 미확정 경로 | 현재 LDAP ACL을 기준으로 재수집 |

## 확인할 출력과 권한

- 쓰기 ACE는 GPO 수정 후보이며 권한 상승이나 코드 실행 성공이 아니다.
- ACE와 표시 이름이 일치하면 현재 디렉터리에 제어 관계가 존재하는 후보를 확정한다. 실제 적용 대상과 영향은 GPO 링크, 상속, 필터 및 정책 갱신을 확인하기 전까지 확정하지 않는다.
- 적용 범위는 GPO 링크, 상속 차단, security filtering, WMI filter와 대상의 정책 갱신 상태에 따라 달라진다.
- 이 문서는 조회 전용이다. GPO 변경은 다수 endpoint에 영향을 줄 수 있으므로 별도 세부 기법과 복구 계획 없이 수행하지 않는다.

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[AD ACL 권한 열거와 공격 경로 식별]]

## 관련 도구

- [[PowerView]]
- [[BloodHound]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
