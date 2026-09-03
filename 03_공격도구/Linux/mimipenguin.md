---
tags:
  - 환경/linux
  - 기능/자격증명수집
실행환경: ["Linux"]
필요권한: ["root 권한 또는 높은 로컬 권한"]
필요조건: ["대상 Linux 환경과 호환되는 mimipenguin script 또는 binary"]
결과: ["자격증명", "평문 비밀번호"]
---

# mimipenguin

## 도구 개요

`mimipenguin`은 Linux의 현재 로그온 세션, 프로세스 메모리와 관련 파일에 남은 평문 사용자명·비밀번호 후보를 찾는 도구다. 메모리에 잔존한 로그인 자격 증명 흔적을 점검하는 데 특화되어 있다.

## 필요한 입력과 실행 환경

- 실행 위치: 지원되는 Linux 호스트의 로컬 shell
- 권한 조건: 프로세스 메모리와 credential store 접근을 위해 root 또는 높은 로컬 권한
- 필요한 입력: 대상 환경에 맞는 script/binary; 출력된 평문은 실제 인증 전까지 후보로 취급한다.


## 표준 사용법

```bash
sudo python3 mimipenguin.py
```

root 권한으로 실행해 메모리와 세션에서 자격증명 후보를 출력한다.

## 대표 예시

### Python3 실행

```bash
sudo python3 mimipenguin.py
```

확인할 출력:

- `[SYSTEM - GNOME] <user>:<password>` 형태의 평문 자격증명 후보

### Bash 버전 실행

```bash
sudo bash mimipenguin.sh
```

확인할 출력:

- 사용자명과 비밀번호 후보

## 주요 옵션과 요소

| 항목 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `mimipenguin.py` | Python 버전 | Python3 사용 가능 환경 |
| `mimipenguin.sh` | Bash 버전 | Python 실행이 제한될 때 |
| `sudo` | 높은 권한으로 실행 | 메모리/세션 접근 필요 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `<user>:<password>` 출력 | 평문 credential 후보 확보 | SSH, sudo, 내부 서비스 재사용 확인 |
| 결과 없음 | 저장된 credential 없음 또는 접근 제한 | 다른 사용자 세션, LaZagne, 수동 파일 검색 |
| permission denied | 권한 부족 | root 권한 확보 후 재실행 |
| 일부 모듈 실패 | 환경/패키지 차이 | 출력 가능한 결과만 검증하고 다른 도구로 보완 |

## 관련 공격기법

- [[Linux 프로세스 메모리 자격증명 수집]]
