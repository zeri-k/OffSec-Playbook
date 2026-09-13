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

`<TARGET>`은 인증 서비스 호스트/IP(예: `ssh.example.test`), `<SERVICE>`는 해당 서비스의 Hydra 모듈, 목록 입력은 Linux 호스트의 newline 파일이다. `<USER_LIST>`·`<PASSWORD_LIST>`는 조합 입력, `<USERPASS_FILE>`은 `login:password` 한 쌍씩인 파일이며, `<FAILURE_MARKER>`는 대상 HTTP 실패 응답에서 관찰한 문자열이다.


## 표준 사용법

```bash
hydra -L '<USER_LIST>' -P '<PASSWORD_LIST>' <SERVICE>://<TARGET>
```

`-L`/`-P`는 두 목록의 조합을 시도한다. 유출 자격 증명처럼 확인된 `login:password` 쌍의 대응을 보존해야 할 때는 `-C`를 쓴다. 로컬 결과·debug·restore 파일을 다른 작업과 구분하도록 고유 디렉터리에서 실행한다.

```bash
HYDRA_ORIGINAL_WORKDIR="$PWD"
HYDRA_WORKDIR="$(mktemp -d "${PWD}/hydra.XXXXXX")"
printf 'Hydra workdir: %s\n' "$HYDRA_WORKDIR"
cd -- "$HYDRA_WORKDIR"
```

## 대표 예시

### SSH 사용자/비밀번호 목록 공격

```bash
hydra -L '<USER_LIST>' -P '<PASSWORD_LIST>' ssh://<TARGET>
```

이 예시는 사용자·비밀번호의 모든 조합을 시도한다. 유출 pair 재사용 검증을 의미하지 않는다.

### 유출 `login:password` pair 검증

```bash
hydra -C '<USERPASS_FILE>' -t 1 -f -o result.txt ssh://<TARGET>
```

확인할 출력:

- `-C`는 파일의 각 `login:password` 행을 하나의 쌍으로 처리하며 `-L`/`-P` 교차 조합을 만들지 않는다.
- `result.txt`의 양성 조합은 해당 SSH 서비스의 인증 후보이며, 정상 SSH client로 로그인·identity·권한을 다시 확인한다.

### FTP 단일 사용자 비밀번호 추측

```bash
hydra -l '<USER>' -P '<PASSWORD_LIST>' ftp://<TARGET>
```

FTP 서버가 병렬 로그인 시도에 민감하거나 `550` 같은 비정상 응답을 섞어 반환하면 병렬 작업 수를 1로 낮춰 재시도한다.

```bash
hydra -t 1 -V -l '<USER>' -P '<PASSWORD_LIST>' ftp://<TARGET>
```

실행 중 FTP 응답 코드를 확인하려면 debug 출력과 로그 저장을 같이 사용한다.

```bash
hydra -t 1 -V -d -l '<USER>' -P '<PASSWORD_LIST>' ftp://<TARGET> 2>&1 | tee hydra-ftp.log
tail -f hydra-ftp.log | grep -E '550|530|230|421|Login|incorrect|denied'
```

판단:

- `-V`는 시도 중인 사용자/비밀번호 조합을 보여준다.
- `-d`는 FTP 서버 응답을 포함한 debug 출력을 보여준다.
- `530`은 일반적인 로그인 실패, `230`은 로그인 성공이다.
- `550` 또는 `421`이 반복되면 병렬 연결, rate limit, FTP 서버 상태 처리 문제 가능성을 본다.

### 웹 로그인 폼 password spraying

```bash
hydra -L '<USER_LIST>' -p '<PASSWORD>' <TARGET> http-post-form '/login:username=^USER^&password=^PASS^:<FAILURE_MARKER>'
```


### IMAP/SMTP credential 검증

```bash
hydra -L '<USER_LIST>' -p '<PASSWORD>' -f imap://<TARGET>
hydra -L '<USER_LIST>' -p '<PASSWORD>' -f smtp://<TARGET>
```

서비스별 로그인 성공 여부가 다를 수 있으므로 POP3, IMAP, SMTP 응답을 각각 확인한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-l`, `-L` | 단일 사용자 또는 사용자 목록 |
| `-p`, `-P` | 단일 비밀번호 또는 비밀번호 목록 |
| `-C` | `login:password` 형식의 파일을 쌍 단위로 사용. `-L`/`-P`와 대체 관계 |
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

## 변경 영향과 로컬 산출물 정리

- Hydra는 인증 시도와 서버 로그를 남길 수 있고 계정 잠금을 유발할 수 있다. 로컬 파일을 삭제해도 이 영향은 되돌려지지 않으므로 [[원격 비밀번호 공격]]의 중단·잠금 분기를 따른다.
- `-o result.txt`는 양성 credential을, debug pipeline은 시도 조합과 서버 응답을 담을 수 있다. 중단된 작업은 현재 작업 디렉터리에 `hydra.restore`를 남길 수 있다.
- 필요한 분석이 끝나면 Hydra를 실행한 Linux host에서 이번 작업이 만든 정확한 파일만 삭제한다. 작업 디렉터리에 다른 파일이 있으면 원인을 확인하고 디렉터리 제거를 중단한다.

```bash
cd -- "$HYDRA_ORIGINAL_WORKDIR"
rm -f -- "$HYDRA_WORKDIR/result.txt" "$HYDRA_WORKDIR/hydra-ftp.log" "$HYDRA_WORKDIR/hydra.restore"
rmdir -- "$HYDRA_WORKDIR"
test ! -e "$HYDRA_WORKDIR"
```

`rmdir` 실패 시 먼저 지정 경로가 이번 작업의 고유 디렉터리인지와 남은 파일의 소유자·용도를 확인한다. 파일명 pattern이나 process 이름으로 일괄 삭제·종료하지 않는다.

## 관련 공격기법

- [[원격 비밀번호 공격]]

## 참고 링크

- [THC Hydra 전체 사용 법과 option](https://github.com/vanhauser-thc/thc-hydra/blob/master/hydra.1)
