---
tags:
  - 서비스/http
  - 기능/프로토콜접근
실행환경: ["Linux", "Windows"]
필요조건: ["대상 URL"]
결과: ["정보", "파일"]
---

# curl

## 도구 개요

`curl`은 URL에 HTTP·HTTPS 요청을 보내고 응답 상태·헤더·본문을 확인하거나 파일을 전송하는 명령줄 클라이언트다. 웹 서버 지문·인증·메서드·요청 데이터를 확인하고 간단한 업로드·다운로드를 재현할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 URL에 접근 가능한 Linux 또는 Windows 호스트
- 필요한 입력: 프로토콜을 포함한 URL
- 요청별 입력: HTTP method, header, body, 인증 정보, 업로드/다운로드할 파일과 저장 경로
- `<URL>`은 scheme·host·port·path를 가진 전체 URL, local input/output은 curl 실행 host의 path다. HTTP status와 TLS·authentication 결과를 분리하며 upload 성공은 서버의 실행이나 공개를 뜻하지 않는다.


## 표준 사용법

`<URL>`은 scheme·host·port·path를 포함한 full URL, `<LOCAL_FILE>`은 curl 실행 host의 input/output path다. `<USER>:<PASSWORD>`는 HTTP authentication input이고 `-k`는 자체 서명 인증서로 validation failure 원인이 확인된 경우에만 쓴다. response status와 expected content type/body는 file hash 없이 download·upload 결과를 해석하는 기준이다.

```bash
curl [options] <url>
```

## 대표 예시

### 웹 서버 헤더 확인

```bash
curl -I http://<TARGET>
```

### 파일 다운로드

```bash
curl -o LinEnum.sh https://example.com/LinEnum.sh
```

### 인증이 필요한 HTTPS 요청

```bash
curl -k -u <USER>:<PASSWORD> https://<TARGET>/admin
```

### 로그인 폼 POST 요청 테스트

```bash
curl -X POST -d 'username=admin&password=admin' http://<TARGET>/login
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-I` | 헤더만 요청 |
| `-o`, `-O` | 출력 파일명 지정 또는 원격 파일명으로 저장. 성공 상태·예상 content type·필요한 응답 본문을 함께 확인하며, 비교 기준이 없는 file hash는 기본 확인으로 쓰지 않음 |
| `-k` | 자체 서명 인증서로 검증 실패 원인이 확인된 경우에만 TLS 인증서 검증 무시 |
| `-u` | Basic/서비스 인증 정보 지정 |
| `-X` | HTTP method 지정 |
| `-d` | POST body 데이터 전송 |
| `--data-binary` | 줄바꿈 변환 없이 요청 body 전송 |
| `-T`, `--upload-file` | 지정한 로컬 파일을 URL로 업로드 |
| `--basic` | 서버와 협상하지 않고 HTTP Basic 인증 사용 |
| `--path-as-is` | URL 경로의 `/../`·`/./` 시퀀스를 정규화하지 않고 전송 |
| `-H` | 요청 헤더 추가 |
| `-G` | 데이터를 query string으로 전송 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 상태 코드와 헤더 확인 | 웹 서버 응답 특성 파악 | `Server`, `Location`, `Set-Cookie`, 인증 헤더 확인 |
| `200/301/302/401/403` | 접근 가능, 리다이렉트, 인증/권한 필요 여부 확인 | 브라우저 재확인, 인증 정보 적용, 우회 헤더/경로 검토 |
| 파일 다운로드/업로드 성공 | 파일 전송 또는 기능 검증 가능 | 응답 status·예상 content type 또는 필요한 본문, 업로드 위치, 실행 가능 여부 확인 |
| TLS/연결 오류 | 인증서, SNI, 프록시, 포트 문제 | `-k`, `--resolve`, Host 헤더, 프록시/터널 확인 |

## 관련 공격기법

- [[Linux HTTP 파일 반입]]
- [[Linux HTTP 파일 회수]]
- [[웹 지문 확인과 공격면 분류]]
- [[웹 단서 기반 기능 열거]]
- [[공개 클라우드 스토리지 익명 접근 검증]]
- [[상황별 파일 전송]]
- [[제한 환경 파일 반입]]
- [[WebDAV PUT 업로드와 실행]]
- [[CoreFTP HTTP PUT Path Traversal]]

## 참고 링크

- [curl manual](https://curl.se/docs/manpage.html)
