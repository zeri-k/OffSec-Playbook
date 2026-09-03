---
tags:
  - 환경/ad
문서역할: 오케스트레이터
시작조건: ["도메인과 DC 식별", "현재 사용할 AD 계정 또는 도메인 사용자 세션 확보"]
필요권한: ["현재 AD 계정으로 디렉터리 객체를 읽을 권한"]
필요조건: ["DC LDAP 또는 SMB 접근"]
결과: ["사용자 객체 열거 경로", "컴퓨터 객체 열거 경로", "Password Spraying 대상 사용자 목록 후보"]
---

# 인증 후 AD 사용자와 컴퓨터 객체 열거

## 한 줄 판단

도메인·DC와 사용할 AD 계정을 확인했으면 필요한 결과가 계정 속성인지 호스트 목록인지 먼저 구분하고 사용자 객체와 컴퓨터 객체를 각각 독립된 열거 문서에서 확인한다.

## 사용할 때

- 기존 플레이북이나 링크에서 인증 후 AD 객체 열거의 공통 진입점으로 들어왔을 때.
- 사용자와 컴퓨터를 한 번에 조회한 결과를 서로 다른 후속 판단으로 분리해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 미충족 시 다음 확인 |
|---|---|---|
| 현재 AD 계정과 도메인 | 어느 도메인의 어떤 계정으로 조회하는지 확인 | [[AD 도메인 컨텍스트 기본 확인]] |
| 필요한 결과 | 사용자 속성 또는 컴퓨터·FQDN 후보 중 먼저 필요한 결과 선택 | 현재 목표와 다음 기법의 입력을 다시 확인 |

## 실행

### Windows 공격 호스트에서 실행

Active Directory PowerShell 모듈을 사용할 수 있으면 사용자와 컴퓨터 객체를 각각 조회한다.

```powershell
Import-Module ActiveDirectory
Get-ADUser -Filter * -Properties Enabled,LastLogonDate,PasswordLastSet,ServicePrincipalName
Get-ADComputer -Filter * -Properties DNSHostName,OperatingSystem,LastLogonDate
```

Windows 기본 명령만 사용할 수 있는 경우에는 사용 가능한 경로를 선택한다.

```cmd
net user /domain
dsquery user
dsquery computer
```

PowerView를 사용할 수 있으면 사용자와 컴퓨터 객체를 각각 조회한다.

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser
Get-DomainComputer
```

Password Spraying에 사용할 사용자명 후보 파일이 필요하면 `samaccountname`만 추출하고 빈값과 중복을 제거한다.

```powershell
Get-DomainUser -Identity '*' |
    Select-Object -ExpandProperty samaccountname |
    ForEach-Object { $_.Trim() } |
    Where-Object { $_ } |
    Sort-Object -Unique |
    Set-Content -LiteralPath .\adusers.txt -Encoding ascii
```

`adusers.txt`는 디렉터리에 존재하는 사용자명 후보 목록이다. 비활성·잠금 임박 계정과 Fine-Grained Password Policy 적용 대상을 구분하지 않은 상태이므로 바로 인증 시도에 사용하지 않는다.

### Linux 공격 호스트에서 실행

```bash
crackmapexec smb <DC> -u <USER> -p '<PASSWORD>' --users
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -U
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -C
```

확인할 출력:

- 사용자 조회에서는 계정명·DN·활성 상태와 SPN 같은 계정 속성.
- 컴퓨터 조회에서는 컴퓨터명·DN·FQDN·운영체제와 마지막 로그온 단서.
- `adusers.txt`에는 빈 줄과 중복 없이 사용자 `samaccountname`이 한 줄에 하나씩 저장돼야 한다.
- 인증 성공만 표시되고 객체가 반환되지 않으면 Base DN, 대상 도메인, 현재 계정의 읽기 범위와 도구 필터를 먼저 확인한다.
- 상세 속성과 후속 판단은 [[AD 사용자 객체 열거]]와 [[AD 컴퓨터 객체 열거]]에서 이어 간다.

## 관찰과 상태 전환

| 필요한 결과 | 실행 문서 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 사용자명·DN·활성 상태·계정 속성 | [[AD 사용자 객체 열거]] | 사용자 목록과 계정 속성 | 그룹·SPN·계정 위험 속성 열거 |
| 중복과 빈값을 제거한 `samaccountname` 목록 | 이 문서의 PowerView 추출 명령 | Password Spraying 대상 사용자 목록 후보 | [[AD 비밀번호 정책 열거 및 조회]]에서 잠금 정책과 계정별 적용 정책을 확인한 뒤 [[내부 AD Password Spraying]] |
| 컴퓨터명·FQDN·운영체제 단서 | [[AD 컴퓨터 객체 열거]] | AD 컴퓨터 목록 | DNS·서비스·세션·원격 접근 확인 |

## 확인할 출력과 권한

- 사용자 객체 반환과 컴퓨터 객체 반환은 독립된 결과다.
- 컴퓨터 객체는 현재 활성 호스트를, 사용자 객체는 비밀번호 유효성이나 현재 권한을 뜻하지 않는다.
- Password Spraying 전에는 disabled 상태, `badpwdcount`, 도메인 기본 정책과 Fine-Grained Password Policy를 확인하고 단일 비밀번호 후보·최대 시도 횟수·간격을 정한다.

## 관련 공격기법

- [[AD 사용자 객체 열거]]
- [[AD 컴퓨터 객체 열거]]
- [[AD 비밀번호 정책 열거 및 조회]]
- [[내부 AD Password Spraying]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
