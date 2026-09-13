---
tags:
  - 환경/windows
  - 서비스/wmi
시작조건: ["공격 호스트에서 <TARGET>:135 및 WMI RPC 경로 접근 가능", "Windows 계정 plaintext password, NT hash 또는 Kerberos ccache 확보"]
필요권한: ["대상 호스트의 로컬 관리자 권한과 원격 WMI 실행 권한"]
필요조건: ["<TARGET>에서 유효한 사용자 plaintext password, NT hash 또는 Kerberos ccache", "공격 호스트에서 <TARGET>:135/RPC와 필요한 동적 RPC 포트 및 445/SMB 연결 가능", "Kerberos 사용 시 <HOST_FQDN>·realm·시간 정합과 TGT로 새 TGS를 요청할 때 <DC_FQDN>:88 접근 가능"]
결과: ["대상 계정의 WMI 원격 명령 실행", "대상 Windows 호스트의 원격 shell"]
---

# WMI 원격 명령 실행

## 한 줄 판단

공격 호스트에서 `<TARGET>`의 RPC·SMB 경로에 연결할 수 있고 확보한 plaintext password, NT hash 또는 Kerberos ccache가 대상의 원격 WMI 실행이 가능한 로컬 관리자 계정에 해당하면, WMI로 그 호스트에서 명령을 실행하거나 shell을 연다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트에서 `<TARGET>:135`, 필요한 동적 RPC 포트와 `445/TCP` 연결 가능 | 포트 연결과 `impacket-wmiexec` 오류 단계를 확인 | RPC endpoint mapper, 동적 포트 범위, SMB와 호스트 방화벽을 분리 확인 |
| 현재 계정 또는 인증 수단 | `<TARGET>`에서 유효한 plaintext password 또는 NT hash, 또는 준비된 Kerberos ccache | `netexec smb` 인증 결과와 선택한 인증 방식 확인 | 사용자 이름의 로컬·도메인 범위, hash 형식, ticket 대상·만료 확인 |
| 현재 권한 | 대상의 로컬 관리자이면서 원격 WMI 실행이 허용됨 | `Pwn3d!`·Administrators 멤버십을 후보로 보고 실제 `-x whoami` 또는 WMI shell로 검증 | UAC token filtering, DCOM·WMI namespace 권한과 방화벽 확인 |
| 공격 대상의 조건 | WMI와 RPC 서비스가 동작하고 원격 관리가 허용됨 | WMI 실행 결과 또는 구체적인 RPC 오류 확인 | WinRM, PsExec, SMBExec 등 허용된 대체 원격 실행 경로 확인 |
| 필요한 파일·목록·주소 | `<DOMAIN>`, `<USER>`, credential, `<TARGET>`, Kerberos 사용 시 `<CCACHE_FILE>`, `<HOST_FQDN>`, `<DC_IP>` | 로컬·도메인 계정 표기, 대상 주소와 ccache의 principal·SPN 확인 | 인증 범위, 대상 FQDN·realm과 ticket 파일 수정 |

## 실행

`<TARGET>`은 RPC·SMB에 도달하는 대상 주소, `<DOMAIN>/<USER>`·`<PASSWORD>` 또는 `<NTLM_HASH>`는 같은 원격 실행 요청자 입력이다. Kerberos 분기에서는 `<CCACHE_FILE>`이 공격 호스트의 ticket 파일이고 `<HOST_FQDN>`은 ticket SPN과 맞는 대상 FQDN, `<DC_IP>`는 TGS 요청에 쓰는 domain controller 주소다. 아래 client 명령은 모두 공격 호스트에서 실행한다.

### 인증 방식 선택

| 보유 인증 자료 | 실행 위치·추가 조건 | 사용할 방식 | 성공 결과 |
|---|---|---|---|
| `<USER>`의 plaintext password | RPC·SMB 경로에 도달하는 공격 호스트 | Impacket credential 문자열의 `<PASSWORD>` | 대상에서 인증 계정으로 실행된 WMI 명령 출력 또는 shell |
| `<USER>`의 NT hash | RPC·SMB 경로에 도달하고 대상이 NTLM을 허용함 | `-hashes :<NTLM_HASH>` | Pass the Hash 인증 뒤 WMI 명령 출력 또는 shell |
| `<USER>`의 Kerberos ccache | ccache를 읽을 Linux 공격 호스트, 대상 FQDN·realm·시간 정합 | `KRB5CCNAME`과 `-k -no-pass` | ticket 주체로 Kerberos 인증한 WMI 명령 출력 또는 shell |

세 방식 모두 대상 호스트의 원격 WMI 실행 권한이 필요하다. password·hash·ticket이 유효해도 SMB 인증만 성공하고 WMI 실행은 거부될 수 있다.

