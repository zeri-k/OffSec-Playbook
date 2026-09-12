---
tags:
  - 환경/linux
  - 서비스/ssh
시작조건: ["공격 호스트에서 <PIVOT_IP>:22/TCP SSH 인증 성공", "피벗 호스트에서 <INTERNAL_IP>:<PORT> TCP 연결 가능", "피벗 뒤 <INTERNAL_CIDR> 확인"]
필요권한: ["공격 호스트에서 sshuttle 방화벽 규칙을 만들 sudo 권한"]
필요조건: ["피벗 호스트용 SSH password 또는 private key", "피벗 호스트에서 <INTERNAL_CIDR> route와 목표 TCP 포트 확인", "공격 호스트에서 iptables 또는 nftables 기반 라우팅 설정 가능", "양쪽에서 설치된 sshuttle 버전이 요구하는 Python 실행 가능"]
결과: ["공격 호스트의 <INTERNAL_CIDR> TCP 라우팅", "공격 호스트에서 <INTERNAL_IP>:<PORT>까지의 TCP 경로"]
---

# sshuttle SSH 피벗팅

## 한 줄 판단

공격 호스트에서 `<PIVOT_IP>:22`에 SSH 인증할 수 있고 피벗 호스트에서 `<INTERNAL_IP>:<PORT>`에 연결할 수 있으면, 공격 호스트의 `<INTERNAL_CIDR>` TCP 트래픽을 피벗으로 전달해 ProxyChains 없이 내부 서비스에 도달한다.

## 사용할 때

- 현재 네트워크 위치: 공격 호스트에서는 `<INTERNAL_IP>:<PORT>`에 직접 연결할 수 없지만 `<PIVOT_IP>:22`에는 연결할 수 있고, 피벗 호스트에서는 `<INTERNAL_CIDR>`의 목표 TCP 서비스에 연결할 수 있다.
- 명령 실행 위치: `sshuttle`, `nmap`, `curl`, `xfreerdp` 같은 최종 클라이언트는 공격 호스트에서 실행하며, SSH 서버는 `<PIVOT_IP>`에서 트래픽을 내부 대역으로 전달한다.
- 보유 계정·인증 자료: password 또는 private key는 피벗 SSH 인증용이다. 내부 RDP·DB·웹 서비스에는 해당 서비스에서 유효한 별도 인증 자료가 필요하다.
- 현재 권한: 피벗 계정의 관리자/root 권한은 필수가 아니지만, 공격 호스트에서는 방화벽·라우팅 규칙을 만들 sudo 권한이 필요하다.
- 지금 가능한 행동: 이 문서의 기본 `auto` 방식으로 `<INTERNAL_CIDR>`의 여러 TCP 호스트·포트를 일반 클라이언트로 반복 확인한다. 일반 ICMP는 전달되지 않으며, UDP는 Linux TPROXY 방식과 추가 조건을 명시적으로 선택한 별도 범위이므로 기본 절차의 성공으로 간주하지 않는다.
- 성공 범위: 공격 호스트에서 `<INTERNAL_IP>:<PORT>`의 TCP 응답 또는 로그인 단계까지 도달한다. 내부 서비스 인증, 원격 세션과 관리자 권한은 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트→`<PIVOT_IP>:22`와 피벗 호스트→`<INTERNAL_IP>:<PORT>` TCP 연결 가능 | 공격 호스트의 SSH 접속과 피벗 호스트의 `nc -vz`를 각각 확인 | SSH listener·방화벽과 내부 route·대상 포트를 분리 확인 |
| 현재 계정 또는 인증 수단 | `<PIVOT_IP>`용 SSH 사용자와 password 또는 private key | `ssh <USER>@<PIVOT_IP>` | 계정, 키 권한과 SSH 인증 방식 확인 |
| 현재 권한 | 공격 호스트에서 sshuttle 방화벽 규칙 생성 가능 | `sudo -v`와 sshuttle 시작 로그 확인 | sudo 권한과 iptables·nftables backend 확인 |
| 공격 대상의 조건 | 피벗 호스트가 `<INTERNAL_CIDR>` route를 가지고 목표 TCP 포트에 연결 가능 | 피벗 호스트의 `ip route`와 `nc -vz <INTERNAL_IP> <PORT>` | CIDR, gateway, 내부 방화벽과 대상 listener 확인 |
| 필요한 파일·목록·주소 | sshuttle과 `<PIVOT_IP>`, `<INTERNAL_CIDR>`, `<INTERNAL_IP>`, `<PORT>`, 호환되는 양쪽 Python | `sshuttle --version`, 공격 호스트와 피벗 호스트의 `python3 --version`, route 범위를 비교 | 설치한 sshuttle 버전의 공식 Requirements에 맞는 Python을 선택하고 잘못된 CIDR 수정 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 내부 CIDR을 여러 도구로 반복 접근 | sshuttle이 편함 | CIDR 단위로 라우팅 |
| UDP/ICMP 기반 검증 필요 | 기본 `nat`·`nft` 방식은 일반 UDP를 전달하지 않고 ICMP도 이 절차의 범위가 아님 | TCP 도구 중심으로 확인한다. UDP가 반드시 필요하면 공식 요구사항에 따라 Linux TPROXY를 별도로 검토하고, ICMP는 다른 피벗 기법을 선택 |
| route가 잡혔는데 일부 포트만 실패 | 내부 방화벽 또는 서비스 문제 | 피벗 호스트에서 직접 연결 재확인 |

