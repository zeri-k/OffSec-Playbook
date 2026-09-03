---
tags:
  - 환경/ad
  - 환경/windows
  - 서비스/smb
시작조건: ["로컬 또는 도메인 계정의 재사용 가능한 NT hash 확보", "공격 호스트에서 대상 Windows 원격 서비스 포트 도달 가능"]
필요권한: ["인증은 hash 주체의 대상 서비스 로그온 권한", "SMB·WMI·PsExec 원격 명령 실행은 대상 로컬 관리자 권한", "RDP는 Remote Desktop 로그온 권한과 Restricted Admin Mode"]
필요조건: ["NT hash 주체의 로컬·도메인 범위 확인", "SMB 445, WinRM 5985/5986, WMI 135·동적 RPC·44445, RDP 3389 중 선택한 서비스 경로", "RDP PtH는 대상의 Restricted Admin Mode 활성"]
결과: ["NT hash로 대상 서비스 인증 성공", "share 접근 또는 원격 명령 실행", "WinRM shell 또는 RDP GUI 세션", "대상 호스트의 실제 권한 확인"]
---

# Pass the Hash

## 한 줄 판단

공격 호스트에서 로컬 또는 도메인 `<USER>`의 재사용 가능한 NT hash를 보유하고 `<TARGET>`의 SMB·WinRM·WMI·RDP 포트에 도달할 수 있으면, 비밀번호 대신 hash로 인증한 뒤 서비스 접근, 원격 명령 실행, GUI·셀 세션과 대상 관리자 권한을 순서대로 구분한다.

## 사용할 때

- 현재 보유 정보: SAM·LSASS·NTDS에서 수집한 `<USER>`의 NT hash 또는 `LM:NTLM` 형식과 해당 계정의 로컬·도메인 범위를 알고 있다. Responder·PCAP에서 캡처한 NetNTLMv1/v2 challenge-response는 NT hash와 다르며 그대로 Pass the Hash에 사용하지 않는다.
- 명령 실행 위치: NetExec·Impacket·Evil-WinRM·xfreerdp를 실행할 공격 호스트에서 `<TARGET>`의 선택한 서비스 주소·포트까지 직접 또는 검증된 피벗 경로로 도달해야 한다.
- 현재 계정·권한: hash 보유는 아직 `<TARGET>` 인증 성공이 아니다. 로컬 계정은 `--local-auth`, 도메인 계정은 `<DOMAIN>\\<USER>` 범위로 검증하며, 인증 성공과 대상 로컬 관리자 권한을 따로 확인한다.
- 지금 가능한 행동: SMB `445/TCP`로 인증과 share·관리자 단서를 먼저 보고, 해당 서비스 조건과 권한이 맞을 때만 PsExec·WMI·WinRM·RDP로 확장한다.
- 성공 범위: `[+]`는 해당 서비스 인증, `(Pwn3d!)`는 관리자급 원격 실행 가능성, shell prompt·`whoami` 출력은 원격 명령 실행, GUI 세션은 RDP 로그온 성공을 각각 의미한다. Domain Admin 멤버십은 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트에서 `<TARGET>:445`·`5985/5986`·`135` 및 동적 RPC·`3389` 중 사용할 서비스 도달 가능 | 서비스별 TCP 연결과 피벗 경로 확인 | 대상 주소·서비스 listener·방화벽·피벗 route 확인 |
| 현재 계정 또는 인증 수단 | `<USER>`의 재사용 가능한 NT hash 또는 `LM:NTLM` | SAM·LSASS·NTDS 출처와 hash 형식 확인 | NetNTLM challenge-response와 NT hash를 구분하고 username·hash 쌍 재확인 |
| 현재 권한 | 공격 호스트에서 필요 클라이언트 실행 가능, 대상 권한은 아직 미확인 | 도구 실행 가능 여부와 `netexec` 인증 결과 | 로컬 도구 환경 복구, 대상은 인증 성공 후 서비스별 권한 확인 |
| 공격 대상의 조건 | 계정 범위가 대상과 일치하고 선택 서비스가 NTLM hash 인증을 허용 | 로컬은 `--local-auth`, 도메인은 `DOMAIN\\user`·`user@domain`; RDP는 Restricted Admin Mode 확인 | 계정 범위, UAC Remote Restrictions, NTLM·Restricted Admin 정책 확인 |
| 필요한 파일·목록·주소 | `<TARGET>`, `<USER>`, `<DOMAIN>` 필요 시, `<NTLM_HASH>` | 계정·hash·대상 쌍과 주소 확인 | 로컬·도메인 계정 표기와 대상 주소 수정 |

## 실행

### 서비스 선택

