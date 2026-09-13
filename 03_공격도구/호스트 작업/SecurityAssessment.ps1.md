---
tags:
  - 환경/ad
  - 서비스/rpc
  - 기능/열거
실행환경: ["Windows PowerShell"]
필요권한: ["현재 도메인 사용자 세션 권한"]
필요조건: ["SecurityAssessment.ps1", "대상 컴퓨터 FQDN"]
결과: ["Print Spooler 원격 인터페이스 상태"]
---

# SecurityAssessment.ps1

## 도구 개요

`SecurityAssessment.ps1`의 `Get-SpoolStatus`는 원격 Windows 호스트의 Print Spooler RPC 인터페이스 응답 여부를 확인한다. Printer Bug 인증 강제 후보를 빠르게 선별할 때 사용하며, `Status True`만으로 취약점이나 relay 성공을 확정하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 RPC 서비스에 접근 가능한 Windows PowerShell
- 필요한 입력: `SecurityAssessment.ps1` 파일과 대상 컴퓨터 FQDN
- 권한 조건: 현재 도메인 사용자 세션에서 함수가 원격 상태를 확인할 수 있어야 함
- `<TARGET_FQDN>`은 spooler/RPC target, `<DC_FQDN>`은 domain controller FQDN이며 둘은 다른 역할일 수 있다. `Status: True`는 함수의 확인 결과이지 relay·인증·권한 상승 결과가 아니다.

## 표준 사용법

`<TARGET_FQDN>`은 remote spooler/RPC target, `<DC_FQDN>`은 domain controller FQDN이며 PowerShell 실행 host에서 각각 해석한다. 둘은 같은 host일 필요가 없고 function의 `Status` output은 remote state 확인 결과일 뿐 relay·authentication·권한 변경 결과가 아니다.

```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName <TARGET_FQDN>
```

## 대표 예시

```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName <DC_FQDN>
```

확인할 출력:

- `ComputerName`과 `Status` 열.
- 요청한 대상의 `Status`가 `True`인지 여부.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-ComputerName <FQDN>` | 확인할 원격 Windows 호스트 | 대상별 Spooler 상태 조회 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Status True` | Printer Bug 인증 강제 후보 | relay 조건과 listener 도달성을 별도 확인 |
| 결과 없음·오류 | FQDN·모듈·RPC 경로 또는 대상 상태 미확정 | 이름 해석, 모듈 로드와 네트워크 경로 확인 |

## 관련 공격기법

- [[Print Spooler 원격 인터페이스 노출 확인]]
