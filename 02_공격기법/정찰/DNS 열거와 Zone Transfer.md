---
tags:
  - 서비스/dns
시작조건: ["DNS 서비스 식별", "도메인 후보 확보"]
필요조건: ["도메인 후보", "DNS 질의 가능"]
결과: ["정보", "대상 확장"]
---

# DNS 열거와 Zone Transfer

## 한 줄 판단

현재 명령 실행 위치에서 대상 도메인의 DNS 서버에 질의할 수 있다면, NS·SOA·MX·TXT·PTR 레코드와 허용된 경우 AXFR Zone Transfer를 확인해 가상 호스트 이름, 메일 서버, 내부 호스트명과 추가 연결 대상 후보를 얻는다.

## 사용할 때

- 53/TCP 또는 53/UDP가 열려 있고 도메인 이름을 알고 있을 때.
- 웹 인증서, 메일 주소, SMB 도메인명, 배너에서 도메인 후보가 나왔을 때.
- 포트 스캔 대상이 IP 하나뿐인데 호스트명 기반 서비스가 의심될 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 도메인 후보 | 웹 인증서, SMTP 배너, WHOIS, `/etc/hosts` | `<DOMAIN>` 형태 확보 |
| DNS 서버 접근 | `dig @<DNS> <DOMAIN> SOA` | 응답 또는 timeout 여부 확인 |
| AXFR 가능성 | 권한 NS 대상 `dig axfr` | 전체 zone이 내려오면 영향 큼 |

## 실행

1. 기본 레코드로 권한 NS, 메일, TXT, 서비스 단서를 확인한다.
2. 권한 NS마다 AXFR을 시도한다.
3. AXFR이 실패하면 wordlist 기반 서브도메인 열거와 PTR 조회로 대상을 넓힌다.
4. 발견한 호스트는 웹 vhost, 포트 스캔, 메일/AD 후보로 넘긴다.

### 명령과 확인할 출력

#### 방식 선택

| 현재 보유 정보 | 선택할 방식 | 성공 시 얻는 결과 | 실패 시 다음 확인 |
|---|---|---|---|
| 도메인과 질의할 DNS 서버를 알고 있음 | `dig` 기본 레코드 조회 | NS·SOA·MX·TXT 레코드 | DNS 서버 주소, TCP·UDP 53과 도메인 표기 확인 |
| 도메인과 권한 네임서버를 알고 있음 | `dig axfr` | 전송이 허용된 zone 레코드 집합 | AXFR 거부를 정상 응답으로 기록하고 호스트 후보 열거로 전환 |
| 특정 DNS 서버에 직접 질의하며 여러 레코드·호스트 후보를 자동 수집 | `dnsenum --dnsserver` | 지정 DNS가 반환한 레코드와 발견 호스트 후보 | wildcard 응답, DNS 서버 주소와 wordlist 범위를 확인 |
| 현재 resolver 또는 별도로 지정한 nameserver로 host discovery 수행 | `fierce` | 권한 NS·AXFR 결과와 해석되는 서브도메인 후보 | 실제 사용된 nameserver, wildcard와 wordlist 범위를 확인 |
| 공개 도메인의 외부 정보원에서 서브도메인 후보 수집 | `subfinder` | 수동 검증이 필요한 서브도메인 후보 | 내부 전용 zone에는 결과가 없을 수 있으므로 직접 DNS 열거 사용 |
| wordlist와 resolver 목록을 보유 | `subbrute.py` | 직접 DNS로 해석된 이름 후보 | resolver 응답, wildcard와 wordlist 적합성 확인 |
| IP 주소를 알고 PTR 이름을 확인 | `dig -x` | 역방향 DNS가 반환한 호스트명 후보 | PTR 미등록을 호스트 부재로 해석하지 않음 |

#### 기본 레코드 확인

```bash
dig @<DNS> <DOMAIN> NS
dig @<DNS> <DOMAIN> SOA
dig @<DNS> <DOMAIN> MX
dig @<DNS> <DOMAIN> TXT
```

확인할 출력:

- 권한 네임서버, 메일 서버, SPF/DMARC, 검증 토큰, 내부 도메인명.

#### Zone Transfer

```bash
dig axfr <DOMAIN> @<DNS>
```

확인할 출력:

- `XFR size`, SOA 시작·종료와 함께 zone의 여러 A·CNAME·MX·SRV 레코드가 반환되면 AXFR 성공이다.
- `Transfer failed`, `REFUSED`, `NOTAUTH`는 AXFR 거부다. 일반 DNS 응답이나 서브도메인 발견을 Zone Transfer 성공으로 기록하지 않는다.

