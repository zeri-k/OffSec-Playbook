---
tags:
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows"]
필요조건: ["송수신 호스트 사이의 TCP 또는 UDP 도달성"]
결과: ["연결 상태", "평문 배너와 데이터 스트림", "수신 listener"]
---

# netcat

## 도구 개요

Netcat은 TCP·UDP 연결을 열거나 수신해 포트 도달성, 평문 배너와 데이터 스트림을 직접 확인하는 범용 네트워크 도구다. 이 문서는 연결 진단과 수신 listener의 대표 절차를 다룬다. 파일 전송과 셸은 양쪽 명령·무결성 또는 실제 명령 실행을 함께 판단해야 하므로 관련 공격기법에서 다룬다.

## 필요한 입력과 실행 환경

- 실행 환경: `nc`, `netcat` 또는 Ncat이 설치된 Linux/Windows 호스트
- 입력: 접속할 호스트와 포트 또는 열어 둘 로컬 listen 포트
- 프로토콜: 기본 TCP, 필요한 경우 `-u`로 UDP

## 표준 사용법

```shell
nc [옵션] <host> <port>
nc -lvnp <port>
```

Netcat은 배포판에 따라 OpenBSD netcat, traditional netcat, Ncat 등 옵션 차이가 있다. 아래 `nc` 예시는 Linux의 OpenBSD/traditional 계열 문법이며 실행 전에 `nc -h`로 확인한다. Nmap Ncat은 connect mode `ncat <HOST> <PORT>`, listen mode `ncat -l [<LISTEN_ADDR>] <PORT>`를 사용한다. `-e`, `-c`, `-q`, listen 시 `-p` 같은 option은 구현마다 지원·의미가 다르다.

## 대표 예시

### 포트 연결 확인

```shell
nc -nv <TARGET> 80
```

대상 포트에 직접 연결해 응답을 확인한다. HTTP처럼 텍스트 기반이면 직접 요청을 입력할 수 있다.

### 포트 스캔

```shell
nc -nvz <TARGET> 1-1000
```

간단한 포트 열림 여부를 확인한다. 대규모 스캔은 Nmap이 더 적합하다.

### 리스너 생성

```shell
nc -lvnp <LISTEN_PORT> &
NC_LISTENER_PID=$!
ps -p "$NC_LISTENER_PID" -o pid=,args=
ss -ltnp 'sport = :<LISTEN_PORT>'
```

리버스 셸이나 임시 연결을 받을 때 가장 자주 쓰는 형태다. `ps`의 정확한 PID·명령행과 `ss`의 local address·port가 의도한 listener인지 기록한다. 포트가 이미 사용 중이거나 PID가 즉시 사라지면 실행 성공으로 보지 않는다.


## 주요 옵션

| 옵션 | 의미 |
| --- | --- |
| `-l` | listen 모드 |
| `-v` | 자세한 출력 |
| `-n` | DNS 조회 비활성화 |
| `-p <port>` | 로컬 출발지 포트 지정 |
| `-s <ip>` | 로컬 출발지 IP 지정 |
| `-u` | UDP 모드 |
| `-z` | 데이터 전송 없이 포트 확인 |
| `-w <sec>` | 연결/읽기 타임아웃 |
| `-q <sec>` | EOF 후 지정 시간 뒤 종료. 구현체별 지원 차이 있음 |
| `-e`/`-c` | 프로그램 실행 연결. 많은 구현체에서 제거되거나 비활성화됨 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 연결 성공 또는 배너 출력 | TCP/UDP 서비스와 통신 가능 | 수동 프로토콜 대화 또는 서비스별 클라이언트로 전환 |
| listener에 연결 수신 | TCP/UDP 연결과 데이터 스트림 도착 | 셸이면 실제 `id`·`whoami`, 파일이면 양쪽 크기·hash를 관련 기법에서 확인 |
| connection refused/timeout | 서비스 비활성화, 방화벽, IP/포트 오류 | 포트 상태와 라우팅, listen 주소 확인 |
| 입력 후 응답 없음 | 프로토콜 불일치 또는 blind 연결 | verbose 옵션, 다른 클라이언트, 패킷 캡처로 확인 |

## 변경 영향과 복구

위 listener를 종료할 때 이름으로 모든 `nc`·`ncat` process를 종료하지 않고 기록한 PID만 대상으로 한다.

```shell
kill "$NC_LISTENER_PID"
wait "$NC_LISTENER_PID" 2>/dev/null || true
ps -p "$NC_LISTENER_PID" -o pid=,args=
ss -ltnp 'sport = :<LISTEN_PORT>'
```

`ps`에 해당 PID가 없고 `ss`에 같은 listener가 없으면 이번 process 정리가 확인된다. 포트가 계속 열려 있으면 다른 PID의 기존 listener인지 먼저 확인하며 이름 기반 일괄 종료를 사용하지 않는다. 연결이 이미 끊겨 process가 끝났다면 PID 재사용 여부를 `ps`의 명령행으로 확인한 뒤 별도 `kill`을 실행하지 않는다.

## 관련 공격기법

- [[Bind Shell 획득]]
- [[Reverse Shell 획득]]
- [[상황별 파일 전송]]
- [[제한 환경 파일 반입]]

## 참고 링크

- [Ncat Users' Guide — Basic usage](https://nmap.org/ncat/guide/ncat-usage.html)
- [Ncat Users' Guide — Source and timing options](https://nmap.org/ncat/guide/ncat-other-options.html)
