---
tags:
  - 서비스/http
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 URL 또는 도메인", "모드에 맞는 wordlist"]
결과: ["정보", "경로"]
---

# gobuster

## 도구 개요

`gobuster`는 wordlist를 대입해 웹 경로·파일, 가상 호스트, DNS 이름 또는 URL의 지정 위치를 빠르게 열거한다. 숨은 리소스와 이름 후보를 찾을 때 모드별로 사용할 수 있으며, wildcard·custom 404와 공통 응답을 거르기 위해 baseline과 status·length 차이를 함께 비교해야 한다.

## 필요한 입력과 실행 환경

- 실행 위치와 도달성: `dir`·`vhost`·`fuzz`는 대상 HTTP(S) 포트에, `dns`는 사용할 DNS resolver에 도달할 수 있는 Linux 호스트에서 실행한다.
- `dir`·`fuzz` 입력: 스킴, 포트, 가상 호스트와 기준 경로를 포함한 URL, 모드에 맞는 wordlist, 존재하지 않는 경로의 baseline status·length·title
- `vhost` 입력: 실제 웹 서버로 연결되는 기준 URL, Host 헤더에 붙일 기준 domain과 vhost wordlist. IP로 접속하더라도 Host 헤더가 달라지면 별도 애플리케이션이 반환될 수 있다.
- `dns` 입력: URL이 아닌 기준 domain과 DNS 이름 wordlist. DNS 레코드 발견은 해당 호스트의 HTTP 서비스 존재를 의미하지 않는다.
- 필요한 인증 정보: 아래 예시는 익명 요청이다. 보호된 경로를 열거하려면 대상이 요구하는 `Authorization`·`Cookie` 등을 `-H`로 전달하고, 인증 전·후 baseline을 분리한다.
- vhost·TLS 조건: 기준 domain의 name resolution과 Host 값을 맞춘다. HTTPS는 URL 이름이 TLS SNI와 인증서 검증에도 영향을 주므로 IP URL과 `--domain`만으로 충분한지 먼저 수동 요청으로 확인한다.
- 필요한 파일·값: 모드에 맞는 wordlist, 필요하면 확장자, 포함·제외 status, 제외할 응답 길이, thread 수와 결과 파일 경로


## 표준 사용법

```shell
gobuster <mode> [옵션]
```

자주 쓰는 모드는 `dir`, `vhost`, `dns`, `fuzz`다.

```shell
gobuster dir -u http://target/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
gobuster vhost -u http://target/ -w subdomains.txt --append-domain
gobuster dns -d example.com -w subdomains.txt
```

## 대표 예시

### 디렉터리/파일 열거

```shell
gobuster dir -u http://<TARGET>/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 50 -o gobuster-dir.txt
```

웹 루트 아래 흔한 경로를 찾는다.

확인할 출력:

- 시작 시 `[+] Url`, `[+] Wordlist`, status 설정과 `Starting gobuster`가 의도한 값인지 확인한다.
- `/경로 (Status: ..., Size: ...)` 형태의 후보를 baseline과 비교한다. `200`도 soft 404일 수 있고 `401`·`403`은 인증·인가 경계 후보일 뿐 현재 읽기 권한을 뜻하지 않는다.

### 확장자 포함 파일 탐색

```shell
gobuster dir -u http://<TARGET>/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt,bak,conf
```

PHP, 백업 파일, 설정 파일처럼 확장자가 붙은 후보를 함께 찾는다.

확인할 출력:

- 원래 단어와 각 확장자를 붙인 경로 중 baseline과 status·length가 다른 항목을 확인한다.
- 파일명 후보 출력만으로 파일 내용, 민감도나 다운로드 권한을 확정하지 않고 개별 HTTP 요청으로 검증한다.

### 상태 코드 필터링

```shell
gobuster dir -u http://<TARGET>/ -w common.txt -b 404,403
```

노이즈가 되는 상태 코드를 제외한다. 반대로 `-s 200,204,301,302,307,401,403`처럼 보고 싶은 코드만 지정할 수도 있다.

확인할 출력:

- `404`, `403`이 결과에서 제외되는지와 남은 후보의 status·length가 baseline과 구분되는지 확인한다.
- 필터에서 빠진 항목은 검사 대상 부재가 아니라 출력 제외일 수 있으므로 `-b`와 `-s` 설정을 함께 기록한다.

### HTTPS 인증서 오류 무시

```shell
gobuster dir -k -u https://<TARGET>/ -w common.txt
```

자체 서명 인증서나 호스트명 불일치가 있는 HTTPS 대상에서 사용한다.

확인할 출력:

- TLS 검증 오류 없이 HTTP status가 수집되는지 확인한다. `-k`는 인증서 검증만 생략하며 올바른 vhost·SNI나 HTTP 인증을 제공하지 않는다.

### vhost 열거

```shell
gobuster vhost -u http://<DOMAIN>/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain -t 60
```

Host 헤더를 바꿔가며 가상 호스트를 찾는다. 기준 도메인이 로컬에서 해석되지 않으면 `/etc/hosts`에 먼저 등록한다.

확인할 출력:

- `Found:` 또는 vhost 후보 행의 host·status·length가 임의 Host baseline과 다른지 확인한다.
- 후보 이름이 출력돼도 DNS 레코드나 별도 서버가 존재한다는 뜻은 아니다. 동일 IP에서 Host 값에 따라 다른 HTTP 응답이 나온다는 점을 수동 요청으로 확인한다.

### 별도 도메인 지정 vhost 열거

```shell
gobuster vhost -u http://<TARGET>:32272 -w subdomains.txt --append-domain --domain <DOMAIN> -t 60
```

