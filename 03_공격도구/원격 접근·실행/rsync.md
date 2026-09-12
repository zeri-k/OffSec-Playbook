---
tags:
  - 서비스/rsync
  - 기능/프로토콜접근
  - 기능/파일전송
실행환경: ["Linux"]
필요조건: ["rsync daemon module 또는 SSH 원격 경로"]
결과: ["module 정보", "파일"]
---

# rsync

## 도구 개요

`rsync`는 daemon module이나 SSH 원격 경로의 파일 목록을 확인하고 파일을 동기화·전송하는 도구다. 공개 module의 읽기·쓰기 범위를 검증하거나 SSH를 통한 파일 수집·업로드에 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- daemon 입력: rsync 호스트와 module 이름
- SSH transport 입력: SSH 계정과 원격 경로
- 전송 입력: 로컬 source/destination 경로와 필요한 인증 정보

## 표준 사용법

```bash
rsync [options] <source> <destination>
```

## 대표 예시

### 공개 rsync module 확인

```bash
rsync --list-only rsync://<TARGET>/
```

### module 내용 다운로드

```bash
rsync -av rsync://<TARGET>/module/ ./loot/
```

### 인증이 필요한 daemon module

```bash
rsync --list-only rsync://<RSYNC_USER>@<TARGET>/<MODULE>/
```

비밀번호는 prompt에 입력한다. 이 계정은 Rsync daemon의 module 인증 주체이며 SSH 계정과 같다고 가정하지 않는다.

### SSH transport로 파일 업로드

```bash
rsync -av -e ssh ./file.txt user@<TARGET>:/tmp/
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `--list-only` | 파일 전송 없이 목록만 확인 |
| `-a` | archive mode. 권한/시간 등 보존 |
| `-v` | 상세 출력 |
| `-z` | 압축 전송 |
| `-e ssh` | SSH transport 사용 |
| `--delete` | 대상에서 원본에 없는 파일 삭제. 주의 필요 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| module 목록 출력 | rsync daemon이 공개 module을 노출 | module별 `--list-only`로 파일 목록 확인 |
| 파일 목록 출력 | 읽기 가능 | 백업, 설정, credential, 소스코드 파일 다운로드 검토 |
| upload 성공 | 쓰기 가능 module 존재 | 쓰기 위치와 실행 가능 경로 여부를 분리해 판단 |
| auth failed/permission denied/connection refused | 인증 필요, 권한 부족, 서비스 접근 불가 | module 이름, 계정, TCP/873 접근성, SSH transport 여부 확인 |

## 참고 링크

- [rsync(1) official manual](https://rsync.samba.org/ftp/rsync/rsync.1.html)
- [rsyncd.conf(5) official manual](https://rsync.samba.org/ftp/rsync/rsyncd.conf.5.html)

## 관련 공격기법

- [[Rsync 공개 모듈 수집]]
