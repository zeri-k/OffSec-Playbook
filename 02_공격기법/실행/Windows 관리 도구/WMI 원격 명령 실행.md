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

## 사용할 때

- 현재 네트워크 위치: 공격 호스트에서 `<TARGET>:135/RPC`, 필요한 동적 RPC 포트와 도구가 사용하는 `445/SMB` 경로에 연결할 수 있다.
- 명령 실행 위치: `impacket-wmiexec`와 NetExec은 공격 호스트에서 실행하고, 성공한 명령과 shell은 `<TARGET>`에서 인증된 Windows 계정 컨텍스트로 실행된다.
- 보유 계정·인증 자료: `<TARGET>`에서 유효할 가능성이 있는 plaintext password, NT hash 또는 Kerberos ccache를 보유한다. NetNTLM challenge-response는 `-hashes` 입력에 사용할 NT hash가 아니며, credential 보유나 SMB 인증 성공만으로 WMI 실행을 확정하지 않는다.
- 현재 권한: 대상 호스트의 로컬 관리자 권한과 원격 WMI 실행 허용이 필요하다. `Pwn3d!`는 관리자급 실행 가능성 신호이며, 실제 WMI 명령 출력과 원격 `whoami`로 확인한다.
- 지금 가능한 행동: PsExec 서비스 설치에 의존하지 않고 WMI/RPC로 단일 명령을 실행하거나 반대화형 shell을 연다.
- 성공 범위: `<TARGET>`에서 현재 계정 Identity로 명령 출력이나 shell을 얻는다. SMB 인증 성공, WMI 실행 성공, 관리자 token과 도메인 권한은 각각 별도로 판정한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트에서 `<TARGET>:135`, 필요한 동적 RPC 포트와 `445/TCP` 연결 가능 | 포트 연결과 `impacket-wmiexec` 오류 단계를 확인 | RPC endpoint mapper, 동적 포트 범위, SMB와 호스트 방화벽을 분리 확인 |
| 현재 계정 또는 인증 수단 | `<TARGET>`에서 유효한 plaintext password 또는 NT hash, 또는 준비된 Kerberos ccache | `netexec smb` 인증 결과와 선택한 인증 방식 확인 | 사용자 이름의 로컬·도메인 범위, hash 형식, ticket 대상·만료 확인 |
| 현재 권한 | 대상의 로컬 관리자이면서 원격 WMI 실행이 허용됨 | `Pwn3d!`·Administrators 멤버십을 후보로 보고 실제 `-x whoami` 또는 WMI shell로 검증 | UAC token filtering, DCOM·WMI namespace 권한과 방화벽 확인 |
| 공격 대상의 조건 | WMI와 RPC 서비스가 동작하고 원격 관리가 허용됨 | WMI 실행 결과 또는 구체적인 RPC 오류 확인 | WinRM, PsExec, SMBExec 등 허용된 대체 원격 실행 경로 확인 |
| 필요한 파일·목록·주소 | `<DOMAIN>`, `<USER>`, credential, `<TARGET>`, Kerberos 사용 시 `<CCACHE_FILE>`, `<HOST_FQDN>`, `<DC_IP>` | 로컬·도메인 계정 표기, 대상 주소와 ccache의 principal·SPN 확인 | 인증 범위, 대상 FQDN·realm과 ticket 파일 수정 |

## 실행

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
export KRB5CCNAME=<CCACHE_FILE>
klist
impacket-wmiexec -k -no-pass -dc-ip <DC_IP> '<DOMAIN>/<USER>@<HOST_FQDN>'
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

- [[135_WMI]]
- [[445_SMB]]

## 관련 상태 라우터

- 원격 명령 실행 주체와 권한을 확인할 때: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 도구

- [[impacket-wmiexec]]
- [[netexec]]
- [[crackmapexec]]
- [[powershell]]
