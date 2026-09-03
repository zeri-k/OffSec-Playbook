---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 서비스/ldap
시작조건: ["도메인과 DC 식별", "유효한 AD 계정 또는 도메인 사용자 세션 확보"]
필요권한: ["일반 도메인 사용자 수준의 사용자·SPN 속성 읽기 권한"]
필요조건: ["Linux에서 DC LDAP·Kerberos 접근 또는 Windows 도메인 세션", "현재 도메인 또는 조회 가능한 신뢰 대상 도메인"]
결과: ["SPN이 설정된 사용자 기반 서비스 계정", "SPN·서비스·호스트와 계정의 대응 관계", "Kerberoasting 대상 후보"]
---

# SPN 계정 열거

## 한 줄 판단

유효한 AD 계정 또는 도메인 사용자 세션이 있으면 현재 도메인이나 조회 가능한 신뢰 대상 도메인에서 Service Principal Name(SPN)이 설정된 객체를 열거하고, 사용자 기반 서비스 계정과 컴퓨터·관리형 계정을 구분하여 Kerberoasting 대상 후보를 만든다.

## 사용할 때

- [[Kerberoasting]] 전에 실제 SPN 계정과 서비스 이름을 먼저 확인할 때.
- MSSQL·HTTP·백업·ADFS 같은 서비스 단서가 어느 AD 계정의 SPN에 연결되는지 확인할 때.
- trust를 확인한 뒤 대상 도메인의 SPN 계정을 별도로 조회할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 요청자 계정 | 유효한 AD 계정 또는 도메인 사용자 세션 | LDAP·Kerberos 인증 결과 | 계정·도메인 형식과 ticket 주체 확인 |
| 현재 도메인 경로 | DC LDAP 389·636과 필요 시 Kerberos 88 접근 | 연결·bind 응답 확인 | DNS·FQDN·피벗 경로 확인 |
| 신뢰 대상 경로 | 대상 도메인 LDAP 조회와 Kerberos 요청 방향 확인 | trust Source·Target과 대상 객체 반환 | [[AD 도메인 트러스트 열거와 공격 경로 식별]] |

## 실행

### 현재 도메인 대상

#### Linux 공격 호스트에서 실행

TGS를 요청하지 않고 SPN 계정만 나열한다. `-request`는 이 단계에서 사용하지 않는다.

```bash
GetUserSPNs.py -dc-ip <DC_IP> '<DOMAIN>/<REQUESTER>'
```

확인할 출력:

- `ServicePrincipalName`, `Name`, `MemberOf`, `PasswordLastSet`, `LastLogon`.
- SPN 목록 반환과 `$krb5tgs$` hash 획득은 다른 단계다.

#### Windows 공격 호스트에서 실행

```powershell
Import-Module ActiveDirectory
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

특정 서비스 SPN이 주어졌다면 계정명만 출력하지 말고 SPN 속성을 함께 표시해 소유 계정을 대조한다.

```powershell
Get-DomainUser * -SPN | Select-Object SamAccountName,ServicePrincipalName
```

확인할 출력:

- `MSSQLSvc/<MSSQL_FQDN>:<PORT>` 같은 찾고 있는 SPN과 같은 행의 `SamAccountName`.
- 계정 이름만 일치하는 것이 아니라 `ServicePrincipalName` 값에 대상 호스트·포트가 함께 표시되는지 확인한다.

기본 도구만 사용할 수 있으면 전체 SPN 등록을 먼저 확인한다.

```cmd
setspn.exe -Q */*
```

`setspn` 결과에는 컴퓨터 객체의 SPN도 섞일 수 있으므로 계정 유형을 다시 확인한다.

### 신뢰 대상 도메인 대상

#### Linux 공격 호스트에서 실행

```bash
GetUserSPNs.py -target-domain <TARGET_TRUST_DOMAIN> '<SOURCE_DOMAIN>/<REQUESTER>'
```

#### Windows 공격 호스트에서 실행

```powershell
Get-DomainUser -SPN -Domain <TARGET_TRUST_DOMAIN> | Select-Object SamAccountName,MemberOf,ServicePrincipalName
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 사용자 계정과 SPN이 반환됨 | TGS 요청 가능한 서비스 계정 후보 | Kerberoasting 대상 목록 | 비밀번호 설정 시점·그룹·서비스 중요도를 확인한 뒤 [[Kerberoasting]] |
| 알려진 `MSSQLSvc/<MSSQL_FQDN>:<PORT>`와 사용자 계정이 같은 객체에서 반환됨 | 해당 MSSQL SPN의 소유 계정 식별 | 특정 Kerberoasting 대상 계정 | [[Kerberoasting]]에서 해당 계정만 TGS 요청 |
| 컴퓨터 계정 또는 관리형 계정만 반환됨 | 일반 사용자 기반 서비스 계정과 다른 비밀번호 관리 특성 | 낮은 우선순위 또는 별도 검토 대상 | 계정 유형과 실제 서비스 용도 확인 |
| 신뢰 대상 도메인 SPN 반환 | 현재 계정으로 대상 도메인 객체 조회 가능 | 교차 도메인 Kerberoasting 후보 | [[Kerberoasting]]의 `신뢰 대상 도메인` 절차 |
| 결과가 비거나 LDAP 오류 | SPN 부재보다 domain·Base DN·조회 권한 문제 가능 | 대상 미확정 | 현재 도메인, `-target-domain`, DC·LDAP 접근과 요청자 인증 확인 |

## 확인할 출력과 권한

- SPN이 설정된 계정은 Kerberoasting 후보일 뿐이며 TGS hash, 평문 비밀번호 또는 서비스 권한을 확보한 상태가 아니다.
- `MemberOf`, `admincount`, 서비스 이름은 우선순위 단서다. 실제 그룹·서비스 권한은 별도로 검증한다.

## 관련 도구

- [[impacket-GetUserSPNs]]
- [[ActiveDirectory PowerShell 모듈]]
- [[PowerView]]
- `setspn.exe`

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
