---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/프로토콜접근
  - 기능/파일전송
실행환경: ["Linux"]
필요조건: ["SMB 접근", "익명, guest, 계정 또는 Kerberos 인증 조건"]
결과: ["공유 정보", "파일", "쓰기 결과"]
---

# smbclient

## 도구 개요

`smbclient`는 SMB 공유 목록과 디렉터리를 탐색하고 파일을 다운로드·업로드하는 Samba 명령줄 클라이언트다. 익명·Guest·계정·Kerberos 세션별 실제 파일 접근 범위를 직접 확인하거나 파일을 전송할 때 사용하기 좋다.

## 필요한 입력과 실행 환경

- 실행 환경: Samba client가 설치된 Linux 호스트
- 입력: SMB 서버와 share 이름
- 인증 입력: null/guest 조건, 계정·비밀번호 또는 Kerberos ticket


## 표준 사용법

```bash
smbclient //<target>/<share> [options]
```

## 대표 예시

`<TARGET>`은 SMB 서버 IP/FQDN(예: `smb.example.test`)이고, `<DOMAIN>`은 도메인 계정에만 쓰는 workgroup/DNS 도메인 값(예: `CORP`)이다. `<USER>`·`<PASSWORD>`는 도메인 계정 인증 입력, `notes`는 접속할 share 이름이며, 로컬 계정에는 대상 호스트명 또는 `WORKGROUP`을 사용한다.

### anonymous/null session으로 share 목록 확인

```bash
smbclient -N -L //<TARGET>
```

### 사용자 credential로 share 접속

```bash
smbclient //<TARGET>/notes -U '<USER>'
```

### 도메인 계정으로 share 목록 확인

```bash
smbclient -L //<TARGET> -W <DOMAIN> -U '<USER>%<PASSWORD>'
```

### 로컬 계정 또는 workgroup 지정

```bash
smbclient -L //<TARGET> -W WORKGROUP -U 'administrator%<PASSWORD>'
```



## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-h`, `--help` | 도움말 확인 | 지원 옵션과 모듈 확인 |
| `-L //<target>` | 대상의 share 목록 조회 | 접근 가능한 share를 먼저 확인 |
| `-N`, `--no-pass` | 비밀번호 없이 접속 | null session, guest/anonymous 확인 |
| `-U <user>` | 사용자 지정 | 프롬프트로 비밀번호 입력 |
| `-U '<user>%<pass>'` | 사용자와 비밀번호를 한 번에 지정 | 비대화형 명령 실행 |
| `-W <DOMAIN>` | workgroup/domain 지정 | 도메인 계정, 로컬 계정 구분 |
| `-k` | Kerberos 인증 사용 | `kinit`/ccache 확보 후 ticket으로 접근 |
| `-c '<commands>'` | 접속 후 실행할 명령 지정 | `ls`, `get`, `mget`, `put` 자동 실행 |
| `-m <dialect>` | SMB dialect 지정 | `SMB2`, `SMB3`, 구형 서버 호환성 확인 |
| `-p <port>` | SMB 포트 지정 | 445가 아닌 포트 또는 139 테스트 |
| `--option='client min protocol=SMB2'` | Samba client 세부 옵션 지정 | SMB 버전 협상 문제 우회 |
| `--use-kerberos=required` | Kerberos 사용 강제 | NTLM fallback을 피하고 Kerberos 여부 확인 |

## smbclient 내부 명령

| 명령 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `ls`, `dir` | 파일/디렉터리 목록 조회 | 읽기 가능 범위 확인 |
| `cd <dir>`, `pwd` | 경로 이동/현재 경로 확인 | share 탐색 |
| `get <file>` | 단일 파일 다운로드 | 민감 파일 수집 |
| `mget *` | 여러 파일 다운로드 | 디렉터리 내 파일 일괄 수집 |
| `recurse ON` | 재귀 탐색/다운로드 활성화 | 하위 디렉터리까지 수집 |
| `prompt OFF` | `mget` 확인 프롬프트 비활성화 | 자동 다운로드 |
| `put <file>` | 파일 업로드 | 쓰기 권한 검증 |
| `mkdir`, `rmdir`, `del` | 디렉터리/파일 변경 | 쓰기 영향 확인 시 주의 |
| `allinfo <file>` | 파일 상세 정보 조회 | timestamp/속성 확인 |
| `exit`, `quit` | 세션 종료 | 작업 종료 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| share 목록 출력 | SMB 인증 또는 익명 열거 성공 | share별 읽기/쓰기 가능 여부 확인 |
| `smb: \\>` 프롬프트 | share 접속 성공 | `ls`, `get`, `put`, `recurse`로 파일 접근 범위 확인 |
| `NT_STATUS_ACCESS_DENIED` | share 또는 파일 권한 부족 | 다른 share, 다른 계정, guest/null session 차이 확인 |
| `NT_STATUS_LOGON_FAILURE` | 계정 형식, 도메인 또는 인증 정보 불일치 | `DOMAIN\\user`, `user%pass`, Kerberos/NTLM 구분 |
| 목록은 보이나 다운로드 실패 | 파일 권한, 경로 또는 quoting 문제 | 파일명 quoting과 `get`, `mget`, `recurse` 확인 |
| dialect 또는 연결 오류 | SMB 버전, 포트 또는 방화벽 문제 | `-m`, 포트 상태와 NetExec 결과 확인 |

## 관련 공격기법

- [[SMB 익명 열거와 공유 권한 확인]]
- [[SMB 인증 공유 파일 수집]]
- [[SMB 공유 자격증명 수집]]
- [[SMB 쓰기 가능한 공유 검증]]
- [[Pass the Ticket]]

## 참고 링크

- [Samba smbclient manual](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)
