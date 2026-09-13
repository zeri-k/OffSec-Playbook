---
tags:
  - 환경/ad
  - 기능/열거
실행환경: ["Windows"]
필요권한: ["도메인 인증 세션"]
필요조건: ["SharpView.exe", "AD 서비스 접근"]
결과: ["LDAP 검색 조건", "AD 사용자와 객체 속성"]
---

# SharpView

## 도구 개요

SharpView는 PowerView 기능을 .NET 실행 파일 형태로 제공해 AD 객체·그룹·ACL을 조회한다. PowerShell 스크립트 실행이 제한된 Windows 환경에서 PowerView와 유사한 열거가 필요할 때 적합하지만 `SharpView.exe <METHOD>` 문법을 사용하며 PowerShell pipeline과는 호환되지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: 도메인 서비스에 접근 가능한 Windows 호스트
- 필요한 입력: SharpView method와 대상 identity
- 권한 조건: 현재 사용 중인 AD 계정이 읽을 수 있는 LDAP 범위

## 표준 사용법

```powershell
.\SharpView.exe <METHOD> [OPTIONS]
```

## 대표 예시

### method 인수 확인

```powershell
.\SharpView.exe Get-DomainUser -Help
```

확인할 출력:

- `Get_DomainUser`가 받는 `-Identity`, `-SPN`, `-LDAPFilter`, `-Server` 등의 인수 목록.

### 특정 사용자 조회

```powershell
.\SharpView.exe Get-DomainUser -Identity <USER>
```

`<USER>`는 현재 도메인에서 조회할 사용자명(예: `alice`)이며, 명령은 `SharpView.exe`가 있는 Windows 호스트에서 실행한다.

확인할 출력:

- `[Get-DomainSearcher] search base`와 `[Get-DomainUser] filter string`.
- `samaccountname`, `distinguishedname`, `memberof`, `useraccountcontrol`, `badpwdcount` 등 반환 속성.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-Help` | 선택한 method의 인수 나열 | PowerView와 문법을 혼동하지 않도록 먼저 확인 |
| `-Identity <VALUE>` | 조회 대상을 사용자명 등으로 제한 | 특정 AD 계정 상세 확인 |
| `-Properties <LIST>` | 반환 속성 제한 | 필요한 속성만 확인할 때 |
| `-Server <DC>` | 질의할 서버 지정 | 기본 DC 탐색이 실패하거나 대상을 고정할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `search base` | 실제 LDAP 검색 기준 DN | 예상 도메인과 OU인지 확인 |
| `filter string` | 적용된 LDAP 필터 | 대상 identity와 필터 조건이 맞는지 확인 |
| 사용자 속성 반환 | 현재 사용 중인 계정으로 객체 읽기 성공 | 그룹·SPN·UAC·계정 상태를 대응 기법에서 해석 |
| 빈 결과 | 대상 불일치·검색 범위·권한 문제 가능성 | identity, domain, server와 현재 사용 중인 계정 확인 |

## 버전과 환경 차이

- SharpView는 PowerView의 .NET 포트다. `Import-Module`이나 PowerShell pipeline을 전제로 한 PowerView 문법과 `SharpView.exe <METHOD>` 문법을 혼용하지 않는다.

## 관련 공격기법

- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[SPN 계정 열거]]
- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[AD 계정 위험 속성과 Description 열거]]
