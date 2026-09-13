---
tags:
  - 환경/ad
시작조건: ["Windows·Linux 세션, AD 계정·ticket 또는 AD 서비스 단서 중 하나 확보"]
필요권한: ["현재 운영체제 계정으로 로컬 컨텍스트 조회", "원격 확인 시 RootDSE 또는 해당 인증 주체에 허용된 디렉터리 조회"]
필요조건: ["로컬 Windows·Linux 명령 실행 또는 DC LDAP 접근", "도메인명·DC·DNS 중 하나 이상의 단서"]
결과: ["도메인명과 DC", "현재 AD 계정", "그룹·객체 권한 단서", "AD 서비스 도달성"]
문서역할: 수동절차
---

# AD 도메인 컨텍스트 기본 확인

## 한 줄 판단

Windows·Linux 세션, AD 계정·Kerberos ticket 또는 AD 서비스 단서 중 하나가 있으면, 명령을 실행하는 호스트의 도메인 연결과 현재 운영체제 계정, ticket에 기록된 AD principal, DC 이름과 서비스 도달성을 따로 확인하여 실제 조건이 충족된 AD 기법을 선택한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 로컬 셸 또는 DC LDAP에 닿는 원격 실행 호스트 | 셸 종류, 인터페이스·route, DC 이름 해석과 LDAP 응답 확인 | 로컬 확인만 가능한지 원격 확인도 가능한지 나누고 피벗·DNS 경로 확인 |
| 현재 계정 또는 인증 수단 | 현재 OS 계정, Kerberos ticket 주체 또는 LDAP 인증 수단 중 확인할 대상 | `whoami`·`id`, `klist`, LDAP bind 결과를 각각 기록 | 세 주체를 같은 계정으로 가정하지 말고 사용할 인증 수단부터 식별 |
| 현재 권한 | 로컬 컨텍스트 조회 또는 RootDSE base query에 필요한 읽기 권한 | 각 명령이 access denied 없이 필요한 필드를 반환하는지 확인 | 로컬 파일·명령 권한과 LDAP 조회 권한을 별도로 확인 |
| 공격 대상의 조건 | AD에 연결됐거나 AD DS 서비스를 제공하는 도메인·DC 후보 | host 설정, DNS SRV, LDAP RootDSE를 교차 확인 | 단일 호스트명이나 계정 접두사만으로 도메인 연결을 단정하지 않음 |
| 필요한 파일·목록·주소 | 도메인명·realm·DC·DNS 중 하나 이상의 단서 | 계정 형식, ticket, `/etc` 설정 또는 서비스 응답에서 확보 | [[무인증 내부 네트워크에서 AD 단서 확인]]에서 서비스·이름 단서 보강 |

## 실행

### 현재 세션의 AD·Kerberos 컨텍스트 확인

`<DOMAIN>`은 DNS 도메인(예: `corp.example`)이며, `nltest`와 SRV 조회는 현재 명령을 실행하는 호스트에서 해당 도메인의 DC를 찾는다. 현재 OS 계정, `klist` principal, 후속 LDAP bind 주체는 서로 다른 값일 수 있다.

#### Windows 실행 환경

현재 Windows 호스트의 로그온 주체, 도메인 연결과 Kerberos ticket을 서로 분리해 확인한다.

```cmd
whoami /all
echo %USERDOMAIN%
echo %USERDNSDOMAIN%
nltest /dsgetdc:<DOMAIN>
klist
nslookup -type=SRV _ldap._tcp.dc._msdcs.<DOMAIN>
```

확인할 출력:

- 현재 로그온 계정이 로컬 계정인지 도메인 계정인지 구분한다.
- `nltest`와 DNS SRV 결과에서 DC FQDN, 도메인명과 사이트를 확인한다.
- `klist`에서 티켓에 표시된 클라이언트 계정, realm, ticket 만료 시간과 서비스 이름을 확인한다.
- `nltest`가 DC를 찾지 못하면 도메인명·DNS와 DC 도달성을 확인하고, `klist`가 비어 있으면 현재 로그온 계정에 사용할 ticket이 없다고만 판정한다.

#### Linux 실행 환경

현재 Linux 계정, 호스트의 AD 통합 설정과 Kerberos credential cache의 주체를 따로 확인한다.

```bash
id
realm list
klist
getent passwd <USER>
```

확인할 출력:

- SSSD·Winbind·realmd 등 도메인 통합 설정과 domain name.
- 현재 Linux 계정과 티켓에 표시된 계정이 같은 Identity인지 여부.
- domain 사용자·그룹이 NSS를 통해 조회되는지 여부.
- `realm list` 또는 `getent`가 비어 있어도 별도 Kerberos ticket이나 원격 LDAP 접근까지 부정하지 않는다. 명령 부재, 미가입 호스트와 NSS 통합 실패를 구분한다.

