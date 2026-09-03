---
tags:
  - 환경/ad
  - 환경/windows
  - 서비스/ldap
  - 기능/열거
실행환경: ["Windows PowerShell"]
필요권한: ["도메인 인증 세션", "평문 비밀번호 확인에는 LAPS 비밀번호 읽기 권한"]
필요조건: ["LAPSToolkit.ps1", "도메인 LDAP 접근"]
결과: ["LAPS 위임 정보", "LAPS 적용 컴퓨터", "접근 가능할 경우 평문 로컬 관리자 비밀번호"]
---

# LAPSToolkit

## 도구 개요

LAPSToolkit은 AD에서 Local Administrator Password Solution(LAPS) 적용 컴퓨터와 비밀번호 읽기 위임 관계를 열거한다. LAPS 배포 범위와 노출된 위임을 점검하고 현재 계정이 읽을 수 있는 로컬 관리자 비밀번호를 식별할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: 도메인에 인증되어 있고 LDAP에 접근 가능한 Windows PowerShell
- 입력: `LAPSToolkit.ps1`과 현재 도메인 컨텍스트
- 권한 조건: 위임 구조 열거에는 일반 도메인 사용자 권한을 사용하며, 평문 비밀번호 출력에는 해당 컴퓨터의 LAPS 비밀번호 읽기 권한이 필요

## 표준 사용법

```powershell
Import-Module .\LAPSToolkit.ps1
<LAPS_FUNCTION>
```

## 대표 예시

### LAPS 비밀번호 읽기가 위임된 그룹 열거

```powershell
Find-LAPSDelegatedGroups
```

확인할 출력:

- `OrgUnit`과 `Delegated Groups` 열
- 각 OU에서 LAPS 비밀번호 읽기를 명시적으로 위임받은 그룹

### 컴퓨터별 확장 권한 보유 계정 또는 그룹 열거

```powershell
Find-AdmPwdExtendedRights
```

확인할 출력:

- `ComputerName`, `Identity`, `Reason` 열
- `Delegated` 또는 `All Extended Rights`로 비밀번호를 읽을 수 있는 사용자·그룹

### LAPS 적용 컴퓨터와 읽을 수 있는 비밀번호 확인

```powershell
Get-LAPSComputers
```

확인할 출력:

- `ComputerName`, `Password`, `Expiration` 열
- 읽기 권한이 있을 때만 표시되는 평문 `Password`

## 주요 함수

| 함수 | 의미 | 사용하는 상황 |
|---|---|---|
| `Find-LAPSDelegatedGroups` | OU별로 LAPS 읽기를 위임받은 그룹 열거 | 공격 가능한 위임 그룹과 보호 그룹 관계 확인 |
| `Find-AdmPwdExtendedRights` | LAPS 컴퓨터별 확장 권한 보유 계정 또는 그룹 열거 | `All Extended Rights` 등 직접 읽기 경로 확인 |
| `Get-LAPSComputers` | LAPS 적용 컴퓨터, 비밀번호, 만료 시점 조회 | 현재 사용 중인 계정의 실제 비밀번호 읽기 가능 여부 확인 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Delegated Groups` | OU에 LAPS 읽기 권한을 위임받은 그룹 | 현재 사용자 또는 제어 가능한 사용자의 그룹 구성원 관계 확인 |
| `Reason: Delegated` | 해당 계정 또는 그룹이 명시적 위임으로 읽기 가능 | 위임 범위가 특정 OU 또는 컴퓨터에 한정되는지 확인 |
| `All Extended Rights` | LAPS 비밀번호 읽기를 포함할 수 있는 광범위한 권한 | 권한 상속과 대상 컴퓨터 범위 확인 |
| `Password` 값 표시 | 현재 사용 중인 계정이 해당 LAPS 비밀번호를 평문으로 읽을 수 있음 | 만료 시점과 대상 호스트의 로컬 관리자 계정명 확인 |
| `Password`가 비어 있음 | LAPS 적용 여부 또는 현재 사용 중인 계정의 읽기 권한 부족 | 컴퓨터 속성, 위임 ACL, 스키마 버전 확인 |

## 관련 공격기법

- [[LAPS 비밀번호 읽기 권한과 자격 증명 수집]]
