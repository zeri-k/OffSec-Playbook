---
tags:
  - 기능/인증검증
실행환경: ["Linux"]
필요조건: ["사용자 또는 사용자 목록", "비밀번호 후보 또는 목록"]
결과: ["자격증명"]
---

# medusa

## 도구 개요

`medusa`는 서비스 module을 선택해 하나 이상의 호스트에 사용자·비밀번호 조합을 병렬로 시도하는 원격 인증 검증 도구다. 대상·계정 목록을 함께 처리하거나 서비스별 동시 시도 수를 조절해야 하는 작업에 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 인증 서비스에 접근 가능한 Linux 호스트
- 필요한 입력: host/host list, 사용자/user list, 비밀번호/password list와 서비스 module
- 환경별 입력: 비표준 port, TLS와 module option
- spraying 조건: 계정 잠금 정책과 rate limit을 확인하고 병렬 수와 시도 간격을 정한다.


## 표준 사용법

```bash
medusa -h <target> -U <user_list> -P <password_list> -M <module>
```

## 대표 예시

`<TARGET>`은 SSH 서비스 IP/FQDN(예: `ssh.example.test`)이며 사용자 목록과 모듈 입력은 실행 호스트의 파일·값이다.

### SSH 로그인 브루트포스

```bash
medusa -h <TARGET> -U users.txt -P passwords.txt -M ssh
```

### SMB 로컬 계정 검증

```bash
medusa -H hosts.txt -u admin -p 'Password1!' -M smbnt
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-h`, `-H` | 단일 대상 또는 대상 목록 |
| `-u`, `-U` | 단일 사용자 또는 사용자 목록 |
| `-p`, `-P` | 단일 비밀번호 또는 비밀번호 목록 |
| `-M` | 사용할 모듈 지정 |
| `-n` | 포트 지정 |
| `-t` | 병렬 스레드 수 |
| `-f` | 성공 시 해당 대상 중단 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 유효 credential 출력 | 계정/비밀번호 조합이 인증에 성공함 | 해당 서비스에 직접 로그인하고 SMB/WinRM/SSH/FTP/Web 등 재사용 가능성 확인 |
| 모든 조합 실패 | 사용자 목록, 비밀번호 목록, 모듈 또는 실패 문자열이 맞지 않을 수 있음 | 사용자 열거 결과와 서비스별 옵션을 다시 확인 |
| 지연, 차단, 잠금 징후 | rate limit, lockout policy, 방어 장비 영향 가능 | 병렬 수와 시도 빈도를 줄이고 password spraying 방식으로 전환 |
| 연결 또는 모듈 오류 | 포트, TLS, 서비스 모듈, 폼 파라미터가 맞지 않음 | 서비스별 help, 포트, TLS 옵션, 실패 문자열을 재확인 |

## 관련 공격기법

- [[원격 비밀번호 공격]]
