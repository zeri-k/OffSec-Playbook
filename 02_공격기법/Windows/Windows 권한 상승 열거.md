---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 사용자 셸 또는 세션 확보", "현재 실행 Identity 확인 가능"]
필요권한: ["현재 Windows 프로세스 토큰으로 허용된 로컬 조회 권한"]
필요조건: ["대상 호스트에서 CMD·PowerShell 또는 RDP·WinRM 명령 실행", "수동 명령 또는 허용된 열거 도구"]
결과: ["현재 토큰의 그룹·privilege와 무결성 단서", "제어 가능한 서비스·작업·파일·설정 후보", "관리자 또는 SYSTEM 권한 상승 후보"]
---

# Windows 권한 상승 열거

## 한 줄 판단

대상 Windows 호스트의 현재 사용자 프로세스에서 로컬 명령을 실행하여 token·그룹·운영체제 정보와 서비스·예약 작업·저장된 비밀번호·key·소프트웨어 단서를 수집하고, 현재 사용자가 실제로 제어할 수 있는 객체만 관리자 또는 SYSTEM 권한 상승 후보로 남긴다.

## 사용할 때

- 현재 보유 정보: WinRM, RDP, Web Shell 또는 Meterpreter로 대상 Windows 호스트의 셸·세션을 확보했고 현재 사용자 이름을 확인할 수 있다.
- 명령 실행 위치와 도달성: 명령은 원격 공격 호스트가 아니라 권한을 평가할 대상 Windows 호스트의 현재 세션 안에서 실행한다.
- 현재 계정과 권한: 현재 프로세스 토큰이 일반 사용자이거나 권한 범위를 아직 모른다. 그룹 구성원 자격과 현재 토큰의 활성 privilege·무결성은 별도로 확인한다.
- 지금 가능한 행동과 결과: 시스템 상태를 조회해 현재 사용자가 수정할 수 있고 고권한 주체가 소비하는 서비스·작업·파일·설정 후보를 찾는다. 단서 발견은 권한 상승 완료가 아니다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 권한 상승 대상을 평가할 Windows 호스트의 셸 또는 세션 | `hostname`과 세션 유형 확인 | 공격 호스트와 대상 호스트를 혼동하지 말고 실행 위치 재확인 |
| 현재 계정 | 현재 프로세스를 실행하는 로컬·도메인 사용자 식별 | `whoami /all`의 사용자 SID와 그룹 확인 | 계정 이름만 보이면 로컬·도메인 범위와 토큰 정보를 추가 확인 |
| 현재 권한 | 현재 토큰의 그룹, 활성 privilege와 무결성 수준을 조회 가능 | `whoami /all` 결과와 접근 거부 여부 확인 | 관리자 그룹 구성원 자격과 현재 프로세스의 elevated 상태를 분리 |
| 공격 대상의 조건 | 운영체제 build·역할과 고권한으로 실행되는 서비스·작업 후보 식별 가능 | `systeminfo`, `hostname`과 후속 수동·자동 열거 결과 확인 | 시스템 정보 조회 실패와 후보 부재를 구분 |
| 실행 가능한 도구 | CMD 기본 명령, PowerShell 또는 허용된 EXE 중 하나 | 명령별 실행 성공과 차단 메시지 확인 | 도구 차단이면 수동 명령으로 전환하고 정책·AV·파일 오류를 구분 |

## 실행

### 기본 확인

대상 Windows 호스트에서 현재 프로세스 Identity와 토큰을 먼저 확인한 뒤, 운영체제 build와 로컬 Administrators 그룹 목록을 기준 정보로 수집한다. 계정·그룹 목록과 현재 process token이 달라질 수 있는 이유는 [[Windows 액세스 토큰과 특권 활성화]]의 로그온 시점 경계를 따른다.

```cmd
whoami /all
hostname
systeminfo
net localgroup administrators
```

확인할 출력:

- `whoami /all`의 사용자 SID, 그룹, 활성·비활성 privilege와 무결성 수준.
- `hostname`, `systeminfo`의 대상 호스트명, 운영체제 build와 역할 단서.
- `net localgroup administrators`의 로컬 Administrators 그룹 구성원. 목록에 이름이 있다는 사실과 현재 프로세스가 elevated 상태라는 사실을 구분한다.

### 운영체제 패치와 설치 소프트웨어 확인

`systeminfo`의 build와 아래 QFE 목록은 후보를 좁히는 입력이다. `Get-HotFix`는 CBS가 제공한 QFE만 반환하므로 목록에 KB가 없다는 이유만으로 해당 패치가 없거나 exploit가 가능하다고 단정하지 않는다.

