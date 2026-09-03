---
tags:
  - 환경/windows
  - 환경/ad
  - 기능/열거
시작상태:
  - Windows 호스트에서 명령 실행 세션 확보
목표:
  - 현재 계정과 실제 권한 확인
  - 도메인 연결과 DC 접근 경로 확인
  - 자격 증명 또는 내부망 단서 확보
필요권한:
  - Windows 사용자 명령 실행 세션
필요정보:
  - 접근한 Windows 호스트
네트워크위치:
  - 접근한 Windows 호스트 내부
---

# Windows 서버 접근 성공 후 초기 열거

## 시나리오 개요

RDP, WinRM, Web Shell 또는 Meterpreter로 Windows 호스트에서 명령을 실행할 수 있을 때 현재 계정·권한·도메인 연결·네트워크 경로·자격 증명 단서를 순서대로 확인하고, 확인한 상태에 맞는 후속 공격으로 이어간다.

## 기준 구조

```text
공격 호스트
  -> Windows 대상 호스트의 명령 실행 세션
       -> 로컬 권한 상승
       -> 자격 증명 수집
       -> DC와 내부 호스트 열거
       -> 원격 접근 또는 피벗
```

## 시작 상태

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | Windows 대상 호스트 | `hostname`, `whoami` | 세션 유형과 대상 호스트를 다시 확인 |
| 현재 계정 | 로컬 또는 도메인 사용자 | `whoami /all` | 토큰과 세션 계정을 먼저 식별 |
| 현재 가능한 행위 | CMD 또는 PowerShell 명령 실행 | `echo %COMSPEC%`, `$PSVersionTable` | 사용할 수 있는 셸 문법으로 변경 |
| 네트워크 경로 | 대상 호스트에서 로컬·내부 주소 확인 가능 | `ipconfig /all`, `route print -4` | 인터페이스와 라우팅 테이블부터 확인 |

## 공격 경로 요약

| 단계 | 실행 위치 | 수행할 행동 | 확인할 출력·상태 | 다음 단계 |
|---|---|---|---|---|
| 1 | Windows 대상 호스트 | 현재 계정과 토큰 확인 | 계정 유형, 그룹, 특권 | 로컬 또는 도메인 분기 |
| 2 | Windows 대상 호스트 | 호스트와 네트워크 열거 | 인터페이스, route, 연결 중인 원격 주소 | 내부망·DC 접근 확인 |
| 3 | Windows 대상 호스트 | 도메인 조인과 DC 확인 | 도메인명, 로그온 서버, DC 주소 | AD 객체 열거 |
| 4 | Windows 대상 호스트 | 저장 자격 증명과 파일 단서 확인 | `cmdkey`, ticket, history, 구성 파일 | 자격 증명 검증 |
| 5 | Linux 공격 호스트 | 확보한 자격 증명과 원격 서비스 검증 | 인증 성공, 관리자 표시, 셸 가능 서비스 | 원격 접근 또는 고권한 후속 행동 |

## 1. 현재 계정과 권한 확인

Windows 대상 호스트에서 현재 토큰의 계정, 그룹과 특권을 확인한다.

```cmd
hostname
whoami
whoami /all
whoami /groups
whoami /priv
net user %USERNAME%
net localgroup administrators
```

PowerShell을 사용할 수 있으면 현재 토큰의 SID도 함께 확인한다.

```powershell
$me = [Security.Principal.WindowsIdentity]::GetCurrent()
$me.Name
$me.Groups | Select-Object Value
```

| 관찰 | 판단 | 다음 행동 |
|---|---|---|
| `HOSTNAME\user` | 로컬 계정으로 실행 중 | [[Windows 권한 상승 열거]]와 로컬 자격 증명 검색 |
| `DOMAIN\user` | 도메인 계정 토큰으로 실행 중 | 도메인명과 DC 연결 확인 |
| 로컬 Administrators SID 포함 | 이 호스트의 관리자 가능성 | [[Windows SAM 로컬 계정 해시 추출]], [[Windows LSA Secrets 추출]] |
| `SeImpersonatePrivilege` 등 위험 특권 활성화 가능 | 토큰 기반 권한 상승 후보 | 현재 OS·서비스 조건과 맞는 권한 상승 기법 선택 |
| Domain Admins 등 도메인 고권한 그룹 포함 | 도메인 권한 후보 | 실제 DC 접근과 DCSync·원격 관리 권한을 별도 검증 |

