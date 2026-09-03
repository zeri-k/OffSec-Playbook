---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/원격실행
실행환경: ["Linux"]
필요권한: ["로컬 관리자 권한"]
필요조건: ["SMB 인증 정보 또는 hash", "ADMIN$ 접근과 서비스 생성 가능"]
결과: ["세션", "명령 실행"]
---

# impacket-psexec

## 도구 개요

`impacket-psexec`는 SMB의 ADMIN$ 공유와 Service Control Manager를 이용해 임시 서비스를 만들고 원격 명령 shell을 연다. Linux에서 관리자 자격 증명으로 서비스 기반 원격 실행이 필요할 때 사용하며, 대상에 서비스와 파일 흔적을 남기는 방식이다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 SMB/RPC TCP/445에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 주소, 도메인/로컬 관리자 credential 또는 NTLM hash
- 대상 조건: `ADMIN$` 접근과 Service Control Manager를 통한 서비스 생성 권한이 필요하다.


## 표준 사용법

```bash
impacket-psexec <domain>/<user>:<password>@<target>
```

## 대표 예시

### NTLM hash로 SYSTEM shell 획득

```bash
impacket-psexec administrator@<TARGET> -hashes :<NTLM_HASH>
```

### 도메인 credential로 원격 명령 실행 세션 획득

```bash
impacket-psexec <DOMAIN>/<USER>:'<PASSWORD>'@<DC_HOST>
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-hashes` | LM:NT hash로 인증 |
| `-k` | Kerberos 인증 사용 |
| `-no-pass` | 비밀번호 없이 Kerberos/ccache 사용 |
| `-dc-ip` | 도메인 컨트롤러 IP 지정 |
| `-target-ip` | 이름 해석과 별도 대상 IP 지정 |
| `-service-name` | 생성할 서비스 이름 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| command output 또는 shell 획득 | Windows 원격 명령 실행 성공 | `whoami`, `hostname`, `ipconfig`로 컨텍스트 확인 |
| service 생성/실행 로그 | 관리자 권한으로 실행 경로 접근 | AV/EDR, 서비스 cleanup, 파일 쓰기 가능 경로 확인 |
| `STATUS_ACCESS_DENIED` | 관리자 권한 부족 또는 UAC 제한 | 로컬 관리자 여부, admin share 접근, UAC remote restriction 확인 |
| logon/network 오류 | credential, SMB/RPC/SCM 접근 문제 | 포트, 계정 형식, NTLM/Kerberos, 방화벽 확인 |
| 명령 실행 실패 | Service Control Manager/SMB/RPC 제한 | 445 접근성, admin share, 서비스 생성 권한, AV/EDR 확인 |
| 출력 없음 | 실행은 됐지만 stdout 회수 실패 | 파일로 출력 저장, 다른 exec 방식, 방화벽 확인 |

## 관련 공격기법

- [[Pass the Hash]]
- [[Pass the Ticket]]
