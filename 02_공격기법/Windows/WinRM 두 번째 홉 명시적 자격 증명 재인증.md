---
tags:
  - 환경/windows
  - 환경/ad
  - 서비스/winrm
시작조건: ["<JUMP_HOST>의 WinRM PowerShell 세션에서 로컬 명령은 성공하지만 DC 또는 두 번째 Kerberos 서비스 접근 실패"]
필요권한: ["원래 AD 계정의 대상 리소스 읽기 권한", "RunAs endpoint 사용 시 <JUMP_HOST>의 상승된 로컬 관리자 권한"]
필요조건: ["원래 AD 계정의 plaintext password", "<JUMP_HOST>에서 DC 또는 두 번째 대상의 FQDN·Kerberos·LDAP 포트 도달 가능", "현재 Kerberos ticket 목록"]
결과: ["명시적 AD 자격 증명으로 두 번째 리소스 조회", "RunAs 선택 시 지정 계정의 TGT를 가진 임시 WinRM 세션"]
---

# WinRM 두 번째 홉 명시적 자격 증명 재인증

## 한 줄 판단

`<JUMP_HOST>`의 WinRM 세션에서 로컬 명령은 성공하지만 DC 또는 두 번째 Kerberos 서비스 접근만 실패하고 원래 AD 계정의 plaintext password가 있으면, 지원되는 명령에 `PSCredential`을 전달해 해당 계정으로 두 번째 서비스에 다시 인증한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | `<JUMP_HOST>`의 WinRM PowerShell 세션 | `whoami`, `hostname` | 공격 호스트 셸과 원격 PowerShell prompt를 구분 |
| 현재 인증 상태 | WinRM 대상 service ticket과 TGT 보유 여부 구분 | `klist` | ticket 주체·SPN·만료와 현재 로그온 계정을 확인 |
| 두 번째 서비스 경로 | `<JUMP_HOST>`에서 DC 또는 `<SECOND_HOST>`의 FQDN과 필요한 포트 도달 | DNS와 포트 연결 확인 | IP 대신 FQDN, DNS suffix, SPN, 방화벽과 시간 확인 |
| 재인증 자료 | 원래 AD 계정의 plaintext password | 동일 계정의 직접 인증 가능 여부 확인 | 계정의 도메인 표기, password 만료·오류를 확인 |
| RunAs 선택 조건 | `<JUMP_HOST>`의 상승된 로컬 관리자 token과 GUI prompt, 고유한 endpoint 이름 | 관리자 token, WinRM service와 endpoint 기준선 확인 | 조건이 없거나 같은 이름의 endpoint가 이미 있으면 `PSCredential` 방식만 사용 |

## 실행

`<JUMP_HOST>`는 현재 WinRM 세션이 열린 Windows 호스트이고, `<SECOND_HOST>`는 그 호스트에서 FQDN으로 도달해야 하는 두 번째 서비스 호스트다. `<DOMAIN>\<USER>`와 `<PASSWORD>`는 최초 WinRM 계정과 동일한 재인증 자료일 때만 함께 사용하며, RunAs endpoint 이름은 기존 endpoint와 겹치지 않는 값만 사용한다.

### 1. Windows 점프 호스트에서 두 번째 홉 후보 확인

```powershell
klist
Get-DomainUser -SPN
```

확인할 출력:

- `klist`에 `HTTP/<JUMP_HOST>` ticket은 있지만 `krbtgt/<DOMAIN>` TGT가 없다.
- 로컬 명령은 성공하지만 PowerView 조회에서 `FindAll` 또는 `An operations error occurred`가 나타난다.
- 이 조합만으로 원인을 확정하지 않는다. DNS, DC 포트, 시간, PowerView 모듈과 현재 계정의 LDAP 읽기 권한을 먼저 분리 확인한다.

