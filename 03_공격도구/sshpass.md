---
tags:
  - 서비스/ssh
  - 기능/프로토콜접근
실행환경: ["Linux"]
필요조건: ["SSH 비밀번호"]
결과: ["비대화형 인증 결과", "세션", "파일"]
---

# sshpass

## 도구 개요

`sshpass`는 `ssh`·`scp`·`rsync` 같은 하위 명령의 대화형 비밀번호 프롬프트에 값을 전달해 비대화형 실행을 가능하게 한다. 비밀번호 기반 작업의 짧은 자동화에 적합하지만 명령행·프로세스 목록·shell history에 평문 비밀번호가 노출될 수 있다.

## 필요한 입력과 실행 환경

- 실행 환경: `sshpass`와 SSH 계열 client가 설치된 Linux 호스트
- 입력: SSH 비밀번호와 함께 실행할 `ssh`, `scp` 또는 `rsync` 명령


## 표준 사용법

```bash
sshpass -p '<password>' ssh <user>@<target>
```

## 대표 예시

### SSH 비밀번호 입력 자동화

```bash
sshpass -p '<PASSWORD>' ssh user@<TARGET>
```

### SCP 파일 전송 자동화

```bash
sshpass -f password.txt scp file.txt user@<TARGET>:/tmp/
```

### 환경 변수로 비밀번호 전달

```bash
SSHPASS='<PASSWORD>' sshpass -e ssh user@<TARGET>
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-p` | 명령행에서 비밀번호 지정 |
| `-f` | 파일에서 비밀번호 읽기 |
| `-e` | `SSHPASS` 환경 변수에서 비밀번호 읽기 |
| `-P` | 다른 password prompt 문자열 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SSH/SCP/rsync 명령이 비대화식으로 완료 | 비밀번호 자동 입력 성공 | 같은 credential을 필요한 파일 전송/명령 실행 흐름에 적용 |
| `Permission denied` | 비밀번호 또는 사용자 오류 | 동일 credential을 수동 SSH로 재확인 |
| host key 확인에서 멈춤 | 최초 접속 확인 프롬프트 영향 | `StrictHostKeyChecking` 처리 또는 수동 접속으로 known_hosts 등록 |
| 명령 인자 노출 우려 | 프로세스 목록에 비밀번호가 보일 수 있음 | 임시 사용 후 히스토리/프로세스 노출을 최소화 |

## 관련 공격기법

- [[SSH credential 및 키 인증 검증]]
- [[상황별 파일 전송]]
