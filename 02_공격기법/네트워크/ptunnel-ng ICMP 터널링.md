---
tags:
  - 환경/linux
시작조건: ["공격 호스트에서 <PIVOT_IP>로의 일반 TCP 경로 제한", "공격 호스트와 <PIVOT_IP> 사이 ICMP 왕복 가능", "피벗 호스트 셀 확보"]
필요권한: ["공격 호스트와 피벗 호스트에서 raw ICMP socket을 열 sudo 또는 capability", "피벗 호스트 SSH 로그인 권한"]
필요조건: ["양쪽에서 ptunnel-ng 실행 가능", "피벗 호스트의 <PIVOT_IP>:22 SSH 서비스 실행", "SOCKS 확장 시 피벗 호스트에서 <INTERNAL_IP>:<PORT> 연결 가능", "피벗 SSH 계정의 비밀번호 또는 키"]
결과: ["공격 호스트의 127.0.0.1:2222에서 <PIVOT_IP>:22로의 ICMP 터널", "피벗 호스트 SSH 세션", "공격 호스트의 SOCKS 프록시"]
---

# ptunnel-ng ICMP 터널링

## 한 줄 판단

공격 호스트에서 `<PIVOT_IP>`로의 TCP가 제한되지만 ICMP가 왕복하고 피벗 호스트에서 raw socket을 열 수 있으면, 피벗에서 ptunnel-ng server를 실행해 공격 호스트의 `127.0.0.1:2222`를 피벗의 `22/TCP` SSH로 전달하고 필요하면 SOCKS로 확장한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트↔`<PIVOT_IP>` ICMP 왕복, 피벗의 `22/TCP` listener | `ping <PIVOT_IP>`과 피벗 호스트의 SSH 서비스 상태 | ICMP 패킷 캡처, SSH listener·방화벽 확인 |
| 현재 계정 또는 인증 수단 | 피벗 셀과 피벗 SSH `<USER>`의 비밀번호 또는 키 | `whoami`, SSH 계정·키 소유 확인 | SSH 인증 수단을 먼저 확보 |
| 현재 권한 | 공격·피벗 호스트에서 raw socket 사용 가능 | `sudo ./ptunnel-ng ...` | sudo 권한 또는 raw socket capability 확인 |
| 공격 대상의 조건 | 1차로 `<PIVOT_IP>:22`, SOCKS 후 `<INTERNAL_IP>:<PORT>` 응답 | 피벗 SSH listener와 피벗에서 내부 서비스 응답 확인 | SSH 서비스와 피벗→내부 경로를 별도 진단 |
| 필요한 파일·목록·주소 | 양쪽의 ptunnel-ng, `<PIVOT_IP>`, `<USER>`, SSH 인증 수단 | 바이너리 실행과 placeholder 대응 확인 | OS·아키텍처에 맞는 바이너리와 올바른 주소 사용 |

## 실행

`<PIVOT_IP>`는 ICMP를 주고받는 피벗 주소(가상 예시 `192.0.2.70`)이고, `<ATTACKER_IP>`는 client를 실행하는 공격 호스트 주소다. `<USER>`·`<PASSWORD>`는 ptunnel과 분리된 SSH 인증 자료이며 `<PTUNNEL_SERVER_PID>`·`<PTUNNEL_CLIENT_PID>`는 양쪽 시작 출력의 PID다. server는 피벗 호스트에서, client와 후속 SSH·SOCKS 확인은 공격 호스트에서 실행해 ICMP 전달·SSH 인증·내부 TCP 도달성을 구분한다.

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| TCP outbound 제한 | SSH/Chisel이 실패할 수 있음 | ICMP 터널 검토 |
| ICMP reply는 정상 | ptunnel-ng 가능성 | 서버/클라이언트 실행 |
| 터널 후 SSH 가능 | SOCKS dynamic forwarding 확장 가능 | `ssh -D` + ProxyChains |

### 절차
1. 공격 호스트와 피벗 호스트에 ptunnel-ng를 준비하고 양쪽 raw socket 권한을 확인한다.
2. 피벗 호스트에서 ICMP 터널 server를 실행해 `<PIVOT_IP>:22`로 전달하도록 대기한다.
3. 공격 호스트에서 client를 실행해 `127.0.0.1:2222`를 피벗 호스트 SSH 포트로 연결한다.
4. 공격 호스트에서 로컬 `2222/TCP`로 SSH에 접속해 ICMP 전달과 `<USER>` 로그인을 각각 검증한다.
5. 필요하면 공격 호스트에서 SSH `-D 9050`으로 SOCKS 프록시를 만들고 `<INTERNAL_IP>:<PORT>`를 확인한다.

