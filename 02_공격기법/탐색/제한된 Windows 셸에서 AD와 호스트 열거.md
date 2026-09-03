---
tags:
  - 환경/windows
  - 환경/ad
시작조건: ["대상 Windows 호스트의 셸 또는 세션 확보", "외부 도구 반입 제한", "현재 실행 계정 확인 가능"]
필요권한: ["현재 Windows 프로세스 토큰으로 로컬·도메인 정보를 조회할 권한"]
필요조건: ["대상 호스트에서 Windows 기본 명령 사용 가능", "도메인 가입·DNS suffix·로그온 서버 중 AD 연결 단서"]
결과: ["현재 호스트와 계정", "도메인·DC 단서", "네트워크 경로 후보", "사용자·컴퓨터 객체 열거 경로"]
문서역할: 수동절차
---

# 제한된 Windows 셸에서 AD와 호스트 열거

## 한 줄 판단

외부 도구를 반입할 수 없는 Windows 셸이 있으면 기본 명령으로 현재 호스트·프로세스 계정·도메인 연결·네트워크 경로를 구분하고, 확인된 상태를 도메인 컨텍스트와 사용자·컴퓨터 객체 열거 문서로 넘긴다.

## 사용할 때

- 대상 Windows 호스트에서 명령은 실행할 수 있지만 PowerView·ADRecon 같은 도구를 반입하기 어려울 때.
- 도메인 가입 호스트라는 사실과 현재 프로세스가 사용하는 로컬·AD 계정을 구분해야 할 때.
- 로컬 route·ARP 단서와 실제 DC·내부 서비스 도달성을 분리해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 열거 대상 Windows 호스트의 셸 | `hostname`, `whoami` | 공격 호스트와 대상 호스트를 다시 구분 |
| 현재 계정과 토큰 | 로컬·도메인 계정과 현재 token 확인 가능 | `whoami /all` | 계정명과 호스트 가입 도메인을 별도로 기록 |
| 네트워크 단서 | 인터페이스·route와 DNS suffix 확인 가능 | `ipconfig /all`, `route print` | 로컬 경로와 실제 서비스 연결 실패를 구분 |

## 실행

### 1. 현재 호스트와 실행 계정 확인

대상 Windows 호스트의 현재 셸에서 호스트, 실행 계정, 도메인 연결 단서를 먼저 확인한다.

```cmd
whoami /all
hostname
systeminfo
ipconfig /all
echo %USERDOMAIN%
echo %LOGONSERVER%
```

확인할 출력:

- 현재 프로세스의 계정·그룹·privilege, 대상 hostname과 OS 버전.
- DNS suffix, 로그온 도메인과 로그온 서버. 빈 값이나 로컬 호스트 값이면 도메인 계정으로 단정하지 않는다.

### 2. 세션과 내부망 경로 확인

```cmd
qwinsta
arp -a
route print
netsh advfirewall show allprofiles
```

확인할 출력:

- 현재 열린 사용자 세션과 상태.
- 인터페이스별 ARP 이웃, 직접 연결·정적 route와 방화벽 profile.
- ARP와 route는 로컬 단서다. 원격 호스트 생존이나 서비스 연결 성공은 별도로 확인한다.

### 3. 도메인과 기본 AD 객체 확인

```cmd
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
net accounts /domain
net user /domain
net group /domain
net group "Domain Admins" /domain
net group "Domain Controllers" /domain
```

확인할 출력:

- `wmic ntdomain`이 반환하는 도메인·forest 이름과 Domain Controller 주소.
- 도메인 비밀번호·잠금 정책, 사용자·그룹과 Domain Controller 계정.
- `wmic` 명령 부재, 이름 해석·DC 연결 실패와 현재 계정의 조회 거부를 서로 다른 실패로 구분한다.

### 4. `dsquery`로 사용자와 컴퓨터 객체 확인

```cmd
dsquery user
dsquery computer
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -attr sAMAccountName
```

확인할 출력:

- 사용자·컴퓨터 DN과 Domain Controller 계정.
- `dsquery`가 없거나 접근이 거부되면 객체가 없다고 판단하지 않고 [[AD 사용자 객체 열거]] 또는 [[AD 컴퓨터 객체 열거]]의 다른 실행 경로를 사용한다.

도메인·DC와 현재 계정을 확인한 뒤 필요한 결과별 문서로 이동한다.

| 확인할 상태 | 실행 문서 | 성공 결과 |
|---|---|---|
| 현재 계정·도메인·DC·인증 방식 | [[AD 도메인 컨텍스트 기본 확인]] | 사용할 AD 계정과 조회 대상 도메인 확정 |
| 사용자명·계정 속성 | [[AD 사용자 객체 열거]] | 사용자 객체와 후속 그룹·SPN 열거 입력 |
| 컴퓨터명·FQDN·운영체제 단서 | [[AD 컴퓨터 객체 열거]] | DNS·서비스 확인 대상 호스트 목록 |
| 별도 인터페이스·route·내부 주소 | [[피벗팅 경로 식별과 내부망 열거]] | 현재 호스트에서 도달 가능한 내부망 후보 |

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 도메인 사용자와 로그온 DC가 확인됨 | 현재 프로세스가 AD 계정을 사용함 | AD 계정과 DC 단서 | [[AD 도메인 컨텍스트 기본 확인]]에서 LDAP·Kerberos 경로 확인 |
| 로컬 계정이지만 도메인 DNS suffix가 있음 | 호스트 가입 상태와 현재 계정이 다름 | 도메인 가입 호스트의 로컬 사용자 셸 | 별도 AD 자격 증명 유무 확인 |
| 별도 route 또는 ARP 이웃 발견 | 현재 호스트의 경로·최근 이웃 단서 | 내부망 경로 후보 | [[피벗팅 경로 식별과 내부망 열거]]에서 실제 포트 도달성 확인 |
| 비밀번호·잠금 정책이 반환됨 | Password Spraying 시도 간격과 잠금 위험을 계산할 수 있음 | 도메인 비밀번호 정책 | [[AD 비밀번호 정책 열거 및 조회]]에서 정책 값 해석 |
| 사용자·컴퓨터 DN이 반환됨 | 현재 세션에서 해당 디렉터리 객체 범위를 읽음 | AD 사용자·컴퓨터 후보 | [[AD 사용자 객체 열거]]와 [[AD 컴퓨터 객체 열거]]에서 속성과 현재성 확인 |
| `qwinsta`에 다른 사용자의 활성 세션이 표시됨 | 현재 호스트에 다른 사용자가 로그인함 | 로그온 사용자 단서 | [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]]에서 세션과 권한을 구분 |
| 기본 명령이 없거나 접근 거부 | 해당 조회 경로를 사용할 수 없음 | 부분 열거 상태 | 명령 부재·권한 거부·이름 해석·DC 미도달을 분리 |

## 확인할 출력과 권한

- `whoami /all`은 현재 프로세스 토큰을, DNS suffix와 `%LOGONSERVER%`는 호스트·로그온 구성을 보여 준다.
- `route print`와 `arp -a`는 내부 서비스 도달 성공이나 피벗 완료를 뜻하지 않는다.
- 이 문서의 기본 명령은 제한 셸에서 즉시 사용할 기준선이다. 더 많은 객체 속성과 Linux 원격 열거는 각 원자 기법 문서에서 수행한다.

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[내부망 경로 확보 후 피벗 구성]]
