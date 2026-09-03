---
tags:
  - 환경/ad
  - 환경/windows
시작조건: ["인증된 AD 계정 확보", "LAPS 사용 가능성 확인"]
필요권한: ["AD 객체 기본 읽기 권한", "LAPS 비밀번호 속성 읽기는 위임된 확장 권한 또는 동등한 객체 권한"]
필요조건: ["DC의 LDAP 접근", "LAPSToolkit을 실행할 수 있는 PowerShell 환경"]
결과: ["LAPS 위임 그룹", "컴퓨터별 비밀번호 읽기 권한 후보", "LAPS 로컬 관리자 비밀번호"]
---

# LAPS 비밀번호 읽기 권한과 자격 증명 수집

## 한 줄 판단

인증된 AD 계정으로 LAPS 위임 그룹과 컴퓨터별 확장 권한을 먼저 확인하고, 현재 사용 중인 계정이 실제 비밀번호 속성을 읽을 수 있을 때만 해당 컴퓨터의 로컬 관리자 credential을 확보한 것으로 판정한다.

## 사용할 때

- 도메인에서 LAPS가 배포된 컴퓨터와 비밀번호 읽기 위임 범위를 확인할 때.
- 현재 보유 정보: 제어 중인 AD 계정으로 인증된 Windows PowerShell 세션과 대상 도메인·OU 또는 컴퓨터 후보. 비밀번호·NT hash·Kerberos ticket만 따로 보유했다면 먼저 그 자료가 속한 계정의 실행·LDAP 인증 컨텍스트를 준비해야 한다.
- 명령 실행 위치: DC 또는 LDAP Global Catalog가 아니라, 도메인 DNS와 LDAP/LDAPS에 접근해 현재 AD Identity로 조회할 수 있는 PowerShell 호스트.
- 현재 권한과 대상: 현재 계정 또는 중첩 그룹에 `All Extended Rights` 같은 단서가 있더라도, 대상 컴퓨터 객체의 LAPS 비밀번호 속성 읽기 권한은 별도로 확인한다.
- 획득 결과: 실제 `Password` 값은 그 컴퓨터의 LAPS 로컬 관리자 평문 비밀번호다. 도메인 계정 비밀번호나 다른 컴퓨터의 관리자 권한은 아니다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|
| 명령 실행 위치와 LDAP 경로 | PowerShell 호스트에서 도메인 DNS와 DC LDAP/LDAPS에 접근 가능 | [[AD 도메인 컨텍스트 기본 확인]]과 LDAP 조회 | DNS, Kerberos/LDAP 포트와 현재 네트워크 위치 확인 |
| 현재 인증 수단 | LAPSToolkit을 실행하는 PowerShell의 현재 Windows Identity로 LDAP 인증 가능 | `whoami`, 현재 Windows logon과 LDAP 조회 결과 | 별도로 보유한 비밀번호·NT hash·ticket을 함수 인자로 직접 넣는 것으로 해석하지 말고 해당 계정의 실행 컨텍스트부터 준비 |
| 기본 조회 | OU, 컴퓨터와 ACL을 읽을 수 있음 | LDAP 또는 LAPSToolkit 열거 결과 | 대상 OU 범위와 기본 읽기 권한 확인 |
| 비밀번호 읽기 | 대상 LAPS 비밀번호 속성에 대한 위임 권한 | 위임 그룹·확장 권한과 현재 사용 중인 계정의 관계 확인 | 직접·중첩 그룹, ACL 상속과 대상 컴퓨터 객체 재확인 |
| 입력·환경 | LAPSToolkit을 로드할 수 있는 PowerShell 환경 | 모듈 로드와 함수 존재 확인 | 스크립트 경로와 실행 정책 확인 |

## 실행

LAPSToolkit의 아래 함수는 `<PASSWORD>`, `<NT_HASH>` 또는 `<CCACHE_FILE>`을 직접 받는 명령이 아니다. `whoami`에 표시되는 현재 Windows Identity가 LDAP 요청자이며, 위임 그룹·ACL과 실제 LAPS 비밀번호 읽기 권한도 이 요청자 기준으로 판정한다.

### 1. LAPS 위임 그룹과 확장 권한 확인

```powershell
Import-Module .\LAPSToolkit.ps1
Find-LAPSDelegatedGroups
Find-AdmPwdExtendedRights
```

확인할 출력:

- OU별로 LAPS 비밀번호 읽기를 위임받은 그룹.
- 컴퓨터별 `Delegated` 또는 `All Extended Rights` Identity.
- 위임된 그룹 이름만으로 현재 사용 중인 계정의 읽기 권한을 확정하지 않고 직접·중첩 그룹 구성원 관계를 확인한다.

### 2. 현재 사용 중인 계정으로 실제 비밀번호 읽기

```powershell
Get-LAPSComputers
```

확인할 출력:

- LAPS가 적용된 컴퓨터 이름과 비밀번호 만료 시각.
- 현재 사용 중인 계정에 읽기 권한이 있을 때만 표시되는 비어 있지 않은 `Password` 값.
- 비밀번호가 비어 있거나 가려진 컴퓨터는 credential 미확보 상태로 유지한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| OU와 위임 그룹만 확인됨 | LAPS 배포 및 위임 구조 확인, 현재 사용 중인 계정의 권한은 미확정 | LAPS 권한 후보 | 현재 사용 중인 계정의 직접·중첩 그룹 관계와 컴퓨터별 확장 권한 확인 |
| 현재 사용 중인 계정 또는 소속 그룹이 특정 컴퓨터의 읽기 권한을 가짐 | 비밀번호 속성 조회 조건 충족 가능 | 컴퓨터별 LAPS 읽기 권한 | `Get-LAPSComputers`에서 실제 값 반환 여부 확인 |
| 컴퓨터와 만료 시각은 보이나 `Password`가 비어 있음 | LAPS 배포는 확인됐지만 비밀번호 읽기 실패 | 자격 증명 미확보 | 권한 상속, 대상 컴퓨터와 현재 사용 중인 계정을 재확인 |
| 비어 있지 않은 `Password`와 대상 컴퓨터가 반환됨 | 해당 컴퓨터의 LAPS 로컬 관리자 credential 확보 | 호스트 한정 평문 비밀번호 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 SMB·WinRM·RDP 등 실제 접근과 권한 검증 |
| 원격 인증은 성공하지만 관리자 작업이 거부됨 | 계정 이름, LAPS 적용 대상 또는 원격 UAC 조건이 다를 수 있음 | 인증 성공, 관리자 권한 미확정 | 서비스별 실행 주체와 로컬 관리자 권한을 분리해 확인 |
| 함수가 없거나 LDAP 조회가 실패함 | 도구·모듈 또는 DC 경로 문제 | LAPS 상태 미확정 | 모듈 로드, DNS·LDAP 도달성과 현재 AD Identity 재확인 |

## 확인할 출력과 권한

- 위임 그룹, 확장 권한, 실제 비밀번호 반환과 원격 관리자 권한을 네 단계로 구분한다.
- LAPS 비밀번호는 컴퓨터별 로컬 credential이며 도메인 비밀번호나 다른 호스트의 관리자 권한을 의미하지 않는다.

## 관련 도구

- [[powershell]]
- [[LAPSToolkit]]

## 관련 상태 라우터

- LAPS 비밀번호를 실제로 읽었으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- 현재 사용 중인 계정의 AD 권한 범위를 다시 판단할 때: [[AD Identity 확인 후 도메인 컨텍스트 열거]]
