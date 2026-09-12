---
tags:
  - 서비스/http
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 URL 또는 도메인"]
결과: ["정보"]
---

# FinalRecon

## 도구 개요

`FinalRecon`은 HTTP 헤더, Whois, TLS, DNS, 서브도메인, 크롤링과 디렉터리 열거 모듈을 한 도구에서 선택해 실행하는 웹 정찰 자동화 도구다. 초기 공격면을 넓게 수집하거나 필요한 모듈만 묶어 비교할 때 유용하며, 각 모듈의 성공과 후보는 서로 독립적으로 검증해야 한다.

## 필요한 입력과 실행 환경

- 실행 위치: FinalRecon과 Python 의존성이 설치된 Linux 호스트
- 대상 URL: 스킴, 포트, 가상 호스트와 시작 경로를 포함한 HTTP(S) URL. IP 주소와 도메인이 다른 콘텐츠를 반환하면 의도한 vhost를 URL에 사용하고 DNS·Host·TLS SNI가 그 이름과 맞는지 먼저 확인한다.
- 필요한 파일·값: `finalrecon.py`, `requirements.txt`로 설치한 의존성, 실행할 정찰 모듈과 필요하면 출력 경로
- 필요한 인증 정보: 아래 명령은 익명 응답만 수집한다. 로그인 뒤 페이지, 역할별 기능과 보호된 디렉터리는 이 결과만으로 부재를 판단할 수 없다.
- 대상에서 필요한 서비스와 권한: `--headers`, `--sslinfo`, `--crawl`, `--dir`은 대상 HTTP(S) 서비스에 도달해야 한다. `--whois`, `--dns`, `--sub`, `--wayback`은 외부 DNS·등록정보·공개 데이터 소스에도 연결할 수 있어야 한다.
- 모듈 경계: 웹 연결 성공과 Whois·DNS·서브도메인 조회 성공은 서로 독립적이다. 한 모듈의 성공을 전체 정찰 성공으로 해석하지 않는다.


## 표준 사용법

```bash
./finalrecon.py [options] --url <url>
```