#### 특정 DNS 서버로 자동 레코드 열거

```bash
dnsenum --dnsserver <DNS> --enum <DOMAIN>
```

확인할 출력:

- 지정한 DNS가 반환한 NS·MX·A 레코드, AXFR 시도 결과와 서브도메인 후보.
- 호스트를 찾았더라도 출력에서 AXFR 성공을 명시적으로 확인하지 않았다면 서브도메인 열거 결과로 기록한다.

#### resolver 기반 host discovery

```bash
fierce --domain <DOMAIN>
```

확인할 출력:

- 실제 질의에 사용된 nameserver, 권한 NS·AXFR 상태와 해석된 호스트명·주소 후보.
- 결과가 없으면 현재 resolver, 필요 시 nameserver 지정 옵션, wildcard와 wordlist 범위를 확인한다.

#### 공개 정보원에서 서브도메인 후보 수집

```bash
subfinder -d <DOMAIN> -o subdomains.txt
```

확인할 출력:

- 공개 source에서 수집한 서브도메인 후보와 결과 파일.
- 외부 정보원 후보는 현재 DNS 해석이나 서비스 도달성을 증명하지 않으므로 대상 DNS에서 다시 확인한다.

#### wordlist와 resolver로 직접 이름 확인

```bash
./subbrute.py <DOMAIN> -s ./names.txt -r ./resolvers.txt
```

확인할 출력:

- 지정 resolver가 실제로 해석한 호스트명과 주소 후보.
- 결과가 없으면 resolver 응답, wildcard 기준값, names wordlist와 내부·외부 zone 구분을 확인한다.

#### 역방향 조회

```bash
dig @<DNS> -x <IP>
```

확인할 출력:

- PTR 응답의 FQDN과 `dev`, `admin`, `vpn`, `dc`, `mail`, `intranet` 같은 역할성 호스트명.
- `NXDOMAIN`이나 빈 PTR은 해당 주소에 호스트가 없다는 증거가 아니다. 정방향 레코드와 포트 도달성을 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| AXFR로 zone 파일 전체 또는 다수의 내부 호스트명이 나온다. | zone transfer 허용 확인 | DNS 레코드 집합 | 레코드별 주소와 포트를 확인해 해당 서비스 문서로 분기 |
| 서브도메인/역방향 조회로 새 웹 vhost나 내부 서비스가 확인된다. | 추가 호스트 식별 | 서비스 후보 | 웹 호스트는 [[웹 정찰과 경로 열거]], 나머지는 식별된 서비스 기법으로 전환 |
| TXT/SRV/MX에서 메일, AD 또는 클라우드 단서가 나온다. | 서비스 역할 식별 | 도메인·서비스 단서 | 메일은 [[SMTP 사용자 열거]], SMB는 [[SMB 익명 열거와 공유 권한 확인]] 조건 검토 |
| AXFR 거부 | 정상적인 전송 제한 | 시작 상태 유지 | wordlist, 인증서 SAN, 웹 링크에서 호스트 후보 수집 |
| 응답 없음 | UDP/TCP 차단, 잘못된 DNS 서버 | 시작 상태 유지 | TCP 53, 다른 NS, `/etc/hosts` 후보 확인 |
| 결과 빈약 | 도메인 후보 오류 | 시작 상태 유지 | 웹 리다이렉션, SMTP 배너, 인증서 CN/SAN 재확인 |

## 확인할 출력과 권한

- 판정 기준: AXFR 레코드, 새 호스트명, NS·MX·TXT·PTR 응답을 확인하고 단순 DNS 응답과 대상 확장을 구분한다.
- 권한 구분: 인증 전·익명·유효 계정 상태를 구분하고, 서비스 응답만으로 실제 권한을 추정하지 않는다.

## 후속 공격 연결

- 새 웹 호스트: [[웹 정찰과 경로 열거]]
- 메일 서버: [[SMTP 사용자 열거]], [[25_SMTP#Open Relay 검증|SMTP Open Relay 검증]], [[IMAP POP3 메일함 수집]]
- AD 도메인 단서: [[원격 비밀번호 공격]], [[SMB 익명 열거와 공유 권한 확인]]

## 관련 서비스

- [[53_DNS]]

## 관련 도구

- [[dig]]
- [[dnsenum]]
- [[fierce]]
- [[subfinder]]
- [[subbrute]]
- [[nmap]]