로컬 관리자 그룹과 도메인 전체 권한은 같은 상태가 아니다. 그룹 이름만 보고 결론 내리지 않고 해당 호스트와 DC에서 가능한 실제 작업을 확인한다.

## 2. 호스트와 네트워크 위치 확인

Windows 대상 호스트에서 인터페이스, route, DNS, ARP와 기존 연결을 확인한다.

```cmd
ipconfig /all
route print -4
arp -a
netstat -ano
```

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias,IPAddress,PrefixLength
Get-NetRoute -AddressFamily IPv4 | Sort-Object DestinationPrefix
Get-NetTCPConnection | Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,State
```

새 내부 대역이나 현재 공격 호스트에서 보이지 않던 원격 주소가 나오면 [[피벗팅 경로 식별과 내부망 열거]]로 접근 경로를 확인한다. 실제 새 대역이 확보되면 [[내부망 경로 확보 후 피벗 구성]]에서 사용할 피벗 방법을 선택한다.

## 3. 도메인 조인과 DC 연결 확인

Windows 대상 호스트에서 장비의 도메인 조인 상태와 현재 세션의 도메인 정보를 구분한다.

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain
$env:USERDOMAIN
$env:USERDNSDOMAIN
$env:LOGONSERVER
```

도메인명이 확인되면 DC를 찾는다.

```cmd
nltest /dsgetdc:<DOMAIN>
nltest /dclist:<DOMAIN>
nslookup -type=SRV _ldap._tcp.dc._msdcs.<DOMAIN>
```

제한된 CMD에서도 도메인 객체와 DC 단서를 확인할 수 있다.

```cmd
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
net view /domain
net user /domain
net group "Domain Admins" /domain
setspn -Q */*
```

DC 후보가 확인되면 Windows 대상 호스트에서 필요한 포트에 실제로 연결되는지 본다.

```powershell
Test-NetConnection <DC_IP> -Port 53
Test-NetConnection <DC_IP> -Port 88
Test-NetConnection <DC_IP> -Port 389
Test-NetConnection <DC_IP> -Port 445
```

| 출력·상황 | 판단 | 다음 행동 |
|---|---|---|
| `PartOfDomain : True` | 호스트가 도메인에 조인됨 | 현재 토큰이 로컬 계정이어도 도메인명과 DC 확인 가능 |
| `$env:USERDNSDOMAIN`이 비어 있음 | 로컬 계정 세션일 가능성 | 장비의 `Domain` 값과 저장 자격 증명 확인 |
| `$env:LOGONSERVER`가 `\\<DC_HOST>` | 로그온을 처리한 DC 단서 | 이름 해석과 88·389·445 연결 확인 |
| `nltest /dsgetdc` 성공 | DC 이름·주소·사이트 확인 | [[AD 도메인 컨텍스트 기본 확인]]과 AD 객체 열거 |
| 이름은 확인되지만 TCP 연결 실패 | 라우팅·방화벽·피벗 문제 | route와 실제 연결 가능 위치 재확인 |

## 4. 저장 자격 증명과 파일 단서 확인

현재 사용자 범위에서 저장된 자격 증명, Kerberos ticket, PowerShell history와 구성 파일을 확인한다.

```cmd
cmdkey /list
klist
dir C:\Users
dir C:\Users\Public
dir C:\ProgramData
```

```powershell
Get-ChildItem -Path C:\Users -Recurse -Force -Include ConsoleHost_history.txt -ErrorAction SilentlyContinue
Get-ChildItem C:\Users -Recurse -Force -ErrorAction SilentlyContinue -Include *.txt,*.xml,*.config,*.ini,*.ps1,*.bat,*.cmd |
  Select-String -Pattern 'password|passwd|pwd|credential|secret|token|key' -ErrorAction SilentlyContinue
```

| 단서 | 다음 공격기법 또는 상태 라우터 |
|---|---|
| `cmdkey /list`에 저장 대상 존재 | [[Windows 저장 자격증명 수집]] |
| Kerberos ticket 존재 | ticket의 사용자·서비스·만료를 확인한 뒤 [[Pass the Ticket]] |
| history·구성 파일에 비밀번호·token·key 후보 | [[Windows 파일 자격증명 검색]] |
| 관리자 권한과 SAM·SYSTEM 접근 | [[Windows SAM 로컬 계정 해시 추출]] |
| 관리자 권한과 LSA secret 접근 | [[Windows LSA Secrets 추출]] |
| 계정명이 연결된 자격 증명 확보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |

