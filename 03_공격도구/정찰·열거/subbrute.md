---
tags:
  - 서비스/dns
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["루트 도메인", "서브도메인 wordlist", "resolver 목록"]
결과: ["서브도메인", "호스트명", "DNS 레코드"]
---

# subbrute

## 도구 개요

`subbrute`는 wordlist의 이름을 루트 도메인에 대입하고 지정한 resolver로 DNS 해석을 시도해 존재하는 서브도메인 후보를 찾는 도구다. 공개 자료 수집보다 직접적인 DNS brute force가 필요하거나 내부 resolver를 명시해야 하는 작업에 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: `subbrute.py`와 의존성을 실행할 수 있는 Linux 호스트
- 입력: 루트 도메인, 서브도메인 wordlist와 DNS resolver 목록

## 표준 사용법

```bash
./subbrute.py <DOMAIN> -s <NAMES_FILE> -r <RESOLVERS_FILE>
```

`<DOMAIN>`은 루트 도메인(예: `corp.example.test`)이고, `<NAMES_FILE>`은 Linux 실행 호스트의 줄바꿈 이름 목록, `<RESOLVERS_FILE>`은 같은 호스트의 resolver IP/FQDN 목록이다. `<DNS_HOST>`는 그 resolver 목록에 쓸 단일 서버 예시다. `-s`에는 서브도메인 후보 목록, `-r`에는 사용할 resolver 목록을 지정한다.

## 대표 예시

### 내부 resolver 지정

```bash
echo "<DNS_HOST>" > ./resolvers.txt
./subbrute.py <DOMAIN> -s ./names.txt -r ./resolvers.txt
```

`<DNS_HOST>`는 내부 resolver IP/FQDN의 가상 값(예: `dns.example.test`)이다.

확인할 출력:

- 존재하는 서브도메인 목록
- resolver 부족 경고가 나오면 resolver 목록을 보강한다.

### 발견 도메인 CNAME 확인

```bash
./subbrute.py <DOMAIN> -s ./names.txt -r ./resolvers.txt > subbrute-found.txt
while read sub; do host "$sub"; done < subbrute-found.txt
```

확인할 출력:

- CNAME, A 레코드, 내부/외부 서비스 연결 단서

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-s <FILE>` | 서브도메인 후보 wordlist | brute force 대상 이름 지정 |
| `-r <FILE>` | resolver 목록 | 내부 DNS나 특정 네임서버 사용 |
| `<DOMAIN>` | 대상 루트 도메인 | 기본 대상 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 서브도메인 출력 | DNS brute force 성공 | 포트 스캔, vhost, CNAME 확인 |
| resolver 부족 경고 | resolver 수가 적어 안정성 낮음 | resolver 목록 추가 |
| 결과 없음 | wordlist 부족 또는 DNS 제한 | wordlist 변경, `dnsenum`, `subfinder` 교차 확인 |
| wildcard 의심 | 없는 이름도 응답 | baseline 쿼리로 false positive 제거 |

## 관련 공격기법

- [[DNS 열거와 Zone Transfer]]
- [[DNS 서비스#외부 CNAME과 Subdomain Takeover 후보 확인|Subdomain Takeover 후보 확인]]

## 관련 서비스

- [[DNS 서비스]]

## 참고 링크

- [SubBrute upstream repository and usage](https://github.com/TheRook/subbrute)
