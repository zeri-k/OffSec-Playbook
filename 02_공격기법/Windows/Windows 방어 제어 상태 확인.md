---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트에서 명령 실행 또는 셸 확보", "현재 실행 Identity 확인 가능"]
필요권한: ["현재 Windows 사용자에게 허용된 방어 상태 조회 권한"]
필요조건: ["대상 호스트에서 CMD 또는 PowerShell 실행", "각 제어의 조회 명령 사용 가능"]
결과: ["조회 가능한 Defender 구성 요소 상태", "Windows Firewall 프로필별 정책", "조회 가능한 AppLocker 유효 정책", "현재 PowerShell 런스페이스의 LanguageMode"]
문서역할: 수동절차
---

# Windows 방어 제어 상태 확인

## 한 줄 판단

대상 Windows 호스트의 현재 사용자 세션에서 Defender 구성 요소, 방화벽 프로필, AppLocker 유효 정책과 현재 PowerShell 런스페이스의 LanguageMode를 각각 조회하여, 보이는 제약 범위 안에서 실행 가능한 후속 열거 방법을 선택한다.

## 사용할 때

- 현재 보유 정보: 대상 Windows 호스트에서 셸이나 원격 명령 실행을 확보했고 현재 사용자와 호스트명을 확인할 수 있다.
- 명령 실행 위치와 도달성: 방어 상태를 평가할 대상 호스트의 CMD 또는 PowerShell에서 로컬 조회를 실행한다. 별도 원격 관리 포트 도달성은 이 절차의 결과가 아니다.
- 현재 계정과 권한: 현재 Windows 사용자에게 보이는 범위만 조회한다. 관리자 권한이 없을 때의 접근 거부와 제어 비활성을 구분한다.
- 지금 가능한 행동과 결과: 같은 도메인 안에서도 호스트별로 다른 파일·스크립트 실행 제약을 확인한다. 한 제어의 상태나 조회 실패만으로 전체 방어 제품의 존재·부재를 확정하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 방어 상태를 평가할 대상 Windows 호스트의 세션 | `whoami`, `hostname` 응답 | 다른 호스트나 원격 공격 호스트의 상태와 섞지 않음 |
| 현재 계정 | 로컬·도메인 사용자와 현재 프로세스 토큰 식별 | `whoami`와 현재 셸 확인 | 계정 범위와 elevated 여부를 별도 확인 |
| 현재 조회 권한 | 각 제어의 상태 또는 명시적 접근 거부를 관찰 가능 | 명령별 정상 출력·접근 거부·cmdlet 부재 구분 | 권한 부족을 제어 비활성으로 해석하지 않음 |
| 실행 환경 | CMD와 PowerShell 중 하나 이상, 필요한 cmdlet 사용 가능 | 셸 종류, PowerShell 버전과 cmdlet 존재 확인 | CMD 대체 조회와 PowerShell 조회 결과를 비교 |
| 공격 대상의 조건 | Defender·방화벽·AppLocker·PowerShell 중 실제 설치·적용된 제어 식별 | 서비스, 구성 요소와 유효 정책 출력 교차 확인 | 타사 AV·EDR, WDAC 등 이 명령이 다루지 않는 제어를 별도 상태로 유지 |

## 실행

### 1. Defender 서비스와 보호 상태 확인

```cmd
sc query windefend
```

확인할 출력:

- `STATE : 4 RUNNING`이면 Defender 서비스가 실행 중이다.
- 서비스 미존재나 중지 상태만으로 다른 보안 제품 또는 전체 보호 부재를 단정하지 않는다.

```powershell
Get-MpComputerStatus | Select-Object AMServiceEnabled,AntivirusEnabled,RealTimeProtectionEnabled,BehaviorMonitorEnabled,IoavProtectionEnabled,IsTamperProtected,DefenderSignaturesOutOfDate
```

확인할 출력:

- 실시간 보호, 동작 모니터링, 다운로드·첨부 파일 검사, 변조 방지와 서명 최신 상태를 각각 확인한다.
- cmdlet 오류는 Defender 부재, 모듈 사용 불가 또는 권한 제한을 구분해 기록한다.

### 2. Windows Firewall 유효 프로필 확인

```cmd
netsh advfirewall show allprofiles
```

확인할 출력:

- Domain, Private, Public 각 프로필의 `State`와 기본 inbound/outbound 정책.
- 현재 연결에 적용되는 프로필과 전체 프로필 설정을 혼동하지 않는다.

