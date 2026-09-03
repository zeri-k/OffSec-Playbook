---
tags:
  - 서비스/ssh
  - 기능/피벗
실행환경: ["Linux", "Windows"]
필요조건: ["SSH 서버 주소", "SSH 계정과 비밀번호 또는 개인키"]
결과: ["셸", "명령 출력", "포트 포워딩", "SOCKS 프록시"]
---

# ssh

## 도구 개요

`ssh`는 암호화된 원격 셸과 단일 명령 실행을 제공하고 로컬·리버스·동적 포트 포워딩도 구성하는 클라이언트다. 원격 관리 접근을 검증하거나 SSH 세션을 내부 서비스 피벗 경로로 사용할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: OpenSSH client가 설치된 Linux 또는 Windows 호스트
- 입력: SSH 서버, 포트와 사용자명
- 인증 입력: 비밀번호, private key 또는 사용 가능한 GSSAPI/Kerberos 컨텍스트
- 포워딩 입력: 로컬·원격 listen 포트와 목적지 주소·포트


## 표준 사용법

```bash
ssh <user>@<target>
```

## 대표 예시

### 비밀번호 기반 원격 로그인

```bash
ssh <USER>@<TARGET>
```

### private key로 로그인

```bash
ssh -i id_rsa user@<TARGET>
```

### 원격 명령만 실행

```bash
ssh user@<TARGET> 'id; hostname'
```

### SOCKS proxy 생성

```bash
ssh -N -D <LOCAL_SOCKS_PORT> <USER>@<PIVOT_IP>
```

### 단일 내부 서비스의 로컬 포워딩

```bash
ssh -N -L <LOCAL_PORT>:<INTERNAL_IP>:<INTERNAL_PORT> <USER>@<PIVOT_IP>
```

`127.0.0.1:<LOCAL_PORT>`의 서비스 응답은 피벗 호스트를 통한 TCP 전달을 뜻하며 내부 서비스 인증 성공을 뜻하지 않는다.

### 피벗 호스트 수신 포트의 역방향 포워딩

```bash
ssh -vN -R <PIVOT_BIND_IP>:<PIVOT_LISTEN_PORT>:<ATTACKER_LISTEN_IP>:<ATTACKER_LISTEN_PORT> <USER>@<PIVOT_IP>
```

verbose 출력의 `forwarded-tcpip`와 공격 호스트 listener 도착을 확인한다. SSH 포워딩 성공과 listener가 처리하는 payload session 획득은 서로 다른 단계다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-i` | private key 지정 |
| `-p` | 포트 지정 |
| `-L` | 로컬 포트 포워딩 |
| `-R` | 리버스 포트 포워딩 |
| `-D` | SOCKS dynamic forwarding |
| `-N` | 명령 실행 없이 터널만 유지 |
| `-o` | SSH 세부 옵션 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 원격 셸 또는 명령 출력 | SSH 인증 성공 | `id`, `hostname`, `sudo -l`, 파일 접근 범위 확인 |
| `Permission denied` | credential/key 오류 또는 인증 방식 제한 | 사용자명, 키 권한, `PreferredAuthentications`, 계정 잠금 확인 |
| 로컬 포트 listen 또는 동적 SOCKS listen | `-L` 또는 `-D` 포워딩 준비 | 내부 서비스 접근을 `curl`, `nmap -sT`, 서비스 클라이언트로 확인 |
| verbose 출력의 `forwarded-tcpip` | 피벗 호스트의 `-R` 수신 연결이 공격 호스트 쪽으로 전달됨 | 공격 호스트 listener와 실제 payload session을 별도 확인 |
| host key / algorithm 오류 | 키 신뢰 또는 구형 알고리즘 문제 | `known_hosts`, `HostKeyAlgorithms`, `PubkeyAcceptedAlgorithms` 확인 |

## 관련 공격기법

- [[SSH credential 및 키 인증 검증]]
- [[SSH credential 및 키 인증 검증]]
- [[SSH 포트 포워딩 피벗팅]]