### 피벗 호스트에서 서버 시작

```bash
sudo '<PIVOT_PTUNNEL_PATH>' -r<PIVOT_IP> -R22 &
PTUNNEL_SERVER_PID=$!
ps -p "$PTUNNEL_SERVER_PID" -o pid=,args=
```

확인할 출력:

- `Starting ptunnel-ng`와 연결 대기 로그.

### 공격 호스트에서 클라이언트 시작

```bash
sudo '<ATTACKER_PTUNNEL_PATH>' -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22 &
PTUNNEL_CLIENT_PID=$!
ps -p "$PTUNNEL_CLIENT_PID" -o pid=,args=
```

확인할 출력:

- 공격 호스트의 `127.0.0.1:2222`가 ICMP 터널을 거쳐 피벗 호스트의 `<PIVOT_IP>:22` SSH로 이어진다. 로컬 listener 생성만으로 SSH 인증 성공을 판정하지 않는다.

### ICMP 터널 위 SSH/SOCKS

```bash
ssh -p2222 -l<USER> 127.0.0.1
ssh -N -D 127.0.0.1:9050 -p2222 -l<USER> 127.0.0.1 &
SSH_SOCKS_PID=$!
ps -p "$SSH_SOCKS_PID" -o pid=,args=
proxychains nc -vz <INTERNAL_IP> 3389
proxychains nmap -sT -Pn -n -p3389 <INTERNAL_IP>
```

확인할 출력:

- SSH 로그인이 성공하면 `<USER>` 권한의 피벗 호스트 셀을 획득한 것이며 root 권한을 의미하지 않는다.
- `proxychains nc -vz`에서 `127.0.0.1:9050 ... <INTERNAL_IP>:3389 ... OK`와 `open`이 보이면 공격 호스트에서 ICMP·SSH·SOCKS를 거쳐 내부 TCP 포트까지 도달한 것이다. RDP 인증과 권한은 아직 미확인 상태다.
- ProxyChains를 통한 Nmap은 TCP connect scan인 `-sT`를 사용한다.
- ICMP 터널은 느릴 수 있으므로 Nmap이 `filtered`를 보여도 `nc`가 `open`이면 포트 접근은 `nc` 결과를 우선한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `ping` 또는 패킷 관찰로 ICMP가 왕복함 | ICMP 기반 운반 경로 후보가 있음 | ICMP baseline 확보 | 양쪽 ptunnel-ng 실행 |
| ptunnel-ng 양쪽 로그에 세션과 트래픽이 보임 | ICMP 터널이 생성됨 | 로컬 TCP 포워딩 | `127.0.0.1:2222` SSH 확인 |
| 공격 호스트의 로컬 `2222`로 SSH 인증 성공 | ICMP 터널 위 TCP와 피벗 SSH `<USER>` 인증이 모두 동작함 | `<USER>` 권한의 피벗 SSH 세션 | 필요하면 `ssh -D`로 SOCKS 생성 |
| ProxyChains `nc` 체인이 `OK`이고 `<INTERNAL_IP>:<PORT>`가 `open` | ICMP·SSH·SOCKS의 전체 hop이 통과함 | 공격 호스트에서 내부 TCP 서비스 접근 | 최종 서비스 클라이언트로 인증·권한을 별도 확인 |
| raw socket 오류로 시작하지 못함 | sudo 또는 capability가 부족함 | 터널 미생성 | 실행 권한과 raw socket 조건 확인 |
| ICMP 세션이 생기지 않음 | ICMP 차단 또는 `-p`·`-r`·`-R` 방향 오류 | ICMP baseline만 있거나 경로 미확정 | 방화벽과 양쪽 주소를 재확인 |
| SSH 인증만 실패함 | 터널은 동작하지만 계정·키가 맞지 않음 | TCP 포워딩만 확보 | SSH 인증을 별도 진단 |
| `nc`는 `open`인데 Nmap만 `filtered` | 느린 터널에서 Nmap timeout 또는 hook 판정이 흔들림 | 내부 TCP 접근 유지 | `nc`와 서비스 클라이언트를 우선하고 Nmap timeout 조정 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| ICMP baseline | echo reply 또는 packet capture | TCP가 아닌 ICMP 경로 자체를 확인 |
| raw socket | ptunnel-ng 시작 로그와 권한 오류 없음 | 양쪽 실행 위치의 sudo/capability 확인 |
| TCP listener | `127.0.0.1:2222` SSH handshake | ICMP 터널 위 TCP 전달 확인 |
| SOCKS와 최종 hop | `socks4`·`socks5` 일치, ProxyChains `OK`, `nc open` | SSH 인증과 내부 서비스 접근을 분리 |

