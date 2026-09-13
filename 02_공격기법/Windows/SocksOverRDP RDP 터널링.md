---
tags:
  - 환경/windows
  - 서비스/rdp
시작조건: ["운영 호스트에서 <PIVOT_IP>:3389/TCP RDP 세션 확보", "피벗 호스트에서 <INTERNAL_IP>:<PORT> TCP 연결 가능"]
필요권한: ["RDP client의 plugin DLL 등록은 상승된 Windows 관리자 권한", "RDP server의 SocksOverRDP 실행은 현재 사용자 권한"]
필요조건: ["Windows mstsc.exe client에서 <PIVOT_IP> RDP 세션 유지", "첫 RDP 피벗에 로그인할 계정", "최종 내부 서비스에 인증할 별도 계정 또는 인증 수단", "RDP 세션에서 파일 전송 또는 도구 실행 가능", "client와 server 아키텍처에 맞는 SocksOverRDP 구성 요소"]
결과: ["RDP client 측 127.0.0.1:1080 SOCKS 프록시", "운영 호스트에서 <INTERNAL_IP>:<PORT>까지의 TCP 경로"]
---

# SocksOverRDP RDP 터널링

## 한 줄 판단

Windows RDP client의 `mstsc.exe`에서 `<PIVOT_IP>:3389`로 로그인할 수 있고 피벗 호스트에서 `<INTERNAL_IP>:<PORT>`에 연결할 수 있으면, RDP Dynamic Virtual Channel을 통해 client 측 `127.0.0.1:1080` SOCKS 프록시와 내부 TCP 경로를 만든다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 운영 호스트→`<PIVOT_IP>:3389`와 피벗→`<INTERNAL_IP>:<PORT>` TCP 연결 가능 | 첫 RDP 접속 후 피벗 세션에서 최종 포트를 확인 | 첫 RDP 방화벽·NLA와 피벗 호스트의 route·대상 포트를 분리 확인 |
| 현재 계정 또는 인증 수단 | 피벗 호스트용 RDP 계정과 최종 내부 서비스용 인증 자료를 역할별로 보유 | `xfreerdp` 또는 `mstsc`로 첫 세션을 열고 최종 계정은 별도 검증 | 계정의 대상 호스트, 도메인, 로그온 유형과 만료 상태 확인 |
| 현재 권한 | client는 상승된 plugin 등록 권한, server는 현재 사용자 실행 권한 | 각 실행 위치에서 `whoami`, client의 상승 여부와 파일 실행 가능 여부 확인 | client의 UAC elevation, 양쪽 x86·x64 아키텍처와 파일 경로 확인 |
| 공격 대상의 조건 | 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` 서비스가 응답 | 피벗 RDP 세션에서 TCP 연결 또는 서비스 클라이언트로 확인 | 내부 DNS·route·방화벽과 대상 listener 확인 |
| 필요한 파일·목록·주소 | client/server 구성 요소와 `<PIVOT_IP>`, `<INTERNAL_IP>`, `<PORT>` | RDP drive redirection 또는 기존 전송 경로로 파일 존재 확인 | 파일 전송 방식과 구성 요소 버전 재확인 |

## 실행

`<PIVOT_IP>`는 첫 RDP 세션의 피벗 Windows 주소, `<INTERNAL_IP>:<PORT>`는 피벗에서만 도달하는 최종 TCP 서비스다. client plugin 경로·server 경로·PID는 각 실행 단계의 출력에서 얻는다. 첫 RDP 계정과 최종 서비스 계정은 역할상 별도 입력이며, 같은 계정을 쓰려면 두 호스트·서비스에서 각각 유효함을 따로 확인한다.

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| RDP만 연결 가능한 Windows 구간 | SSH/Chisel 대안 필요 | SocksOverRDP 검토 |
| plugin 등록 성공 | DVC 플러그인 준비 | RDP 재접속 후 listener 확인 |
| `127.0.0.1:1080` listen | SOCKS proxy 생성 | Proxifier 또는 SOCKS 지원 도구 연결 |

### 절차
1. Windows RDP client에서 기존 plugin registry key·DLL 파일·`127.0.0.1:1080` listener를, 피벗 호스트에서 server 파일과 같은 이름의 기존 프로세스를 확인한다.
2. client에서 plugin key가 없고 이번 작업에 사용할 DLL 경로가 새 파일임을 확인한 경우에만 상승된 명령 프롬프트로 DLL을 등록한다. 기존 key가 있으면 덮어쓰지 말고 등록된 경로·설정을 검토한다.
3. Windows client의 `mstsc.exe`에서 `<PIVOT_IP>:3389`로 RDP 로그인하고 피벗 세션의 현재 Identity를 확인한다.
4. 피벗 호스트의 RDP 세션에서 SocksOverRDP server를 시작하고 반환된 PID·경로·시작 시각을 기록한다.
5. RDP client 측 `127.0.0.1:1080` SOCKS listener의 owning PID를 확인한다.
6. Proxifier에 고유한 이름의 SOCKS5 `127.0.0.1:1080` proxy와 애플리케이션 rule을 등록한다.
7. `<INTERNAL_IP>:<PORT>`까지 TCP 응답을 확인한 뒤, 최종 대상용 인증 자료로 로그인 또는 명령 실행을 별도 검증한다.

### 작업 전 기준선

Windows RDP client의 상승된 `cmd.exe`:

```cmd
reg query "HKCU\SOFTWARE\Microsoft\Terminal Server Client\Default\AddIns\SocksOverRDP-Plugin" /s
netstat -ano | findstr "127.0.0.1:1080"
if exist "<CLIENT_PLUGIN_PATH>" (echo EXISTS) else (echo ABSENT)
```

Windows RDP 피벗 호스트의 PowerShell:

```powershell
Test-Path -LiteralPath '<SERVER_EXE_PATH>'
Get-CimInstance Win32_Process -Filter "Name = 'SocksOverRDP-Server.exe'" | Select-Object ProcessId,ExecutablePath,CreationDate,CommandLine
```

plugin registry key나 listener가 이미 있으면 이번 작업이 만든 자원과 구분할 수 없으므로 그대로 덮어쓰거나 정리하지 않는다. client와 server 파일의 작업 전 존재 여부를 각각 기록한다.

### Windows RDP 클라이언트에서 plugin DLL 등록

```cmd
regsvr32.exe "<CLIENT_PLUGIN_PATH>"
```

확인할 출력:

- DLL 등록 성공 대화상자 또는 성공 메시지. 이 출력은 plugin 등록만 입증하며 SOCKS listener나 내부 서비스 접근을 입증하지 않는다.

### Windows RDP 클라이언트에서 SOCKS listener 확인

```cmd
netstat -antb | findstr 1080
```

확인할 출력:

- RDP client 측 `127.0.0.1:1080` listen 상태. listener 생성 뒤에도 `<INTERNAL_IP>:<PORT>`의 실제 응답을 별도로 확인한다.

### Windows RDP 피벗 호스트에서 server 실행

```powershell
$SocksOverRdpServer = Start-Process -FilePath '<SERVER_EXE_PATH>' -PassThru
$SocksOverRdpServer | Select-Object Id,Path,StartTime
```

확인할 출력:

- `<SERVER_PID>`로 기록할 PID·경로·시작 시각과 server가 RDP Dynamic Virtual Channel 연결을 처리하는 상태가 보여야 한다. server는 일반 사용자로 실행할 수 있다. 프로세스 시작만으로 client 측 listener나 내부 서비스 접근을 판단하지 않는다.

### Windows RDP 클라이언트에서 Proxifier 설정

- Proxy Server: `127.0.0.1:1080`
- Protocol: `SOCKS5`
- Proxification Rule: `mstsc.exe` 또는 실제로 내부 서비스에 연결할 애플리케이션

확인할 출력:

- Proxifier connection log에 `<INTERNAL_IP>:<PORT>`가 표시되고 대상 서비스가 응답해야 한다.
- listener는 있으나 connection log가 없으면 애플리케이션 rule을, log는 있으나 대상 응답이 없으면 SocksOverRDP server·피벗 route·최종 listener를 확인한다.

### Linux 공격 호스트에서 Windows RDP client까지 접속하는 경우

```bash
xfreerdp /v:<WINDOWS_CLIENT_IP> /u:<USER> /p:'<PASSWORD>' /drive:share,<LOCAL_PATH>
```

확인할 출력:

- 이 명령은 SocksOverRDP plugin을 직접 로드하지 않는다. Linux에서 접속한 `<WINDOWS_CLIENT_IP>` 작업 호스트 안에서 `mstsc.exe`를 실행해 `<PIVOT_IP>`로 다음 RDP 세션을 열 때만 그 Windows 작업 호스트를 위 절차의 RDP client로 사용한다. 첫 Windows 세션과 최종 내부 호스트 권한은 각각 미확인 상태다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 첫 Windows 피벗에 RDP 로그인 성공 | `<PIVOT_IP>`에서 DVC와 server를 운반할 인증된 RDP 세션이 확보됨 | 피벗 호스트의 Windows 사용자 세션 | 현재 Identity·관리자 권한과 plugin·server 구성 요소 확인 |
| DLL 등록 성공 후 `127.0.0.1:1080`이 listen | RDP DVC를 통한 SOCKS listener가 생성됨 | SOCKS 프록시 | Proxifier의 SOCKS protocol·port 확인 |
| SOCKS 규칙을 적용한 `<INTERNAL_IP>:<PORT>`의 RDP·서비스가 응답함 | DVC와 최종 TCP 경로가 모두 동작함 | 운영 호스트에서 내부 서비스 접근 | 최종 대상 계정의 인증, 원격 세션과 실제 권한을 차례로 확인 |
| DLL 등록이 실패함 | 관리자 권한, x86·x64 또는 경로가 맞지 않음 | plugin 미등록 | 권한과 아키텍처를 재확인 |
| 등록은 됐지만 `1080` listen이 없음 | RDP 재접속 또는 server 활성화가 필요함 | DVC 구성만 일부 완료 | RDP 세션 재연결과 server 실행 여부 확인 |
| SOCKS 연결은 되지만 RDP 인증이 실패함 | 터널보다 대상 계정·NLA·로그온 권한 문제일 수 있음 | 내부 RDP 인증 경로 | [[RDP 로그인과 GUI 세션]]으로 분기 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 첫 RDP hop | Windows client의 `mstsc.exe` GUI 세션과 피벗의 현재 `whoami` | client의 plugin 등록 권한과 server의 현재 사용자 실행을 분리 |
| plugin | DLL 등록 성공과 RDP 재접속 | 파일 존재와 DVC 활성화를 구분 |
| listener·protocol | `127.0.0.1:1080` listen, SOCKS client 규칙 | 포트 생성과 실제 proxy 사용을 분리 |
| 최종 hop | 내부 RDP 로그인 화면·인증 결과 | TCP 접근과 대상 호스트 권한을 구분 |

## 변경 영향과 복구

연결이 살아 있을 때 최종 내부 세션과 그 세션이 만든 자원을 먼저 정리한다. 이어 피벗 RDP 세션 안에서 기록한 server를 종료한 뒤, Windows client에서 Proxifier의 이번 proxy·rule, RDP 세션, 이번 plugin 등록 순서로 정리한다. RDP부터 끊어 server 종료를 확인할 수 없게 만들지 않는다.

피벗 호스트의 PowerShell:

```powershell
Get-CimInstance Win32_Process -Filter "ProcessId = <SERVER_PID>" | Select-Object ProcessId,ExecutablePath,CreationDate,CommandLine
Stop-Process -Id <SERVER_PID>
Get-Process -Id <SERVER_PID> -ErrorAction SilentlyContinue
```

조회한 경로·시작 시각이 기록과 다르면 PID 재사용 가능성이 있으므로 종료하지 않는다. server 종료 뒤 client의 `netstat -ano | findstr "127.0.0.1:1080"` 결과가 작업 전 상태로 돌아오는지 확인한다. Proxifier에서는 이번에 붙인 고유 proxy와 rule만 제거하고 다른 기존 항목은 유지한다.

이번 작업 전 registry key가 없었고 이번 DLL을 등록한 경우에만 Windows client의 상승된 `cmd.exe`에서 해제한다.

```cmd
regsvr32.exe /u "<CLIENT_PLUGIN_PATH>"
reg query "HKCU\SOFTWARE\Microsoft\Terminal Server Client\Default\AddIns\SocksOverRDP-Plugin" /s
```

작업 전 없었고 이번에 전송한 것으로 확인된 파일만 각 호스트에서 정확한 경로로 제거한다.

```powershell
Remove-Item -LiteralPath '<SERVER_EXE_PATH>' -Force
Test-Path -LiteralPath '<SERVER_EXE_PATH>'
```

```cmd
del /f /q "<CLIENT_PLUGIN_PATH>"
if exist "<CLIENT_PLUGIN_PATH>" (echo REMAINS) else (echo REMOVED)
```

기존 key·파일·Proxifier 항목은 제거하지 않는다. RDP가 먼저 끊어져 `<SERVER_PID>`와 피벗 파일을 확인하지 못하면 client listener·plugin을 정리하더라도 원격 정리 완료로 표현하지 않는다.

## 관련 상태 라우터

- RDP 경유 SOCKS로 내부 대상에 도달했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[SocksOverRDP]]
- [[Proxifier]]
- [[xfreerdp]]

## 관련 노트

- [[RDP 로그인과 GUI 세션]]

## 참고 링크

- [SocksOverRDP 공식 README](https://github.com/nccgroup/SocksOverRDP)
