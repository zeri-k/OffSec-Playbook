---
tags:
  - 서비스/http
  - 기능/워드리스트
실행환경: ["Linux"]
필요조건: ["대상 URL"]
결과: ["정보", "wordlist"]
---

# cewl

## 도구 개요

`cewl`은 웹사이트를 크롤링해 페이지에 노출된 단어·이메일 주소와 연결된 문서 metadata를 수집하고 대상 맞춤형 후보 파일로 저장한다. 조직명·제품명·연도처럼 대상 문맥을 반영한 비밀번호 크래킹 후보나 사용자명 단서를 만들 때 유용하지만, 수집 항목 자체가 유효한 계정이나 비밀번호를 뜻하지는 않는다.

## 필요한 입력과 실행 환경

- 실행 위치와 도달성: `cewl`을 실행할 수 있고 대상 URL의 HTTP 또는 HTTPS 서비스까지 연결할 수 있는 Linux 호스트
- 대상 URL: 스킴, 포트, 가상 호스트와 시작 경로를 포함한 크롤링 기준 URL. IP 주소와 도메인이 서로 다른 페이지를 반환하면 의도한 vhost의 URL을 사용한다.
- 필요한 파일·값: 필요하면 크롤링 깊이, 최소 단어 길이, 숫자 포함 여부, wordlist·이메일·metadata 출력과 임시 파일 경로. metadata 수집에는 ExifTool 필요
- 인증 정보: 아래 예시는 익명으로 읽을 수 있는 콘텐츠만 수집한다. 로그인 뒤 페이지가 필요하면 대상 인증 방식과 현재 CeWL 실행이 해당 인증 상태를 전달할 수 있는지 별도로 확인한다.
- 대상에서 필요한 서비스와 권한: HTTP 응답 본문을 읽을 수 있어야 한다. 연결 성공이나 로그인 페이지 수신만으로 인증된 콘텐츠 접근을 의미하지 않는다.
- `<TARGET_URL>`은 scheme·vhost·port·start path를 포함한 URL(예: `https://site.example.invalid/docs/`)이고, `<TARGET>`은 단순 HTTP 예시의 host명이다. `<PUBLIC_HOST>`는 공개 문서 수집에 쓰는 FQDN, `<CRAWL_DEPTH>`·`<MIN_WORD_LENGTH>`는 양의 정수다. `<WORDLIST_FILE>`·`<EMAIL_FILE>`·`<METADATA_FILE>`은 `$CEWL_OUTPUT_DIR` 아래에 이번 실행이 만든 파일명이며, wordlist·email·metadata 출력과 `--meta-temp-dir`은 CeWL 실행 host의 새 경로다. public content 후보는 credential이나 인증 결과가 아니다.


## 표준 사용법

```bash
cewl [options] <url>
```

## 대표 예시

### 웹사이트 기반 맞춤 wordlist 생성

```bash
cewl -d 2 -m 5 -w words.txt https://example.com
```

확인할 출력:

- `words.txt`가 생성되고 최소 길이 5 이상의 수집 단어가 행 단위로 저장되는지 확인한다.
- 파일의 단어는 페이지 노출 기반 비밀번호·사용자명 **후보**일 뿐, 계정 존재나 비밀번호 유효성을 확정하지 않는다.

### 숫자가 포함된 단어까지 수집

```bash
cewl --with-numbers -d 3 -w cewl.txt http://<TARGET>
```

확인할 출력:

- `cewl.txt`에 숫자가 포함된 페이지 단어도 저장되는지 확인한다. 숫자가 포함됐다는 사실만으로 조직의 비밀번호 규칙이나 실제 비밀번호 사용을 확정하지 않는다.

### 이메일 주소 수집

```bash
cewl -e --email_file emails.txt https://example.com
```

확인할 출력:

- `emails.txt`에 페이지에서 추출한 이메일 형식 문자열이 저장되는지 확인한다.
- 발견 문자열은 계정·메일함의 존재나 현재 사용 여부를 확정하지 않으므로 별도 계정 열거 절차에서 검증한다.

### 공개 문서 metadata와 이메일을 함께 수집

```bash
CEWL_OUTPUT_DIR="$(mktemp -d "${PWD}/cewl.XXXXXX")"
install -d -m 700 "$CEWL_OUTPUT_DIR/meta-temp"
cewl -d '<CRAWL_DEPTH>' -m '<MIN_WORD_LENGTH>' \
  -w "$CEWL_OUTPUT_DIR/words.txt" \
  -e --email_file "$CEWL_OUTPUT_DIR/emails.txt" \
  -a --meta_file "$CEWL_OUTPUT_DIR/metadata.txt" \
  --meta-temp-dir "$CEWL_OUTPUT_DIR/meta-temp" \
  'https://<PUBLIC_HOST>/'
```

확인할 출력:

