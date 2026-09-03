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

## 사용할 때

- [[AD ACL 권한 열거와 공격 경로 식별]]에서 대상 사용자에 대한 `User-Force-Change-Password`가 확인되었을 때.
- 기존 비밀번호를 알지 못해도 비밀번호를 재설정할 수 있는 ACL 영향도를 검증해야 할 때.
- 계정 소유자 영향과 원복 방법을 사전에 합의했을 때.

## 전제 조건

| 구분 | 조건 | 확인 방법 |
|---|---|---|
| 시작 상태 | 인증된 AD 계정 | 현재 사용자와 도메인 확인 |
| 필요 권한 | 대상 사용자 비밀번호 재설정 권한 | 제어 중인 SID와 대상 객체의 ACE 대조 |
| 입력·환경 | 임시 비밀번호와 복구 계획 | 잠금·복잡도 정책 및 계정 소유자 영향 확인 |

## 실행

### 1. 대상과 권한 재확인

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid '<CONTROLLED_USER>'
Get-DomainObjectACL -ResolveGUIDs -Identity '<TARGET_USER>' |
  Where-Object { $_.SecurityIdentifier -eq $sid }
```

확인할 출력:

- 대상 사용자의 `ObjectDN`과 현재 계정 SID가 같은 행에 표시된다.
- `ObjectAceType`에 `User-Force-Change-Password`가 표시된다.

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

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상 사용자의 비밀번호 | 기존 세션·서비스·예약 작업이 실패할 수 있음 | 계정 소유자와 관련 서비스 상태 확인 | 원래 비밀번호를 알고 있는 경우 같은 절차로 원래 값 복원 |
| 원래 비밀번호를 모르는 계정 | 자동 원복 불가 | 재설정 전 영향과 복구 담당자 확인 | 계정 소유자 또는 관리자가 새 비밀번호를 재설정하도록 인계 |

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[원격 비밀번호 공격]]

## 관련 도구

- [[PowerView]]

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]
