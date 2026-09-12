---
tags:
  - 서비스/smtp
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["SMTP 접근", "사용자 목록", "RCPT 모드 사용 시 도메인"]
결과: ["유효 사용자 후보", "SMTP 응답"]
---

# smtp-user-enum

## 도구 개요

`smtp-user-enum`은 SMTP의 `VRFY`, `EXPN`, `RCPT` 명령에 대한 응답 차이로 유효한 사용자 후보를 찾는 열거 도구다. 서버가 사용자 존재 여부를 구분해 응답하는지 여러 방식으로 비교할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: `smtp-user-enum`이 설치된 Linux 호스트
- 입력: SMTP 서버와 사용자 목록
- 선택 입력: `VRFY`, `EXPN`, `RCPT` 방식, 포트와 도메인


## 표준 사용법

```bash
smtp-user-enum -M <method> -U <user_list> -t <target>
```

## 대표 예시

### VRFY로 SMTP 사용자 열거

```bash
smtp-user-enum -M VRFY -U users.txt -t <TARGET> -m 1
```

확인할 출력:

- 명백히 존재하지 않는 기준 사용자와 후보 사용자별 valid/invalid 응답 차이.
- 먼저 단일 process로 응답과 rate limit을 확인하고, 병렬 수는 허가 범위와 서버 제한을 확인한 뒤에만 조정한다.

### RCPT 모드로 도메인 포함 열거

```bash
smtp-user-enum -M RCPT -U userlist.txt -D <DOMAIN> -t <TARGET>
```

확인할 출력:

- `<user>@<DOMAIN> exists` 형태의 유효 계정 후보.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-M` | 열거 방식. `VRFY`, `EXPN`, `RCPT` 등 |
| `-u`, `-U` | 단일 사용자 또는 사용자 목록 |
| `-t`, `-T` | 단일 대상 또는 대상 목록 |
| `-m` | 최대 프로세스 수 |
| `-w` | 응답 대기 시간 |
| `-p` | 포트 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| valid user / user exists | SMTP 반응으로 유효 사용자 후보 확인 | 가짜 사용자 기준값과 다른 방식으로 응답 차이 재확인 |
| rejected / user unknown | 해당 사용자가 없거나 명령이 차단됨 | 다른 사용자 목록과 `VRFY`, `EXPN`, `RCPT` 모드 비교 |
| 모든 사용자가 valid처럼 보임 | catch-all 또는 동일 응답 정책 가능 | 응답 코드, 응답 길이, 명백히 가짜 사용자로 기준값 확인 |
| timeout / connection closed | SMTP 명령 제한, STARTTLS, rate limit 가능 | 포트, TLS, 연결 속도, 명령 모드 확인 |
| 모든 후보가 거부됨 | SMTP 명령 비활성화 또는 사용자 형식 불일치 | 수동 SMTP 대화와 계정명·전체 주소 형식을 비교 |

## 관련 공격기법

- [[SMTP 사용자 열거]]

## 참고 링크

- [smtp-user-enum 공식 사용자 문서](https://pentestmonkey.net/tools/smtp-user-enum/smtp-user-enum-user-docs.pdf)