### 원격 DC의 LDAP RootDSE 확인

실행 호스트에서 DC LDAP 서비스가 응답하고 어떤 AD naming context를 제공하는지 확인한다.

```bash
ldapsearch -x -H ldap://<DC> -s base -b "" defaultNamingContext dnsHostName supportedCapabilities
```

확인할 출력:

- `defaultNamingContext`에서 도메인 DN을 확인한다.
- `dnsHostName`과 `supportedCapabilities`로 DC FQDN과 AD DS 구현 단서를 확인한다.
- 연결 실패는 DNS·포트·TLS·피벗 경로를 확인한다. RootDSE 응답이 비어 있거나 거부되면 익명 base query 정책을 확인하되 인증된 객체 읽기 가능 여부와 혼동하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 도메인명·DC·DNS와 현재 사용 중인 계정이 확인됨 | AD Identity 컨텍스트가 확정됨 | AD 계정과 사용 가능한 서비스 | [[AD Identity 확인 후 도메인 컨텍스트 열거]]에서 실제 권한과 기법 조건 재평가 |
| 호스트는 도메인에 연결됐지만 현재 로그온은 로컬 계정임 | 호스트의 AD 연결 상태와 현재 로그온 계정이 다름 | AD 연결 Windows·Linux 호스트의 로컬 사용자 세션 | 현재 플랫폼 상태 라우터를 유지하고 별도 AD 계정의 비밀번호·NT hash·Kerberos ticket 유무 확인 |
| 유효 ticket과 DC 서비스 도달성이 확인됨 | Kerberos 인증을 검증할 수 있음 | ticket 주체의 AD 접근 후보 | [[Pass the Ticket]]에서 실제 서비스 접근 확인 |
| 그룹 멤버십·ACL·복제 권한 단서만 확인됨 | 고권한 후보이며 실제 권한은 미확정 | AD 객체 권한 후보 | 대상 객체와 권한 종류를 확인하고, 복제 권한이 실제로 확인된 경우에만 [[DCSync]] |
| DC 이름은 확인됐지만 필요한 서비스가 닿지 않음 | Identity 문제가 아니라 네트워크 경계일 수 있음 | AD 경로 미확보 | [[피벗팅 경로 식별과 내부망 열거]] 후 [[내부망 경로 확보 후 피벗 구성]] |
| RootDSE는 응답하지만 인증된 객체 조회가 거부됨 | AD 서비스만 식별됐고 현재 사용 중인 계정의 권한은 미확정 | LDAP·AD DS 단서 | credential 또는 ticket을 확보한 뒤 인증된 읽기 범위 확인 |

도메인과 현재 사용 중인 계정이 확정되면 사용자는 [[AD 사용자 객체 열거]], 컴퓨터는 [[AD 컴퓨터 객체 열거]], 중첩 그룹은 [[AD 고권한 그룹과 중첩 구성원 열거]], 서비스 계정은 [[SPN 계정 열거]]로 넘긴다. 여러 객체의 ACL·GPO·세션 관계를 함께 볼 때는 [[AD 관계 그래프 수집과 공격 경로 식별]], 개별 객체의 위임 권한은 [[AD ACL 권한 열거와 공격 경로 식별]]에서 직접 확인한다. 제한된 셸에서 AD 전용 모듈을 사용할 수 없으면 [[제한된 Windows 셸에서 AD와 호스트 열거]]를 사용한다.

## 확인할 출력과 권한

- 도메인 조인 호스트, 현재 운영체제 계정과 티켓에 표시된 계정을 같은 Identity로 간주하지 않는다.
- `whoami`·`id`는 현재 OS 계정을, `klist`는 credential cache의 Kerberos 주체를, RootDSE는 접속한 LDAP 서버의 도메인 정보를 각각 확정한다. 어느 하나만으로 다른 두 상태를 확정하지 않는다.
- 그룹 멤버십, 로컬 관리자, 원격 로그인 권한, AD 객체 쓰기와 디렉터리 복제 권한을 각각 구분한다.
- Kerberos 오류는 credential뿐 아니라 DNS, FQDN, realm, 시간과 DC 도달성을 함께 확인한다.

## 관련 서비스

- [[DNS 서비스]]
- [[Kerberos 서비스]]
- [[LDAP 서비스]]
- [[SMB 서비스]]

## 관련 도구

- [[klist]]
- [[ldapsearch]]
- [[ActiveDirectory PowerShell 모듈]]

## 관련 공격기법

- [[AD 사용자 객체 열거]]
- [[AD 컴퓨터 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[SPN 계정 열거]]
- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[제한된 Windows 셸에서 AD와 호스트 열거]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[무인증 내부 네트워크에서 AD 단서 확인]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- [[내부망 경로 확보 후 피벗 구성]]