먼저 필요한 의존성을 설치한 뒤 실행한다.

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
```

## 대표 예시


### 헤더와 Whois 정보 수집

```bash
./finalrecon.py --headers --whois --url http://<DOMAIN>
```

웹 서버 헤더와 도메인 등록 정보를 함께 확인한다.

확인할 출력:

- 헤더 결과에서 HTTP status와 `Server` 등 응답 헤더가, Whois 결과에서 등록정보 필드가 각각 출력되는지 확인한다.
- HTTP 응답은 지정 URL의 도달성만, Whois 결과는 조회 시점의 등록정보만 확정한다. 애플리케이션 인증 성공이나 현재 서비스 소유권은 확정하지 않는다.

### SSL 인증서 정보 확인

```bash
./finalrecon.py --sslinfo --url https://<DOMAIN>
```

인증서 발급자, 유효 기간, TLS 관련 단서를 확인한다.

확인할 출력:

- SSL 인증서 정보 섹션에서 subject·issuer·유효 기간과 이름 단서를 확인한다.
- 인증서의 CN·SAN은 vhost 후보일 수 있지만 해당 이름에서 HTTP 서비스가 실제 응답하거나 취약하다는 뜻은 아니다.

### 웹 크롤링

```bash
./finalrecon.py --crawl --url http://<DOMAIN>
```

내부/외부 링크, 리소스, JavaScript 파일 등 웹 구조 파악에 필요한 정보를 수집한다.

확인할 출력:

- Crawler 결과의 내부·외부 링크, JavaScript, `robots.txt`, `sitemap.xml`과 리소스 URL을 확인한다.
- 수집 URL은 HTML·응답에서 관찰한 후보다. 현재 접근 가능성, 인증 필요 여부, 파일 내용과 취약성은 개별 요청으로 확인한다.

### 서브도메인 열거

```bash
./finalrecon.py --sub --url http://<DOMAIN>
```

다양한 공개 데이터 소스를 활용해 서브도메인 후보를 찾는다.

확인할 출력:

- Subdomain Enumeration 결과에 호스트 이름 후보가 표시되는지 확인한다.
- 공개 데이터 소스의 이름 발견만 확정하며, 현재 DNS 해석·HTTP 도달성·평가 범위 포함 여부는 별도로 확인한다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
| --- | --- | --- |
| `--url <URL>` | 스킴과 vhost를 포함한 대상 URL 지정 | 모듈이 요청·조회할 기준 대상을 정할 때 |
| `--headers` | 대상 URL의 HTTP 응답 헤더 수집 | status, redirect, 서버·프레임워크 헤더 단서를 볼 때 |
| `--sslinfo` | 지정 HTTPS 대상의 SSL/TLS 인증서 정보 수집 | issuer, 유효 기간과 CN·SAN 단서를 볼 때 |
| `--whois` | URL에서 얻은 도메인의 Whois 조회 | 등록정보 단서를 별도로 확인할 때 |
| `--crawl` | 기준 URL에서 웹사이트 크롤링 | 링크·리소스·JavaScript 후보를 모을 때 |
| `--dns` | 대상 도메인의 DNS 레코드 열거 | 현재 DNS 이름과 메일·인증 관련 레코드 단서를 볼 때 |
| `--sub` | 공개 데이터 소스에서 서브도메인 열거 | 추가 호스트 이름 후보를 수집할 때 |
| `--dir` | 대상 URL에서 디렉터리 열거 | 숨은 경로 후보를 찾고 수동 검증할 때 |
| `--wayback` | Wayback Machine URL 수집 | 과거 URL 후보를 현재 서비스와 비교할 때 |
| `--ps` | 대상 시스템에 빠른 포트 스캔 수행 | 웹 외 노출 포트 후보를 좁힐 때 |
| `--full` | 지원 정찰 모듈을 넓게 실행 | 모듈별 성공·실패를 분리해 검토할 수 있을 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 확정 범위와 다음 확인 |
|---|---|---|
| Header Information에 status·응답 헤더가 표시됨 | 지정 URL에서 HTTP 응답 수신 | 해당 URL의 현재 응답만 확정한다. 올바른 vhost·인증 상태·backend 제품은 redirect와 본문을 교차 확인한다. |
| Whois Lookup, SSL Certificate Information 또는 DNS Enumeration 섹션에 필드가 표시됨 | 해당 외부 조회 또는 TLS handshake 성공 | 등록정보·인증서·DNS 관찰값만 확정한다. 애플리케이션 소유권, HTTP 접근과 취약성은 확정하지 않는다. |
| Subdomain Enumeration에 이름이 표시됨 | 공개 데이터 소스에서 호스트 이름 후보 발견 | DNS 해석, 대상 서비스 포트, HTTP 응답과 범위 포함 여부를 별도로 확인한다. |
| Crawler·Directory·Wayback 결과에 URL이나 파일이 표시됨 | 링크·과거 URL·경로 후보 확보 | `gobuster`, `curl`, 브라우저로 현재 접근성과 응답 본문을 확인한다. 후보 출력만으로 경로 실재나 민감 정보 노출을 확정하지 않는다. |
| 특정 모듈 결과가 비어 있음 | 해당 모듈이 현재 조건에서 단서를 반환하지 못함 | 정보 부재로 단정하지 말고 인증, vhost, redirect, API source와 모듈별 도달성을 확인한다. |
| 연결·TLS·DNS·rate limit·proxy 오류가 반복됨 | 해당 모듈 요청 실패 | 전체 대상 부재로 묶지 말고 `curl`, `dig`, `whatweb`로 웹·DNS·외부 source의 기준 응답을 각각 확인한다. |

## 관련 공격기법

- [[웹 정찰과 경로 열거]]
- [[웹 지문 확인과 공격면 분류]]
- [[웹 단서 기반 기능 열거]]
- [[DNS 열거와 Zone Transfer]]

## 참고 링크

- [FinalRecon GitHub](https://github.com/thewhiteh4t/FinalRecon)
