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


## 표준 사용법

```bash
ftp <target>
```

## 대표 예시

### anonymous 로그인과 파일 확인

```bash
ftp <TARGET>
Name: anonymous
Password: anonymous
ftp> ls
ftp> get backup.zip
```

### 쓰기 권한 확인

```text
ftp> put test.txt
```

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-h`, `--help` | 도움말 확인 | 지원 옵션과 모듈 확인 |

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