### 절차
1. 피벗 호스트에서 `<INTERNAL_CIDR>` route와 `<INTERNAL_IP>:<PORT>` TCP 응답을 확인한다.
2. 공격 호스트에서 `sshuttle -r`로 SSH 피벗 호스트와 라우팅할 CIDR을 지정한다.
3. sshuttle 로그에서 firewall/routing 규칙 생성을 확인한다.
4. 공격 호스트에서 별도 ProxyChains 없이 내부 IP의 TCP 포트를 먼저 확인한다.
5. 서비스 클라이언트로 인증한 뒤 세션 Identity와 실제 권한을 별도로 확인한다.

### CIDR 라우팅

`--method auto`는 공격 호스트에서 사용 가능한 firewall backend를 고릅니다. 이 문서의 완료 기준은 선택된 방식으로 TCP 연결이 전달되는지까지입니다. UDP는 Linux의 TPROXY 방식에서만 지원되며, 해당 방식이 선택되면 자동으로 활성화됩니다. TPROXY에는 root 권한과 별도 route·policy rule이 필요하므로 아래 기본 명령의 범위에 포함하지 않습니다.

현재 sshuttle 1.3.2 공식 Requirements는 공격 호스트와 피벗 호스트 모두 Python 3.10 이상을 요구합니다. 배포판에 포함된 이전 sshuttle은 요구 버전이 다를 수 있으므로 명령을 실행하기 전에 `sshuttle --version`과 양쪽 `python3 --version`을 확인하고 설치된 버전의 문서를 따릅니다.

```bash
sudo sshuttle -r <USER>@<PIVOT_IP> <INTERNAL_CIDR> -v
```

확인할 출력:

- `Starting sshuttle proxy`
- `iptables` 또는 firewall manager 규칙 생성 로그.

### 내부 RDP 포트 확인

```bash
sudo nmap -sT -Pn -n -p3389 <INTERNAL_IP>
xfreerdp /v:<INTERNAL_IP> /u:<USER> /p:'<PASSWORD>' /dynamic-resolution
```

확인할 출력:

- 공격 호스트에서 `<INTERNAL_IP>:3389`의 TCP 또는 RDP 응답. 이 단계는 RDP 서비스 접근이다.
- `<USER>` 인증과 Remote Desktop 로그온 권한이 모두 충족되면 RDP GUI 세션이 열린다. 세션 내 관리자 권한은 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 피벗 호스트에서 내부 포트가 직접 열림 | 라우팅 대상 CIDR과 최종 hop이 유효함 | 내부 TCP baseline 확보 | sshuttle에 정확한 CIDR 지정 |
| `Starting sshuttle proxy`와 firewall 규칙 로그가 보임 | SSH 세션과 로컬 라우팅 규칙이 생성됨 | 내부망 라우팅 구성 | 공격 호스트에서 내부 IP로 `nc` 확인 |
| 공격 호스트에서 내부 IP `nc -vz`가 성공함 | sshuttle TCP 경로가 동작함 | 내부 서비스 접근 | ProxyChains 없이 최종 클라이언트 실행 |
| RDP·웹·DB 클라이언트가 응답함 | 라우팅과 최종 서비스가 모두 동작하지만 인증은 아직 별도임 | 공격 호스트에서 내부 서비스 접근 | 서비스별 계정 인증, 세션과 실제 권한 확인 |
| sshuttle이 시작하지 못함 | sudo, iptables·nftables 또는 method 문제 | 라우팅 미생성 | 로컬 권한과 firewall backend 확인 |
| SSH 인증이 실패함 | 계정·키·SSH 경로 문제 | 첫 hop 미확보 | `ssh -v`로 인증 분리 |
| 모든 내부 TCP가 실패함 | CIDR 또는 피벗 baseline이 잘못됨 | 내부 경로 미확보 | 피벗 호스트 직접 `nc`와 route 재확인 |
| 기본 실행에서 UDP·ICMP 도구만 실패함 | TCP 경로 성공과 모순되지 않음. 일반 UDP는 선택된 method의 지원 여부에 따라 달라지고 ICMP는 범위 밖임 | TCP 라우팅만 확보 | UDP는 TPROXY 전제와 기존 route·policy rule 영향을 별도 검토하고, ICMP는 다른 피벗 기법 사용 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| SSH 첫 hop | SSH 인증 성공 | 피벗 계정과 연결 유지 가능 여부 확인 |
| 로컬 권한 | sshuttle firewall 규칙 생성 성공 | 공격 호스트의 sudo 권한 확인 |
| 내부 baseline | 피벗 호스트의 직접 `nc -vz` | 내부 대상과 CIDR이 유효한지 확인 |
| 라우팅 후 | 공격 호스트 `nc open`과 서비스 응답 | 라우팅 생성과 최종 접근을 분리 |

## 변경 영향과 복구

sshuttle은 공격 호스트에 임시 방화벽·라우팅 규칙을 만든다. 작업을 마치면 실행 중인 sshuttle을 정상 종료하고, 해당 프로세스가 만든 규칙이 정리됐는지 확인한다.

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 공격 호스트의 임시 firewall·routing 규칙 | `<INTERNAL_CIDR>` TCP 트래픽이 SSH 피벗으로 전달됨 | sshuttle verbose 로그와 내부 TCP 연결 확인 | sshuttle을 정상 종료한 뒤 route와 firewall 상태에서 이번 규칙이 남지 않았는지 확인 |

## 관련 상태 라우터

- 라우팅 뒤 내부 서비스에 도달했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[sshuttle]]
- [[ssh]]
- [[nmap]]
- [[xfreerdp]]

## 참고 링크

- [sshuttle 공식 Usage](https://sshuttle.readthedocs.io/en/stable/usage.html)
- [sshuttle 공식 Requirements](https://sshuttle.readthedocs.io/en/stable/requirements.html)
- [sshuttle 공식 TPROXY 안내](https://sshuttle.readthedocs.io/en/stable/tproxy.html)
