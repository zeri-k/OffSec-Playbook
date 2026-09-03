---
tags:
  - 서비스/dns
대표포트:
  - "T:53"
  - "U:53"
서비스:
  - DNS
---

# 53_DNS

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Domain Name System(DNS) UDP 또는 TCP 53에 질의할 수 있고, 계정 권한 없이도 확인 가능한 `<DOMAIN>` 후보가 있는 상태에서 시작한다. 권한 서버 여부와 도메인을 확정한 뒤 Name Server(NS)·Start of Authority(SOA)·메일·텍스트·서비스·역방향 레코드, zone transfer(AXFR), 서브도메인 순으로 대상명을 확장한다.

**첫 화면 상태:** 지금 가능한 일은 이름·레코드·zone 공개 범위를 확인하는 것이다. 성공하면 추가 호스트·IP·서비스 역할 후보를 얻는다. `NXDOMAIN`은 이름 부재를, `REFUSED`는 질의·재귀·AXFR 정책을, `SERVFAIL`은 권한 서버·DNSSEC·후단 오류를, timeout은 UDP/TCP·source 제한을 나눠 확인한다. 레코드 하나만으로 새 호스트의 도달성이나 takeover 영향은 확정되지 않는다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | 권한 서버와 주요 레코드 | `dig @<TARGET> <DOMAIN> NS`, `dig @<TARGET> <DOMAIN> SOA`, `dig @<TARGET> <DOMAIN> MX`, `dig @<TARGET> <DOMAIN> TXT` | 네임서버, 관리 정보, 메일 서버, 검증 토큰과 서비스 단서를 정리한다. |
| 2 | Zone Transfer | `dig axfr <DOMAIN> @<TARGET>` | AXFR 성공 시 전체 zone의 호스트명·IP·서비스명을 수집한다. |
| 3 | 레코드 타입별 응답 | `dig @<TARGET> <DOMAIN> ANY` | 응답이 제한되면 A, AAAA, CNAME, MX, TXT, SRV를 개별 질의한다. |
| 4 | 서브도메인 | `dnsenum --dnsserver <TARGET> --enum <DOMAIN>`, `subfinder -d <DOMAIN>`, `subbrute` | 새 웹·관리·개발 호스트와 vhost 후보를 확인한다. |
| 5 | TCP와 UDP 응답 차이 | `dig`로 같은 질의를 UDP와 TCP에서 비교 | UDP 제한, 큰 응답의 TCP 전환, 권한 서버별 정책 차이를 구분한다. |

**출력 해석 경계:** NS/SOA 응답은 응답 서버가 제시한 위임·관리 정보를 확정하지만, 대상 서버가 해당 zone의 권한 서버임을 단독으로 보장하지 않는다. AXFR의 완전한 전송 결과만 해당 서버가 공개한 zone 내용을 확정한다. `ANY` 실패, timeout, recursion 응답은 각 질의 유형·전송 경로의 결과일 뿐 레코드 부재나 취약점 부재를 확정하지 않는다.

### 외부 CNAME과 Subdomain Takeover 후보 확인

외부 SaaS·클라우드·호스팅 서비스로 이어지는 CNAME을 발견하면 실제 리소스를 등록하지 않고 DNS chain과 공급자별 미점유 신호까지만 확인한다.

```bash
dig +noall +answer <SUBDOMAIN> CNAME
host <SUBDOMAIN>
curl -k -sS -D headers.txt -o body.txt https://<SUBDOMAIN>/
openssl s_client -connect <SUBDOMAIN>:443 -servername <SUBDOMAIN> </dev/null
```

확인할 출력:

- 전체 CNAME chain과 최종 DNS 상태.
- HTTP status·header·body의 공급자 고유 오류와 SNI 인증서.
- CNAME target의 `NXDOMAIN`만으로 takeover라고 판단하지 않는다. 공급자의 현재 custom domain 조건과 미점유 응답이 함께 일치해야 후보로 기록한다.
- 정상 콘텐츠, 잘못된 region, 인증서 오류와 일시 장애를 미점유 상태와 구분한다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| NS/SOA/MX/TXT/SRV, PTR 응답 또는 recursion 허용 | [[DNS 열거와 Zone Transfer]] | `dig` | 도메인·호스트 역할과 추가 서비스 후보 |
| AXFR 성공 | [[DNS 열거와 Zone Transfer]] | `dig`, `dnsenum`, `fierce` | 전체 zone의 내부 호스트명·IP·서비스명 |
| 서브도메인 후보 필요 또는 새 이름 발견 | [[DNS 열거와 Zone Transfer]] | `dnsenum`, `subfinder`, `subbrute` | 웹·관리·개발 호스트 후보 |
| CNAME이 S3, GitHub Pages, CDN 등 외부 서비스로 연결 | 이 문서의 외부 CNAME과 Subdomain Takeover 후보 확인 | `dig`, `curl`, `openssl` | 외부 CNAME과 공급자별 미점유 신호가 일치한 후보 |
| 피해자와 게이트웨이 사이 L2 MITM 가능 | [[Ettercap L2 MITM DNS Spoofing]] | `ettercap` | 관찰한 DNS 질의의 응답 조작과 HTTP 트래픽 유도 가능성 |

## 서비스 고유 주의 사항

- `ANY`와 AXFR 실패는 정상 정책일 수 있으며 레코드 타입별 조회 실패를 뜻하지 않는다.
- UDP 53이 제한되면 TCP 53도 확인한다.
- 도메인 이름이 없으면 인증서, 웹 리다이렉션, SMB 배너에서 후보를 먼저 모은다.
- CNAME 외부 연결은 미점유 오류와 리소스 등록 가능성을 모두 확인해야 Subdomain Takeover 후보가 된다.