```powershell
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object HotFixID,Description,InstalledOn
$uninstallKeys = @(
  'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
  'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
)
Get-ItemProperty -Path $uninstallKeys -ErrorAction SilentlyContinue |
  Where-Object DisplayName |
  Select-Object DisplayName,DisplayVersion,Publisher,InstallLocation
```

확인할 출력:

- KB ID·설치 시각과 `systeminfo`의 OS build를 함께 기록한다. 누락 여부는 해당 제품의 공식 보안 공지와 실제 file/package version으로 다시 확인한다.
- 설치 프로그램의 이름·버전·설치 경로는 취약 후보와 서비스·저장 credential 위치를 좁히는 단서다. 설치 항목만으로 해당 프로그램이 실행 중이거나 현재 사용자가 제어할 수 있다고 판단하지 않는다.
- 교육 원천의 `Win32_Product` 전체 조회는 MSI consistency check·repair를 시작할 수 있으므로 대표 열거 명령으로 사용하지 않는다. uninstall registry 목록도 모든 설치 방식을 포괄하지 않으므로 결과 없음은 소프트웨어 부재가 아니다.

### 프로세스·서비스·listener 대응

`tasklist /svc`로 프로세스와 호스팅 service를, `netstat -ano`로 listener·연결과 PID를 먼저 대응시킨다.

```cmd
tasklist /svc
netstat -ano
```

관심 PID는 대상 Windows PowerShell에서 process와 service 정보로 다시 확인한다.

```powershell
Get-Process -Id <LISTENER_PID> | Select-Object Id,ProcessName,Path
Get-CimInstance Win32_Service | Where-Object ProcessId -eq <LISTENER_PID> |
  Select-Object Name,StartName,State,PathName,ProcessId
```

확인할 출력:

- `127.0.0.1`·`::1` listener는 현재 호스트에서만 접근할 로컬 서비스 후보이고, `0.0.0.0`·`::`는 모든 interface에 bind한 후보다. 어느 경우든 방화벽·서비스 인증과 실제 연결 성공은 별도로 확인한다.
- PID가 어떤 process·service와 연결되고 그 service가 어떤 계정과 binary path로 실행되는지 확인한다. listener 발견만으로 취약점이나 고권한 실행을 확정하지 않는다.
- 추가 NIC·route·새 내부 주소가 보이면 로컬 권한 상승과 별개로 [[피벗팅 경로 식별과 내부망 열거]]에서 실제 대상 port 도달성을 확인한다.

### 로컬 사용자·그룹과 세션 정책 확인

```cmd
query user
net user
net localgroup
net accounts
```

확인할 출력:

- 활성·연결 해제 session의 사용자와 ID, 로컬 사용자·그룹, 로컬 비밀번호·잠금 정책.
- 다른 사용자의 session은 credential·token 접근을 보장하지 않는다. 고권한 그룹 구성원 이름, 현재 process token의 그룹 SID와 실제 접근 권한도 분리한다.
- Named Pipe가 보이거나 process 간 통신 객체의 권한을 확인해야 하면 [[Windows Named Pipe 권한 열거]]로 분기한다.

### 고권한 서비스·예약 작업의 제어 지점 확인

전체 자동화 결과를 곧바로 취약점으로 보지 말고, 먼저 고권한으로 실행되는 항목의 실행 경로를 수집한 뒤 관심 경로의 ACL만 현재 SID·그룹과 대조한다.

```powershell
Get-CimInstance Win32_Service | Select-Object Name,StartName,State,PathName
schtasks /query /fo LIST /v
Get-Acl -LiteralPath '<EXECUTABLE_OR_SCRIPT_PATH>' | Format-List Owner,AccessToString
icacls '<EXECUTABLE_OR_SCRIPT_PATH>'
```

확인할 출력:

