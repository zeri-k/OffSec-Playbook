---
tags:
  - 환경/linux
  - 환경/unix
문서역할: 수동절차
시작조건: ["제한된 Linux 셸 확보"]
필요권한: ["제한된 셸 세션"]
필요조건: ["Python 또는 script 사용 가능", "로컬 터미널 제어 가능"]
결과: ["대화형 TTY"]
---

# TTY 업그레이드

## 한 줄 판단

공격 호스트의 터미널에서 Linux Reverse Shell 또는 Bind Shell을 제어하고 있지만 `tty`가 `not a tty`를 반환한다면, 원격 pseudo-terminal과 로컬 터미널 모드를 맞춰 같은 사용자 권한의 안정적인 대화형 셸로 바꾼다. 이 과정만으로 사용자 권한은 상승하지 않는다.

## 사용할 때

- 제한된 Reverse Shell 또는 Bind Shell에서 작업 제어와 대화형 입력이 필요할 때.
- `sudo`, 편집기, 전체 화면 프로그램이 TTY 부족으로 정상 동작하지 않을 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 제한된 셸 | `echo $0`, `tty` | `not a tty` 또는 작업 제어 불가 |
| 인터프리터 | `command -v python3 python script` | 하나 이상 사용 가능 |
| 로컬 터미널 | `stty size`, `echo $TERM` | 행/열과 TERM 확인 가능 |

## 실행

1. 원격 셸에서 pseudo-terminal을 생성한다.
2. 셸을 백그라운드로 보내고 로컬 터미널을 raw mode로 전환한다.
3. 셸을 다시 foreground로 가져온 뒤 `TERM`과 터미널 크기를 맞춘다.
4. `sudo -l`, 편집기, 작업 제어가 정상 동작하는지 확인한다.

### 명령과 확인할 출력

#### 원격 셸

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Python이 없으면
script -qc /bin/bash /dev/null
```

`Ctrl+Z`로 로컬 터미널로 돌아간다.

#### 로컬 터미널

```bash
stty raw -echo
fg
```

화면이 비어 있으면 `Enter`를 누른 뒤 원격 셸에서 다음을 실행한다.

```bash
reset
export TERM=xterm-256color
stty rows <ROWS> columns <COLUMNS>
```

로컬 터미널 크기는 별도 창에서 확인한다.

```bash
echo $TERM
stty size
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `tty`가 `/dev/pts/<N>` 형태를 반환한다. | PTY 할당 확인 | 대화형 TTY | 터미널 크기와 `TERM` 값을 맞춘 뒤 입력 제어 확인 |
| 방향키, 명령 기록, `Ctrl+C`, `sudo -l`이 정상 동작한다. | 대화형 제어 확인 | 안정화된 TTY | [[상황별 파일 전송]] 또는 로컬 권한 상승 열거로 전환 |
| Python 없음 | 조건 미충족 | 시작 상태 유지 | `script`, `socat`, SSH 전환 가능성 확인 |
| 화면이 깨짐 | 조건 미충족 | 시작 상태 유지 | `reset`, `export TERM=xterm`, 행/열 재설정 |
| 로컬 터미널 입력이 보이지 않음 | 조건 미충족 | 시작 상태 유지 | 새 로컬 터미널에서 `stty sane` 실행 |
| 셸이 즉시 종료됨 | 조건 미충족 | 시작 상태 유지 | 현재 셸 프로세스와 listener 안정성 확인 |

## 확인할 출력과 권한

- 판정 기준: `tty`가 `/dev/pts/<N>`을 반환하고 작업 제어·`sudo -l`이 실제로 동작하는지 확인한다.

## 관련 공격기법

- [[Reverse Shell 획득]]
- [[Bind Shell 획득]]
- [[상황별 파일 전송]]

## 참고 링크

- https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/