접속 URL과 Host 헤더 도메인을 분리해야 할 때 사용한다.

확인할 출력:

- 연결 대상 `<TARGET>:32272`는 유지되고 Host 후보가 `<DOMAIN>` 아래 이름으로 생성되는지 확인한다.
- HTTP vhost 응답과 DNS 이름 존재는 별개이며, HTTPS라면 같은 이름의 SNI 처리도 별도로 확인한다.

### DNS 서브도메인 열거

```shell
gobuster dns -d <DOMAIN> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

DNS 질의로 존재하는 서브도메인을 찾는다.

확인할 출력:

- `Found: <HOST>.<DOMAIN>` 형태의 DNS 결과를 확인한다.
- 이 출력은 DNS 응답을 확정하지만 해당 이름의 HTTP(S) 포트, 애플리케이션 또는 평가 범위는 확정하지 않는다.

### FUZZ 위치 퍼징

```shell
gobuster fuzz -u 'http://<TARGET>/FUZZ' -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

URL의 특정 위치를 단어 목록으로 치환해 확인한다.

확인할 출력:

- `FUZZ`를 치환한 요청 중 baseline과 status·length가 다른 후보를 확인한다. 파라미터·경로 후보만으로 서버 측 기능이나 취약 동작을 확정하지 않는다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
| --- | --- | --- |
| `dir` | URL 경로에 wordlist를 대입하는 디렉터리·파일 모드 | 확인된 웹 vhost 아래 숨은 경로를 찾을 때 |
| `vhost` | 연결 대상은 유지하고 Host 헤더 후보를 바꾸는 모드 | 같은 IP·포트에서 별도 가상 호스트를 찾을 때 |
| `dns` | DNS 이름을 질의하는 서브도메인 brute force 모드 | HTTP와 독립적으로 현재 DNS 레코드를 찾을 때 |
| `fuzz` | URL의 `FUZZ` 토큰 위치를 치환하는 모드 | 경로·파라미터의 특정 위치만 바꿔 비교할 때 |
| `-u <url>` | 스킴·포트·기준 경로를 포함한 대상 URL | `dir`, `vhost`, `fuzz`의 HTTP 연결 대상을 정할 때 |
| `-d <domain>` | DNS 모드의 기준 도메인 | `dns` 질의 suffix를 정할 때 |
| `-w <wordlist>` | 모드에 대입할 단어 목록 | 경로용·vhost용·DNS용 목록을 구분해 사용할 때 |
| `-x <ext>` | 각 단어에 붙여 검사할 확장자 목록 | 백업·설정·스크립트 파일 후보를 추가할 때 |
| `-t <threads>` | 동시 요청·질의 수 | rate limit과 응답 안정성에 맞춰 속도를 조절할 때 |
| `-k` | TLS 인증서 검증 무시 | 자체 서명·이름 불일치 때문에 HTTPS 요청만 실패할 때 |
| `-H <header>` | 각 HTTP 요청에 헤더 추가 | Host 외 인증·세션·애플리케이션 필수 헤더를 전달할 때 |
| `-b <codes>` | 지정 status를 결과에서 제외 | baseline과 같은 오류 status 노이즈를 숨길 때 |
| `-s <codes>` | 지정 status만 결과에 포함 | 관찰할 응답 범위를 명시적으로 제한할 때 |
| `--exclude-length <len>` | 지정 응답 길이를 결과에서 제외 | wildcard·custom 404가 같은 길이로 반복될 때 |
| `--append-domain` | vhost word 뒤에 기준 domain 추가 | wordlist가 짧은 host label만 포함할 때 |
| `--domain <domain>` | vhost Host 후보의 기준 domain 지정 | 연결 URL의 host와 Host header suffix를 분리할 때 |
| `-o <file>` | 결과 저장 | 후보와 당시 필터 조건을 후속 검증용으로 남길 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 확정 범위와 다음 확인 |
|---|---|---|
| `[+] Url`, `[+] Wordlist`, `Starting gobuster`, `Finished` | 설정을 읽고 열거 루프를 시작·종료함 | 스캔 실행만 확정한다. 결과가 없다는 이유만으로 경로·vhost·DNS 이름 부재를 확정하지 않는다. |
| `/path (Status: 200/301/302/401/403, Size: ...)` | `dir`·`fuzz` 후보가 해당 HTTP 응답을 반환함 | status·length·`Location`·본문을 baseline과 비교한다. 경로 실재, 파일 READ와 취약성은 개별 요청으로 확인한다. |
| `Found:`와 vhost host·status·length | 해당 Host 후보의 응답이 필터를 통과함 | 임의 Host와 응답 차이를 재현한다. DNS 레코드나 별도 물리 서버 존재는 확정하지 않는다. |
| `Found: <HOST>.<DOMAIN>` | `dns` 모드에서 이름이 해석됨 | DNS 응답만 확정한다. 대상 포트·HTTP 서비스와 권한은 별도 확인한다. |
| 모든 후보가 같은 status·length 또는 wildcard 진단 `Error:` | custom 404, wildcard DNS/vhost 또는 공통 차단 응답 가능성 | 임의 값 baseline을 다시 잡고 `-b`, `-s`, `--exclude-length`를 조정한다. |
| 시작 단계 `Error:`, timeout 또는 연결·TLS 실패 | URL, wordlist, DNS, WAF, rate limit, TLS, Host 설정 문제로 열거 미완료 | 결과 없음과 구분하고 스레드, User-Agent, `-k`, `/etc/hosts`, vhost와 입력 파일을 확인한다. |

## 관련 공격기법

- [[웹 정찰과 경로 열거]]
- [[웹 숨은 경로와 민감 파일 열거]]
- [[DNS 열거와 Zone Transfer]]
