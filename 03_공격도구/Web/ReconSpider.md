---
tags:
  - 서비스/http
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 HTTP 또는 HTTPS URL", "ReconSpider Python 스크립트"]
결과: ["크롤링 결과", "링크", "폼", "JavaScript 파일", "이메일"]
---

# ReconSpider

## 도구 개요

`ReconSpider`는 웹사이트를 크롤링해 링크·이메일·JavaScript·외부 파일·폼 필드·주석을 `results.json`에 모으는 Python 스크립트다. 페이지에 흩어진 경로와 입력 지점을 한 번에 정리해 후속 웹 열거 대상을 만들 때 적합하며, 저장된 항목은 응답에서 관찰한 후보로 해석한다.

## 필요한 입력과 실행 환경

- 실행 위치와 도달성: Python 3와 `ReconSpider.py`가 있고 대상 HTTP(S) 서비스까지 연결할 수 있는 호스트
- 대상 URL: 스킴, 포트, 가상 호스트와 시작 경로를 포함한 HTTP(S) URL. IP와 domain이 다른 페이지를 반환하면 의도한 vhost URL을 사용하고 name resolution·Host·TLS SNI를 맞춘다.
- 필요한 인증 정보: 아래 실행 문법은 익명 요청만 사용한다. 로그인 뒤 링크와 역할별 기능은 인증 상태가 전달되지 않으면 수집되지 않을 수 있다.
- 필요한 파일·권한: `ReconSpider.py`와 의존성, 현재 작업 디렉터리에 새 `results.json`을 쓸 수 있는 로컬 파일 권한
- 대상에서 필요한 서비스와 권한: 시작 URL과 크롤러가 따라가는 페이지의 HTTP 응답 본문을 읽을 수 있어야 한다. `401`·`403`이나 로그인 페이지 수신은 인증된 콘텐츠 접근을 의미하지 않는다.
- 결과 파일 주의: 실행 전후 `results.json`의 수정 시각을 비교한다. 기존 파일이 남아 있으면 Python 실행 실패 뒤에도 과거 결과를 새 결과로 오인할 수 있다.

## 표준 사용법

```bash
python3 ReconSpider.py <url>
```

실행 후 수집 결과는 `results.json`에 저장된다.

## 대표 예시

### ReconSpider 준비

승인된 배포 원본에서 `ReconSpider.py`와 필요한 의존성을 확보한다.

확인할 출력:

- `ReconSpider.py`와 의존성이 준비됐는지 확인한다. 파일 준비는 대상 URL 도달성이나 크롤링 성공을 확정하지 않는다.

### 웹사이트 크롤링

```bash
python3 ReconSpider.py http://<DOMAIN>
```

대상 웹사이트를 크롤링하고 결과를 `results.json`에 저장한다.

확인할 출력:

- 실행 뒤 새 수정 시각의 `results.json`이 생성되고 `emails`, `links`, `external_files`, `js_files`, `form_fields`, `images`, `videos`, `audio`, `comments` 키가 기록되는지 확인한다.
- 키에 담긴 값은 크롤러가 응답에서 관찰한 후보다. 현재 URL의 status, 인증 필요 여부와 실제 기능은 개별 요청으로 확인한다.

### JSON 결과 확인

```bash
python3 -m json.tool results.json
```

수집된 링크, 이메일, JavaScript 파일, 폼 필드 등을 읽기 쉽게 확인한다.

확인할 출력:

- JSON parser가 오류 없이 구조를 출력하는지 확인한다. 이는 파일 문법과 저장 성공만 확정하며 각 URL의 실재·접근 권한이나 데이터 민감도는 확정하지 않는다.

### 수집된 링크만 빠르게 확인

```bash
grep -i '"links"' -A 20 results.json
```

링크 항목을 우선 확인해 추가 경로 탐색이나 디렉터리 열거 후보로 활용한다.

확인할 출력:

- `"links"` 키와 뒤따르는 URL 후보가 표시되는지 확인한다. `grep` 결과는 일부 문맥만 보여 주므로 전체 JSON과 개별 HTTP 응답을 함께 확인한다.


## 주요 옵션

| 인자·파일 | 의미 | 사용하는 상황 |
|---|---|---|
| `<url>` | 스킴·포트·vhost·시작 경로를 포함한 크롤링 기준 URL | 의도한 웹 애플리케이션에서 익명 크롤링을 시작할 때 |
| `ReconSpider.py` | Python crawler script | Python 3로 실행하며 스크립트 버전과 의존성을 확인할 때 |
| `results.json` | 현재 작업 디렉터리에 쓰는 고정 결과 파일 | 새 수정 시각과 JSON 키를 확인한 뒤 링크·폼·이메일·JavaScript 후보를 검토할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 확정 범위와 다음 확인 |
|---|---|---|
| 새 `results.json`과 예상 JSON 키 생성 | 현재 실행이 결과 파일을 정상 형식으로 저장함 | 저장·파싱 성공만 확정한다. 파일 수정 시각과 시작 URL이 이번 실행과 맞는지 확인한다. |
| `links`, `form_fields`, `js_files`, `emails`에 값이 있음 | 응답 콘텐츠에서 URL·입력·스크립트·이메일 형식 후보 수집 | 폼 파라미터, API endpoint, JS secret 후보를 개별 요청과 소스 검토로 확인한다. 기능 실재·인증 우회·취약성은 확정하지 않는다. |
| `external_files` 또는 `comments`에 값이 있음 | 외부 파일 URL이나 HTML 주석 문자열 관찰 | 파일 status·본문과 주석의 현재 관련성을 확인한다. 발견만으로 민감 정보 노출을 확정하지 않는다. |
| 예상 키가 비었거나 결과가 적음 | 익명 응답, redirect, 인증, 크롤링 범위 또는 동적 콘텐츠 때문에 수집이 제한됐을 수 있음 | 기능 부재로 단정하지 말고 로그인 필요 여부, vhost, `robots.txt`, sitemap과 수동 탐색으로 보완한다. |
| Python 예외·timeout 또는 새 결과 파일 미생성 | 스크립트·의존성·TLS·redirect·rate limit 문제로 실행 실패 | 기존 `results.json`을 이번 결과로 사용하지 말고 `curl`과 브라우저로 기준 응답 및 로컬 쓰기 권한을 확인한다. |

## 관련 공격기법

- [[웹 정찰과 경로 열거]]