- `metadata.txt`에 연결된 문서의 author·creator 같은 metadata 문자열과 `emails.txt`의 이메일 후보가 분리되어 저장되는지 확인한다.
- `-a`를 썼는데 metadata가 없으면 문서 부재로 단정하지 않고 ExifTool 설치, 지원 형식과 크롤링 범위를 확인한다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
| --- | --- | --- |
| `-d` | 시작 URL에서 따라갈 크롤링 깊이 | 첫 페이지만으로 후보가 부족하거나 범위를 제한할 때 |
| `-m` | 저장할 단어의 최소 길이 | 짧고 일반적인 단어 노이즈를 줄일 때 |
| `-w` | 수집 단어를 저장할 wordlist 파일 | 후속 정제·크래킹 입력을 파일로 남길 때 |
| `-e` | 페이지에서 이메일 형식 문자열도 추출 | 사용자명 형식 단서를 함께 수집할 때 |
| `--email_file` | 이메일 결과를 별도 파일에 저장 | 단어 목록과 이메일 후보를 분리할 때 |
| `-a`, `--meta` | 연결된 문서의 metadata를 ExifTool로 처리 | 공개 문서의 작성자·사용자명 형식 후보가 필요할 때 |
| `--meta_file` | metadata 결과를 별도 파일에 저장 | wordlist·이메일과 문서 metadata를 분리할 때 |
| `--meta-temp-dir` | metadata 처리용 임시 문서 경로 지정 | 이번 작업의 임시 파일을 기존 `/tmp`와 분리할 때 |
| `--with-numbers` | 숫자가 포함된 단어도 결과에 포함 | 연도·제품명처럼 숫자가 섞인 후보가 필요할 때 |
| `-v` | 요청과 수집 과정을 상세히 출력 | redirect, 제외된 페이지와 수집 범위를 진단할 때 |


## 도구 고유 출력

| 출력·상태 | 의미 | 확정 범위와 다음 확인 |
|---|---|---|
| `-w`로 지정한 파일이 생성되고 단어가 저장됨 | 지정 URL에서 텍스트 후보 수집 성공 | 해당 문자열의 페이지 노출만 확정한다. `john`, `hashcat`, `hydra` 입력에 맞게 정제하되 실제 비밀번호로 간주하지 않는다. |
| `--email_file` 파일에 이메일 형식 문자열이 저장됨 | 공개 콘텐츠에서 이메일 후보 추출 성공 | 문자열 발견만 확정한다. 실제 계정·메일함 여부와 인증 가능성은 별도 검증한다. |
| `--meta_file`에 author·creator 문자열이 저장됨 | CeWL이 따라간 공개 문서에서 metadata 후보 추출 | 문서 metadata만 확정한다. 작성자 문자열이 현재 계정·직원·AD 사용자라는 뜻은 아니다. |
| 출력 파일이 비었거나 단어가 너무 적음 | 최소 길이, 크롤링 깊이, redirect, 인증 또는 동적 렌더링 때문에 수집 범위가 제한됐을 수 있음 | 계정·단어 부재로 단정하지 말고 기준 URL 응답, vhost, depth와 인증 필요 페이지 여부를 확인한다. |
| 공통 단어·중복·짧은 단어가 많음 | 콘텐츠 수집은 됐지만 후보 품질이 낮음 | 길이 제한, 중복 제거, 조직명·서비스명 규칙으로 후처리한다. |
| HTTP·TLS·이름 해석 오류 또는 연결 실패 | 현재 실행 위치에서 지정 URL의 콘텐츠를 가져오지 못함 | 대상 서비스 부재로 단정하지 말고 `curl`로 스킴, 포트, vhost, redirect, 프록시와 TLS 기준 응답을 확인한다. |

## 관련 공격기법

- [[웹 단서 기반 기능 열거]]
- [[외부 공개 정보로 이메일과 사용자명 후보 수집]]
- [[오프라인 해시 크래킹]]

## 변경 영향과 정리

`-w`·`--email_file`·`--meta_file`은 페이지에서 수집한 단어·이메일·metadata 후보를 로컬 파일에 저장하고 `--meta-temp-dir`에는 처리 중인 문서 사본이 남을 수 있다. 대상에는 반복 HTTP 요청을 남긴다. 실행 전 기존에 없는 고유 출력 디렉터리를 사용하고, 보존하지 않을 때는 이번 실행의 파일만 제거한다. 대상 access log와 rate-limit 영향은 로컬 파일 삭제로 복구되지 않는다.

```bash
CEWL_OUTPUT_DIR="$(mktemp -d "${PWD}/cewl.XXXXXX")"
install -d -m 700 "$CEWL_OUTPUT_DIR/meta-temp"
# 실행 시 wordlist·email·metadata 출력과 --meta-temp-dir을 이 디렉터리 아래로 지정
rm -f -- "$CEWL_OUTPUT_DIR/<WORDLIST_FILE>" "$CEWL_OUTPUT_DIR/<EMAIL_FILE>" "$CEWL_OUTPUT_DIR/<METADATA_FILE>"
find "$CEWL_OUTPUT_DIR/meta-temp" -xdev -depth -type f -delete
find "$CEWL_OUTPUT_DIR/meta-temp" -xdev -depth -type d -empty -delete
rmdir -- "$CEWL_OUTPUT_DIR"
```

디렉터리가 비지 않으면 재귀 삭제하지 말고 예상하지 않은 산출물을 확인한다. 수집한 email·조직 단어는 계정 또는 비밀번호가 아니지만 민감한 후보 자료로 취급하며 Vault에 실제 값을 저장하지 않는다.

## 참고 링크

- [CeWL 공식 저장소와 email·metadata option](https://github.com/digininja/CeWL)
- [ExifTool 공식 command-line 문서](https://exiftool.org/exiftool_pod2.html)