| 목표 | 대상 경로·조건 | 선택할 방식 | 성공 결과 |
|---|---|---|---|
| hash 유효성과 share 권한부터 확인 | `<TARGET>:445`, 올바른 로컬·도메인 계정 범위 | NetExec SMB | `[+]` 인증과 share별 읽기·쓰기 권한. 원격 shell은 아직 아님 |
| 관리자 hash로 서비스 기반 shell 실행 | `<TARGET>:445`, `ADMIN$`·서비스 생성이 가능한 로컬 관리자 | Impacket PsExec | 대상에 임시 서비스를 만들고 얻은 원격 shell·실행 Identity |
| 서비스 생성 없이 WMI 명령 실행 | `<TARGET>:135`, 동적 RPC와 `445`, 원격 WMI가 허용된 로컬 관리자 | Impacket WMIExec | 대상의 WMI 명령 출력 또는 제한된 대화형 shell |
| PowerShell Remoting 세션 필요 | `<TARGET>:5985/5986`, WinRM endpoint 로그온 권한 | Evil-WinRM | 인증된 계정 Identity의 원격 PowerShell prompt |
| GUI 세션 필요 | `<TARGET>:3389`, Remote Desktop 로그온 권한과 Restricted Admin Mode | xfreerdp `/pth` | RDP GUI 세션. 관리자 token은 세션 안에서 별도 확인 |

같은 NT hash라도 서비스별 포트·정책·로그온 권한이 다르므로 한 서비스의 인증 성공을 다른 서비스의 원격 실행 성공으로 확대하지 않는다.

1. NT hash의 `<USER>`가 `<TARGET>`의 로컬 계정인지 `<DOMAIN>` 계정인지 구분하고 명령 실행 호스트에서 대상 서비스 포트까지의 경로를 확인한다.
2. 공격 호스트에서 SMB `445/TCP`로 hash 인증 성공 `[+]`, share 접근, 관리자급 단서 `(Pwn3d!)`를 각각 확인한다.
3. 필요 포트·정책·권한이 맞는 WinRM·WMI·PsExec·RDP 중 하나로 원격 명령 출력 또는 세션을 검증한다.
4. 성공한 호스트에서 `whoami`·그룹·무결성 수준으로 실제 원격 실행 계정과 권한을 확인한 뒤, 해당 권한이 허용할 때만 SAM·LSA·LSASS·공유를 후속 확인한다.

### Linux 공격 호스트에서 실행

#### SMB 권한 확인

```bash
netexec smb <TARGET> -u <USER> -H <NTLM_HASH> --local-auth
netexec smb <TARGET> -u <USER> -H <NTLM_HASH> --shares
```

확인할 출력:

- `[+]`는 `<USER>`의 NT hash가 `<TARGET>` SMB 인증에 성공한 것이다. share 목록·읽기·쓰기는 각 share 권한으로 별도 확인한다.
- `(Pwn3d!)`는 대상에서 관리자급 원격 실행 가능성을 나타내며, 실제 명령 출력과 `whoami`로 실행 주체를 확정한다.

#### Impacket 원격 실행

```bash
impacket-psexec <USER>@<TARGET> -hashes :<NTLM_HASH>
impacket-wmiexec <USER>@<TARGET> -hashes :<NTLM_HASH>
```

확인할 출력:

- shell prompt·`whoami`·명령 출력이 `<TARGET>`에서 반환되면 원격 명령 실행이 확인된다. PsExec의 서비스 생성과 WMI 실행은 실행 방식이 다르므로 출력 주체를 `whoami`로 구분한다.

#### WinRM/RDP PtH

```bash
evil-winrm -i <TARGET> -u <USER> -H <NTLM_HASH>
xfreerdp /v:<TARGET> /u:<USER> /pth:<NTLM_HASH> /dynamic-resolution /drive:linux,<OPERATOR_HOME>/Windows
```

확인할 출력:

- WinRM PowerShell prompt는 `<TARGET>:5985/5986`의 WinRM 인증과 원격 PowerShell 세션, RDP GUI는 `<TARGET>:3389`의 RDP 인증과 Remote Desktop 로그온 권한을 확인한 것이다.
- RDP PtH는 Restricted Admin Mode 조건을 별도 확인하고, GUI 내 `whoami /all`로 관리자·무결성 수준을 확인한다.

### Windows 공격 호스트에서 실행