1. 공격 호스트에서 `<TARGET>`의 RPC·SMB 네트워크 경로를 확인한다.
2. NetExec으로 보유 credential의 SMB 인증 성공과 관리자급 실행 가능성 신호를 구분해 확인한다.
3. `impacket-wmiexec`로 단일 명령 또는 shell을 열어 실제 WMI 실행을 검증한다.
4. `<TARGET>`에서 `whoami`, `hostname`, `ipconfig`로 원격 Identity, 호스트와 네트워크 위치를 확인한다.
5. 확인한 권한 범위 안에서 파일 전송, SAM/LSA dump 또는 내부 정찰로 이어간다.

### Linux 공격 호스트에서 실행

첫 `impacket-wmiexec` 실행 전 같은 인증 방식으로 `impacket-smbclient`를 열어 `ADMIN$`에서 `ls __*`를 수행하고 기존 이름 목록과 실행 시작 시각을 기록한다. 자격 증명이나 목록 원문을 Vault에 저장하지 않는다. 이 기준선은 정상 종료 때 삭제할 대상을 만드는 것이 아니라, client가 중단됐을 때 이번 실행의 임시 출력만 구분하기 위한 것이다.

#### 비밀번호 기반 실행

```bash
impacket-wmiexec '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>'
```

확인할 출력:

- `<TARGET>`에서 반환된 `C:\>` prompt 또는 명령 출력. prompt만 보지 말고 `whoami`와 `hostname`으로 실행 주체와 대상 호스트를 확인한다.

#### hash 기반 실행

```bash
impacket-wmiexec '<DOMAIN>/<USER>@<TARGET>' -hashes :<NTLM_HASH>
```

확인할 출력:

- NTLM hash 인증 뒤 열린 원격 shell과 `<TARGET>`의 `whoami` 결과. hash 보유와 WMI 원격 실행 성공을 구분한다.

#### Kerberos ccache 기반 실행

```bash
KRB5CCNAME='<CCACHE_FILE>' klist
KRB5CCNAME='<CCACHE_FILE>' impacket-wmiexec -k -no-pass -dc-ip <DC_IP> '<DOMAIN>/<USER>@<HOST_FQDN>'
```

확인할 출력:

- `klist`에서 ticket principal·TGT/TGS·만료 시각을 확인하고 IP가 아니라 ticket의 SPN과 맞는 `<HOST_FQDN>`으로 접속한다.
- `<HOST_FQDN>`에서 반환된 WMI shell과 `whoami` 출력이 있어야 Kerberos 인증과 원격 WMI 실행이 모두 확인된다. `-no-pass`는 무인증이 아니라 ccache ticket을 사용하므로 password 입력을 생략한다는 뜻이다.

#### NetExec 검증

```bash
netexec smb <TARGET> -u <USER> -p '<PASSWORD>' -x whoami
netexec smb <TARGET> -u <USER> -H <NTLM_HASH> -x whoami
```

확인할 출력:

- SMB 인증 결과와 원격 `whoami` 명령 출력을 구분한다. `-x whoami` 출력이 반환되어야 원격 명령 실행 성공이다.

#### 기존 CrackMapExec 명령 재현

내부 대상이 SOCKS 뒤에 있고 기존 `cme`·`crackmapexec` 문법을 재현해야 할 때 사용한다.

```bash
proxychains -q crackmapexec smb <TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>' -x 'whoami /all'
```

확인할 출력:

- ProxyChains `OK`: SOCKS를 통한 대상 TCP 연결 성공.
- CrackMapExec `[+]`: SMB 인증 성공.
- `Pwn3d!`: 관리자급 원격 작업 가능성.
- `Executed command`와 `whoami /all` 출력: 대상에서 실제 원격 명령 실행 성공.

CrackMapExec 버전에 따라 `-x`의 내부 실행 방법이 달라질 수 있으므로 도구 출력과 `--exec-method` 지원 여부를 확인한다. WMI 실행을 특정해 검증해야 하면 `impacket-wmiexec` 결과를 기준으로 삼는다.

## 변경 영향과 복구

현재 Impacket `wmiexec.py`는 command output을 기본 `ADMIN$` share의 `\__<timestamp>` 임시 파일로 redirect하고 읽은 뒤 `deleteFile`로 제거한다. 기본 `ADMIN$`은 일반적으로 Windows 디렉터리에 매핑되므로 원격 경로는 `%SystemRoot%\__<timestamp>`다. 정상 shell에서는 `exit`로 SMB·DCOM session을 닫고, 중단된 경우에만 남은 임시 파일을 확인한다.

