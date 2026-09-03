---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 서비스/smb
시작조건: ["실행 호스트에서 내부 AD 인증 서비스 접근 가능", "유효한 공격 대상 사용자 목록 확보", "비밀번호 정책 확인"]
필요권한: ["DomainPasswordSpray 자동 열거 시 요청자 도메인 사용자 세션"]
필요조건: ["Domain Controller와 도메인명", "공격 대상 사용자 목록", "단일 평문 비밀번호 후보", "계정별 시도 횟수와 간격"]
결과: ["사용자명과 평문 비밀번호가 일치하는 유효한 도메인 credential", "서비스별 인증 성공", "인증 실패·잠금·rate limit 신호"]
---

# 내부 AD Password Spraying

## 한 줄 판단

실행 호스트에서 DC의 Kerberos·SMB/RPC에 접근할 수 있고 유효한 공격 대상 사용자 목록과 잠금 정책을 확보했으면 단일 평문 비밀번호를 계정마다 한 번씩 검증해 일치하는 도메인 credential을 찾고 첫 성공 또는 잠금 징후에서 멈춘다.

## 사용할 때

- 현재 보유 정보: 내부 AD의 도메인명·DC, 유효한 공격 대상 사용자 목록, 단일 평문 비밀번호 후보와 계정별 적용 잠금 정책을 확보했다.
- 명령 실행 위치와 도달성: Linux 또는 Windows 공격 호스트에서 DC의 Kerberos에 접근해 Kerbrute를 실행할 수 있다. DomainPasswordSpray는 요청자 도메인 사용자로 로그인한 Windows 호스트에서 실행한다.
- 현재 가능한 행동과 결과: 잠금 임계값에 여유를 둔 한 라운드만 실행해 사용자명·비밀번호 일치 여부를 확인하고, 인증 성공을 원격 세션·로컬 관리자·도메인 고권한으로 확대 해석하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | Linux 또는 Windows 공격 호스트에서 DC의 Kerberos 접근 가능. rpcclient·CrackMapExec은 Linux에서 SMB/RPC 접근 가능. DomainPasswordSpray는 도메인 Windows 사용자 세션에서 실행 가능 | DC 이름·도메인, DNS·시간과 대상 서비스 연결 확인 | realm·DNS·시간·라우팅을 바로잡기 전에는 인증 시도 금지 |
| 현재 계정 또는 인증 수단 | Kerbrute·rpcclient·CrackMapExec은 인증 전 대상 목록 사용, DomainPasswordSpray는 요청자 도메인 사용자 컨텍스트 사용 | 현재 실행 계정과 도구별 인증 요구 사항 확인 | 요청자 계정과 spraying 공격 대상 계정 목록을 분리해 다시 확인 |
| 현재 권한 | 정책·사용자 자동 열거에는 요청자 도메인 사용자 권한, 인증 검증 자체에는 별도 관리자 권한 불필요 | [[AD 비밀번호 정책 열거 및 조회]]와 [[인증 전 AD 사용자 목록 수집]] 또는 [[인증 후 AD 사용자와 컴퓨터 객체 열거]] 결과 확인 | 정책과 유효 사용자 목록을 확인할 다른 경로 확보 |
| 공격 대상의 조건 | 비활성·잠금 임박 계정을 제외한 유효 사용자와 계정별 잠금 정책 식별 | Fine-Grained Password Policy, `badpwdcount`, disabled 상태 확인 | 정책을 확인하지 못했거나 계정 상태가 불명확하면 실행하지 않음 |
| 필요한 파일·목록·주소 | DC·도메인, 중복 제거한 사용자 목록, 단일 비밀번호와 라운드 간격 확정 | 계획한 사용자 수·비밀번호 수·최대 시도 횟수 확인 | 여러 비밀번호를 한 라운드에 넣지 말고 입력과 간격 재설계 |

## 비밀번호 후보 선택

다음 순서로 한 라운드에 사용할 단일 비밀번호 후보를 정한다.

1. 설정 파일·스크립트·문서·메일 등에서 이미 확인한 비밀번호가 있으면 그 값을 우선한다.
2. 신규 계정이나 비밀번호 재설정에 사용하는 조직의 초기·기본 비밀번호를 확인했으면 그 값을 사용한다.
3. 환경에서 확인한 후보가 없으면 일반적인 초기 비밀번호 패턴인 `Welcome1`을 첫 공통 후보로 한 번 시도할 수 있다.

