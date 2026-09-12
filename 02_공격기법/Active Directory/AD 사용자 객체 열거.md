---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["도메인과 DC 식별", "현재 사용할 AD 계정 또는 도메인 사용자 세션 확보"]
필요권한: ["현재 AD 계정으로 사용자 객체를 읽을 권한"]
필요조건: ["Windows에서 AD 모듈·PowerView·기본 명령 중 하나 또는 Linux에서 인증 가능한 SMB·LDAP 열거 도구", "DC LDAP 또는 SMB 접근"]
결과: ["AD 사용자 객체와 계정 속성", "후속 그룹·SPN·계정 위험 속성 열거 입력"]
---

# AD 사용자 객체 열거

## 한 줄 판단

도메인·DC와 사용할 AD 계정을 확인했으면 Windows 도메인 세션 또는 Linux 공격 호스트에서 사용자 객체를 조회하여 계정명·DN·활성 상태와 주요 속성을 수집하고 그룹·SPN·계정 위험 속성 열거의 입력으로 사용한다.

## 사용할 때

- 도메인 사용자 목록과 개별 계정 속성이 필요한 경우.
- 그룹 구성원, SPN 서비스 계정 또는 비활성 계정 후보를 조사하기 전에 입력 목록을 만들 때.
- 인증 성공과 실제 사용자 객체 반환 범위를 구분해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 도메인 컨텍스트 | 도메인·DC·현재 계정 확인 | [[AD 도메인 컨텍스트 기본 확인]] | DNS·Base DN·계정 형식을 먼저 수정 |
| Linux 인증 경로 | DC SMB 445 또는 LDAP 389·636 접근과 현재 계정 비밀번호 | bind·SMB 인증 응답 확인 | 연결·인증·객체 조회 오류를 분리 |
| Windows 실행 경로 | AD 모듈, PowerView 또는 `dsquery` 사용 가능 | 모듈·명령 존재 확인 | [[제한된 Windows 셸에서 AD와 호스트 열거]]에서 사용할 경로 선택 |

## 실행

### Linux 공격 호스트에서 실행

```bash
crackmapexec smb <DC> -u <USER> -p '<PASSWORD>' --users
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -U
```

확인할 출력:

- 계정명, DN과 도구가 반환한 `badpwdcount`, 비활성·잠금 상태 단서.
- SMB 또는 LDAP 인증 성공 뒤 실제 사용자 객체가 반환됐는지 확인한다.

### Windows 공격 호스트에서 실행

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser -Identity <USER> -Domain <DOMAIN> | Select-Object name,samaccountname,memberof,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol
```

AD 모듈이나 PowerView를 사용할 수 없으면 기본 명령이 반환하는 범위만 확인한다.

```cmd
net user /domain
dsquery user
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 사용자 객체와 속성이 반환됨 | 현재 계정으로 해당 사용자 범위를 읽음 | 사용자 목록과 계정 속성 | [[AD 고권한 그룹과 중첩 구성원 열거]], [[SPN 계정 열거]], [[AD 계정 위험 속성과 Description 열거]] |
| 오래된 로그온 시각 또는 비활성 속성 | 현재 사용 여부를 직접 확인해야 하는 계정 후보 | 현재성 미확정 사용자 | 서비스 인증과 관련 호스트 활동을 별도로 확인 |
| 인증 성공 후 결과가 비거나 접근 거부 | 검색 범위·권한·도구 필터 문제 가능 | 사용자 범위 미확정 | Base DN, domain, 필터와 현재 계정의 읽기 범위 확인 |

## 확인할 출력과 권한

- 사용자 이름과 `admincount`는 비밀번호 유효성이나 현재 고권한 토큰을 뜻하지 않는다.
- 객체 반환 범위와 계정 활성 상태, 실제 서비스 인증 성공을 각각 구분한다.

## 관련 도구

- [[crackmapexec]]
- [[windapsearch]]
- [[PowerView]]
- [[ActiveDirectory PowerShell 모듈]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