#### Invoke-TheHash로 SMB 또는 WMI 명령 실행

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "whoami /all" -Verbose
Invoke-WMIExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "whoami /all"
```

확인할 출력:

- `successfully authenticated`는 NT hash로 대상 인증에 성공한 단계다.
- `Service Control Manager write privilege`, 임시 서비스 생성·삭제 메시지는 SMB 방식의 원격 실행 경로가 동작한 결과다.
- `Command executed with process id <PID>`는 WMI가 대상에서 프로세스를 만들었다는 결과이며, 명령 출력 회수 여부와 실제 실행 계정은 별도로 확인한다.

#### Windows 대상 호스트에서 Restricted Admin Mode 확인과 활성화

RDP PtH가 필요하고 대상 호스트에서 이미 관리자 명령을 실행할 수 있을 때만 기존 값을 기록한 뒤 변경한다.

```cmd
reg query HKLM\System\CurrentControlSet\Control\Lsa /v DisableRestrictedAdmin
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
reg query HKLM\System\CurrentControlSet\Control\Lsa /v DisableRestrictedAdmin
```

`DisableRestrictedAdmin    REG_DWORD    0x0`을 확인한 뒤 Linux 공격 호스트의 `xfreerdp /pth` 절차로 돌아간다. 이 설정 변경 자체는 RDP 로그온 성공을 뜻하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 인증 실패 | 계정 범위 오류, hash 오타 | 시작 상태 유지 | 로컬/도메인, LM:NTLM 형식, username 확인 |
| `[+]` 인증 성공, 실행 실패 | hash는 유효하지만 해당 원격 실행 권한 또는 서비스 조건 부족 | 인증된 서비스 접근, 원격 명령 미확보 | share·LDAP·WinRM 권한, UAC·RPC·방화벽 조건 확인 |
| 로컬 admin인데 실패 | UAC Remote Restrictions | 시작 상태 유지 | RID-500 여부, `LocalAccountTokenFilterPolicy` |
| RDP 실패 | Restricted Admin Mode 비활성 | 시작 상태 유지 | WinRM/SMB/WMI로 대체 |
| WinRM·PsExec·WMI prompt 또는 RDP GUI가 열리고 `<TARGET>`의 명령 출력이 보임 | 해당 서비스에서 원격 명령 실행 또는 GUI 로그온 성공 | `<TARGET>` 원격 세션 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 계정·호스트·무결성 수준 확인 |
| `(Pwn3d!)` 후 `whoami /groups` 또는 원격 실행으로 관리자 권한 확인 | 대상 로컬 관리자 권한으로 실행 가능 | `<TARGET>` 로컬 관리자 원격 실행 | [[고권한 세션 확보 후 후속 판단]]에서 SAM·LSA·LSASS 접근 영향 재평가 |
| 여러 호스트에서 같은 hash 성공 | 재사용 영향 확보 | 재사용 영향 | hash가 재사용되는 호스트와 각 서비스 권한 차이 확인 |

## 확인할 출력과 권한

- hash 보유: username·NT hash 쌍을 알고 있지만 `<TARGET>` 인증은 아직 미확인이다.
- 인증 성공: NetExec `[+]`, WinRM·RDP 인증 성공으로 해당 서비스에서 hash·계정 쌍이 유효함을 확인한다.
- 서비스 접근: SMB share 목록·읽기·쓰기, WinRM prompt, RDP GUI 등 서비스별 실제 허용 행동으로 확인한다.
- 원격 명령 실행: PsExec·WMI·WinRM의 `whoami`·`hostname` 출력으로 실행 호스트와 주체를 확정한다.
- 관리자·도메인 권한: `(Pwn3d!)`를 단서로 삼되 실제 원격 명령·그룹·무결성 수준으로 로컬 관리자 여부를 확정하고, Domain Admin 멤버십과 DCSync 권한은 별도로 확인한다.

## 변경 영향과 복구

Restricted Admin Mode 값을 변경했다면 실행 전 `reg query` 결과를 기준으로 되돌린다.

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d <PREVIOUS_DWORD> /f
```

실행 전 값이 존재하지 않았다면 이번에 만든 값만 삭제한다.

```cmd
reg delete HKLM\System\CurrentControlSet\Control\Lsa /v DisableRestrictedAdmin /f
reg query HKLM\System\CurrentControlSet\Control\Lsa /v DisableRestrictedAdmin
```

## 후속 공격 연결

- SMB 관리자 권한 확인: [[Windows SAM SECURITY SYSTEM 덤프]]
- 활성 세션/메모리 접근 가능: [[LSASS 메모리 덤프]]
- 원격 명령 실행: [[WMI 원격 명령 실행]], [[WinRM 원격 PowerShell 세션]]
- GUI 확인 필요: [[RDP 로그인과 GUI 세션]]
- 공유 접근만 가능한 경우: [[SMB 공유 자격증명 수집]]

## 관련 서비스

- [[445_SMB]]
- [[5985_5986_WinRM]]
- [[135_WMI]]
- [[3389_RDP]]

## 관련 상태 라우터

- 인증으로 Windows 세션 또는 명령 실행을 얻었으면: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[netexec]]
- [[impacket-psexec]]
- [[impacket-wmiexec]]
- [[evil-winrm]]
- [[xfreerdp]]
- [[mimikatz]]
- [[Invoke-TheHash]]
