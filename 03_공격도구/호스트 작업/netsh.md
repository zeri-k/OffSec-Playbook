---
tags:
  - 환경/windows
  - 기능/피벗
실행환경: ["Windows"]
필요권한: ["Windows 로컬 관리자 권한"]
필요조건: ["피벗 호스트에서 내부 목적지 TCP 포트로의 도달성", "공격 호스트에서 listen 주소와 포트로의 도달성"]
결과: ["포트 포워딩", "네트워크 접근"]
---

# netsh

## 도구 개요

`netsh interface portproxy`는 Windows 호스트의 수신 주소·포트로 들어온 TCP 연결을 다른 주소·포트로 전달한다. 별도 터널 도구 없이 단일 TCP 서비스를 포워딩할 때 유용하지만 Windows 네트워크 설정을 변경하므로 생성한 규칙과 방화벽 예외를 추적해야 한다.

## 필요한 입력과 실행 환경

- 실행 환경: 로컬 관리자 권한을 가진 Windows 피벗 호스트
- 입력: `listenaddress`, `listenport`, `connectaddress`, `connectport`
- 네트워크 조건: 공격 호스트에서 listen 포트로, 피벗 호스트에서 내부 목적지 TCP 포트로 각각 도달 가능
- listen address·port는 Windows pivot에 만들 endpoint, connect address·port는 pivot 관점 destination(예: `192.0.2.50:443`)이다. account는 명령 실행 권한과 별개이고 portproxy 추가는 listener가 실제로 동작함을 보장하지 않는다.

## 표준 사용법

`<LISTEN_ADDRESS>:<LISTEN_PORT>`는 Windows pivot의 portproxy listener, `<CONNECT_ADDRESS>:<CONNECT_PORT>`는 pivot 관점 destination이다. add·show·delete에는 같은 listen address/port를 재사용하며 listener 등록, final TCP response, destination authentication은 separate results다.

```cmd
netsh.exe interface portproxy add v4tov4 listenport=<LISTEN_PORT> listenaddress=<PIVOT_IP> connectport=<DEST_PORT> connectaddress=<DEST_IP>
netsh.exe interface portproxy show v4tov4
```

피벗 호스트의 listen 포트로 들어온 TCP 연결을 내부 대상 IP/포트로 전달한다.

## 대표 예시

### 내부 RDP 포워딩 추가

```cmd
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=<PIVOT_IP> connectport=3389 connectaddress=<INTERNAL_IP>
```

확인할 출력:

- 명령 오류 없이 rule 추가.

### 설정 확인

```cmd
netsh.exe interface portproxy show v4tov4
```

확인할 출력:

- listen address/port와 connect address/port가 의도대로 표시된다.

### 공격 호스트에서 접속

```bash
xfreerdp /v:<PIVOT_IP>:8080 /u:<USER> /p:'<PASSWORD>'
```

확인할 출력:

- 내부 RDP 인증 단계 또는 GUI 세션.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `interface portproxy add v4tov4` | IPv4 to IPv4 TCP 포워딩 추가 | 기본 피벗 |
| `listenport` | 피벗 호스트에서 열 포트 | 공격 호스트가 접속할 포트 |
| `listenaddress` | 피벗 호스트 listen IP | 특정 NIC/IP 바인딩 |
| `connectport` | 내부 대상 포트 | RDP 3389 등 |
| `connectaddress` | 내부 대상 IP | 피벗 호스트가 접근할 내부 호스트 |
| `show v4tov4` | 설정 확인 | rule 검증 |
| `delete v4tov4` | 설정 삭제 | 정리 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| show에 rule 표시 | portproxy 설정됨 | 외부에서 listen 포트 접속 |
| listen 포트 응답 | 포워딩 동작 | 서비스 클라이언트 실행 |
| add 실패 | 관리자 권한 부족 | elevated cmd, UAC, 그룹 확인 |
| rule은 있으나 접속 실패 | 방화벽 또는 listenaddress 오류 | 방화벽 rule, `0.0.0.0`/특정 IP 검토 |
| 내부 포트 실패 | connectaddress/connectport 오류 | 피벗 호스트에서 내부 포트 연결 확인 |

## 관련 공격기법

- [[Windows Netsh Portproxy 포트 포워딩]]