- 서비스의 `StartName` 또는 예약 작업의 실행 계정이 고권한 주체인지, `PathName`·`Task To Run`이 어느 실행 파일·script·인자를 사용하는지 구분한다.
- `<EXECUTABLE_OR_SCRIPT_PATH>`와 필요한 상위 디렉터리에서 현재 사용자 또는 현재 token의 그룹 SID에 쓰기·수정 권리가 실제로 있는지 확인한다.
- 조회 접근 거부는 해당 객체의 세부 정보가 미확인이라는 뜻이며, 쓰기 가능이나 후보 부재로 해석하지 않는다. `Get-CimInstance`를 사용할 수 없으면 `sc query`와 관심 서비스의 `sc qc <SERVICE_NAME>`으로 범위를 좁힌다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 현재 계정이 Administrators 구성원이지만 토큰이 elevated 상태가 아님 | 관리자 Identity와 현재 프로세스 권한이 다름 | 제한된 관리자 토큰 후보 | 현재 세션의 무결성 수준과 권한 상승 여부를 확인 |
| 서비스 계정의 현재 토큰에 `SeImpersonatePrivilege Enabled`가 표시됨 | token 가장 기반 권한 상승 전제 조건 후보 | PrintSpoofer 선택 단서 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]에서 서비스 계정 컨텍스트, 대상 build·arch, 실행 파일과 쓰기·실행 경로를 확인한다. `Enabled`만으로 성공을 판정하지 않는다. |
| `SeAssignPrimaryTokenPrivilege`가 표시되고 legacy build·`SeIncreaseQuotaPrivilege`·CLSID 조건을 확인할 수 있음 | primary token을 process에 지정하는 JuicyPotato legacy 경로 후보 | SeAssignPrimaryToken 선택 단서 | [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]에서 exact build·edition, CLSID, 빈 로컬 포트와 `CreateProcessAsUser`·자식 Identity를 확인한다. PrintSpoofer의 SeImpersonate 경로로 바꾸지 않는다. |
| 현재 token에 `SeDebugPrivilege`가 표시됨 | 보호 process 접근 또는 SYSTEM parent 기반 process 생성의 privilege 후보 | SeDebugPrivilege 선택 단서 | LSASS 자료가 목표면 [[LSASS 메모리 덤프]], SYSTEM 명령 실행이 목표면 [[SeDebugPrivilege로 SYSTEM 자식 프로세스 생성]]에서 privilege 활성화·대상 process handle·실제 자식 Identity를 확인한다. `Disabled`를 미보유로 처리하지 않고 이름만으로 성공을 판정하지 않는다. |
| 현재 token에 `SeLoadDriverPrivilege`가 표시됨 | kernel driver load 권한 상승 전제 조건 후보 | SeLoadDriverPrivilege 선택 단서 | [[SeLoadDriverPrivilege로 취약 드라이버 권한 상승]]에서 privilege 활성화, driver signature·architecture, Code Integrity·blocklist, 실제 driver load와 SYSTEM child Identity를 확인한다. `Disabled` 또는 Print Operators 멤버십만으로 성공을 판정하지 않는다. |
| 현재 token에 Event Log Readers가 표시되고 event log에서 자격 증명 단서를 찾는 것이 목표임 | 채널별 event read 후보 | event log 가시성 단서 | [[Windows 이벤트 로그에서 민감 명령줄 검색]]에서 Security·PowerShell channel read, audit 설정과 실제 명령줄 값을 확인한다. |
| 현재 token에 DnsAdmins가 표시되고 관리 대상 Windows DNS 서버가 확인됨 | DNS server-level 설정·zone record 권한 후보 | DNS 서비스 설정 단서 | service account 실행 목표면 [[DnsAdmins DNS 서버 플러그인 DLL 실행]], WPAD 인증 유도 목표면 [[DnsAdmins WPAD DNS 레코드로 NTLM 인증 유도]]에서 현재 token, 기존 설정과 영향 승인을 확인한다. |
| 현재 token에 Hyper-V Administrators가 표시되고 승인된 VM을 관리할 수 있음 | VM export·disk 접근 후보 | guest 가상 디스크 수집 단서 | [[Hyper-V VM 내보내기와 가상 디스크 오프라인 수집]]에서 VM ID·disk chain·export 경로와 read-only 사본 분석을 확인한다. |
| 그 밖의 흥미로운 privilege가 표시됨 | privilege 기반 권한 상승 전제 조건 후보 | 토큰 권한 단서 | privilege의 활성 상태, 대상 서비스·객체와 해당 세부 기법의 조건을 직접 확인 |
| 관리자 또는 SYSTEM으로 실행되는 서비스·작업에서 현재 사용자가 제어 가능한 파일, 경로 또는 설정이 발견된다. | 실행 주체와 제어 지점 확인 | 권한 상승 단서 | 변경 가능한 객체와 trigger를 확인해 해당 권한 상승 기법으로 분기 |
| localhost listener와 service PID가 대응됨 | 같은 호스트에서만 접근할 수 있는 서비스 후보 | 로컬 서비스 공격면 단서 | service의 계정·version·인증·protocol과 실제 연결 결과 확인 |
| 현재 token이나 활성 그룹에 write가 허용된 Named Pipe가 확인됨 | process 간 통신 객체의 write 후보 | Named Pipe 권한 단서 | [[Windows Named Pipe 권한 열거]]에서 pipe server·protocol·실제 요청 권한을 확인 |
| 설치 소프트웨어·KB 단서만 확인됨 | version·patch 기반 후보 | 추가 검증 필요 | 공식 보안 공지, 실제 binary version과 구성 조건을 대조 |
| 저장 비밀번호·token·ticket 또는 재사용 가능한 credential 단서가 발견됨 | 현재 세션 밖의 인증 수단 후보 | 자격 증명 후보 | 계정 종류, 대상 서비스와 유효성을 확인한 뒤 [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| 새 세션에서 로컬 관리자 또는 SYSTEM 토큰이 확인된다. | 권한 상승 완료 | 로컬 관리자 또는 SYSTEM | [[Windows SAM SECURITY SYSTEM 덤프]]와 [[LSASS 메모리 덤프]]의 필요권한 확인 |
| 도구가 차단되거나 실행되지 않음 | AV, 애플리케이션 제어, 셸 제약 또는 파일 문제 가능 | 자동 열거 미실행 | 오류 메시지를 구분하고 수동 명령 또는 허용된 작은 모듈 사용 |
| 특정 조회가 접근 거부됨 | 현재 토큰의 해당 객체 조회 권한 부족 | 부분 열거 | 다른 조회 결과까지 실패한 것으로 일반화하지 말고 객체별 권한 기록 |
| 단서 없음 | 현재 조회 범위에서 권한 상승 후보를 찾지 못함 | 권한 상승 후보 미확인 | [[Windows 저장 자격증명 수집]], [[Windows 파일 자격증명 검색]]과 원격 접근 후보를 별도 확인 |
| exploit 위험 | crash/탐지 가능성 | 시작 상태 유지 | 설정 오남용/credential 경로 우선 |

## 확인할 출력과 권한

- `whoami /all`은 현재 프로세스 토큰만 보여 준다. 로컬 Administrators 그룹 구성원, elevated 토큰, SYSTEM 실행 컨텍스트를 같은 권한 상태로 취급하지 않는다.
- 서비스·작업이 고권한으로 실행된다는 사실만으로 상승할 수 없다. 현재 사용자가 실행 파일, 경로, 서비스 설정 또는 trigger를 실제로 제어할 수 있어야 후보가 된다.
- 운영체제 build나 자동 도구의 취약 표시만으로 exploit 가능성을 확정하지 않고 패치·구성·아키텍처와 재현 조건을 해당 기법에서 검증한다.
- process·listener·Named Pipe 이름은 공격면 단서다. 실행 계정, object DACL, client protocol과 실제 제어 가능성을 확인하기 전에는 권한 상승 후보를 확정하지 않는다.
- 저장 credential 단서는 자격 증명 유효성이나 관리자 권한을 뜻하지 않는다. 계정, 대상 서비스와 실제 로그인 결과를 분리한다.

## 후속 공격 연결

- [[Windows 저장 자격증명 수집]]
- [[Windows 파일 자격증명 검색]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]
- [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]
- [[SeDebugPrivilege로 SYSTEM 자식 프로세스 생성]]
- [[SeLoadDriverPrivilege로 취약 드라이버 권한 상승]]
- [[Windows 이벤트 로그에서 민감 명령줄 검색]]
- [[DnsAdmins DNS 서버 플러그인 DLL 실행]]
- [[DnsAdmins WPAD DNS 레코드로 NTLM 인증 유도]]
- [[Hyper-V VM 내보내기와 가상 디스크 오프라인 수집]]
- [[Windows Named Pipe 권한 열거]]
- [[Windows 방어 제어 상태 확인]]
- [[피벗팅 경로 식별과 내부망 열거]]
- [[Windows SAM SECURITY SYSTEM 덤프]]
- [[LSASS 메모리 덤프]]

## 관련 상태 라우터

- 비밀번호·token·ticket을 확인했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- 로컬 관리자 또는 SYSTEM 실행 컨텍스트를 확인했으면: [[고권한 세션 확보 후 후속 판단]]
- 권한 상승 단서만 확인했으면: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[powershell]]
- [[PEASS]]

## 참고 링크

- [Microsoft Learn: tasklist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tasklist)
- [Microsoft Learn: netstat](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netstat)
- [Microsoft Learn: Get-HotFix 범위](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix)
- [Microsoft Learn: 설치 소프트웨어 조회와 Win32_Product 영향](https://learn.microsoft.com/powershell/scripting/samples/working-with-software-installations)
