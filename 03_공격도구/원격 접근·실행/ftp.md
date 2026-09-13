---
tags:
  - 서비스/ftp
  - 기능/프로토콜접근
  - 기능/파일전송
실행환경: ["Linux", "Windows"]
필요조건: ["대상 FTP 서버"]
결과: ["정보", "파일"]
---

# ftp

## 도구 개요

`ftp`는 FTP 서버에 로그인해 원격 경로와 파일 목록을 확인하고 파일을 내려받거나 올리는 대화형 클라이언트다. 익명·사용자 인증과 실제 읽기·쓰기 권한을 빠르게 검증할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 FTP TCP/21에 접근 가능한 Linux 또는 Windows 호스트
- 필요한 입력: 서버 주소와 포트, anonymous 또는 사용자 credential
- 파일 전송 입력: 로컬/원격 경로와 텍스트·바이너리 전송 모드
- 로컬 경로는 FTP client를 실행한 호스트 기준(예: `./report.bin`)이고 원격 경로는 FTP server의 현재 디렉터리 기준(예: `incoming/report.bin`)이다. `get`은 READ 결과, `put`은 WRITE 결과이므로 전송 성공을 실행 권한으로 해석하지 않는다.


## 표준 사용법

`<TARGET>`은 FTP control listener의 IP/FQDN, `<REMOTE_PATH>`는 server의 현재 directory 기준 파일명·경로, `<LOCAL_PATH>`는 FTP client 실행 host의 파일 경로다. 가상 예시는 `ftp.example.invalid`, `incoming/report.bin`, `./report.bin`이며 get과 put은 각각 별도의 READ·WRITE 결과다.

```bash
ftp <target>
```

## 대표 예시

### anonymous 로그인과 파일 확인

```bash
ftp <TARGET>
Name: anonymous
Password: anonymous
ftp> binary
ftp> ls
ftp> get <REMOTE_FILE> <LOCAL_FILE>
```

### 쓰기 권한 확인

```text
ftp> put <LOCAL_FILE> <UNIQUE_REMOTE_FILE>
```

### 데이터 채널 모드 확인과 전환

GNU Inetutils `ftp`에서는 `passive`가 Active(`PORT`)와 Passive(`PASV`) 데이터 연결을 토글한다. 다른 클라이언트는 기본값이나 명령이 다를 수 있으므로 먼저 도움말과 현재 상태를 확인한다.

```text
ftp> help passive
ftp> status
ftp> passive
ftp> ls
```

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-h`, `--help` | 도움말 확인 | 지원 옵션과 모듈 확인 |
| `binary` | binary 전송 유형 사용 | 압축 파일·실행 파일처럼 텍스트 변환을 피해야 하는 파일 |
| `passive` | 지원 구현체에서 Active·Passive 데이터 연결 토글 | 로그인 뒤 목록·전송만 실패할 때 데이터 연결 방향 비교 |
| `prompt` | `mget`·`mput`의 파일별 확인 토글 | 선택한 여러 파일을 일괄 처리하기 전 범위 확인 |
| `get <REMOTE> <LOCAL>` | 원격 파일을 지정한 로컬 경로로 저장 | 기존 로컬 파일과 겹치지 않는 목적지로 회수 |
| `put <LOCAL> <REMOTE>` | 로컬 파일을 지정한 원격 이름으로 저장 | 고유 원격 파일명으로 쓰기 권한 확인 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `230 Login successful` | 인증 성공 또는 anonymous 접근 가능 | `ls`, `pwd`, `get`, `put`으로 읽기/쓰기 권한 확인 |
| 파일 목록 출력 | 디렉터리 열람 가능 | 민감 파일 다운로드, 업로드 가능 경로 분리 확인 |
| `550 Permission denied` | 파일/디렉터리 권한 제한 | 다른 경로, 파일명 quoting, 쓰기 가능 디렉터리 확인 |
| passive/active 연결 실패 | 데이터 채널 문제 | passive mode, 방화벽, 포트 포워딩 여부 확인 |

## 관련 공격기법

- [[FTP 익명 접근과 파일 수집]]
- [[FTP Bounce]]
- [[상황별 파일 전송]]

## 참고 링크

- [GNU Inetutils ftp](https://www.gnu.org/software/inetutils/manual/html_node/ftp-invocation.html)
- [RFC 959: File Transfer Protocol](https://www.rfc-editor.org/rfc/rfc959)
