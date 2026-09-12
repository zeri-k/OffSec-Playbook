---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/열거
  - 기능/파일전송
실행환경: ["Linux"]
필요조건: ["SMB 접근", "익명, guest 또는 계정 인증 조건"]
결과: ["공유 권한", "파일 목록", "파일"]
---

# smbmap

## 도구 개요

`smbmap`은 SMB 서버의 공유별 READ·WRITE 권한을 빠르게 분류하고 하위 파일을 탐색하거나 전송하는 도구다. 여러 공유의 접근 수준을 먼저 비교해 조사 우선순위를 정할 때 유용하며, 공유 단위 표시는 개별 파일 ACL과 구분해 검증한다.

## 필요한 입력과 실행 환경

- 실행 환경: `smbmap`이 설치된 Linux 호스트
- 입력: SMB 서버 주소
- 인증 입력: null/guest 조건 또는 계정·비밀번호
- 선택 입력: 탐색할 share와 하위 경로


## 표준 사용법

```bash
smbmap -H <target> [options]
```

## 대표 예시

### SMB share와 권한 빠르게 확인

```bash
smbmap -H <TARGET>
```

### 인증 정보로 share 권한 확인

```bash
smbmap -H <TARGET> -u user -p '<PASSWORD>'
```

### null과 guest 접근을 분리 확인

```bash
smbmap -H <TARGET> -P 445 -u '' -p ''
smbmap -H <TARGET> -P 445 -u guest -p ''
```

`smbclient -N`으로 share 목록이 보여도 `smbmap -u '' -p ''`에서 authenticated session이 0개일 수 있다. 이때 `guest` 빈 비밀번호로 다시 시도한다.


### READ 가능한 share 재귀 탐색

```bash
smbmap -H <TARGET> -P 445 -u guest -p '' -r Home
smbmap -H <TARGET> -P 445 -u guest -p '' -r Home --depth 10
```


## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-H` | 대상 호스트 지정 |
| `-u`, `-p` | 인증 정보 지정 |
| `-P` | SMB 포트 지정. 보통 `445` |
| `-d` | 도메인 지정 |
| `-r [PATH]` | share 또는 하위 경로 탐색. 예: `-r Home`, `-r 'Home\IT'` |
| `--depth <N>` | `-r` 탐색 깊이 조정 |
| `--download`, `--upload` | 파일 다운로드/업로드 |
| `-x` | 권한이 있을 때 명령 실행 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| share별 READ/WRITE/NO ACCESS | SMB share 권한 빠른 분류 | READ share에서 민감 파일 검색, WRITE share에서 업로드 영향 검토 |
| `Established ... 0 authenticated session(s)` | SMB 연결은 됐지만 해당 인증 방식으로 세션 생성 실패 | null 대신 `guest` 빈 비밀번호, 도메인/포트 지정 |
| `guest`로 `Authenticated`와 `READ ONLY` 출력 | guest 빈 비밀번호로 읽기 가능 | `-r <SHARE>`와 `--depth`로 탐색, 필요 시 `smbclient`로 다운로드 |
| admin share 접근 가능 | 관리자 권한 가능성 | 원격 실행, SAM dump, 파일 시스템 접근 가능성 확인 |
| `ACCESS DENIED` | 권한 부족 | 다른 credential, 도메인/로컬 계정 형식 확인 |
| null session은 실패하고 `smbclient -N`은 목록 표시 | 도구별 null/guest 처리 차이 | `smbmap -u guest -p ''`, `smbclient -U 'guest%'`로 비교 |
| 인증 실패 | credential 또는 도메인 형식 오류 | `DOMAIN\\user`, 로컬 계정, NTLM/Kerberos 구분 |
| 재귀 탐색 실패 | 경로 권한 또는 timeout | 특정 share만 지정하고 depth/timeout 조정 |
| 연결 실패 | SMB 포트, 방화벽 또는 dialect 문제 | 포트 상태, dialect와 signing 여부 확인 |

## 관련 공격기법

- [[SMB 익명 열거와 공유 권한 확인]]
- [[SMB 공유 자격증명 수집]]
- [[SMB 쓰기 가능한 공유 검증]]

## 참고 링크

- [SMBMap official repository](https://github.com/ShawnDEvans/smbmap)