`Welcome1`이 실패했다고 `Welcome1!`, `Welcome123`, 계절·연도 변형을 같은 라운드에서 연속으로 시도하지 않는다. 다음 후보는 적용 정책의 observation window와 계정별 실패 횟수를 다시 확인한 뒤 별도 라운드로 판단한다.

## 계정 잠금 위험과 중지 조건

- 한 라운드에서는 계정마다 같은 비밀번호를 한 번만 시도한다.
- 도메인 기본 정책과 Fine-Grained Password Policy 중 가장 작은 잠금 임계값을 기준으로 잡고, 알려진 `badpwdcount`를 포함해 최소 한 번 이상의 여유를 남긴다.
- 여러 DC의 실패 횟수 차이를 고려해 가능하면 PDC Emulator 또는 계정별 적용 정책을 확인한다.
- 정책을 확인하지 못했다면 실행하지 않는다. 정책 확인 뒤에도 첫 라운드는 좁은 대상에 한 번만 수행하고, 두 번째 라운드는 observation window와 잠금 임계값을 기준으로 간격을 다시 계산한다.
- 다음 중 하나라도 보이면 즉시 전체 실행을 중지한다.
  - `STATUS_ACCOUNT_LOCKED_OUT`, `account locked`, Kerbrute의 lockout·disabled 메시지.
  - 예상보다 빠른 `badpwdcount` 증가, 계정 잠금 신고 또는 방어팀의 중지 요청.
  - rate limit, DC 오류나 정책 불일치로 성공·실패를 신뢰할 수 없는 상태.
  - 첫 유효 credential 확인 또는 정한 최대 시도 횟수 도달.
- 오류 출력을 숨기는 `grep` 필터를 적용하지 않고 성공과 잠금 신호를 함께 확인한다.

## 실행

### Linux 공격 호스트에서 실행

#### 1. Kerbrute로 Kerberos Password Spraying

```bash
kerbrute passwordspray --safe -d <DOMAIN> --dc <DC> valid_users.txt '<PASSWORD>'
```

확인할 출력:

- 실제 성공: `[+] VALID LOGIN: <USER>@<DOMAIN>:<PASSWORD>`.
- 중지 신호: lockout·disabled 메시지, KDC 오류 또는 예상하지 못한 realm 응답. 성공한 대상 계정은 요청자 계정과 별도로 기록한다.

#### 2. rpcclient로 SMB/RPC 인증 확인

```bash
while IFS= read -r u; do
  rpcclient -U "$u%<PASSWORD>" -c 'getusername;quit' <DC>
done < valid_users.txt
```

확인할 출력:

- 실제 성공: `Account Name: <USER>, Authority Name: <DOMAIN>`.
- 실패나 잠금 오류는 표준 오류까지 확인하며 `Authority Name`이 없는 결과를 성공으로 보지 않는다.

#### 3. CrackMapExec으로 SMB Spraying

```bash
crackmapexec smb <DC> -d <DOMAIN> -u valid_users.txt -p '<PASSWORD>'
```

확인할 출력:

- 실제 성공: `[+] <DOMAIN>\<USER>:<PASSWORD>`.
- `Pwn3d!`는 인증 성공과 별개인 관리자급 동작 후보이며 도메인 고권한을 뜻하지 않는다.
- `STATUS_LOGON_FAILURE`와 `STATUS_ACCOUNT_LOCKED_OUT`을 구분한다.

### Windows 공격 호스트에서 실행

#### 4. Kerbrute로 Kerberos Password Spraying

Windows x64 바이너리는 PowerShell 또는 `cmd.exe`에서 실행할 수 있다. Kerbrute 자체에는 현재 도메인 사용자 세션이 필요하지 않지만, 실행 호스트에서 `<DC>:88`에 연결할 수 있어야 한다.

```powershell
.\kerbrute_Windows.exe passwordspray -d <DOMAIN> .\adusers.txt '<PASSWORD>'
```

환경에서 확인한 비밀번호 후보가 없고 `Welcome1`을 첫 공통 후보로 선택했다면 다음처럼 한 라운드만 실행한다.

```powershell
.\kerbrute_Windows.exe passwordspray -d <DOMAIN> .\adusers.txt 'Welcome1'
```

DNS로 도메인의 DC를 찾을 수 없거나 특정 DC에만 요청해야 하면 `--dc`를 추가한다. Kerbrute가 잠금 징후를 감지하면 실행을 중단하게 하려면 `--safe`를 함께 사용한다.

