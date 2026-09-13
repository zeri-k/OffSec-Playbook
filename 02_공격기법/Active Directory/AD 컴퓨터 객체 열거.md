---
tags:
  - 환경/ad
  - 환경/windows
  - 서비스/ldap
시작조건: ["도메인과 DC 식별", "현재 사용할 AD 계정 또는 도메인 사용자 세션 확보"]
필요권한: ["현재 AD 계정으로 컴퓨터 객체를 읽을 권한"]
필요조건: ["Windows에서 AD 모듈·PowerView·기본 명령 중 하나 또는 Linux에서 인증 가능한 LDAP 열거 도구", "DC LDAP 접근"]
결과: ["AD 컴퓨터 객체·FQDN·운영체제 후보", "DNS·서비스·세션 열거 입력"]
---

# AD 컴퓨터 객체 열거

## 한 줄 판단

도메인·DC와 사용할 AD 계정을 확인했으면 Windows 도메인 세션 또는 Linux 공격 호스트에서 컴퓨터 객체를 조회하여 hostname·FQDN·DN과 운영체제 단서를 수집하고, DNS와 서비스 응답으로 현재 활성 호스트인지 다시 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 도메인 컨텍스트 | 도메인·DC·현재 계정 확인 | [[AD 도메인 컨텍스트 기본 확인]] | DNS·Base DN·계정 형식을 먼저 수정 |
| Linux 인증 경로 | DC LDAP 389·636 접근과 현재 계정 비밀번호 | LDAP bind와 객체 반환 확인 | 연결·인증·검색 오류를 분리 |
| Windows 실행 경로 | AD 모듈, PowerView 또는 `dsquery` 사용 가능 | 모듈·명령 존재 확인 | [[제한된 Windows 셸에서 AD와 호스트 열거]]에서 사용할 경로 선택 |

## 실행

### Linux 공격 호스트에서 실행

`<DC_IP>`는 LDAP를 제공하는 DC IP, `<USER>@<DOMAIN>`과 `<PASSWORD>`는 LDAP bind 요청자다. `<DC_FQDN>`은 Windows AD module이 조회할 DC FQDN이며, `<OU_DN>`과 `<MAX_OBJECTS>`는 선택적인 검색 범위와 양의 정수 제한이다.

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -C
```

확인할 출력:

- 컴퓨터 이름, DN과 DNS hostname.
- LDAP 인증 성공 뒤 실제 컴퓨터 객체가 반환됐는지 확인한다.

### Windows 공격 호스트에서 실행

Microsoft ActiveDirectory 모듈이 있으면 현재성을 판단할 속성을 명시해 가져온다.

```powershell
Get-Module -ListAvailable ActiveDirectory
Import-Module ActiveDirectory
Get-ADComputer -Filter * -Server '<DC_FQDN>' -Properties DNSHostName,OperatingSystem,OperatingSystemVersion,LastLogonDate |
  Select-Object Name,DNSHostName,DistinguishedName,Enabled,OperatingSystem,OperatingSystemVersion,LastLogonDate
```

`-Filter *`는 디렉터리의 컴퓨터 객체를 넓게 반환한다. 큰 환경에서는 `-SearchBase '<OU_DN>'` 또는 `-ResultSetSize <MAX_OBJECTS>`로 먼저 줄이고, 결과의 `LastLogonDate`·운영체제 문자열만으로 host가 현재 활성이라고 단정하지 않는다.

확인할 출력:

- 컴퓨터 `Name`·`DNSHostName`·DN·활성 속성과 운영체제·마지막 로그온 단서.
- import 성공과 객체 반환을 분리한다. 빈 결과나 오류가 나오면 DC·search base·읽기 권한을 확인한다.

PowerView를 사용할 때는 다음 경로를 사용한다.

```powershell
Import-Module .\PowerView.ps1
Get-DomainComputer
```

기본 명령만 사용할 수 있으면 DN 목록을 확인한다.

```cmd
dsquery computer
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 컴퓨터 객체와 hostname이 반환됨 | 디렉터리에 등록된 호스트 후보 | AD 컴퓨터 목록 | [[AD DNS 레코드 열거]]와 서비스 응답으로 현재성 확인 |
| 오래된 로그온 시각·지원 종료 OS 단서 | 비활성 또는 오래된 레코드 가능 | 현재성 미확정 컴퓨터 | DNS·ICMP·서비스 응답을 교차 확인 |
| 현재성이 확인된 Windows 호스트 | 세션·원격 권한 열거 대상 | 원격 Windows 호스트 목록 | [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]], [[AD 원격 접근 권한 열거]] |
| 인증 성공 후 결과가 비거나 접근 거부 | 검색 범위·권한·도구 필터 문제 가능 | 컴퓨터 범위 미확정 | Base DN, domain, 필터와 현재 계정의 읽기 범위 확인 |

## 확인할 출력과 권한

- 디렉터리의 컴퓨터 객체는 현재 활성 호스트나 특정 서비스의 도달 가능성을 뜻하지 않는다.
- DNS 응답, 포트 도달성, 서비스 인증과 원격 명령 권한을 별도로 확인한다.

## 관련 도구

- [[windapsearch]]
- [[PowerView]]
- [[ActiveDirectory PowerShell 모듈]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 참고 링크

- [Microsoft Get-ADComputer](https://learn.microsoft.com/powershell/module/activedirectory/get-adcomputer)
