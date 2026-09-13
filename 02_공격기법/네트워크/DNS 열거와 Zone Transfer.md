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

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 도메인 후보 | 웹 인증서, SMTP 배너, WHOIS, `/etc/hosts` | `<DOMAIN>` 형태 확보 |
| DNS 서버 접근 | `dig @<DNS> <DOMAIN> SOA` | 응답 또는 timeout 여부 확인 |
| AXFR 가능성 | 권한 NS 대상 `dig axfr` | 전체 zone이 내려오면 영향 큼 |

## 실행

`<DNS>`는 질의할 DNS 서버 주소(예: `192.0.2.53`), `<DOMAIN>`은 확인한 도메인(예: `example.test`)이며 `<PUBLIC_IP>`는 해당 DNS 응답에서 얻은 공개 주소다. 아래 명령은 DNS 서버에 도달하는 명령 실행 호스트에서 실행하고, 같은 도메인·서버 값은 기본 질의와 AXFR·후속 확인에서 재사용한다.

1. 기본 레코드로 권한 NS, 메일, TXT, 서비스 단서를 확인한다.
2. 응답의 RCODE, `aa`·`ra`·`ad`와 answer section을 분리해 해석하고 필요한 RR type을 개별 질의한다.
3. 권한 NS마다 AXFR을 시도한다.
4. AXFR이 실패하면 wildcard 기준값을 먼저 확인한 뒤 wordlist 기반 서브도메인 열거와 PTR 조회로 대상을 넓힌다.
5. 발견한 호스트는 웹 vhost, 포트 스캔, 메일/AD 후보로 넘긴다.

### 명령과 확인할 출력

#### 방식 선택

| 현재 보유 정보 | 선택할 방식 | 성공 시 얻는 결과 | 실패 시 다음 확인 |
|---|---|---|---|
| 도메인과 질의할 DNS 서버를 알고 있음 | `dig` 기본 레코드 조회 | NS·SOA·MX·TXT 레코드 | DNS 서버 주소, TCP·UDP 53과 도메인 표기 확인 |
| 도메인과 권한 네임서버를 알고 있음 | `dig axfr` | 전송이 허용된 zone 레코드 집합 | AXFR 거부를 정상 응답으로 기록하고 호스트 후보 열거로 전환 |
| 특정 DNS 서버에 직접 질의하며 여러 레코드·호스트 후보를 자동 수집 | `dnsenum --dnsserver` | 지정 DNS가 반환한 레코드와 발견 호스트 후보 | wildcard 응답, DNS 서버 주소와 wordlist 범위를 확인 |
| 현재 resolver 또는 별도로 지정한 nameserver로 host discovery 수행 | `fierce` | 권한 NS·AXFR 결과와 해석되는 서브도메인 후보 | 실제 사용된 nameserver, wildcard와 wordlist 범위를 확인 |
| 공개 도메인의 외부 정보원에서 서브도메인 후보 수집 | `subfinder` | 수동 검증이 필요한 서브도메인 후보 | 내부 전용 zone에는 결과가 없을 수 있으므로 직접 DNS 열거 사용 |
| 현재 해석되는 public IP를 제3자 관측 자료와 대조 | `shodan host` | 과거 관측 시점의 port·service banner 후보 | API plan·quota와 관측 시각을 확인하고 현재 서비스는 직접 재검증 |
| wordlist와 resolver 목록을 보유 | `subbrute.py` | 직접 DNS로 해석된 이름 후보 | resolver 응답, wildcard와 wordlist 적합성 확인 |
| IP 주소를 알고 PTR 이름을 확인 | `dig -x` | 역방향 DNS가 반환한 호스트명 후보 | PTR 미등록을 호스트 부재로 해석하지 않음 |

#### 기본 레코드 확인

```bash
dig @<DNS> <DOMAIN> NS
dig @<DNS> <DOMAIN> SOA
dig @<DNS> <DOMAIN> MX
dig @<DNS> <DOMAIN> TXT
dig @<DNS> _<SERVICE>._<PROTO>.<DOMAIN> SRV
```

확인할 출력:

- 권한 네임서버, 메일 서버, SPF/DMARC, 검증 토큰, 서비스 위치와 내부 도메인명.
- `status: NOERROR`와 answer section을 함께 본다. `NOERROR`이면서 `ANSWER: 0`이면 서버 응답은 받았지만 요청한 RRset은 확인하지 못한 상태다.
- `aa`는 해당 응답의 권한 여부, `rd`·`ra`는 재귀 요청·지원 여부다. `ad`는 질의한 resolver가 DNSSEC 정책에 따라 응답을 검증했다는 표지이며 권한 서버 표지가 아니다.
- `TC`가 있거나 UDP 경로가 의심되면 같은 RR type에 `+tcp`를 붙여 다시 확인한다. `dig`의 종료 코드 0에는 `NXDOMAIN` 응답도 포함되므로 shell 종료 코드만으로 이름 존재를 판정하지 않는다.

