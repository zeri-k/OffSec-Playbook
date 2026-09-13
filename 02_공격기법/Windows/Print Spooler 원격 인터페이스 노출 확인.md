---
tags:
  - 환경/ad
  - 서비스/rpc
시작조건: ["대상 Windows 호스트와 FQDN 식별", "Windows PowerShell 실행 위치 확보", "명령 실행 호스트에서 대상 RPC 접근 가능"]
필요권한: ["SecurityAssessment.ps1를 실행할 수 있는 현재 Windows 사용자 권한", "원격 인터페이스 조회에 사용할 도메인 사용자 세션"]
필요조건: ["SecurityAssessment.ps1", "대상 컴퓨터 FQDN"]
결과: ["대상별 Print Spooler 원격 인터페이스 응답 여부", "Printer Bug 인증 강제 후보"]
---

# Print Spooler 원격 인터페이스 노출 확인

## 한 줄 판단

도메인 사용자 세션이 있는 Windows PowerShell 호스트에서 RPC로 도달 가능한 대상 FQDN에 `Get-SpoolStatus`를 실행하여 Print Spooler 원격 인터페이스 응답 여부와 Printer Bug 인증 강제 후보를 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | `SecurityAssessment.ps1`를 읽고 PowerShell 함수를 실행할 수 있는 Windows 호스트 | 모듈 파일 경로와 현재 PowerShell 세션 확인 | 파일 경로, 실행 정책과 모듈 로드 오류를 먼저 구분 |
| 네트워크 경로 | 명령 실행 호스트에서 대상 FQDN의 RPC 원격 인터페이스에 도달 가능 | 이름 해석과 RPC 서비스 도달성 확인 | DNS 결과, 대상 주소와 RPC 경로를 재확인 |
| 현재 계정 또는 인증 수단 | 원격 조회에 사용할 현재 도메인 사용자 세션 | `whoami`로 실행 Identity를 확인하고 함수의 인증 오류 여부 확인 | 로컬 계정과 도메인 계정을 구분하고 사용할 도메인 세션을 다시 선택 |
| 현재 권한 | 현재 사용자로 모듈을 불러오고 원격 상태를 조회할 수 있음 | `Import-Module`과 함수 호출이 각각 성공하는지 확인 | 로컬 모듈 실행 실패와 대상의 원격 조회 거부를 분리 |
| 공격 대상의 조건 | DNS·AD 열거에서 식별한 Windows 호스트의 정확한 FQDN | `ComputerName`이 요청한 FQDN과 일치하는지 확인 | 짧은 호스트명, FQDN과 다른 DNS 레코드를 교차 확인 |

## 실행

도메인 사용자 세션이 있는 Windows PowerShell 호스트에서 모듈을 불러온 뒤, 식별한 대상 FQDN 하나씩 원격 인터페이스 응답을 조회한다.

```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName <TARGET_FQDN>
```

`<TARGET_FQDN>`은 DNS·AD 열거에서 확인한 대상의 FQDN(예: `server01.example.test`)이며, 짧은 이름이나 다른 DNS 별칭으로 바꾸지 않는다.

확인할 출력:

- `ComputerName`이 요청한 대상과 일치하는지 확인한다.
- `Status`가 `True`인지 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 대상 FQDN과 `Status True` | Spooler 원격 인터페이스가 응답하는 인증 강제 후보 | Printer Bug 후보 | [[NTLM Relay 조건 검토]]에서 relay 대상·수신 서비스·계정 권한을 별도 확인 |
| 대상 FQDN과 `Status False` | 이 조회에서는 대상의 원격 인터페이스 응답을 확인하지 못함 | Printer Bug 후보 미확인 | 대상 FQDN과 RPC 도달성이 정상인지 확인한 뒤 대상 상태를 다시 판정 |
| `Import-Module` 오류 | 대상 상태를 조회하기 전에 로컬 모듈 로드가 실패함 | 대상 상태 미조회 | 파일 경로, 스크립트 실행 조건과 함수 존재 여부를 확인 |
| 함수가 결과를 반환하지 않거나 RPC·이름 해석 오류 | 대상 응답이 아니라 네트워크 경로 또는 이름 확인 단계가 실패함 | Printer Bug 가능성 미판정 | FQDN, DNS 결과와 RPC 도달성을 재확인 |
| 함수가 접근 거부 또는 인증 오류를 반환 | 현재 계정으로 원격 조회가 허용되지 않음 | 조회 권한 미확인 | 실행 Identity와 도메인 세션을 확인하고 네트워크 실패와 구분 |
| `True`지만 relay 보호 조건을 모름 | 인증 강제와 relay 성공은 별개 | coercion 후보만 확인 | signing, LDAP/HTTP 보호와 listener 도달성을 확인 |
| PrintNightmare 가능성도 검토 중 | Printer Bug와 RCE 취약성은 다른 판단 | 별도 취약성 후보 | [[PrintNightmare 원격 코드 실행]]의 전제 조건과 rpcdump 결과를 별도로 확인 |

## 확인할 출력과 권한

- `Status True`는 인증 강제 후보라는 의미이며 NTLM relay, 권한 상승 또는 원격 코드 실행 성공을 뜻하지 않는다.
- `Status False`, 무응답과 접근 거부를 같은 결과로 취급하지 않는다. `False`는 함수가 반환한 판정이고, 무응답·오류는 대상 상태를 확정하지 못한 결과다.
- Printer Bug와 PrintNightmare를 같은 취약점이나 성공 조건으로 합치지 않는다.
- 실제 인증 강제와 relay를 수행하기 전 대상 계정, relay endpoint와 변경 영향을 다시 확인한다.

## 관련 서비스

- [[RPC와 NetBIOS 서비스]]
- [[SMB 서비스]]

## 관련 도구

- [[SecurityAssessment.ps1]]

## 관련 공격기법

- [[NTLM Relay 조건 검토]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