```powershell
.\kerbrute_Windows.exe passwordspray --safe -d <DOMAIN> --dc <DC> .\adusers.txt '<PASSWORD>'
```

확인할 출력:

- 실제 성공: `[+] VALID LOGIN: <USER>@<DOMAIN>:<PASSWORD>`.
- 중지 신호: lockout·disabled 메시지, KDC 오류 또는 예상하지 못한 realm 응답.
- `--dc`를 생략한 경우 DC 탐색이 실패하면 도메인 DNS 설정과 SRV 조회를 확인한 뒤 특정 DC를 명시한다.
- Windows Defender 또는 애플리케이션 제어에 차단되면 결과를 인증 실패로 해석하지 않고 실행 차단 여부를 먼저 확인한다.

#### 5. DomainPasswordSpray로 정책 인식 Password Spraying

현재 Windows 세션의 도메인 Identity로 사용자와 정책을 자동 열거할 수 있을 때 사용한다.

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password '<PASSWORD>' -OutFile '<OUTPUT_FILE>'
```

확인할 출력:

- 도구가 확인한 가장 작은 잠금 임계값, 제외한 disabled·잠금 임박 사용자와 대기 시간.
- 실제 성공: `[*] SUCCESS! User:<USER> Password:<PASSWORD>`와 `<OUTPUT_FILE>`의 동일한 성공 항목.
- 확인 프롬프트의 사용자 수와 비밀번호 수가 계획한 대상과 일치하는지 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| Kerbrute `VALID LOGIN`, rpcclient `Authority Name`, CrackMapExec `[+]` 또는 DomainPasswordSpray `SUCCESS!` 확인 | 사용자명과 비밀번호 조합이 유효함 | 도메인 자격 증명 확보 | 추가 spraying을 멈추고 [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 서비스별 접근과 실제 권한 확인 |
| 인증 성공하지만 원격 세션을 열 수 없음 | credential은 유효하나 해당 서비스의 로그온 권한은 없을 수 있음 | 인증 성공·세션 미확보 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 SMB·LDAP·WinRM·RDP 지원 범위를 분리 확인 |
| `Pwn3d!` 표시 | 성공한 대상 계정으로 SMB 관리자급 동작이 가능할 후보 | 대상 호스트 로컬 관리자급 후보 | 원격 실행 주체와 실제 로컬 관리자 토큰을 검증한 뒤 [[고권한 세션 확보 후 후속 판단]] |
| 모든 계정에서 명확한 로그인 실패 | 비밀번호 후보가 맞지 않을 가능성 | 자격 증명 미확보 | 해당 라운드를 종료하고 정책 관찰 창이 지난 뒤에만 새 비밀번호 후보 검토 |
| 잠금·disabled·rate limit 신호 | 추가 시도가 운영 계정 상태에 영향을 줄 수 있음 | 검증 즉시 중단 | 전체 spraying을 중지하고 영향받은 계정 상태와 적용 정책을 확인한 뒤 재시도 금지 |
| KDC·realm·시간·DC 오류 | 인증 결과를 신뢰할 수 없음 | 결과 미판정 | DNS, realm, 시간과 DC 도달성을 바로잡기 전 재시도 금지 |

## 확인할 출력과 권한

- 성공은 도구별 실제 사용자·비밀번호 양성 문자열로 판정하고, 단순한 DC 연결이나 도메인 식별 출력을 성공으로 보지 않는다.
- 유효 credential, 원격 로그온 가능 여부, 파일 접근, 대화형 세션, 로컬 관리자와 도메인 권한을 각각 구분한다.
- 요청자 도메인 계정은 정책·사용자 열거와 스크립트 실행에 쓰는 현재 Identity이고, 양성 결과의 공격 대상 계정은 새로 비밀번호가 일치한 Identity다.
- 첫 성공 뒤 같은 비밀번호를 나머지 사용자에게 계속 시도하지 않고, 확보한 credential의 허용 서비스를 확인하는 흐름으로 전환한다.
- 실패한 Kerberos Pre-Authentication은 계정 실패 횟수에 포함될 수 있으며 이벤트 ID 4771, SMB 인증 실패는 이벤트 ID 4625 등으로 탐지될 수 있다.

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[kerbrute]]
- [[rpcclient]]
- [[crackmapexec]]
- [[DomainPasswordSpray]]

## 관련 공격기법

- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 전 AD 사용자 목록 수집]]
- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[원격 비밀번호 공격]]
