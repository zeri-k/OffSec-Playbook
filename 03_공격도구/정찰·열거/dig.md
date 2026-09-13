---
tags:
  - 서비스/dns
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["조회할 도메인, IP 또는 DNS 서버"]
결과: ["정보"]
---

# dig

## 도구 개요

`dig`는 질의할 DNS 서버와 record type을 지정해 응답 내용을 직접 확인하는 범용 DNS 클라이언트다. 개별 A·MX·NS·TXT 조회, reverse lookup, delegation 추적, AXFR 수락 여부처럼 한 가지 DNS 동작을 정확히 검증할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: 조회할 DNS 서버에 접근 가능한 Linux 호스트
- 필요한 입력: 도메인, record type, 필요하면 질의할 DNS 서버
- zone transfer 입력: 권한 있는 네임서버와 `AXFR`을 요청할 zone 이름


## 표준 사용법

```bash
dig [@dns_server] <name> <record_type> [options]
```

## 대표 예시

### A 레코드 조회

```bash
dig example.com A
```

### zone transfer 가능성 확인

`@<DNS_SERVER>`은 질의할 DNS 서버 IP/FQDN(예: `dns.example.test`)이고, `-x <IP_TO_REVERSE>`는 역조회할 IP(예: `192.0.2.10`)다. 두 역할을 같은 값으로 가정하지 않는다.

```bash
dig @<DNS_SERVER> example.com AXFR
```

### reverse lookup

```bash
dig -x <IP_TO_REVERSE> +short
```

### 네임서버만 간단히 확인

```bash
dig example.com NS +short
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `@server` | 질의할 DNS 서버 지정 |
| `A`, `AAAA`, `MX`, `NS`, `TXT`, `ANY` | 조회할 레코드 타입 |
| `AXFR` | zone transfer 요청 |
| `-x` | reverse lookup |
| `+short` | 간단한 출력 |
| `+trace` | DNS delegation 추적 |
| `+noall +answer` | answer section만 출력 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| A/AAAA/MX/TXT/NS 레코드 출력 | DNS 레코드 확인 성공 | 메일 서버, 네임서버, TXT 정보, 하위 도메인 단서 정리 |
| `NOERROR`, `ANSWER: 0` | 서버 응답은 받았지만 요청한 RRset은 미확인 | authority section, 질의 이름·type과 권한 서버 확인 |
| `NXDOMAIN` | 질의 이름이 존재하지 않는다는 DNS 응답 | shell 종료 코드 0과 이름 존재를 구분하고 다른 권한 NS로 교차 확인 |
| 첫·마지막 SOA와 `XFR size`가 있는 `AXFR` | zone transfer 완료 | 전체 호스트명을 포트 스캔/웹 vhost 확인에 활용 |
| `REFUSED` / `SERVFAIL` | 질의 거부 또는 서버 오류 | 다른 네임서버, 재귀 여부, 레코드 타입 변경 확인 |
| timeout/no servers could be reached | DNS 접근 불가 또는 UDP/TCP 제한 | TCP 질의, 포트 상태, resolver 지정 확인 |

`ANY`는 모든 RRset을 보장하지 않는다. 필요한 A·AAAA·CNAME·MX·TXT·SRV를 개별 질의하고, `aa`·`ra`·`ad`의 의미를 권한 응답·재귀 지원·resolver DNSSEC 검증으로 구분한다.

## 관련 공격기법

- [[DNS 열거와 Zone Transfer]]
- [[DNS 서비스#외부 CNAME과 Subdomain Takeover 후보 확인|Subdomain Takeover 후보 확인]]

## 참고 링크

- [BIND 9 dig manual](https://bind9.readthedocs.io/en/stable/manpages.html#dig-dns-lookup-utility)
- [RFC 5936: DNS Zone Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5936)
- [RFC 8482: Minimal Responses for DNS ANY Queries](https://www.rfc-editor.org/rfc/rfc8482)
