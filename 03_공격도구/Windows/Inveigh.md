---
tags:
  - 환경/windows
  - 서비스/llmnr
  - 기능/자격증명수집
실행환경: ["Windows"]
필요권한: ["패킷 캡처와 수신 포트 바인딩이 가능한 로컬 관리자 권한"]
필요조건: ["이름 해석 요청을 관찰할 수 있는 네트워크 위치", "Inveigh.ps1 또는 Inveigh.exe"]
결과: ["이름 해석 요청", "NetNTLM challenge-response", "사용자 및 출발지 정보"]
---

# Inveigh

## 도구 개요

Inveigh는 Windows에서 LLMNR·NBNS·mDNS·WPAD 이름 해석 요청을 관찰하거나 응답해 NetNTLM 인증 자료와 사용자·출발지 정보를 수집한다. PowerShell 구현과 C# 실행 파일은 실행·콘솔 문법이 다르며, 캡처한 NetNTLM challenge-response는 계정의 NTLM hash 자체가 아니다.

## 필요한 입력과 실행 환경

- 실행 환경: 대상 요청을 관찰할 수 있는 Windows 호스트
- 입력: legacy PowerShell 구현의 `Inveigh.ps1` 또는 C# 구현의 `Inveigh.exe`
- 권한 조건: 패킷 캡처와 HTTP/HTTPS/SMB 등의 수신 포트 바인딩이 가능한 로컬 관리자 권한
- 네트워크 조건: LLMNR/NBNS 요청이 도달하는 위치와 응답에 사용할 로컬 인터페이스

## 표준 사용법

### Legacy PowerShell 구현

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

### C# 구현

```powershell
.\Inveigh.exe [options]
```

실행 중 `ESC`로 대화형 콘솔에 들어간 뒤 `GET NTLMV2UNIQUE`, `GET NTLMV2USERNAMES`, `STOP` 같은 콘솔 명령을 사용한다.

## 대표 예시

### Legacy PowerShell로 LLMNR/NBNS 캡처

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

확인할 출력:

- `Inveigh <VERSION> started`, `LLMNR Spoofer = Enabled`, `NBNS Spoofer ... = Enabled`
- `SMB Capture = Enabled`, `HTTP Capture = Enabled`, `File Output = Enabled`
- 종료 안내인 `Run Stop-Inveigh to stop`

### C# 구현을 기본 설정으로 시작

```powershell
.\Inveigh.exe
```

확인할 출력:

- `Packet Sniffer Addresses`, `Listener Addresses`, `Spoofer Reply Addresses`
- `[+]`로 표시된 활성 기능과 `[ ]`로 표시된 비활성 기능
- `Press ESC to enter/exit interactive console`

### C# 콘솔에서 고유 NTLMv2 hash 확인

```text
GET NTLMV2UNIQUE
```

확인할 출력:

- `Unique NTLMv2 Hashes` 표제와 사용자별 하나의 NetNTLMv2 challenge-response

### C# 콘솔에서 사용자와 출발지 확인

```text
GET NTLMV2USERNAMES
```

확인할 출력:

- `IP Address`, `Host`, `Username`, `Challenge` 열

## 주요 옵션과 콘솔 명령

| 구현 | 옵션·명령 | 의미 |
|---|---|---|
| PowerShell | 첫 번째 위치 인수 `Y` | 근거 예시에서 legacy LLMNR spoofing 활성화 |
| PowerShell | `-NBNS Y` | NBNS spoofing 활성화 |
| PowerShell | `-ConsoleOutput Y` | 실시간 콘솔 출력 활성화 |
| PowerShell | `-FileOutput Y` | 캡처 결과 파일 저장 활성화 |
| C# 콘솔 | `GET NTLMV2UNIQUE` | 사용자별 고유 NTLMv2 hash 확인 |
| C# 콘솔 | `GET NTLMV2USERNAMES` | 캡처 사용자와 출발지 호스트·IP 확인 |
| C# 콘솔 | `RESUME` | 실시간 콘솔 출력으로 복귀 |
| C# 콘솔 | `STOP` | C# Inveigh 종료 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `[response sent]` | 해당 이름 해석 요청에 spoofing 응답 전송 | 이후 NTLM 인증과 hash 캡처 여부 확인 |
| `Unique NTLMv2 Hashes` | 중복을 제거한 NetNTLMv2 challenge-response 확보 | [[오프라인 해시 크래킹]] 또는 relay 조건 검토 |
| `NTLMv2 Usernames` | 사용자, 출발지 IP·호스트, challenge 매핑 확보 | 계정 중요도와 출발지 시스템 확인 |
| `[type ignored]` 또는 `[spoofer disabled]` | 요청은 관찰했지만 유형이 제외되었거나 응답 기능 비활성 | 시작 옵션과 응답 대상 프로토콜 확인 |
| `Error starting HTTP listener` | 권한 부족 또는 다른 프로세스와 수신 포트 충돌 | 관리자 권한과 HTTP/HTTPS/SMB 포트 점유 확인 |
| 요청은 보이지만 hash 없음 | 이름 해석 요청 이후 NTLM 인증이 발생하지 않음 | 인증 유도 조건, 캐시, 대상 프로토콜 확인 |

## 버전과 환경 차이

- `Inveigh.ps1`은 PowerShell 구현이며 `Import-Module`, `Invoke-Inveigh`, `Stop-Inveigh` 같은 cmdlet 문법을 사용한다.
- `Inveigh.exe`는 C# 구현이다. 독립 실행 파일로 시작하고 `ESC`로 여는 대화형 콘솔에서 `GET ...`, `RESUME`, `STOP` 명령을 사용한다.
- C# 대화형 콘솔 명령을 legacy PowerShell 세션에 입력하거나 PowerShell cmdlet을 C# 실행 파일 옵션으로 사용하지 않는다.
- 배포 archive는 대상 Windows와 .NET runtime에 맞는 빌드를 선택한다. C# archive에 `Inveigh.ps1`이 없는 것은 두 구현이 별도 산출물이기 때문이다.

## 관련 공격기법

- [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]
- [[NTLM Relay 조건 검토]]
