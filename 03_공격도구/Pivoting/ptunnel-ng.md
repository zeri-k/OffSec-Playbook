---
tags:
  - 기능/피벗
실행환경: ["Linux"]
필요권한: ["양쪽 호스트의 셸", "raw socket 생성 권한"]
필요조건: ["공격 호스트와 피벗 호스트 사이의 ICMP 도달성", "피벗 호스트의 SSH 서비스와 계정"]
결과: ["ICMP 터널", "SSH 세션", "SOCKS 프록시"]
---

# ptunnel-ng

## 도구 개요

ptunnel-ng는 TCP 스트림을 ICMP 패킷 안에 캡슐화해 양단 사이에 로컬 포트 중계를 만드는 터널링 도구다. TCP 경로를 직접 열기 어렵지만 ICMP가 통과하는 환경에서 SSH 같은 단일 TCP 서비스를 전달하고 그 위에 SOCKS를 구성할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: 공격 호스트와 Linux 피벗 호스트 양쪽의 `ptunnel-ng`
- 입력: ICMP peer, 로컬 listen 포트, 전달할 원격 호스트와 TCP 포트
- 권한 조건: 양쪽에서 raw socket을 열 수 있는 권한
- SSH 연계 입력: 피벗 호스트의 SSH 계정 또는 private key

## 표준 사용법

```bash
sudo ./ptunnel-ng -r<PIVOT_IP> -R22
sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22
```

서버는 피벗 호스트에서 실행하고, 클라이언트는 공격 호스트에서 로컬 포트를 만들어 연결한다.

## 대표 예시

### 피벗 호스트 서버

```bash
sudo ./ptunnel-ng -r<PIVOT_IP> -R22
```

확인할 출력:

- `Starting ptunnel-ng` 로그.

### 공격 호스트 클라이언트

```bash
sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22
```

확인할 출력:

- 로컬 `2222`가 피벗 호스트 SSH로 연결된다.

### 터널 위에 SSH SOCKS 생성

```bash
ssh -D 9050 -p2222 -l<USER> 127.0.0.1
```

확인할 출력:

- SSH 인증 성공 후 로컬 SOCKS 포트 생성.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-p<IP>` | ICMP tunnel peer | 클라이언트에서 피벗 호스트 지정 |
| `-l<PORT>` | 로컬 listen 포트 | 공격 호스트 로컬 SSH 포트 |
| `-r<IP>` | remote host | 피벗 호스트 또는 relay 대상 |
| `-R<PORT>` | remote port | 일반적으로 SSH `22` |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| session/traffic 통계 | ICMP 터널 동작 | SSH 접속 테스트 |
| `127.0.0.1:2222` SSH 성공 | 터널 검증 완료 | `ssh -D` 또는 내부 열거 |
| 연결 없음 | ICMP 차단 가능 | ping, 방화벽, IP 옵션 확인 |
| 권한 오류 | raw socket 권한 부족 | sudo 권한 확인 |
| SSH 실패 | 계정 문제 또는 remote port 오류 | `-R22`, SSH 계정/키 확인 |
| 실행 형식 오류 | 바이너리 아키텍처 또는 실행 권한 불일치 | 파일 형식, 아키텍처, 실행 권한 확인 |

## 관련 공격기법

- [[ptunnel-ng ICMP 터널링]]
