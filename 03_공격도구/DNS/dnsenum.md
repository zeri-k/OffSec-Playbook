---
tags:
  - 서비스/dns
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 도메인"]
결과: ["정보"]
---

# dnsenum

## 도구 개요

`dnsenum`은 도메인의 기본 DNS 레코드와 권한 서버를 수집하고 AXFR, 서브도메인 대입, reverse lookup을 묶어 수행하는 열거 도구다. 여러 DNS 탐색 단계를 한 번에 실행하고 XML이나 서브도메인 목록으로 결과를 남길 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 DNS 서버에 접근 가능한 Linux 호스트
- 필요한 입력: 기준 도메인과 필요하면 nameserver, 하위 도메인 wordlist, 결과 파일
- 대상: DNS record, 권한 있는 nameserver, zone transfer와 서브도메인 후보


## 표준 사용법

```bash
dnsenum [options] <domain>
```

## 대표 예시

### 기본 DNS 열거

```bash
dnsenum example.com
```

### 특정 DNS 서버를 지정해 열거

```bash
dnsenum --dnsserver <TARGET> --enum example.com
```

### wordlist 기반 하위 도메인 탐색

```bash
dnsenum -f subdomains.txt example.com
```

### Google scraping 없이 결과 저장

```bash
dnsenum --dnsserver <TARGET> --enum -p 0 -s 0 -f subdomains.txt --subfile found-subdomains.txt -o dnsenum.xml example.com
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `--enum` | 기본 열거 기능 활성화 |
| `--dnsserver` | 사용할 DNS 서버 지정 |
| `-f` | 하위 도메인 wordlist 지정 |
| `-p <value>`, `--pages <value>` | Google scraping에서 처리할 검색 결과 페이지 수. `-s`와 함께 사용 |
| `-s <value>`, `--scrap <value>` | Google scraping으로 수집할 서브도메인 최대 개수. `-p 0 -s 0`은 Google scraping을 사실상 끄는 용도로 사용 |
| `-o <file>`, `--output <file>` | 결과를 XML 형식으로 저장. 순수 서브도메인 목록은 `--subfile` 사용 |
| `--threads` | 병렬 스레드 수 |
| `--noreverse` | reverse lookup 생략 |
| `--subfile` | 찾은 하위 도메인 저장 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| NS/MX/A/TXT와 하위 도메인 출력 | DNS 기반 대상 확장 성공 | 발견 호스트를 웹 vhost, 포트 스캔, 인증 지점 확인에 활용 |
| zone transfer 성공 | 네임스페이스 대량 노출 | 호스트명과 IP를 정리해 서비스별 우선순위 지정 |
| brute force 결과 없음 | wordlist 부족 또는 wildcard 영향 | wordlist 변경, wildcard 기준값, 다른 DNS 도구로 교차 확인 |
| query timeout/REFUSED | DNS 질의 제한 또는 서버 문제 | 네임서버 지정, TCP 질의, 요청 속도 조정 |

## 관련 공격기법

- [[DNS 열거와 Zone Transfer]]
- [[53_DNS#외부 CNAME과 Subdomain Takeover 후보 확인|Subdomain Takeover 후보 확인]]