## 5. DC 객체와 원격 접근 경로 확인

도메인 계정 토큰과 DC 연결이 있으면 Windows 대상 호스트에서 AD 객체를 열거한다.

```cmd
net user /domain
net group "Domain Users" /domain
net group "Domain Admins" /domain
setspn -Q */*
```

ActiveDirectory 모듈이 있으면 SPN 계정을 명시적으로 확인한다.

```powershell
Get-ADDomain
Get-ADUser -LDAPFilter '(servicePrincipalName=*)' -Properties ServicePrincipalName |
  Select-Object SamAccountName,ServicePrincipalName
Get-ADGroupMember 'Domain Admins'
```

Linux 공격 호스트에서 DC까지 연결되는 경로가 마련되면 현재 확보한 자격 증명의 인증 성공과 원격 권한을 분리해서 확인한다.

```bash
nxc smb <TARGETS> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --continue-on-success
nxc winrm <TARGETS> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --continue-on-success
nxc rdp <TARGETS> -d <DOMAIN> -u <USER> -p '<PASSWORD>'
```

| 출력 | 의미 | 다음 행동 |
|---|---|---|
| SMB `[+]` | SMB 인증 성공 | 공유·세션·원격 권한을 별도 열거 |
| SMB `Pwn3d!` | 대상 로컬 관리자 가능성 | 원격 실행과 원격 자격 증명 덤프 검토 |
| WinRM `[+]` | 원격 PowerShell 사용 가능성 | [[WinRM 원격 PowerShell 세션]] |
| RDP 인증 성공 | RDP 정책과 로그온 권한 후보 | [[RDP 로그인과 GUI 세션]] |
| 인증 실패 | 입력·도메인·이름 해석·시간·대상 서비스 순서로 확인 | 자격 증명 값을 바꾸기 전에 실패 단계를 식별 |

## 실패 시 분기

| 멈춘 단계·출력 | 가능한 원인 | 확인 명령 | 이어갈 단계 |
|---|---|---|---|
| `whoami /all` 정보 부족 | 제한된 셸 또는 명령 필터링 | `whoami`, `set`, `wmic ntdomain ...` | 사용할 수 있는 기본 명령으로 축소 |
| `nltest`가 도메인을 찾지 못함 | DNS, route, 로컬 계정 세션 문제 | `ipconfig /all`, `nslookup`, `route print -4` | DC 이름·주소를 수동 확인 |
| Windows에서는 DC 접근 성공, 공격 호스트에서는 실패 | 피벗 경로 없음 | 양쪽에서 88·389·445 연결 비교 | [[내부망 경로 확보 후 피벗 구성]] |
| `cmdkey` 항목은 있지만 실행 실패 | 저장 대상·사용자·실행 방식 불일치 | 항목의 Target과 User 확인 | [[저장된 자격 증명으로 runas 프로세스 실행]] |
| 원격 인증 성공 후 명령 실행 실패 | 서비스 로그온 권한 또는 관리자 권한 부족 | SMB·WinRM·RDP 결과를 각각 확인 | 사용할 서비스와 권한을 다시 선택 |

## 완료 기준

- 현재 세션의 로컬·도메인 계정과 실제 그룹·특권이 확인됨.
- 호스트의 인터페이스·route와 DC 또는 내부망 접근 가능 위치가 확인됨.
- 도메인 조인·DC 연결 여부와 사용할 수 있는 AD 열거 범위가 확인됨.
- 자격 증명, 고권한 또는 새 내부망 경로를 얻었다면 해당 [[확보한 자격 증명으로 원격 접근 경로 선택]], [[고권한 세션 확보 후 후속 판단]], [[내부망 경로 확보 후 피벗 구성]]으로 상태를 재평가함.

## 관련 노트

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[Windows 권한 상승 열거]]
- [[제한된 Windows 셸에서 AD와 호스트 열거]]
- [[AD 도메인 컨텍스트 기본 확인]]
- [[Windows 파일 자격증명 검색]]
- [[Windows 저장 자격증명 수집]]
- [[피벗팅 경로 식별과 내부망 열거]]
- [[내부망 경로 확보 후 피벗 구성]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
