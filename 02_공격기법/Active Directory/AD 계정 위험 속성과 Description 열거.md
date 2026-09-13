---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["유효한 AD 계정 자격 증명 또는 해당 계정의 Windows 세션 확보", "실행 호스트에서 DC LDAP 접근 가능"]
필요권한: ["현재 인증 주체로 도메인 사용자 Description과 UAC 속성을 읽을 권한"]
필요조건: ["DC LDAP에 닿는 Windows PowerShell 또는 CMD", "PowerView·ActiveDirectory PowerShell 모듈 또는 dsquery"]
결과: ["Description의 민감 정보 후보", "위험한 UAC 설정 계정 후보", "별도 인증이 필요한 자격 증명 후보"]
---

# AD 계정 위험 속성과 Description 열거

## 한 줄 판단

유효한 AD Identity로 DC LDAP에 닿는 Windows PowerShell 세션이 있으면, 읽을 수 있는 사용자 Description과 UAC 속성을 조회하여 민감 정보·약한 설정·자격 증명 후보를 찾되 실제 인증 성공이나 비밀번호 추출과 구분한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | PowerShell 실행 호스트에서 DC LDAP에 도달 | 기본 사용자 객체 조회와 DC 이름 해석 확인 | DNS, LDAP 포트와 피벗 경로를 확인 |
| 현재 계정 또는 인증 수단 | 유효한 AD 자격 증명·ticket 또는 해당 사용자 컨텍스트의 세션 | 현재 사용자·도메인과 LDAP 조회 성공 확인 | [[AD 도메인 컨텍스트 기본 확인]]에서 Identity와 인증 상태 확인 |
| 현재 권한 | 사용자 객체의 Description·UAC 속성 읽기 | 알려진 사용자 한 명의 속성이 access denied 없이 반환되는지 확인 | 객체 존재, 속성 미설정과 읽기 거부를 구분 |
| 공격 대상의 조건 | 대상 도메인의 사용자 객체를 열거할 수 있음 | `samaccountname`과 요청 속성이 함께 반환되는지 확인 | 검색 base, 도메인 컨텍스트와 사용자 필터 확인 |
| 필요한 파일·목록·주소 | PowerView 또는 ActiveDirectory PowerShell module | 모듈 import와 cmdlet 존재 확인 | 사용할 수 있는 모듈에 맞는 명령 경로를 선택 |

## 실행

### 1. Description의 민감 정보 확인

```powershell
Get-DomainUser * |
  Select-Object samaccountname,description |
  Where-Object { $_.Description -ne $null }
```

확인할 출력:

- `samaccountname`과 비어 있지 않은 `description`을 함께 확인해 문자열의 소유 계정을 식별한다.
- 결과가 없으면 Description 미설정, 검색 범위 오류와 속성 읽기 제한을 구분한다. 빈 결과는 민감 정보가 다른 속성에 없다는 뜻이 아니다.

### 2. `PASSWD_NOTREQD` 계정 확인

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD |
  Select-Object samaccountname,useraccountcontrol
```

확인할 출력:

- 해당 플래그가 설정된 `samaccountname`과 `useraccountcontrol`을 확인한다.
- 결과가 없으면 필터 지원 여부와 원시 UAC 값을 확인한 뒤 해당 설정이 없다고 판정한다.

외부 모듈을 사용할 수 없고 `dsquery`가 설치된 제한 셸에서는 AD의 bitwise AND matching rule로 같은 플래그 후보를 조회한다.

```cmd
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName sAMAccountName userAccountControl
```

`1.2.840.113556.1.4.803`은 filter 값에 설정된 모든 bit가 속성에도 설정됐는지 검사한다. `32`가 일치한다는 결과는 `PASSWD_NOTREQD` bit만 확인하며 빈 비밀번호나 인증 성공을 뜻하지 않는다. `dsquery` 부재·DC 연결 실패·접근 거부와 일치 객체 없음은 각각 구분한다.

### 3. 가역 암호화 저장 허용 계정 확인

```powershell
Get-DomainUser -Identity * |
  Where-Object { $_.useraccountcontrol -like '*ENCRYPTED_TEXT_PWD_ALLOWED*' } |
  Select-Object samaccountname,useraccountcontrol
```

ActiveDirectory module을 사용할 수 있다면 다음 결과와 교차 검증한다.

```powershell
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
```

확인할 출력:

- PowerView와 ActiveDirectory module이 같은 계정의 가역 암호화 허용 플래그를 반환하는지 확인한다.
- module 오류는 RSAT·module 가용성과 도메인 연결을 확인한다. 플래그 조회 실패는 현재 비밀번호 저장 형식을 증명하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| Description에 비밀번호 형식 값 | 평문 credential 노출 가능 | credential 후보 | 잠금 정책을 확인하고 [[원격 비밀번호 공격]]으로 최소 검증 |
| `PASSWD_NOTREQD` | 비밀번호 정책 예외 가능 | 약한 계정 설정 후보 | 빈 비밀번호로 단정하지 말고 인증 검증 여부 결정 |
| `ENCRYPTED_TEXT_PWD_ALLOWED` | 비밀번호가 가역 암호화로 저장될 수 있음 | DCSync·NTDS dump 영향 확대 후보 | 설정 시점 이후 비밀번호 변경 여부와 [[DCSync]] 범위 확인 |
| 계정이 disabled | 현재 로그인 가능성 낮음 | 역사적 단서 | 비밀번호 재사용과 다른 계정 사용 여부만 별도 검토 |
| 후보 credential 인증 성공 | 현재 유효한 계정 확인 | 도메인 credential 확보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |

## 확인할 출력과 권한

- `PASSWD_NOTREQD`는 빈 비밀번호의 증거가 아니다.
- 가역 암호화 허용 플래그는 현재 평문을 읽을 수 있다는 뜻이 아니며, 고권한 DCSync·NTDS 추출에서 영향이 나타난다.
- Description의 문자열은 오래된 값일 수 있으므로 실제 인증 성공 전에는 credential 후보로만 취급한다.
- 열거 결과는 속성 값과 설정 플래그가 현재 읽힌다는 사실만 확정한다. 계정 활성 상태, 비밀번호 유효성, 로그인 허용 서비스와 실제 권한은 각각 별도로 검증한다.

## 관련 공격기법

- [[AD 사용자 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[원격 비밀번호 공격]]
- [[DCSync]]

## 관련 도구

- [[PowerView]]
- [[ActiveDirectory PowerShell 모듈]]

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 참고 링크

- [Microsoft LDAP matching rules](https://learn.microsoft.com/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5)
- [Microsoft Directory Service command-line tools](https://learn.microsoft.com/troubleshoot/windows-server/active-directory/directory-service-manage-objects)
