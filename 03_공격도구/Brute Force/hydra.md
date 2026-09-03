---
tags:
  - 기능/인증검증
실행환경: ["Linux"]
필요조건: ["사용자 또는 사용자 목록", "비밀번호 후보 또는 목록"]
결과: ["자격증명"]
---

# hydra

## 도구 개요

`hydra`는 SSH, FTP, 메일, 웹 로그인 폼 등 여러 원격 인증 서비스에 사용자·비밀번호 조합을 시도하는 모듈형 도구다. 서비스별 입력 형식과 병렬 작업 수를 조절할 수 있어 다양한 로그인 지점을 같은 방식으로 검증할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 인증 서비스에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 host/port와 서비스 모듈, 사용자 또는 사용자 목록, 비밀번호 또는 wordlist
- HTTP form 입력: 요청 경로, form parameter와 실패 문자열
- spraying 조건: 계정 잠금 정책과 rate limit을 확인하고 thread/시도 간격을 정한다.


## 표준 사용법

```bash
hydra -L <user_list> -P <password_list> <service>://<target>
```

## 대표 예시

### SSH 사용자/비밀번호 목록 공격

```bash
hydra -L users.txt -P passwords.txt ssh://<TARGET>
```

### FTP 단일 사용자 비밀번호 추측

```bash
hydra -l admin -P passwords.txt ftp://<TARGET>
```

FTP 서버가 병렬 로그인 시도에 민감하거나 `550` 같은 비정상 응답을 섞어 반환하면 병렬 작업 수를 1로 낮춰 재시도한다.

```bash
hydra -t 1 -V -l admin -P passwords.txt ftp://<TARGET>
```

실행 중 FTP 응답 코드를 확인하려면 debug 출력과 로그 저장을 같이 사용한다.

```bash
hydra -t 1 -V -d -l admin -P passwords.txt ftp://<TARGET> 2>&1 | tee hydra-ftp.log
tail -f hydra-ftp.log | grep -E '550|530|230|421|Login|incorrect|denied'
```

판단:

- `-V`는 시도 중인 사용자/비밀번호 조합을 보여준다.
- `-d`는 FTP 서버 응답을 포함한 debug 출력을 보여준다.
- `530`은 일반적인 로그인 실패, `230`은 로그인 성공이다.
- `550` 또는 `421`이 반복되면 병렬 연결, rate limit, FTP 서버 상태 처리 문제 가능성을 본다.

### 웹 로그인 폼 password spraying

```bash
hydra -L users.txt -p '<PASSWORD>' <TARGET> http-post-form '/login:username=^USER^&password=^PASS^:Invalid'
```

### POP3 단일 비밀번호 spraying

```bash
hydra -L users.txt -p '<PASSWORD>' -f pop3://<TARGET>
```

메일 서비스에서 검증된 사용자 후보가 있고 계정 잠금 위험을 낮춰야 할 때 단일 비밀번호로 확인한다.

### IMAP/SMTP credential 검증

```bash
hydra -L users.txt -p '<PASSWORD>' -f imap://<TARGET>
hydra -L users.txt -p '<PASSWORD>' -f smtp://<TARGET>
```

서비스별 로그인 성공 여부가 다를 수 있으므로 POP3, IMAP, SMTP 응답을 각각 확인한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-l`, `-L` | 단일 사용자 또는 사용자 목록 |
| `-p`, `-P` | 단일 비밀번호 또는 비밀번호 목록 |
| `-s` | 기본값이 아닌 포트 지정 |
| `-t` | 병렬 작업 수. FTP/RDP처럼 동시 연결에 민감한 서비스는 `-t 1` 또는 낮은 값으로 안정화 |
| `-V` | 시도 중인 조합 출력 |
| `-f` | 첫 성공 시 중단 |
| `-o` | 결과 파일 저장 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 유효 credential 출력 | 계정/비밀번호 조합이 인증에 성공함 | 해당 서비스에 직접 로그인하고 SMB/WinRM/SSH/FTP/Web 등 재사용 가능성 확인 |
| 모든 조합 실패 | 사용자 목록, 비밀번호 목록, 모듈 또는 실패 문자열이 맞지 않을 수 있음 | 사용자 열거 결과와 서비스별 옵션을 다시 확인 |
| FTP에서 `550` 또는 비정상 응답 반복 | 서버의 동시 세션 제한, rate limit, FTP 상태 처리 문제 가능성 | `-t 1`, `-V`로 낮은 병렬 수에서 재시도하고 수동 FTP 로그인으로 응답 비교 |
| 지연, 차단, 잠금 징후 | rate limit, lockout policy, 방어 장비 영향 가능 | 병렬 수와 시도 빈도를 줄이고 password spraying 방식으로 전환 |
| 연결 또는 모듈 오류 | 포트, TLS, 서비스 모듈, 폼 파라미터가 맞지 않음 | 서비스별 help, 포트, TLS 옵션, 실패 문자열을 재확인 |

## 관련 공격기법

- [[원격 비밀번호 공격]]