### 2. Windows 점프 호스트에서 PSCredential로 재인증

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER>', $SecPassword)
Get-DomainUser -SPN -Credential $Cred | Select-Object samaccountname
Remove-Variable Cred,SecPassword
```

확인할 출력:

- `samaccountname` 목록이 반환되면 해당 명령이 `<DOMAIN>\<USER>`로 DC에 재인증해 객체를 읽은 것이다.
- 다른 PowerView 명령도 `-Credential`을 지원할 때만 같은 객체를 전달한다.
- 조회 성공은 원래 계정의 AD 읽기 권한을 보여 줄 뿐, 점프 호스트 또는 두 번째 대상의 관리자 권한을 의미하지 않는다.
- `Remove-Variable`은 현재 PowerShell scope의 참조를 제거할 뿐 managed memory에서 plaintext 파생 자료가 즉시 소거됐음을 보장하지 않는다. 작업 후 원격 PowerShell session 자체도 종료한다.

### 3. Windows 점프 호스트에서 RunAs endpoint 사용

관리자 권한 Windows PowerShell 콘솔과 GUI credential prompt를 사용할 수 있고, 현재 WinRM 세션 단절 가능성을 확인한 경우에만 수행한다. 기존 endpoint를 덮어쓰지 않도록 작업 전 목록과 WinRM 상태를 먼저 기록하고 `<TEMP_ENDPOINT>`가 없을 때만 진행한다.

```powershell
Get-Service WinRM | Select-Object Name,Status,StartType
Get-PSSessionConfiguration | Select-Object Name,PSVersion,Permission
Get-PSSessionConfiguration -Name '<TEMP_ENDPOINT>' -ErrorAction SilentlyContinue
(Get-Command Register-PSSessionConfiguration).Parameters.ContainsKey('NoServiceRestart')
(Get-Command Unregister-PSSessionConfiguration).Parameters.ContainsKey('NoServiceRestart')
```

마지막 두 결과가 모두 `True`인 PowerShell 7.5 이상에서는 등록·삭제와 service restart를 분리하여 변경마다 한 번만 재시작한다.

```powershell
Register-PSSessionConfiguration -Name '<TEMP_ENDPOINT>' -RunAsCredential '<DOMAIN>\<USER>' -NoServiceRestart
Restart-Service WinRM
Enter-PSSession -ComputerName '<JUMP_HOST>' -Credential '<DOMAIN>\<USER>' -ConfigurationName '<TEMP_ENDPOINT>'
klist
```

`NoServiceRestart`를 지원하지 않는 Windows PowerShell에서는 `Register-PSSessionConfiguration -Name '<TEMP_ENDPOINT>' -RunAsCredential '<DOMAIN>\<USER>'`가 표시하는 restart prompt를 로컬 관리자 콘솔에서 한 번 확인한다. 이는 cmdlet의 UI 입력 동작이며, 이 경우 뒤에서 `Restart-Service`를 중복 실행하지 않는다. 등록 또는 restart 때 기존 원격 session이 끊길 수 있으므로, 별도 관리 경로 없이 현재 WinRM session 하나만 가진 상태에서는 이 분기를 사용하지 않는다.

확인할 출력:

- `<TEMP_ENDPOINT>`가 `WSManConfig`에 등록된다.
- 새 세션의 `klist`에 `krbtgt/<DOMAIN>` TGT가 표시된다.
- credential 매개변수 없이 두 번째 Kerberos 서비스 조회가 성공한다.

## 관찰과 판단

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 명시적 credential 조회 성공 | 원래 AD 계정으로 두 번째 서비스에 재인증됨 | 점프 호스트에서 해당 계정 권한의 AD 조회 가능 | [[AD Identity 확인 후 도메인 컨텍스트 열거]] |
| RunAs 세션에서 TGT와 AD 조회 확인 | 임시 endpoint가 지정 계정으로 동작함 | 지정 계정으로 두 번째 Kerberos 리소스 접근 가능 | 필요한 작업 후 endpoint 복구 |
| 동일 오류 지속 | Double Hop 외 원인이 남아 있음 | AD 접근 실패 | DNS·DC 포트·LDAP 권한·도메인 지정·시간·모듈 상태를 분리 확인 |
| WinRM 세션 단절 | 서비스 재시작 영향 | 원격 세션 없음 | 새 endpoint 또는 기존 endpoint로 다시 연결 |
| RunAs 등록 실패 | GUI prompt 또는 상승된 관리자 조건 미충족 | 구성 변경 불가 | `PSCredential` 방식 사용 |

## 결과 상태

- WinRM 로그인 성공, `HTTP/<JUMP_HOST>` service ticket, 사용자 TGT와 두 번째 서비스 인증 성공은 서로 다른 상태다.
- 두 번째 서비스 인증 성공도 최종 리소스의 읽기·쓰기 또는 관리자 권한을 뜻하지 않는다. 원래 AD 계정의 ACL과 그룹 권한을 별도로 확인한다.
- `PSCredential` 방식은 지원되는 명령 하나에만 자격 증명을 전달한다. RunAs endpoint는 새 세션 전체의 실행 Identity와 WinRM 구성을 바꾼다.

## 다음 행동

- AD 객체 조회가 가능해지면 [[AD Identity 확인 후 도메인 컨텍스트 열거]]에서 현재 계정의 읽기·객체 권한을 확인한다.
- 새 원격 Windows 세션을 얻으면 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 실행 주체와 token을 확인한다.

## 변경 영향과 복구

`PSCredential` 방식은 영속 구성을 바꾸지 않는다. RunAs endpoint를 만들었다면 연결이 살아 있을 때 후속 작업을 끝낸 다음 정확한 임시 이름만 제거한다. 삭제 명령도 설치된 PowerShell에서 `NoServiceRestart` 지원 여부를 확인해 WinRM을 한 번만 재시작한다.

```powershell
Exit-PSSession
Get-PSSessionConfiguration -Name '<TEMP_ENDPOINT>'
Unregister-PSSessionConfiguration -Name '<TEMP_ENDPOINT>' -NoServiceRestart -Force
Restart-Service WinRM
Get-PSSessionConfiguration | Where-Object { $_.Name -eq '<TEMP_ENDPOINT>' }
Get-Service WinRM | Select-Object Name,Status,StartType
Get-PSSessionConfiguration | Select-Object Name,PSVersion,Permission
```

위 block은 `Unregister-PSSessionConfiguration`이 `NoServiceRestart`를 지원할 때 사용한다. 지원하지 않으면 `Unregister-PSSessionConfiguration -Name '<TEMP_ENDPOINT>' -Force`가 WinRM을 재시작하므로 별도 `Restart-Service`를 실행하지 않는다. 임시 endpoint가 없어지고 WinRM status·start type과 기존 endpoint 목록이 작업 전 기준과 일치해야 복구 완료다. 연결이 이미 끊겨 이 상태를 확인하지 못하면 `원격 복구 미확인`으로 남긴다.

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| WinRM PSSession endpoint | 지정 계정으로 실행되는 endpoint 생성 | `Get-PSSessionConfiguration` | `<TEMP_ENDPOINT>`만 제거하고 빈 조회 확인 |
| WinRM 서비스 | 등록·삭제를 적용하는 restart로 연결된 세션 단절 | 작업 전 status·start type, 기존 endpoint 목록과 재접속 | 지원되는 `NoServiceRestart` 분기에서는 변경마다 명시적으로 한 번만 재시작하고, 미지원 분기에서는 cmdlet 자체 restart만 사용한 뒤 기준 상태 확인 |

## 관련 공격기법

- [[WinRM 원격 PowerShell 세션]]
- [[AD 도메인 컨텍스트 기본 확인]]
- [[SPN 계정 열거]]

## 관련 도구

- [[PowerView]]
- [[evil-winrm]]
- [[klist]]
- [[powershell]]

## 관련 상태 라우터

- [[WinRM Kerberos Double Hop 진단과 재인증]]
- [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 참고 링크

- [Microsoft: Making the second hop in PowerShell Remoting](https://learn.microsoft.com/powershell/scripting/security/remoting/ps-remoting-second-hop)
- [Microsoft: Register-PSSessionConfiguration](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/register-pssessionconfiguration)
- [Microsoft: Unregister-PSSessionConfiguration](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/unregister-pssessionconfiguration)