#### `ANY`와 구현 정보의 경계

```bash
dig @<DNS> <DOMAIN> ANY
dig @<DNS> version.bind TXT CH
```

확인할 출력:

- `ANY` 응답은 단일 RRset이나 합성 HINFO처럼 최소 응답일 수 있다. 모든 레코드가 필요하면 A·AAAA·CNAME·MX·TXT·SRV를 개별 질의한다.
- `version.bind` 응답은 BIND가 노출한 구현 문자열 후보다. 빈 응답이나 거부는 BIND 부재를 뜻하지 않고, 반환 문자열도 실제 패치 수준을 단독으로 확정하지 않는다.

#### Zone Transfer

```bash
dig axfr <DOMAIN> @<DNS>
```

확인할 출력:

- `XFR size`, SOA 시작·종료와 함께 zone의 여러 A·CNAME·MX·SRV 레코드가 반환되면 AXFR 성공이다.
- `Transfer failed`, `REFUSED`, `NOTAUTH`는 AXFR 거부다. 일반 DNS 응답이나 서브도메인 발견을 Zone Transfer 성공으로 기록하지 않는다.
- AXFR은 TCP 기반이다. 첫 응답의 SOA와 마지막의 동일 SOA, 오류 없는 종료와 `XFR size`를 확인하기 전에는 중간 레코드 일부만으로 완전한 전송이라고 판정하지 않는다.

#### wildcard 기준값 확인

```bash
dig @<DNS> <RANDOM_LABEL>.<DOMAIN> A
dig @<DNS> <RANDOM_LABEL>.<DOMAIN> CNAME
```

확인할 출력:

- 존재하지 않도록 만든 고유 label도 실제 후보와 같은 주소·CNAME으로 응답하면 wildcard 기준값으로 기록한다.
- 이후 도구 결과가 기준값과 같으면 이름 존재를 바로 확정하지 않고 레코드 차이와 해당 Host의 실제 서비스를 추가 확인한다.

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
test ! -e <WORKDIR>/subdomains.txt
subfinder -d <DOMAIN> -o <WORKDIR>/subdomains.txt
wc -l <WORKDIR>/subdomains.txt
```

확인할 출력:

- 공개 source에서 수집한 서브도메인 후보와 새 결과 파일.
- 외부 정보원 후보는 현재 DNS 해석이나 서비스 도달성을 증명하지 않으므로 대상 DNS에서 다시 확인한다.

후보별 현재 A·AAAA 응답을 확인한다. 빈 줄과 wildcard는 제외하고, CNAME·공유 CDN 주소는 해당 조직 소유 IP로 확대하지 않는다.

```bash
while IFS= read -r hostname; do
  test -n "$hostname" || continue
  printf '\n[%s]\n' "$hostname"
  dig +short "$hostname" A
  dig +short "$hostname" AAAA
