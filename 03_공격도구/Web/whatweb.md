---
tags:
  - 서비스/http
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 HTTP 또는 HTTPS URL"]
결과: ["웹 기술 fingerprint", "버전 단서"]
---

# whatweb

## 도구 개요

`whatweb`은 HTTP(S) 응답의 status·헤더·본문을 plugin signature와 비교해 CMS·프레임워크·서버·플러그인과 버전 후보를 식별하는 웹 지문 도구다. 웹 기술 스택을 빠르게 분류하고 후속 수동 검증 대상을 좁힐 때 적합하며, signature 일치만으로 실제 설치 버전이나 취약성을 확정하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치와 도달성: `whatweb`이 설치되어 있고 대상 HTTP(S) 포트까지 연결할 수 있는 Linux 호스트
- 대상 URL: 스킴, 포트, 가상 호스트와 기준 경로를 포함한 URL. IP와 domain이 다른 콘텐츠를 반환하면 의도한 vhost URL을 사용하고 DNS·Host·TLS SNI를 맞춘다.
- 필요한 인증 정보: 아래 예시는 익명 응답을 지문화한다. `401`·`403`·로그인 페이지를 스캔하면 보호된 애플리케이션이 아니라 그 오류·인증 응답의 기술만 식별될 수 있다.
- 선택 입력: 탐지 강도, proxy와 출력 파일
- 대상에서 필요한 서비스와 권한: HTTP 응답을 읽을 수 있어야 한다. 연결 성공과 status 수신은 애플리케이션 인증 성공이나 기능 접근 권한을 의미하지 않는다.


## 표준 사용법

```bash
whatweb [options] <url_or_host>
```

## 대표 예시

### 웹 기술 스택 빠른 식별

```bash
whatweb http://<TARGET>
```

확인할 출력:

- `<URL> [200 OK]` 같은 status와 `Apache[...]`, `HTTPServer[...]`, `Title[...]`, CMS·framework plugin 토큰을 확인한다.
- status는 해당 URL 응답을, plugin 토큰은 signature 일치를 뜻한다. 실제 backend 구성·설치 버전과 취약성은 확정하지 않는다.

### 더 적극적인 탐지와 상세 출력

```bash
whatweb -a 3 -v https://example.com
```

확인할 출력:

- 더 많은 요청과 상세 plugin 근거가 출력되는지 확인한다. 탐지 강도를 높여도 인증 뒤 콘텐츠가 자동으로 열리거나 지문이 확정 사실로 바뀌지는 않는다.

### 결과를 JSON으로 저장

```bash
whatweb --log-json whatweb.json http://<TARGET>
```

확인할 출력:

- `whatweb.json`이 생성되고 URL, status와 plugin 결과가 기록되는지 확인한다. 로그 생성만으로 모든 요청 성공이나 version 검증을 확정하지 않는다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
| --- | --- | --- |
| `-a` | 탐지 강도 조정. 값이 높을수록 적극적 | 기본 응답만으로 plugin 근거가 부족하고 추가 요청이 허용될 때 |
| `-v` | plugin 탐지 근거와 요청 정보를 상세 출력 | 제품·버전 후보가 어떤 응답에서 나왔는지 검토할 때 |
| `--log-json` | 결과를 JSON 로그로 저장 | URL·status·plugin 결과를 구조화해 후속 비교할 때 |
| `--log-brief` | 간단한 결과 저장 | 많은 대상의 핵심 fingerprint만 빠르게 비교할 때 |
| `--user-agent` | HTTP User-Agent 지정 | 기본 User-Agent 차단 여부와 응답 차이를 확인할 때 |
| `--proxy` | HTTP proxy 사용 | 지정 network path 또는 request inspection이 필요할 때 |


## 도구 고유 출력

| 출력·상태 | 의미 | 확정 범위와 다음 확인 |
|---|---|---|
| `<URL> [200 OK]`와 `Title[...]` | 지정 URL이 해당 status·title 응답을 반환함 | URL 응답만 확정한다. redirect 최종 목적지, vhost와 인증 상태가 의도와 맞는지 확인한다. |
| `Apache[...]`, `HTTPServer[...]`, CMS·framework·plugin 토큰 | 응답의 header·HTML·cookie가 plugin signature와 일치함 | 기술 스택 후보만 확정한다. 수동 header·source·정적 파일로 제품을 교차 확인한다. |
| plugin 토큰에 버전 정보가 포함됨 | 응답에 버전 signature 또는 banner가 노출됨 | 검토할 버전 후보이다. 실제 설치·patch 버전과 영향 범위는 `searchsploit`, vendor advisory, changelog와 직접 기능으로 확인한다. |
| `[301]`, `[302]`, `[401]`, `[403]` 또는 일부 plugin만 출력 | redirect·인증·차단 응답의 fingerprint가 수집됐을 수 있음 | 최종 URL, Host, 인증 전·후 응답을 확인하고 원본 애플리케이션 지문과 구분한다. |
| 출력 없음, 연결·이름 해석·TLS 오류 | 응답이 단순한 것이 아니라 요청 자체가 실패했을 수 있음 | 오탐·기술 부재로 단정하지 말고 `curl`, vhost, redirect, proxy와 인증 필요 여부를 확인한다. |

## 관련 공격기법

- [[웹 지문 확인과 공격면 분류]]
- [[웹 정찰과 경로 열거]]
