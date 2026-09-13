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

### 모듈 선택과 옵션 확인

```text
msf6 > use exploit/windows/smb/ms17_010_psexec
msf6 exploit(windows/smb/ms17_010_psexec) > show options
msf6 exploit(windows/smb/ms17_010_psexec) > show payloads
```

`show options`로 필수 옵션을 확인하고, 모듈에 맞는 payload 목록을 확인한다.

검색 결과의 번호는 현재 검색 목록에서만 유효하므로 자동화·기록에는 module의 전체 경로를 사용한다. `info`에서 지원 target, module rank, disclosure/reference와 `check` 지원을 확인하고, 제품명·platform이 같다는 이유만으로 target을 선택하지 않는다.

### 대상/인증 정보 설정

```text
msf6 exploit(...) > set RHOSTS <TARGET>
msf6 exploit(...) > set SMBUser <USER>
msf6 exploit(...) > set SMBPass '<PASSWORD>'
msf6 exploit(...) > set LHOST <ATTACK_INTERFACE_OR_IP>
```

`RHOSTS`는 대상, `LHOST`는 리버스 연결을 받을 인터페이스/IP다. 옵션명은 모듈마다 다르므로 `show options`를 기준으로 맞춘다.

비밀번호·token 같은 민감 값은 console history·DB·session metadata에 남을 수 있으므로 실제 값을 Vault에 기록하지 않는다. 현재 module에만 필요한 값은 `set`을 사용하고, `setg`는 이후 다른 module에도 적용되는 전역 상태이므로 필요한 범위를 확인하지 않았다면 사용하지 않는다.

### 취약 여부 확인

```text
msf6 exploit(...) > check
```

모듈이 `check`를 지원하면 실제 실행 전 취약 가능성을 먼저 확인한다. `Vulnerable`은 module이 취약 동작을 직접 확인한 결과, `Appears`는 version·구성 같은 간접 조건 일치, `Detected`는 서비스 식별, `Unknown`은 충분한 판정 자료를 얻지 못한 상태다. `Unknown` 또는 미지원은 취약·비취약 어느 쪽도 확정하지 못한다. `check`라는 이름만으로 비파괴성을 보장하지 않으므로 module 문서와 구현에서 요청·상태 변경을 먼저 확인한다.


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

## session·job 상태와 정리

`background`나 `Ctrl+Z`는 session을 종료하지 않는다. handler·서버 module을 `run -j`로 실행하면 session과 별도인 job이 생긴다. 작업 전후 목록을 비교해 이번에 생성된 `<SESSION_ID>`와 `<JOB_ID>`를 기록한다.

```text
msf6 > sessions -l
msf6 > jobs -l
```

원격 session으로 업로드한 파일·시작한 process·변경한 설정은 연결이 살아 있을 때 해당 기법의 exact 복구 절차로 먼저 정리한다. 공격 목표 달성과 원격 복구 완료를 별도로 판정한 뒤 이번 session과 job만 종료한다. 설치된 Framework에서 `sessions -h`와 `jobs -h`로 syntax를 확인한다.

```text
msf6 > sessions -k <SESSION_ID>
msf6 > jobs -k <JOB_ID>
msf6 > sessions -l
msf6 > jobs -l
```

마지막 목록에서 기록한 ID가 없어야 local control resource 정리가 확인된다. 이미 끊어진 session은 원격 정리가 됐다는 증거가 아니며 `sessions -l`에 없다는 이유로 대상 파일·service·process 복구를 완료로 표시하지 않는다. 다른 session/job을 일괄 종료하는 `sessions -K`·`jobs -K`는 사용하지 않는다.

현재 module에 넣은 민감 option은 같은 console을 계속 사용할 때 정확한 key만 해제한다. 전역으로 설정한 경우에만 `unsetg`를 사용한다.

```text
msf6 exploit(...) > unset SMBPass
msf6 exploit(...) > unset SMBUser
msf6 exploit(...) > unset RHOSTS
msf6 exploit(...) > unset LHOST
```

## 기존 Database의 작업별 workspace 사용

이 절차는 `db_status`가 이미 연결된 PostgreSQL backend를 표시할 때만 사용한다. `msfdb init`은 database·사용자·schema·설정 파일을 생성하고 `msfdb reinit`은 기존 database를 삭제하므로, 이 대표 흐름에서는 실행하지 않는다.

작업 전 현재 workspace와 연결 상태를 기록하고, 기존 목록에 없는 고유한 `<MSF_WORKSPACE>`를 만든다.

```text
msf6 > db_status
msf6 > workspace
msf6 > workspace -a <MSF_WORKSPACE>
msf6 > workspace
```

별도 Nmap 실행에서 만든 XML을 현재 workspace에 가져오고, import 성공 문구뿐 아니라 host와 service가 의도한 대상인지 확인한다. 가져온 port-name·version은 원래 scan의 판정 범위를 넘지 않는다.

```text
msf6 > db_import <NMAP_XML_PATH>
msf6 > hosts
msf6 > services
```

