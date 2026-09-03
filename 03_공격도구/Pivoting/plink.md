---
tags:
  - 서비스/ssh
  - 기능/피벗
실행환경: ["Windows"]
필요조건: ["피벗 호스트 SSH 계정"]
결과: ["SSH 세션", "SOCKS 프록시", "포트 포워딩"]
---

# plink.exe

## 도구 개요

Plink는 Windows에서 SSH 연결과 동적 SOCKS 포워딩·로컬 단일 포트 포워딩을 명령줄로 만드는 클라이언트다. Windows 피벗 호스트에서 SOCKS 프록시를 열거나 내부 단일 서비스를 로컬 포트로 가져올 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: `plink.exe`를 실행할 수 있는 Windows 호스트
- SSH 입력: 피벗 호스트 주소와 포트, 사용자명, 비밀번호 또는 PuTTY private key
- 동적 포워딩 입력: 열어 둘 로컬 SOCKS 포트
- 로컬 포워딩 입력: 로컬 listen 포트와 내부 목적지 IP·포트

## 표준 사용법

```cmd
plink.exe -ssh -D <LOCAL_SOCKS_PORT> <USER>@<PIVOT_IP>
plink.exe -ssh -L <LOCAL_PORT>:<INTERNAL_IP>:<INTERNAL_PORT> <USER>@<PIVOT_IP>
```

`-D`는 SOCKS 프록시를 만들고, `-L`은 특정 내부 서비스 하나를 로컬 포트로 당긴다.

## 대표 예시

### SOCKS dynamic forwarding

```cmd
plink.exe -ssh -D 9050 <USER>@<PIVOT_IP>
```

확인할 출력:

- Windows 로컬 `9050` 포트에서 SOCKS proxy가 열린다.

### 내부 RDP 단일 포트 포워딩

```cmd
plink.exe -ssh -L 13389:<INTERNAL_IP>:3389 <USER>@<PIVOT_IP>
```

확인할 출력:

- `127.0.0.1:13389` 접속이 내부 RDP `3389`로 전달된다.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-ssh` | SSH 모드 지정 | SSH 터널 생성 |
| `-D <PORT>` | SOCKS dynamic forwarding | 여러 내부 TCP 서비스 접근 |
| `-L <LPORT>:<RHOST>:<RPORT>` | 로컬 포트 포워딩 | RDP/DB/웹 단일 포트 접근 |
| `-l <USER>` | 사용자 지정 | 사용자명 분리 입력 |
| `-i <KEY>` | private key 지정 | 키 기반 인증 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SSH 세션 유지 | 터널 생성 가능 | 로컬 포트 listen 확인 |
| 로컬 SOCKS 포트 listen | `-D` 성공 | [[Proxifier]] 또는 ProxyChains로 내부 접근 |
| 로컬 포트로 서비스 응답 | `-L` 성공 | 해당 서비스 클라이언트 실행 |
| SSH 로그인 실패 | 계정/키/host key 문제 | `-l`, `-i`, SSH 서버 포트 확인 |
| `-D` 후 RDP가 직접 안 됨 | SOCKS 프록시와 단일 포워딩 혼동 | RDP 하나는 `-L 13389:<TARGET>:3389`로 검증 |
| SOCKS client 실패 | proxy 설정 오류 | SOCKS host/port, SOCKS4/5 지원 확인 |
| 내부 서비스 실패 | 피벗 호스트에서 내부 대상 접근 불가 | 피벗 호스트에서 `nc -vz <INTERNAL_IP> <PORT>` |

## 관련 공격기법

- [[SSH 포트 포워딩 피벗팅]]

## 관련 도구

- [[Proxifier]]