### 3. AppLocker와 PowerShell 제약 확인

```powershell
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
$ExecutionContext.SessionState.LanguageMode
```

확인할 출력:

- AppLocker 규칙의 `Action`, 적용 SID와 경로·게시자·해시 조건.
- `FullLanguage` 또는 `ConstrainedLanguage` 등 현재 PowerShell 세션의 실제 LanguageMode.
- 규칙이 출력되지 않거나 cmdlet이 없으면 AppLocker가 비활성이라고 즉시 단정하지 말고 서비스, 정책 적용과 조회 권한을 분리한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| Defender 서비스와 실시간 보호가 활성화됨 | Defender의 조회된 구성 요소가 활성 상태 | 방어 제어 활성 | 운영체제 기본 명령과 필요한 최소 도구만 선택하고 현재 Windows 상태 라우터 유지 |
| Defender 서비스는 실행 중이나 일부 보호가 비활성화됨 | 구성 요소별 보호 수준이 다름 | 부분 활성 방어 상태 | 비활성 항목을 전체 보호 부재로 확대 해석하지 않고 다른 제어도 확인 |
| 방화벽 프로필이 활성이고 inbound 기본 정책이 차단임 | 새 수신 연결이 제한될 수 있음 | 네트워크 실행 제약 확인 | 기존 outbound 경로와 열려 있는 관리 채널을 기준으로 후속 기법 선택 |
| 현재 적용 SID에 대한 AppLocker `Deny` 규칙이 보임 | 해당 조건의 파일 또는 스크립트 실행 제한 | 애플리케이션 실행 제약 확인 | 실행 가능한 운영체제 기본 명령으로 열거하고 정책 위반 실행은 피함 |
| `ConstrainedLanguage`가 반환됨 | 현재 PowerShell 런스페이스의 기능 제한 | PowerShell 실행 제약 확인 | [[AD 도메인 컨텍스트 기본 확인]] 또는 [[Windows 권한 상승 열거]]의 기본 명령 경로 우선 |
| 명령이 접근 거부로 실패함 | 현재 계정의 조회 가시성이 부족함 | 해당 제어 상태 미확정 | 현재 토큰을 기록하고 권한이 덜 필요한 CMD·PowerShell 대체 조회와 비교 |
| cmdlet 또는 서비스가 존재하지 않음 | 구성 요소 미설치, 제거, 다른 제품 사용 또는 PowerShell 모듈 부재 가능 | 해당 제어 상태 미확정 | 서비스 목록, 모듈 존재와 타사 방어 제품을 별도로 확인 |
| AppLocker 규칙이 비어 있거나 반환되지 않음 | 유효 규칙 부재 또는 조회 실패 가능 | AppLocker 상태 미확정 | AppLocker 서비스·정책 적용 여부와 조회 권한을 구분 |

## 확인할 출력과 권한

- 서비스 실행 상태, 개별 보호 기능, 유효 정책과 현재 세션 LanguageMode를 별도 증거로 취급한다.
- `netsh advfirewall show allprofiles`는 프로필별 설정을 보여 주며 현재 네트워크에 실제 적용된 프로필과 원격 포트 도달성을 자동으로 확정하지 않는다.
- AppLocker 유효 정책 출력은 반환된 SID·규칙 조건의 범위다. WDAC나 다른 애플리케이션 제어 정책의 부재를 뜻하지 않는다.
- `LanguageMode`는 현재 PowerShell 런스페이스의 상태이며 다른 사용자, 새 프로세스 또는 다른 PowerShell 호스트의 모드를 확정하지 않는다.
- 한 제어의 비활성 또는 조회 실패로 EDR, 타사 백신, WDAC 등 다른 제어가 없다고 단정하지 않는다.
- 이 절차는 상태 조회만 수행하며 Defender, 방화벽, AppLocker 또는 PowerShell 정책을 변경하지 않는다.

## 관련 도구

- [[powershell]]
- [[netsh]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 참고 링크

- [Microsoft: Get-MpComputerStatus](https://learn.microsoft.com/powershell/module/defender/get-mpcomputerstatus)
- [Microsoft: Get-AppLockerPolicy](https://learn.microsoft.com/powershell/module/applocker/get-applockerpolicy)
- [Microsoft: about_Language_Modes](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_language_modes)
- [Microsoft: netsh advfirewall](https://learn.microsoft.com/windows-server/administration/windows-commands/netsh-advfirewall)