보존이 필요한 경우 설치된 버전의 `db_export -h`로 Framework·Pro syntax를 먼저 확인하고 새 `<MSF_EXPORT_PATH>`에 현재 workspace를 내보낸다. XML export는 database object 대부분을 담을 수 있지만 loot file·task log 전체의 별도 백업을 보장하지 않으며, credential이 포함될 수 있으므로 일반 Vault에 저장하지 않는다.

```text
msf6 > db_export -h
msf6 > db_export -f xml <MSF_EXPORT_PATH>
```

이번 workspace를 제거하려면 먼저 작업 전 `<ORIGINAL_WORKSPACE>`로 전환한 뒤 정확한 이름만 삭제한다. 삭제는 그 workspace의 host·credential·evidence를 함께 제거하므로 export 필요 여부와 현재 workspace 표시를 먼저 확인한다.

```text
msf6 > workspace <ORIGINAL_WORKSPACE>
msf6 > workspace -d <MSF_WORKSPACE>
msf6 > workspace
```

목록에 `<MSF_WORKSPACE>`가 없고 `*`가 원래 workspace에 있어야 database 정리가 확인된다. 생성한 export를 보존하지 않을 때는 msfconsole 밖의 공격 호스트에서 정확한 `<MSF_EXPORT_PATH>`만 제거하고 부재를 확인한다. 기존 workspace에 직접 import했다면 원천별 행을 안전하게 구분할 수 없으므로 전체 workspace 삭제로 복구하지 않는다.

## 검토한 외부 module을 사용자 경로에서 로드

Exploit-DB의 `.rb` 확장자만으로 Metasploit module 호환성이나 안전성을 확정할 수 없다. source URL·commit/hash를 기록하고 `MetasploitModule` class, module type, mixin, target·option, `check`와 `Notes`의 side effect, 생성 파일·service·account 정리를 먼저 검토한다.

공식 설치 경로를 덮어쓰지 않고 현재 실행 사용자의 기본 private module 경로인 `${HOME}/.msf4/modules` 아래에 기존에 없던 고유 category를 만든다.

```shell
MSF_PRIVATE_MODULE_DIR="${HOME}/.msf4/modules/exploits/review_<BATCH_ID>"
test ! -e "$MSF_PRIVATE_MODULE_DIR"
mkdir -p -- "$MSF_PRIVATE_MODULE_DIR"
install -m 0600 -- <REVIEWED_RB_PATH> "$MSF_PRIVATE_MODULE_DIR/<MODULE_NAME>.rb"
sha256sum "$MSF_PRIVATE_MODULE_DIR/<MODULE_NAME>.rb"
```

실행 중인 console에서는 `reload_all` 후 전체 경로로 검색·선택하고 `info`·`show options`를 다시 확인한다. load 성공은 대상 취약성이나 module 실행 성공이 아니다.

```text
msf6 > reload_all
msf6 > search <MODULE_NAME>
msf6 > use exploit/review_<BATCH_ID>/<MODULE_NAME>
msf6 exploit(...) > info
msf6 exploit(...) > show options
```

검토가 끝나면 active module에서 빠져나와 공격 호스트에서 이번에 설치한 정확한 파일과 고유 leaf directory만 제거한다. system module directory와 `~/.msf4/modules` 전체를 삭제하지 않는다.

```text
msf6 exploit(...) > back
msf6 > exit
```

```shell
rm -- "$MSF_PRIVATE_MODULE_DIR/<MODULE_NAME>.rb"
rmdir -- "$MSF_PRIVATE_MODULE_DIR"
test ! -e "$MSF_PRIVATE_MODULE_DIR"
```

새 console에서 같은 전체 module 경로가 검색되지 않아야 향후 load 경로 정리가 확인된다. module을 실제 실행했다면 local module 파일 제거와 대상 복구는 별개이며, module이 만든 원격 자원은 해당 module의 확인된 복구 절차가 없으면 완료로 표시하지 않는다.

## 관련 공격기법

- [[Public Exploit 검토와 검증]]
- [[MS17-010 EternalBlue SMB RCE]]
- [[MSFVenom Payload 생성과 Handler 수신]]
- [[Reverse Shell 획득]]
- [[Metasploit Web Delivery로 Meterpreter 세션 획득]]

## 참고 링크

- [Metasploit Framework documentation](https://docs.metasploit.com/)
- [Metasploit check method와 CheckCode](https://docs.metasploit.com/docs/development/developing-modules/guides/how-to-write-a-check-method.html)
- [Rapid7 API reference — session and job state](https://docs.rapid7.com/metasploit/standard-api-methods-reference/)
- [Rapid7 — Managing Workspaces](https://docs.rapid7.com/metasploit/managing-workspaces/)
- [Rapid7 — Exporting and Importing Data](https://docs.rapid7.com/metasploit/exporting-and-importing-data/)
- [Metasploit — Running Private Modules](https://docs.metasploit.com/docs/using-metasploit/intermediate/running-private-modules.html)