| 생성 항목 | 기존 상태·식별값 | 종료·정리 | 완료 확인 |
|---|---|---|---|
| Impacket WMI shell | 대상·계정, 공격 호스트 client PID와 terminal | 원격 prompt에서 `exit` | 공격 호스트 client가 끝나고 새 명령이 실행되지 않음 |
| 원격 command output 임시 파일 | 실행 시작 시각, 실행 전 `ADMIN$\__*` 이름 목록, 중단 시 새로 생긴 `<WMI_OUTPUT_NAME>` | WMI 연결이 살아 있으면 먼저 shell을 종료한다. 별도 SMB 관리자 session에서 `ADMIN$`의 `<WMI_OUTPUT_NAME>`을 exact 이름으로 제거 | 실행 전 목록과 대조해 이번 실행의 새 파일이 없고, 기존 `ADMIN$\__*`는 그대로 존재 |
| Kerberos 환경 지정 | 이 문서는 기존 `<CCACHE_FILE>`을 command별 `KRB5CCNAME`으로만 전달 | 별도 shell 환경 복원 불필요. 입력 ccache는 삭제하지 않음 | 호출 command 종료 후 상위 shell의 `KRB5CCNAME` 값이 바뀌지 않음 |

중단 후 임시 파일을 확인할 때는 기존에 검증한 credential로 `impacket-smbclient`를 열어 `ADMIN$`에서 `ls __*`를 실행하고, 실행 전 목록에 없으며 시간대가 일치하는 exact `<WMI_OUTPUT_NAME>`만 `rm <WMI_OUTPUT_NAME>`으로 제거한 뒤 `exit`한다. 이름·시간 대응을 확정할 수 없거나 원격 연결이 끊겼으면 광범위하게 `%SystemRoot%\__*`를 삭제하지 않고 `원격 복구 미확인`으로 기록한다.

NetExec·CrackMapExec의 `-x`는 버전과 `--exec-method`에 따라 생성 자원이 달라질 수 있다. exact 정리가 필요한 이 절차의 기준 실행은 `impacket-wmiexec`로 두며, 다른 method를 선택했다면 해당 버전의 생성 서비스·파일을 확인하기 전에는 복구 완료로 판정하지 않는다. WMI·SMB 인증과 원격 명령의 감사 기록은 되돌리지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| NetExec SMB 인증은 성공하지만 원격 명령 출력은 없음 | credential은 SMB에서 유효하지만 WMI 실행 권한은 미확정 | 인증된 SMB 계정 | 관리자 token, WMI 권한과 RPC 경로 확인 |
| 대상에서 `whoami`, `hostname` 같은 명령 출력이 반환됨 | `<TARGET>`의 WMI 원격 실행 확인 | 대상 호스트의 명령 실행 | 계정, 호스트와 token 상태를 확인 |
| 원격 shell에서 파일 쓰기, 명령 실행, 후속 열거가 가능함 | 재사용 가능한 원격 실행 경로 확인 | 대상 계정 Identity의 원격 shell | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 권한을 확인한 뒤 [[Windows 권한 상승 열거]] 또는 [[상황별 파일 전송]]으로 전환 |
| login success, exec fail | 인증은 성공했지만 관리자·WMI 실행 권한 또는 RPC 경로가 부족함 | 인증된 계정, 원격 실행 미확보 | SMB share/WinRM 권한과 WMI·RPC 조건을 분리 확인 |
| Kerberos ticket은 보이지만 WMI 접속 실패 | FQDN·SPN·realm·시간 또는 필요한 service ticket 문제 | Kerberos ticket 보유, WMI 실행 미확보 | `<HOST_FQDN>`, `<DC_FQDN>:88`, `klist`의 Server와 시간 정합 확인 |
| RPC 오류 | 동적 포트/방화벽 차단 | 시작 상태 유지 | WinRM, PsExec, SMBExec 대체 |
| UAC 제한 | 로컬 관리자 토큰 필터링 | 시작 상태 유지 | RID-500, 도메인 계정, WinRM 확인 |

## 확인할 출력과 권한

- credential 보유, SMB 인증, `Pwn3d!`, WMI 명령 출력과 원격 shell은 서로 다른 확인 지점이다.
- `whoami`, `hostname`, `whoami /all`로 실행 주체, 대상 호스트와 실제 관리자 token을 확인한다. 로컬 관리자 권한은 도메인 관리자 권한을 뜻하지 않는다.

## 후속 공격 연결

- 로컬 관리자 권한 확인: [[Windows SAM SECURITY SYSTEM 덤프]]
- 활성 사용자 credential 수집: [[LSASS 메모리 덤프]]
- 파일 반입/회수: [[상황별 파일 전송]]
- 다른 원격 실행 방식 확인: [[WinRM 원격 PowerShell 세션]]

## 관련 서비스

- [[WMI 서비스]]
- [[SMB 서비스]]

## 관련 상태 라우터

- 원격 명령 실행 주체와 권한을 확인할 때: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[impacket-wmiexec]]
- [[netexec]]
- [[crackmapexec]]
- [[powershell]]

## 참고 링크

- [Impacket wmiexec](https://github.com/fortra/impacket/blob/master/examples/wmiexec.py)
