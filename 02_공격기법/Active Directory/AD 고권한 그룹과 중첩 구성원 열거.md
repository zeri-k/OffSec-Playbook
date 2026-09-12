---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["도메인과 DC 식별", "현재 사용할 AD 계정 또는 도메인 사용자 세션 확보"]
필요권한: ["현재 AD 계정에 허용된 그룹·구성원 속성 읽기 권한"]
필요조건: ["확인할 고권한·업무 그룹 이름", "Windows AD 모듈·PowerView 또는 Linux windapsearch·SMB 열거 경로"]
결과: ["직접 그룹 구성원", "중첩 그룹을 거쳐 권한을 상속받는 사용자·그룹 후보", "후속 권한 검증 대상"]
---

# AD 고권한 그룹과 중첩 구성원 열거

## 한 줄 판단

도메인·DC와 인증된 AD 계정이 있으면 고권한·업무 그룹의 직접 구성원과 중첩 그룹을 재귀 조회하여 권한 후보 계정을 식별하고, 실제 객체 권한·로컬 관리자·원격 로그온 가능성은 후속 기법에서 따로 검증한다.

## 사용할 때

- 사용자·그룹 목록에서 `Domain Admins`, `Backup Operators`, 운영·백업·보안 관련 그룹 등 확인할 그룹을 골랐을 때.
- 특정 사용자가 어느 중첩 경로로 권한을 얻는지 확인해야 할 때.
- 그룹 이름이나 `admincount`만 있고 실제 구성원 관계는 확인하지 못했을 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 계정 | 그룹 객체를 읽을 수 있는 AD 계정 | LDAP bind 또는 AD cmdlet 반환 | [[AD 도메인 컨텍스트 기본 확인]] |
| 대상 그룹 | 정확한 그룹 이름·DN·SID 중 하나 | 전체 그룹 목록과 상세 객체 조회 | 동명 로컬 그룹과 도메인 그룹 구분 |
| 중첩 범위 | 직접 구성원과 재귀 구성원을 구분 | `-Recurse` 사용 전 직접 결과 저장 | 빈 결과와 잘못된 그룹 이름 구분 |

## 실행

### Linux 공격 호스트에서 실행

```bash
crackmapexec smb <DC> -u <USER> -p '<PASSWORD>' --groups
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' --da
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -PU
```

확인할 출력:

- 그룹별 `membercount`, Domain Admins 직접 구성원과 고권한 그룹별 `nested users`.
- bind 성공과 구성원 반환을 분리하고, 그룹 이름만으로 현재 권한을 확정하지 않는다.

### Windows 공격 호스트에서 실행

```powershell
Import-Module ActiveDirectory
Get-ADGroup -Filter * | Select-Object Name
Get-ADGroupMember -Identity '<GROUP>'
```

PowerView에서 중첩 구성원까지 추적한다.

```powershell
Import-Module .\PowerView.ps1
Get-DomainGroupMember -Identity '<GROUP>' -Recurse
```

확인할 출력:

- `name`, `SamAccountName`, `SID`, `objectClass` 또는 `MemberName`, `MemberSID`, `MemberObjectClass`.
- 구성원이 그룹이면 그 그룹을 거친 사용자까지 반환되는지 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 고권한 그룹의 직접 사용자 구성원 | 고권한 계정 후보 | 우선 조사할 사용자 | [[AD 사용자 객체 열거]]로 계정 상태·속성 확인 |
| 중첩 그룹을 통해 사용자 구성원 확인 | 간접 권한 상속 후보 | 그룹 경로가 연결된 사용자 | [[AD ACL 권한 열거와 공격 경로 식별]]과 대상 서비스 권한 검증 |
| 현재 계정이 원격 접근 그룹에 포함됨 | 원격 로그온 후보 | RDP·WinRM·MSSQL 접근 후보 | [[AD 원격 접근 권한 열거]] |
| 그룹은 존재하지만 구성원 조회가 거부됨 | 객체 존재와 읽기 권한이 분리됨 | 구성원 미확인 | 현재 계정의 읽기 범위와 다른 수집 경로 확인 |

## 확인할 출력과 권한

- 그룹 구성원 관계는 권한 후보이다. 현재 토큰, deny 설정, 대상 서비스의 원격 로그온과 로컬 관리자 권한은 별도 확인한다.
- `Domain Admins` 구성원이라는 출력과 현재 세션에서 Domain Admin 권한을 행사할 수 있다는 판정은 구분한다.

## 관련 도구

- [[windapsearch]]
- [[crackmapexec]]
- [[ActiveDirectory PowerShell 모듈]]
- [[PowerView]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[AD 객체 제어권 확보 후 악용 경로 선택]]
