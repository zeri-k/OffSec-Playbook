---
tags:
  - 환경/windows
문서역할: 수동절차
시작조건: ["Windows PowerShell 세션 확보", "Defender 실시간 보호가 실행을 차단하는 상태 확인"]
필요권한: ["Defender 설정을 변경할 수 있는 상승된 로컬 관리자 권한"]
필요조건: ["Set-MpPreference cmdlet 사용 가능", "변경 전 실시간 보호 상태 확인"]
결과: ["Defender 실시간 보호 비활성화 상태", "변경 뒤 파일·프로세스 실행 결과"]
---

# Windows Defender 실시간 보호 비활성화

## 한 줄 판단

상승된 Windows PowerShell에서 Defender 실시간 보호가 파일·프로세스 실행을 차단하고 설정 변경 권한이 있으면 변경 전 상태를 기록한 뒤 `Set-MpPreference`로 일시 비활성화하고 필요한 실행 후 원래 상태로 복구한다.

## 사용할 때

- 파일 반입은 완료됐지만 Defender 실시간 검사에서 파일 삭제·격리·실행 차단이 확인될 때.
- 현재 PowerShell이 상승된 로컬 관리자 token으로 실행 중일 때.
- Tamper Protection이나 조직 정책이 변경을 차단하는지 구분해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 token | 상승된 로컬 관리자 PowerShell | `whoami /all` | 관리자 그룹과 상승 상태를 구분 |
| Defender 상태 | 실시간 보호 상태와 정책 관리 여부 확인 | `Get-MpPreference`, `Get-MpComputerStatus` | 다른 AV·EDR 제품과 구분 |
| 차단 근거 | Defender가 대상 파일·프로세스를 실제 차단 | Protection History와 파일 상태 | ACL·MOTW·AppLocker 오류와 구분 |
| 복구 기준값 | 변경 전 `DisableRealtimeMonitoring` 값 기록 | `Get-MpPreference` 출력 저장 | 기준값을 모르면 변경하지 않음 |

## 실행

변경 전 현재 계정과 설정을 확인한다.

```powershell
whoami /all
Get-MpPreference | Select-Object DisableRealtimeMonitoring
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled,AntivirusEnabled
```

상승된 PowerShell에서 실시간 보호를 비활성화한다.

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
Get-MpPreference | Select-Object DisableRealtimeMonitoring
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled
```

확인할 출력:

- `DisableRealtimeMonitoring` 값과 `RealTimeProtectionEnabled` 변화.
- 접근 거부, Tamper Protection 또는 Group Policy 재적용으로 값이 유지·복원되는지.
- 설정 변경 성공과 대상 파일·프로세스 실행 성공을 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 실시간 보호 값이 비활성 상태로 바뀜 | Defender 설정 변경 성공 | 실시간 보호 비활성화 | 필요한 단일 실행 후 즉시 복구 |
| 값이 바뀌지 않거나 곧 복원됨 | Tamper Protection·GPO·관리 제품이 설정을 강제함 | 방어 기능 유지 | 정책과 오류를 확인하고 반복 변경하지 않음 |
| 설정은 바뀌었으나 파일 실행은 계속 차단 | 다른 Defender 기능·EDR·AppLocker·MOTW 가능 | 차단 원인 미확정 | 실제 오류와 이벤트를 기준으로 원인 분리 |
| 대상 실행 완료 | 필요한 작업 완료 | 실행 결과 확보 | 변경 전 상태로 복구하고 Defender 상태 재확인 |

## 확인할 출력과 권한

- PowerShell 명령 성공, Defender 상태 변화와 대상 파일·프로세스 실행 결과를 각각 기록한다.
- 실시간 보호 비활성화는 SYSTEM·도메인 권한 상승을 뜻하지 않는다.

## 변경 영향과 복구

변경 전 값이 실시간 보호 활성 상태였다면 작업 직후 다시 활성화한다.

```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
Get-MpPreference | Select-Object DisableRealtimeMonitoring
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled
```

- 변경 전부터 비활성 상태였다면 임의로 활성화하지 않고 기록한 기준값으로 복구한다.
- 정책이 상태를 다시 적용하는 경우 로컬 명령 결과와 최종 적용 상태를 구분한다.

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
