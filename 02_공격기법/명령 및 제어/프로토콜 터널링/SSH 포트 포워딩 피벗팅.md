---
tags:
  - 환경/linux
  - 환경/windows
  - 서비스/ssh
시작조건: ["공격 호스트에서 <PIVOT_IP>:22/TCP SSH 인증 성공", "피벗 호스트에서 <INTERNAL_IP>:<PORT> TCP 연결 가능"]
필요조건: ["피벗 호스트용 SSH password 또는 private key", "공격 호스트에서 <PIVOT_IP>:22/TCP 연결 가능", "피벗 호스트에서 <INTERNAL_IP>:<PORT> 연결 가능", "SSH 서버 정책에서 TCP forwarding 허용"]
결과: ["공격 호스트의 로컬 포트 또는 SOCKS 프록시", "공격 호스트에서 <INTERNAL_IP>:<PORT>까지의 TCP 경로", "피벗 호스트를 경유하는 reverse callback 경로"]
---

# SSH 포트 포워딩 피벗팅

## 한 줄 판단

공격 호스트에서 `<PIVOT_IP>:22`에 SSH 인증할 수 있고 피벗 호스트에서 `<INTERNAL_IP>:<PORT>`에 연결할 수 있으면, `-L`, `-D` 또는 `-R` 포워딩으로 단일 내부 서비스, SOCKS 프록시 또는 역방향 콜백 TCP 경로를 만든다.

## 사용할 때

- 현재 네트워크 위치: 공격 호스트에서는 `<INTERNAL_IP>:<PORT>`에 직접 연결할 수 없지만 `<PIVOT_IP>:22`에는 연결할 수 있고, 피벗 호스트에서는 최종 내부 포트에 연결할 수 있다.
- 명령 실행 위치: `ssh -L`, `ssh -D`, `ssh -R`과 ProxyChains·최종 서비스 클라이언트는 공격 호스트에서 실행하며, 피벗 호스트의 SSH 서버가 내부 연결을 만든다.
- 보유 계정·인증 자료: password 또는 private key는 `<PIVOT_IP>`의 SSH 인증용이다. 이 자료가 `<INTERNAL_IP>`의 DB·RDP·웹 인증에도 유효하다고 간주하지 않는다.
- 현재 권한: 인증된 SSH 사용자로 세션을 유지할 수 있으면 되며 관리자/root 권한은 필수가 아니다. TCP 포워딩 허용 여부는 계정 권한이 아니라 SSH 서버 정책 조건으로 따로 확인한다.
- 지금 가능한 행동: 한 포트는 `-L`, 여러 TCP 서비스는 `-D`, 내부 대상의 reverse callback을 피벗에서 공격 호스트로 전달할 때는 `-R`을 선택한다.
- 성공 범위: 공격 호스트에서 `<INTERNAL_IP>:<PORT>`의 배너나 로그인 단계까지 TCP로 도달하거나 reverse callback을 받을 수 있다. 최종 서비스 인증, 원격 명령 실행과 관리자 권한은 별도 결과다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트→`<PIVOT_IP>:22`와 피벗 호스트→`<INTERNAL_IP>:<PORT>` TCP 연결 가능 | 공격 호스트의 `ssh -v`와 피벗 호스트의 `nc -vz <INTERNAL_IP> <PORT>`를 각각 확인 | SSH listener·방화벽과 피벗의 route·내부 방화벽을 분리 확인 |
| 현재 계정 또는 인증 수단 | `<PIVOT_IP>`에 로그인할 SSH 사용자와 password 또는 private key | `ssh <USER>@<PIVOT_IP>` | 계정 이름, 키 권한, SSH 인증 방식과 포트 확인 |
| 현재 권한 | 인증된 SSH 사용자로 세션 유지 가능하며 관리자/root 권한은 불필요 | `ssh -vN`에서 인증 성공과 세션 유지 확인 | 계정의 SSH 로그온 권한과 세션 제한 확인 |
| 공격 대상의 조건 | SSH 서버 정책이 TCP forwarding을 허용하고 피벗 호스트에서 `<INTERNAL_IP>:<PORT>`가 응답 | forwarding 요청 로그와 피벗 호스트의 `nc -vz <INTERNAL_IP> <PORT>` | 서버의 `AllowTcpForwarding`·bind 제한과 내부 주소·포트·방화벽 확인 |
| 필요한 파일·목록·주소 | `<PIVOT_IP>`, `<INTERNAL_IP>`, `<PORT>`와 필요 시 ProxyChains 설정 | 연결 방향을 도식화하고 `-L`, `-D`, `-R` 중 선택 | listener가 생길 호스트와 connect 대상의 주소·포트 재정리 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 특정 내부 포트 하나만 필요 | 로컬 포트 포워딩이 간단함 | `ssh -L` 사용 |
| 여러 내부 서비스 탐색 필요 | SOCKS 프록시가 유리함 | `ssh -D` + [[proxychains]] 사용 |
| reverse payload가 공격 호스트로 직접 못 옴 | 피벗 호스트 포트로 콜백을 받아야 함 | `ssh -R` 사용 |
| Windows 공격 호스트에서 SSH 클라이언트만 가능 | OpenSSH 대신 Plink 사용 가능 | [[plink]]로 `-D` 또는 `-L` 구성 |

