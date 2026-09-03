---
tags:
  - 서비스/ssh
  - 기능/피벗
실행환경: ["Linux"]
필요권한: ["공격 호스트 sudo 권한"]
필요조건: ["피벗 호스트 SSH 계정"]
결과: ["내부망 라우팅", "네트워크 접근"]
---

# sshuttle

## 도구 개요

`sshuttle`은 SSH 세션과 로컬 방화벽 규칙을 이용해 선택한 내부 CIDR의 TCP 연결을 직접 주소 형식으로 전달하는 피벗 도구다. 명령마다 SOCKS 설정을 붙이지 않고 내부 대역을 다룰 때 편리하며, 일반 ICMP와 모든 UDP를 전달하는 완전한 VPN은 아니다.

## 필요한 입력과 실행 환경

- 실행 환경: `sshuttle`과 로컬 firewall backend를 사용할 수 있는 Linux 호스트
- 입력: SSH 피벗 호스트, 계정 또는 key와 내부 CIDR
- 필요 권한: 공격 호스트에서 route/firewall 규칙을 만들 수 있는 sudo 권한

## 표준 사용법

```bash
sudo sshuttle -r <USER>@<PIVOT_IP> <INTERNAL_CIDR> -v
```

로컬 방화벽/라우팅 규칙을 만들어 내부 CIDR로 향하는 TCP 트래픽을 SSH를 통해 보낸다.

## 대표 예시

### 내부 대역 라우팅

```bash
sudo sshuttle -r <USER>@<PIVOT_IP> <INTERNAL_CIDR> -v
```

확인할 출력:

- `Starting sshuttle proxy`
- firewall manager 또는 iptables 규칙 생성 로그.

### 라우팅 후 내부 서비스 확인

```bash
sudo nmap -sT -Pn -n -p3389 <INTERNAL_IP>
```

확인할 출력:

- 내부 IP의 TCP 포트 응답.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-r <USER>@<HOST>` | SSH 피벗 호스트 지정 | 기본 실행 |
| `<CIDR>` | 라우팅할 내부 대역 | 내부망 접근 |
| `-v` | 자세한 로그 | route/firewall 확인 |
| `--dns` | DNS도 터널링 | 내부 DNS 이름 해석 필요 |
| `-x <CIDR>` | 제외할 대역 | 로컬/VPN 대역 제외 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| firewall rule 생성 | 로컬 라우팅 준비 | 내부 IP로 직접 도구 실행 |
| 내부 포트 응답 | 피벗 성공 | 포트별 서비스 노트로 이동 |
| SSH 인증 오류 | 피벗 계정 문제 | `ssh -v`로 인증 확인 |
| 일부 프로토콜 실패 | UDP/raw socket 미지원 가능 | TCP 도구로 재검증 |
| route 생성 실패 | sudo, iptables/nftables 또는 method 문제 | 로컬 권한과 firewall backend 확인 |
| 모든 내부 연결 실패 | 내부 CIDR 또는 피벗 경로 오류 | 피벗 호스트에서 내부 포트를 직접 확인 |
| DNS 이름 실패 | DNS 터널링 미사용 | `--dns`, `/etc/hosts` 또는 IP 직접 지정 |

## 관련 공격기법

- [[sshuttle SSH 피벗팅]]
