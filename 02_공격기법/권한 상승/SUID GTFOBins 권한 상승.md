---
tags:
  - 환경/linux
시작조건: ["Linux shell", "SUID 실행 파일 후보"]
필요권한: ["SUID binary 실행 권한"]
필요조건: ["Linux shell"]
결과: ["root shell", "파일 읽기/쓰기", "명령 실행"]
---

# SUID GTFOBins 권한 상승

## 한 줄 판단

현재 Linux 셸의 계정으로 실행 가능한 SUID 프로그램이 있고 그 프로그램에 셸 실행·파일 읽기·파일 쓰기 기능이 있다면, 프로그램 소유자의 effective user ID로 root 셸 또는 제한된 고권한 기능을 얻는다.

## 사용할 때

- `find / -perm -4000` 결과에 비표준 SUID binary가 있을 때.
- `find`, `bash`, `vim`, `cp`, custom binary처럼 GTFOBins 후보가 보일 때.
- root 소유 SUID 파일이 writable path나 취약한 호출 경로를 사용할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| SUID 파일 | `find / -perm -4000` | root 소유 실행 파일 |
| 악용 기능 | GTFOBins/동작 확인 | shell/read/write 가능 |
| 실행 권한 | 현재 사용자 실행 가능 | SUID bit 유지 |

## 실행

### 방식 선택

| 확인된 SUID 프로그램 | 선택할 실행 | 필요한 입력 | 성공 결과 |
|---|---|---|---|
| `/usr/bin/find`가 root 소유 SUID | `find -exec` 경로 | 실행 가능한 SUID `find`의 정확한 경로 | `find`의 effective UID를 유지한 셸 |
| `/bin/bash`가 root 소유 SUID | `bash -p` 경로 | 실행 가능한 SUID `bash`의 정확한 경로 | privilege mode를 유지한 셸 |
| 셸 기능이 없는 다른 SUID 프로그램 | 해당 프로그램의 GTFOBins 기능 | 프로그램 이름, 소유자, SUID bit와 허용 기능 | 프로그램 소유자 권한의 파일 읽기·쓰기 또는 단일 명령 실행 |

아래 예시는 발견한 모든 명령을 차례로 실행하는 목록이 아니다. `find`가 SUID일 때는 `find` 예시만, `bash` 자체가 SUID일 때는 `bash -p` 예시만 선택한다.

### SUID 탐색

```bash
find / -perm -4000 -type f -ls 2>/dev/null
```

확인할 출력:

- root 소유 SUID binary와 비표준 경로.

### SUID `find`로 셸 실행

실행 위치: SUID 후보를 발견한 대상 Linux 셸.

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

확인할 출력:

- `id`의 effective UID가 SUID `find` 소유자와 같은지 확인한다.
- 셸은 열렸지만 effective UID가 바뀌지 않으면 `/usr/bin/find`의 소유자·SUID bit와 하위 셸의 privilege 유지 여부를 확인한다.

### SUID `bash`로 privilege mode 유지

실행 위치: `/bin/bash` 자체가 SUID 후보로 확인된 대상 Linux 셸.

```bash
/bin/bash -p
```

확인할 출력:

- `id`의 `euid=0` 또는 SUID `bash` 소유자의 effective UID.
- 일반 `/bin/bash`에 `-p`만 지정해도 권한은 상승하지 않는다. `ls -l /bin/bash`에서 소유자와 SUID bit가 확인되지 않으면 이 방식을 사용하지 않는다.

### 다른 SUID 프로그램

프로그램의 실제 버전과 GTFOBins의 `SUID` 항목을 대조해 셸·파일 읽기·파일 쓰기 중 확인된 기능 하나만 실행한다. 실행 후에는 `id` 또는 접근할 수 있게 된 정확한 파일로 결과를 확인하며, 기능이 있다는 이유만으로 root 셸을 얻었다고 기록하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `id`에 `euid=0`이 표시되거나 root 전용 명령이 실행됨 | SUID 소유자 권한으로 실행됨 | root shell 또는 root 명령 실행 | 현재 권한과 접근 범위를 다시 확인 |
| shell은 없지만 root 전용 파일 읽기·쓰기가 됨 | binary 기능 범위 안에서 권한 상승 효과가 있음 | 고권한 파일 읽기·쓰기 | 자격 증명 수집 또는 최소 변경으로 영향 확인 |
| 실행 후 UID가 바뀌지 않음 | binary가 권한을 내려놓거나 호출 옵션이 맞지 않음 | 기존 일반 사용자 shell 유지 | `-p`와 GTFOBins의 정확한 호출 조건 확인 |
| 표준 SUID만 보임 | 즉시 사용할 후보가 없음 | 권한 상승 경로 미확보 | sudo, cron, 자격 증명 경로로 전환 |
| 비표준 custom binary가 보임 | 내부 호출과 경로 분석이 필요함 | SUID 악용 후보 확보 | `strings`, `ltrace`, `strace`로 PATH·파일 호출 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| SUID 소유권 | `find` 결과의 root 소유자와 `-rws` 비트 | 현재 사용자가 실행할 수 있는 root SUID 후보인지 판단 |
| 실행 후 identity | `id`, `whoami`, effective UID | real UID가 아니라 effective UID와 실제 파일 접근 권한으로 상승 여부 확정 |
| 제한 기능 | root 전용 파일 읽기·쓰기 또는 명령 결과 | shell 획득과 제한된 고권한 기능을 구분 |

## 후속 공격 연결

- [[Linux 파일 자격증명 검색]]
- [[Linux Shell History 자격증명 검색]]
- [[Linux 개인키 검색]]
- [[상황별 파일 전송]]

## 관련 상태 라우터

- root 실행 컨텍스트를 확인했으면: [[고권한 세션 확보 후 후속 판단]]
- 다른 사용자 셸 또는 제한된 명령 실행만 확보했으면: [[Linux 셸 확보 후 초기 열거와 권한 상승]]

## 관련 도구

- [[linpeas]]