done < '<WORKDIR>/subdomains.txt'
```

#### 공개 관측 데이터에서 IP 서비스 후보 대조

현재 DNS에서 확인한 public `<PUBLIC_IP>`가 대상과 연결되고, Shodan CLI가 이미 별도 API profile로 구성되어 있으며 host lookup을 사용할 수 있을 때만 조회한다. API key 자체를 명령·Vault에 기록하지 않는다.

```bash
shodan info
shodan host <PUBLIC_IP>
```

확인할 출력:

- 계정 plan·사용 가능 기능 확인 결과와 `<PUBLIC_IP>`에 대해 Shodan이 마지막으로 관측한 시각, port·product·banner 후보.
- Shodan 결과는 제3자의 과거 관측 자료다. 현재 열려 있는 서비스나 대상 조직 소유를 확정하지 않으므로 DNS 관계·범위를 다시 확인하고 해당 포트의 현재 서비스 응답을 검증한다.
- 인증·membership·quota 오류는 대상 서비스 부재가 아니다. 이 분기는 건너뛰고 현재 DNS·직접 서비스 식별 경로를 유지한다.

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
| `_ldap._tcp`, `_kerberos._tcp`, `dc._msdcs` SRV 또는 DC 역할 hostname이 나온다. | AD 서비스 역할 후보이며 실제 DC·도메인 컨텍스트는 미확정 | AD 서비스 단서 | 88·389·445 도달성과 RootDSE를 확인하는 [[AD 도메인 컨텍스트 기본 확인]] |
| MX 또는 mail 역할 hostname이 나온다. | 메일 서버 주소 후보이며 SMTP 도달성과 사용자 후보는 미확정 | 메일 서비스 단서 | 주소와 포트를 확인한 뒤 [[SMTP 서비스]]에서 전송 계층·기능을 확인 |
| CNAME·TXT에서 cloud storage URL이 나온다. | 외부 서비스·저장소 후보이며 소유·익명 접근은 미확정 | 클라우드 서비스 단서 | [[공개 클라우드 스토리지 익명 접근 검증]]의 소유 관계·접근 조건 검토 |
| Shodan에 과거 port·banner가 표시된다. | 제3자 관측 기반 서비스 후보이며 현재 상태 미확정 | 공개 서비스 후보 | 관측 시각·DNS 소유 관계를 확인하고 현재 서비스 응답을 검증한 뒤 해당 서비스 문서로 분기 |
| AXFR 거부 | 정상적인 전송 제한 | 시작 상태 유지 | wordlist, 인증서 SAN, 웹 링크에서 호스트 후보 수집 |
| 응답 없음 | UDP/TCP 차단, 잘못된 DNS 서버 | 시작 상태 유지 | TCP 53, 다른 NS, `/etc/hosts` 후보 확인 |
| 결과 빈약 | 도메인 후보 오류 | 시작 상태 유지 | 웹 리다이렉션, SMTP 배너, 인증서 CN/SAN 재확인 |

## 변경 영향과 복구

기본 `dig`, `dnsenum`, `fierce`, `shodan host`와 PTR 조회는 조회만 수행한다. Shodan API 제공자에는 계정·질의 기록이 남고 plan별 사용 제한이 적용될 수 있으며, 이를 로컬 정리로 되돌릴 수 있다고 표현하지 않는다. `subfinder -o`와 shell redirection을 사용하면 로컬 결과 파일이 생성되므로 실행 전 경로를 확인하고 이번 작업에서 새로 만든 파일만 보존 정책에 따라 이동하거나 제거한다.

```bash
rm -- <WORKDIR>/subdomains.txt
test ! -e <WORKDIR>/subdomains.txt
```

- 기존 파일이 있으면 덮어쓰지 말고 새 고유 경로를 정한다.
- 결과를 보존해야 하면 작업 저장소로 옮긴 뒤 원본 경로와 보존 여부를 확인한다. 실제 대상명·출력은 Vault에 기록하지 않는다.
- 질의는 DNS cache와 서버 로그에 남을 수 있으며 이를 공격 호스트에서 원상복구할 수 있다고 표현하지 않는다.

## 확인할 출력과 권한

- 판정 기준: AXFR 레코드, 새 호스트명, NS·MX·TXT·PTR 응답을 확인하고 단순 DNS 응답과 대상 확장을 구분한다.
- 권한 구분: 인증 전·익명·유효 계정 상태를 구분하고, 서비스 응답만으로 실제 권한을 추정하지 않는다.

## 후속 공격 연결

- 새 웹 호스트: [[웹 정찰과 경로 열거]]
- AD SRV·DC 역할 hostname: [[AD 도메인 컨텍스트 기본 확인]]
- MX·mail 역할 hostname: [[SMTP 서비스]]
- SMTP endpoint와 사용자명·메일 주소 후보를 모두 확인함: [[SMTP 사용자 열거]]
- relay 허용 단서 또는 유효한 메일함 credential까지 확인함: [[SMTP 서비스#Open Relay 검증|SMTP Open Relay 검증]], [[IMAP POP3 메일함 수집]]
- AD 컨텍스트 확인 뒤 계정이 없고 SMB 무인증 응답이 있음: [[SMB 익명 열거와 공유 권한 확인]]
- 공개 cloud storage URL 후보: [[공개 클라우드 스토리지 익명 접근 검증]]

## 관련 서비스

- [[DNS 서비스]]

## 관련 도구

- [[dig]]
- [[dnsenum]]
- [[fierce]]
- [[subfinder]]
- [[shodan]]
- [[subbrute]]
- [[nmap]]

## 참고 링크

- [BIND 9 dig manual](https://bind9.readthedocs.io/en/stable/manpages.html#dig-dns-lookup-utility)
- [RFC 5936: DNS Zone Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5936)
- [RFC 8482: Minimal Responses for DNS ANY Queries](https://www.rfc-editor.org/rfc/rfc8482)
