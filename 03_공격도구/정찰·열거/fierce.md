---
tags:
  - 서비스/dns
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 도메인"]
결과: ["정보", "도메인", "호스트명"]
---

# fierce

## 도구 개요

`fierce`는 도메인의 NS·SOA, AXFR 허용 여부, wildcard DNS와 서브도메인·인접 IP 후보를 빠르게 훑는 DNS 정찰 도구다. 역할성 호스트명과 네트워크 범위 단서를 넓게 찾는 초기 도메인 탐색에 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 DNS 서버에 질의할 수 있는 Linux 호스트
- 필요한 입력: 기준 도메인과 필요하면 지정 nameserver, 서브도메인 wordlist
- 대상: NS/SOA, zone transfer, wildcard DNS와 서브도메인/IP 후보


## 표준 사용법

```bash
fierce --domain <DOMAIN>
```

도메인 이름을 기준으로 DNS 정보를 수집하고, zone transfer가 가능하면 레코드를 대량으로 보여준다.

## 대표 예시

### 기본 도메인 열거

```bash
fierce --domain zonetransfer.me
```

확인할 출력:

- `NS`, `SOA`
- `Zone: success`
- A, MX, TXT, PTR, SRV 등 레코드 출력

### 내부 도메인 후보 확인

```bash
fierce --domain <DOMAIN>
```

확인할 출력:

- `internal`, `dc`, `mail`, `vpn`, `dev` 같은 역할성 호스트명

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `--domain <DOMAIN>` | 대상 도메인 지정 | 기본 실행 |
| `--dns-servers <DNS>` | 질의할 DNS 서버 지정 | 특정 네임서버를 직접 확인 |
| `--subdomains <FILE>` | 서브도메인 wordlist 지정 | 기본 목록이 부족할 때 |
| `--wide` | 더 넓은 IP 범위 확인 | 인접 대역 단서가 필요할 때 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Zone: success` | zone transfer 성공 | 전체 레코드를 저장하고 서비스별 재스캔 |
| NS/SOA 출력 | 권한 네임서버 확인 | 네임서버별 AXFR 교차 확인 |
| 다수의 호스트명 | 대상 확장 가능 | 웹 vhost, 포트 스캔, 메일/AD 단서 확인 |
| 결과가 거의 없음 | zone transfer 제한 또는 wordlist 부족 | `dig`, `dnsenum`, `subfinder`, `subbrute`로 교차 확인 |

## 관련 공격기법

- [[DNS 열거와 Zone Transfer]]

## 참고 링크

- [Fierce upstream repository and usage](https://github.com/mschwager/fierce)
