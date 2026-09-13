---
tags:
  - 서비스/dns
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["루트 도메인", "일부 source 사용 시 API key"]
결과: ["서브도메인", "수집 출처"]
---

# subfinder

## 도구 개요

`subfinder`는 여러 공개 데이터 소스에서 대상 도메인의 서브도메인 후보를 수집하는 수동형 정찰 도구다. DNS 이름을 직접 대입하기 전에 공개 노출을 빠르게 모으는 데 유용하며, 출력은 현재 DNS 해석이나 소유 상태가 확인된 목록이 아니다.

## 필요한 입력과 실행 환경

- 실행 환경: `subfinder`가 설치된 Linux 호스트
- 입력: 루트 도메인. `<DOMAIN>`은 가상 루트 FQDN(예: `example.test`)이며 결과 파일은 Linux 실행 호스트에 만든다.
- 선택 입력: 일부 공개 source에 필요한 API key

## 표준 사용법

```bash
subfinder -d <DOMAIN>
```

결과는 서브도메인 목록으로 나오며, 이후 `dig`, `host`, `curl`로 실제 DNS/HTTP 상태를 확인한다.

## 대표 예시

### 기본 서브도메인 수집

```bash
subfinder -d <DOMAIN> -v
```

확인할 출력:

- `[INF] Enumerating subdomains`
- 출처별 서브도메인
- `[INF] Found <N> subdomains`

### 결과 저장

```bash
subfinder -d <DOMAIN> -o subdomains.txt
```

확인할 출력:

- `subdomains.txt`에 후보 도메인이 저장된다.

### Subdomain Takeover 후보 확인으로 연결

```bash
while read sub; do dig "$sub" CNAME +short; done < subdomains.txt
```

확인할 출력:

- S3, GitHub Pages, Azure, CDN 등 외부 서비스 CNAME

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-d <DOMAIN>` | 대상 도메인 지정 | 기본 실행 |
| `-v` | 상세 출력 | 어떤 소스에서 나왔는지 확인 |
| `-o <FILE>` | 결과 저장 | 후속 `dig`, `httpx`, `curl` 처리 |
| `-silent` | 도메인만 출력 | 파이프라인에 연결 |
| `-all` | 가능한 모든 source 사용 | 빠른 결과보다 포괄성이 중요할 때 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 여러 서브도메인 출력 | 대상 확장 성공 | CNAME, HTTP 상태, vhost 확인 |
| 외부 서비스 CNAME 확인 | takeover 후보 가능성 | [[DNS 서비스#외부 CNAME과 Subdomain Takeover 후보 확인\|Subdomain Takeover 후보 확인]] 조건 확인 |
| 결과가 적음 | 공개 소스에 노출 부족 | `dnsenum`, `subbrute`, 인증서 SAN, 웹 링크 확인 |
| wildcard 의심 | 도메인 전체가 같은 IP로 응답 | baseline 도메인으로 false positive 제거 |

## 관련 공격기법

- [[DNS 서비스#외부 CNAME과 Subdomain Takeover 후보 확인|Subdomain Takeover 후보 확인]]
- [[DNS 열거와 Zone Transfer]]
- [[웹 정찰과 경로 열거]]

## 관련 서비스

- [[DNS 서비스]]
- [[HTTP와 HTTPS 서비스]]

## 참고 링크

- [ProjectDiscovery Subfinder documentation](https://docs.projectdiscovery.io/opensource/subfinder/overview)
- [Subfinder upstream repository and CLI options](https://github.com/projectdiscovery/subfinder)