## 변경 영향과 복구

작업 전 공격 호스트에서 `127.0.0.1:2222`, `127.0.0.1:9050` listener와 실행할 `<ATTACKER_PTUNNEL_PATH>`의 기존 존재 여부를 기록한다. 피벗 호스트에서도 실행할 `<PIVOT_PTUNNEL_PATH>`의 기존 존재 여부를 따로 기록한다. 실행하면서 `<PTUNNEL_SERVER_PID>`, `<PTUNNEL_CLIENT_PID>`, `<SSH_SOCKS_PID>`와 각 전체 명령행을 실행 호스트별로 기록한다. 이 절차의 명령은 route·TUN·방화벽 규칙을 만들지 않는다.

공격 호스트:

```bash
ss -ltnp | grep -E '127\.0\.0\.1:(2222|9050)[[:space:]]'
test -e '<ATTACKER_PTUNNEL_PATH>'
```

피벗 호스트:

```bash
test -e '<PIVOT_PTUNNEL_PATH>'
```

연결이 살아 있을 때는 SOCKS를 통해 접근한 하위 대상의 정리를 먼저 끝낸다. 이후 공격 호스트에서 가장 안쪽 SSH SOCKS, ptunnel-ng client 순서로 종료하고 마지막에 피벗 호스트의 server를 종료한다.

공격 호스트:

```bash
ps -p <SSH_SOCKS_PID> -o pid=,args=
kill <SSH_SOCKS_PID>
ps -p <PTUNNEL_CLIENT_PID> -o pid=,args=
kill <PTUNNEL_CLIENT_PID>
ps -p <SSH_SOCKS_PID>,<PTUNNEL_CLIENT_PID> -o pid=,args=
ss -ltnp | grep -E '127\.0\.0\.1:(2222|9050)[[:space:]]'
```

피벗 호스트:

```bash
ps -p <PTUNNEL_SERVER_PID> -o pid=,args=
kill <PTUNNEL_SERVER_PID>
ps -p <PTUNNEL_SERVER_PID> -o pid=,args=
```

기록한 PID가 다른 명령행으로 바뀌었으면 PID 재사용 가능성이 있으므로 종료하지 않는다. 공격 호스트에서 두 local listener가 작업 전 상태로 돌아오고 양쪽 기록 PID가 사라졌는지 확인한다. ICMP 연결이 먼저 끊겨 하위 호스트 정리를 확인할 수 없으면 원격 정리 완료로 표현하지 않고, 공격 호스트 client와 SOCKS만 로컬에서 정리한 뒤 피벗 접근 복구 시 server PID를 별도 확인한다.

작업 전 `test -e`가 실패해 이번 절차에서 새로 전송한 것으로 확인된 binary만 각 실행 호스트에서 정확한 경로로 제거한다.

공격 호스트:

```bash
rm -- '<ATTACKER_PTUNNEL_PATH>'
test ! -e '<ATTACKER_PTUNNEL_PATH>'
```

피벗 호스트:

```bash
rm -- '<PIVOT_PTUNNEL_PATH>'
test ! -e '<PIVOT_PTUNNEL_PATH>'
```

기존 설치 파일이거나 작업 전 존재 여부를 확인하지 못한 파일은 삭제하지 않는다.

## 관련 상태 라우터

- SOCKS를 거쳐 내부 대상 TCP 접근을 확인했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[ptunnel-ng]]
- [[ssh]]
- [[proxychains]]
- [[nmap]]

## 참고 링크

- [ptunnel-ng 공식 README](https://github.com/utoni/ptunnel-ng)
