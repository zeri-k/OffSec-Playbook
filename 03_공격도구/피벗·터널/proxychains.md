---
tags:
  - 기능/피벗
실행환경: ["Linux"]
필요조건: ["동작 중인 SOCKS4 또는 SOCKS5 프록시", "프록시 주소와 포트가 반영된 설정 파일"]
결과: ["네트워크 접근"]
---

# proxychains

## 도구 개요

ProxyChains는 TCP `connect()`를 사용하는 명령의 연결을 설정된 SOCKS4·SOCKS5 프록시 체인으로 보내는 실행 래퍼다. SOCKS를 직접 지원하지 않는 CLI 도구를 피벗 경로에 태울 때 유용하지만 SYN scan, ICMP와 일반 UDP 같은 raw socket 트래픽은 전달하지 못한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 입력: 동작 중인 SOCKS4/5 주소와 포트가 반영된 ProxyChains 설정 파일
- 실행 입력: SOCKS를 지원하지 않는 TCP 기반 CLI 명령
- 네트워크 조건: 로컬에서 SOCKS listener에 연결되고 SOCKS 서버에서 내부 목적지로 도달 가능
- 제약: raw socket과 UDP 기반 동작은 ProxyChains를 통과하지 않음
- wrapped command의 `<INTERNAL_TARGET>`은 SOCKS server가 도달할 IP/FQDN/URL이며, `<DOMAIN>`·`<USER>`·`<PASSWORD>`는 그 대상 서비스에 전달하는 인증 입력으로 proxy 설정과 별개다. SOCKS 연결 성공은 내부 service 인증·명령 실행을 보장하지 않는다.

## 표준 사용법

1. 먼저 SOCKS proxy를 만든다.
   - 예: Chisel SOCKS, SSH dynamic forwarding(`ssh -D`), Metasploit `socks_proxy`
2. ProxyChains 설정 파일의 `[ProxyList]` 아래에 해당 SOCKS 주소와 포트를 넣는다.
3. 내부망에 접근할 도구 앞에 `proxychains`를 붙여 실행한다.

```bash
sudo vim /etc/proxychains.conf
```

배포판에 따라 설정 파일 경로가 `/etc/proxychains4.conf`일 수도 있다.

```text
[ProxyList]
# 기본 Tor 설정이 있으면 주석 처리하거나 현재 터널 포트와 맞춘다.
# socks4 127.0.0.1 9050

# Chisel/SSH/Metasploit 등으로 연 SOCKS proxy
socks5 127.0.0.1 1080
```

```bash
proxychains <command>
```

필요하면 별도 설정 파일을 지정할 수 있다.

```bash
proxychains -f ./proxychains.conf <command>
```

## 대표 예시

### Chisel SOCKS 포트에 맞춰 설정

```text
[ProxyList]
socks5 127.0.0.1 1080
```

Chisel에서 로컬 SOCKS 포트를 `1080`으로 열었다면 ProxyChains도 같은 포트를 사용해야 한다.


### 설정 파일을 지정해 실행

```bash
proxychains -f ./proxychains.conf -q curl -I http://<INTERNAL_TARGET>
```

터널별로 다른 SOCKS 포트를 쓸 때 전역 설정을 건드리지 않고 별도 config를 사용할 수 있다.

### SOCKS proxy를 통해 내부망 스캔

```bash
proxychains -q nmap -sT -Pn -n <INTERNAL_TARGET>
```

Nmap은 raw SYN scan이 아니라 TCP connect scan인 `-sT`를 사용해야 ProxyChains를 탈 수 있다. IP를 스캔할 때는 `-n`으로 로컬 DNS 조회를 막고, 내부 이름이 필요할 때는 `proxy_dns` 또는 검증된 hosts/DNS 설정을 사용한다.

### SMB 접속을 proxy로 라우팅

```bash
proxychains smbclient //<INTERNAL_TARGET>/HR -U '<USER>'
```

기존 CrackMapExec 명령을 재현하거나 현재 NetExec으로 SMB 인증과 원격 명령을 확인할 때도 같은 SOCKS 경로를 사용할 수 있다.

```bash
proxychains -q crackmapexec smb <INTERNAL_TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>'
proxychains -q crackmapexec smb <INTERNAL_TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>' -x 'whoami'
```

```bash
proxychains -q nxc smb <INTERNAL_TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>' -x 'whoami'
```

ProxyChains의 `<><>-OK`는 SOCKS를 통한 TCP 연결 성공이다. CrackMapExec·NetExec의 `[+]`는 SMB 인증, `Pwn3d!`는 관리자급 원격 작업 후보, `Executed command`와 명령 출력은 실제 원격 실행 성공이므로 각각 분리해서 확인한다.


## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-h`, `--help` | 도움말 확인 | 지원 옵션과 모듈 확인 |
| `-q` | proxychains 자체 출력 최소화 | 도구 결과만 보고 싶을 때 |
| `-f <config>` | 사용할 설정 파일 지정 | 터널별 설정 파일을 따로 둘 때 |

## 주요 설정

| 설정 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `dynamic_chain` | 사용 가능한 proxy를 순서대로 사용 | 여러 proxy 중 일부가 죽어도 진행하고 싶을 때 |
| `strict_chain` | ProxyList 순서대로 모든 proxy를 반드시 통과 | 경로를 정확히 고정해야 할 때 |
| `proxy_dns` | DNS 요청도 proxy를 통해 처리 | 내부 도메인 이름 해석이 필요할 때 |
| `socks4 127.0.0.1 9050` | SOCKS4 proxy 지정 | Tor/SSH 등 SOCKS4 포트 사용 시 |
| `socks5 127.0.0.1 1080` | SOCKS5 proxy 지정 | Chisel, SSH `-D`, Metasploit socks proxy 사용 시 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 프록시 체인을 통해 대상 응답 확인 | SOCKS 경유 접근 성공 | 내부 서비스별 도구를 같은 방식으로 실행 |
| ProxyChains `OK` 뒤 SMB 지문·인증 출력 | SOCKS 경로와 SMB 애플리케이션 요청이 모두 동작함 | 인증 결과와 대상 정보 확인 |
| DNS 이름 해석 실패 | DNS가 프록시 밖에서 처리되거나 내부 DNS 필요 | `proxy_dns`, `/etc/hosts`, 내부 DNS 서버 확인 |
| 일부 도구만 실패 | raw socket 또는 UDP 미지원 | TCP 기반 도구 사용, ligolo 같은 라우팅형 피벗 고려 |
| 모든 연결 실패 | SOCKS 포트 또는 터널 미동작 | `netstat/ss`, chisel/ssh dynamic forward 로그 확인 |
| 스캔 결과 부정확 | proxychains와 도구 방식 불일치 | TCP connect scan, 속도 제한, 라우팅형 피벗 검토 |
| 속도 매우 느림 | 프록시 hop 또는 timeout 문제 | timeout, retry, 스레드 수 조정 |

## 관련 공격기법

- [[SSH 포트 포워딩 피벗팅]]
- [[Chisel SOCKS 터널링]]
- [[Rpivot Reverse SOCKS 피벗팅]]
- [[Meterpreter 라우팅과 포트 포워딩]]
- [[ptunnel-ng ICMP 터널링]]

## 참고 링크

- [ProxyChains-NG 공식 저장소](https://github.com/rofl0r/proxychains-ng)