### 절차
1. 공격 호스트에서 `<PIVOT_IP>:22` SSH 인증을, 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` TCP 연결을 각각 확인한다.
2. 목적에 맞춰 `-L`, `-D`, `-R` 중 하나를 고르고 어느 호스트에 listener가 생기는지 적는다.
3. 공격 호스트에서 포워딩 세션을 `-N`으로 유지한다.
4. 공격 호스트의 로컬 포트나 SOCKS 프록시를 통해 내부 서비스 응답을 확인한다.
5. 최종 서비스 계정으로 인증하고, 세션이 열리면 원격 Identity와 권한을 별도로 확인한다.
6. 실패하면 SSH 인증, 포워딩 listener, 프록시 설정과 최종 클라이언트 문제를 분리한다.

### 단일 내부 서비스 접근

```bash
ssh -N -L 1234:<INTERNAL_IP>:3306 <USER>@<PIVOT_IP>
nmap -sV -p1234 127.0.0.1
```

확인할 출력:

- 공격 호스트의 `127.0.0.1:1234`에서 `<INTERNAL_IP>:3306`의 서비스 배너가 보이면 로컬 포트 포워딩 성공이다. 이는 MySQL 인증 성공이나 DB 권한 획득을 의미하지 않는다.

### SOCKS 동적 포워딩

```bash
ssh -N -D 9050 <USER>@<PIVOT_IP>
```

```text
[ProxyList]
socks4 127.0.0.1 9050
```

```bash
proxychains nc -vz <INTERNAL_IP> 3389
proxychains nmap -sT -Pn -n -p3389 <INTERNAL_IP>
```

확인할 출력:

- ProxyChains 로그에 `127.0.0.1:9050 ... <INTERNAL_IP>:<PORT> ... OK`가 보인다.
- ProxyChains를 통한 Nmap은 TCP connect scan인 `-sT`를 사용한다.
- Nmap이 `filtered` 또는 `no-response`를 보여도 `proxychains nc -vz`에서 `OK`와 `open`이 보이면 해당 TCP 포트 접근은 성공으로 판단한다.

### ProxyChains Nmap 분리 확인

```bash
proxychains nc -vz <INTERNAL_IP> <PORT>
proxychains nmap --unprivileged -sT -Pn -n --reason -p<PORT> <INTERNAL_IP>
```

확인할 출력:

- `nc` 출력에서 `... <INTERNAL_IP>:<PORT> ... OK`와 `open`이 보이면 SOCKS 경유 TCP 연결이 성공한 것이다.
- Nmap 출력에 ProxyChains 체인 로그가 보이지 않고 `filtered`만 나오면 대상 차단으로 단정하지 말고 `nc` 결과와 실제 서비스 클라이언트를 우선한다.
- 피벗 호스트 자체에서 내부 포트가 열리는지 다시 보려면 피벗 호스트에서 `nc -vz <INTERNAL_IP> <PORT>`를 직접 실행한다.

### 역방향 포트 포워딩

```bash
ssh -vN -R <PIVOT_BIND_IP>:8080:0.0.0.0:8000 <USER>@<PIVOT_IP>
```

확인할 출력:

- 피벗 호스트의 `<PIVOT_BIND_IP>:8080`으로 들어온 연결이 공격 호스트의 `0.0.0.0:8000` listener로 전달된다. listener 도착 뒤 실제 payload 세션과 그 Identity를 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 피벗 호스트의 `nc -vz`가 내부 포트에 성공함 | SSH 터널의 최종 TCP 대상은 도달 가능함 | hop baseline 확보 | `-L`, `-D`, `-R` 방향과 listener 결정 |
| `127.0.0.1:<LOCAL_PORT>`에서 내부 서비스 배너가 보임 | 로컬 포워딩 성공 | 단일 내부 서비스 접근 | 실제 서비스 클라이언트로 인증·권한 확인 |
| ProxyChains 로그에 SOCKS listener부터 내부 포트까지 `OK`가 보이고 `nc`가 `open` | 동적 포워딩과 proxy hook이 작동함 | SOCKS 경유 TCP 접근 | `nmap -sT -Pn -n` 또는 최종 클라이언트 실행 |
| RDP·웹·DB 클라이언트가 응답하거나 인증 단계에 도달함 | 터널과 최종 프로토콜이 모두 작동하지만 인증은 아직 별도임 | 공격 호스트에서 내부 서비스 접근 | 서비스별 기법에서 인증, 세션과 권한을 순서대로 확인 |
| 피벗 bind 포트의 연결이 공격 호스트 listener에 도착함 | 역방향 포워딩 성공 | reverse callback 경로 | payload 세션이 열리면 대상 Identity와 권한 확인 |
| SSH 연결 자체가 실패함 | 계정, 키, SSH 포트 또는 인증 방식 문제 | 터널 미생성 | `ssh -v`로 인증 단계 확인 |
| SSH listener는 있지만 피벗 host의 baseline도 실패함 | 내부 주소·포트 또는 방화벽 문제 | 내부 서비스 접근 미확보 | 피벗 호스트의 직접 `nc`부터 재검증 |
| `nc`는 `open`인데 Nmap만 `filtered` | TCP 경로는 성공했고 Nmap hook·timeout 판정 문제일 수 있음 | SOCKS 경유 TCP 접근 유지 | `--unprivileged -sT -Pn -n --reason`과 서비스 클라이언트를 우선 |
| callback이 오지 않음 | `-R` bind 주소와 payload `LHOST`·`LPORT`가 맞지 않을 수 있음 | 역방향 경로 미확정 | 각 listener와 payload 목적지를 독립 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 첫 hop | `ssh -v` 인증 성공과 세션 유지 | 피벗 계정 권한과 포워딩 가능 여부 확인 |
| 내부 hop | 피벗 호스트의 `nc -vz` 성공 | 내부 대상까지의 baseline 확정 |
| listener·protocol | 로컬 listen 포트와 `socks4`·`socks5` 일치 | 포트가 열렸다는 사실과 프록시 형식을 분리 |
| proxy hook | `proxychains nc` 체인의 `OK`와 `open` | TCP connect가 실제 프록시를 통과했는지 확정 |
| 최종 클라이언트 | 배너, 로그인 화면, 인증 성공과 원격 `whoami` | 네트워크 접근과 서비스 권한을 구분 |

## 관련 상태 라우터

- 포워딩을 통해 내부 대상의 응답을 확인했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[ssh]]
- [[plink]]
- [[Proxifier]]
- [[proxychains]]
- [[nmap]]
- [[netcat]]
- [[xfreerdp]]

