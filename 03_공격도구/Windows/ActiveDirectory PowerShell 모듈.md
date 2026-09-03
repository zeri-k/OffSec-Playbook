---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/열거
실행환경: ["Windows PowerShell"]
필요권한: ["도메인 인증 세션"]
필요조건: ["ActiveDirectory PowerShell 모듈", "도메인 서비스 접근"]
결과: ["도메인 정보", "사용자와 그룹 정보", "트러스트 관계", "SPN 계정 목록"]
---

# ActiveDirectory PowerShell 모듈

## 도구 개요

ActiveDirectory PowerShell 모듈은 `Get-AD*` cmdlet으로 도메인·사용자·그룹·컴퓨터와 트러스트 같은 AD 객체를 조회한다. Windows PowerShell에서 Microsoft 관리 도구의 기본 문법으로 AD 구조와 속성을 확인할 때 적합하며, PowerView 함수와는 문법이 다르다.

## 필요한 입력과 실행 환경

- 실행 환경: ActiveDirectory 모듈이 설치된 도메인 가입 또는 관리용 Windows 호스트의 PowerShell
- 입력: 현재 도메인 컨텍스트, 객체 filter, 사용자·그룹 identity와 필요한 property
- 권한 조건: 도메인 인증 세션이 필요하며 반환 범위는 현재 사용 중인 계정의 AD 객체 읽기 권한에 따라 달라짐

## 표준 사용법

```powershell
Import-Module ActiveDirectory
Get-AD<OBJECT> [options]
```

## 대표 예시

### 현재 도메인 기본 정보 조회

```powershell
Import-Module ActiveDirectory
Get-ADDomain
```

확인할 출력:

- `DNSRoot`, `NetBIOSName`, `DomainSID`, `DomainMode`
- `PDCEmulator`, `ReplicaDirectoryServers`, `ChildDomains`

### SPN이 설정된 사용자 검색

```powershell
Get-ADUser -Filter { ServicePrincipalName -ne "$null" } -Properties ServicePrincipalName
```

확인할 출력:

- `SamAccountName`, `Enabled`, `ServicePrincipalName`, `SID`

### 도메인 트러스트 관계 열거

```powershell
Get-ADTrust -Filter *
```

확인할 출력:

- `Source`, `Target`, `Direction`, `IntraForest`, `ForestTransitive`
- SID filtering과 selective authentication 관련 속성

### 특정 그룹의 직접 구성원 확인

```powershell
Get-ADGroupMember -Identity "<GROUP>"
```

확인할 출력:

- `name`, `objectClass`, `SamAccountName`, `SID`

### 부모 도메인의 Enterprise Admins SID 확인

```powershell
Get-ADGroup -Identity "Enterprise Admins" -Server <ROOT_FQDN> | Select-Object DistinguishedName,ObjectSid
```

확인할 출력:

- `DistinguishedName`이 부모 도메인 DN에 속하는지 확인한다.
- `ObjectSid`는 부모 도메인 SID와 RID `519`로 끝나는 `Enterprise Admins` SID여야 한다.

### Protected Users 그룹과 구성원 확인

```powershell
Get-ADGroup -Identity "Protected Users" -Properties Name,Description,Members
```

확인할 출력:

- `Members`에 포함된 계정과 실제 고권한 계정 목록을 비교한다.
- 그룹 멤버십은 NTLM·delegation·ticket 수명 같은 인증 동작을 제한할 수 있으므로, 특정 인증 방식의 실패와 계정 비밀번호 오류를 구분한다.

## 주요 옵션과 cmdlet

| 옵션·cmdlet | 의미 | 사용하는 상황 |
|---|---|---|
| `Import-Module ActiveDirectory` | ActiveDirectory cmdlet 불러오기 | `Get-AD*` 명령을 사용하기 전 |
| `Get-ADDomain` | 현재 또는 지정한 도메인의 기본 속성 반환 | 도메인 SID, 기능 수준, DC와 하위 도메인 확인 |
| `Get-ADUser` | AD 사용자 조회 | SPN, 계정 상태와 사용자 property 열거 |
| `Get-ADTrust` | 도메인 트러스트 조회 | 트러스트 방향과 범위 확인 |
| `Get-ADGroupMember` | 지정한 그룹 구성원 조회 | 고권한 또는 운영 그룹의 직접 구성원 확인 |
| `Get-ADGroup` | 그룹 객체와 SID 조회 | 다른 도메인의 Enterprise Admins 등 그룹 SID 확인 |
| `Get-ADGroup -Properties Members` | 그룹 객체와 직접 구성원 속성 조회 | Protected Users 등 보호·고권한 그룹 감사 |
| `-Server <DOMAIN_OR_DC>` | 조회할 도메인 또는 DC 지정 | 현재 도메인이 아닌 부모·신뢰 대상 도메인 객체 조회 |
| `-Filter <EXPRESSION>` | 반환할 AD 객체 조건 지정 | 전체 객체 또는 특정 속성 보유 객체 검색 |
| `-Properties <NAME>` | 기본 출력 외 추가 속성 요청 | SPN 등 필요한 attribute 확인 |
| `-Identity <VALUE>` | 특정 객체를 이름, DN, GUID, SID 등으로 지정 | 사용자·그룹 상세 조회 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `DomainSID`, `DomainMode` | 현재 AD 도메인과 기능 수준 식별 | DC, 하위 도메인과 트러스트 범위 확인 |
| `ServicePrincipalName` | 서비스에 연결된 사용자 계정 발견 | [[Kerberoasting]] 대상 적합성 확인 |
| `Direction`, `IntraForest`, `ForestTransitive` | 트러스트 방향과 포리스트 경계 확인 | 신뢰 방향에 따른 인증·열거 가능 범위 검토 |
| `SamAccountName`, `SID` | 그룹의 직접 구성원 식별 | 중첩 그룹과 실제 권한 별도 확인 |
| `The specified module 'ActiveDirectory' was not loaded` | 모듈이 설치되지 않았거나 module path에 없음 | RSAT/AD DS 관리 도구 설치 여부와 `Get-Module -ListAvailable` 확인 |
| `Cannot find an object with identity` | identity 형식 또는 검색 도메인 불일치 | DN, SID, 이름과 현재 도메인 컨텍스트 확인 |
| `IntraForest`, `ForestTransitive`, `SelectiveAuthentication`, SID filtering 속성 | 포리스트 경계와 인증 제한 단서 | 실제 대상 도메인 객체 조회·서비스 인증으로 방향과 제한 확인 |

## 버전과 환경 차이

- ActiveDirectory 모듈은 모든 Windows 호스트에 기본 설치되는 구성요소가 아니다. RSAT 또는 AD DS 관리 도구가 제공되는지 `Get-Module -ListAvailable ActiveDirectory`로 먼저 확인한다.
- 이 모듈의 `Get-AD*` cmdlet은 PowerView의 `Get-Domain*` 함수와 별개다. 이름이 비슷한 객체 조회 명령의 옵션을 서로 혼용하지 않는다.

## 관련 공격기법

- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[SPN 계정 열거]]
- [[Kerberoasting]]
- [[AD 도메인 트러스트 열거와 공격 경로 식별]]
- [[자식 도메인 ExtraSids Golden Ticket]]
- [[AD 보안 구성과 GPO 감사]]
