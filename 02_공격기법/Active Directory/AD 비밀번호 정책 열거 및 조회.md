---
tags:
  - 환경/ad
  - 서비스/smb
  - 서비스/ldap
시작조건: ["Domain Controller와 도메인 식별", "실행 호스트에서 정책 조회 경로 하나 이상 확보"]
필요권한: ["서버가 허용한 익명 정책 조회 또는 현재 도메인 계정의 정책 읽기 권한"]
필요조건: ["DC SMB·RPC 또는 LDAP 접근, 혹은 도메인 Windows 세션", "선택한 경로에 맞는 도구와 인증 수단"]
결과: ["도메인 비밀번호 정책", "계정 잠금 임계값과 관찰 창", "Password Spraying 시도 기준"]
---

# AD 비밀번호 정책 열거 및 조회

## 한 줄 판단

식별한 DC의 SMB·RPC 또는 LDAP에 닿는 실행 호스트나 도메인 Windows 세션이 있으면, 서버가 허용한 익명 조회 또는 유효한 도메인 계정으로 비밀번호·잠금 정책을 읽어 Password Spraying 전에 확인된 정책 기준의 보수적인 시도 상한과 대기 시간을 정한다.

## 사용할 때

- 현재 보유 정보: 도메인명과 DC, 익명 SMB·LDAP 가능성 또는 유효한 도메인 사용자 자격 증명·도메인 Windows 세션 중 하나가 있다.
- 명령 실행 위치와 도달 대상: Linux·Windows 도구 실행 호스트에서 DC의 SMB·RPC 445/TCP 또는 LDAP 389/TCP에 닿거나, 도메인 Windows 세션에서 DC를 찾을 수 있다.
- 현재 계정과 권한: 익명 경로는 서버가 정책 조회를 허용해야 하고, 인증 경로는 현재 도메인 계정으로 정책 속성을 읽을 수 있어야 한다. 관리자 권한은 일반적인 도메인 정책 조회의 전제 조건이 아니다.
- 지금 가능한 행동과 결과: 내부 AD에서 Password Spraying이나 비밀번호 후보 검증을 계획하기 전에 최소 길이·복잡성뿐 아니라 잠금 임계값·지속 시간·관찰 창을 수집해 확인된 정책 기준의 반복 시도 위험을 계산할 수 있다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 선택한 도구 실행 호스트에서 DC SMB·RPC 또는 LDAP에 도달하거나 도메인 Windows 세션 사용 | 445/TCP·389/TCP 응답 또는 `net accounts /domain`의 DC 조회 확인 | DC 이름 해석, 포트, 방화벽과 피벗 경로를 확인 |
| 현재 계정 또는 인증 수단 | NULL Session·Anonymous Bind 또는 유효한 도메인 계정 중 하나 | 익명 응답과 인증 성공을 별도로 기록 | 익명 거부 시 보유 AD 계정으로 전환하고, 인증 실패 시 계정 형식·상태 확인 |
| 현재 권한 | 선택한 인증 주체로 도메인 정책 읽기 | 정책 필드가 access denied 없이 반환되는지 확인 | 인증 성공과 정책 읽기 거부를 구분하고 다른 조회 경로 사용 |
| 공격 대상의 조건 | `<DC>`가 대상 도메인의 정책을 제공하는 현재 Domain Controller | RootDSE, DNS SRV, SMB 배너의 도메인 정보 교차 확인 | 잘못된 DC·로컬 호스트 정책 조회 여부를 확인 |
| 필요한 파일·목록·주소 | `<DC>`, `<DOMAIN>`, LDAP는 `<BASE_DN>`, 인증 경로는 credential | 입력값과 선택한 도구의 인증 형식 확인 | [[AD 도메인 컨텍스트 기본 확인]]에서 누락된 도메인·DC·base DN 확보 |

## 실행

### 실행 위치별 조회 경로 선택

| 현재 사용할 수 있는 상태 | 선택할 방식 | 인증에 사용하는 주체 | 이 방식으로 확인하는 결과 |
|---|---|---|---|
| 도메인 자격 증명은 없고 DC 445/TCP에 도달 | SMB NULL Session으로 `rpcclient` 직접 조회 | 사용자명과 비밀번호를 보내지 않는 Null Session | DC가 익명으로 반환한 기본 도메인 정책 |
| 도메인 자격 증명은 없고 DC 445/TCP에 도달하며 여러 RPC 항목을 함께 수집 | `enum4linux-ng` 자동 수집 | 도구가 시도한 Null Session | 자동 수집에서 반환된 비밀번호·잠금 정책과 익명 세션 가능 여부 |
| 도메인 자격 증명은 없고 DC 389/TCP의 Anonymous Bind가 허용됨 | `ldapsearch` 익명 조회 | 인증하지 않은 LDAP bind | Base DN의 기본 도메인 정책 속성 |
| 유효한 도메인 사용자 비밀번호가 있고 DC 445/TCP에 도달 | `crackmapexec --pass-pol` 인증 조회 | `<DOMAIN>\<USER>` 계정 | 해당 계정으로 읽을 수 있는 기본 도메인 정책 |
| 도메인 Windows 세션에서 DC를 찾을 수 있음 | `net accounts /domain` | 현재 Windows 세션의 도메인 Identity | 현재 도메인의 기본 비밀번호·잠금 정책 |
| 도메인 Windows 세션에 ActiveDirectory 모듈이 있음 | AD PowerShell cmdlet | 현재 Windows 세션의 도메인 Identity | 기본 정책, FGPP 목록과 특정 사용자의 resultant PSO |

익명 경로가 거부되면 인증된 경로로 전환한다. 어느 방식으로 기본 정책을 읽어도 특정 사용자에게 Fine-Grained Password Policy가 적용되는지는 별도로 확인해야 한다.

### Linux 공격 호스트에서 실행

#### 방식 1. SMB NULL Session으로 RPC 정책 조회

```bash
rpcclient -U "" -N <DC>
```

```text
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

확인할 출력:

- `Domain`, `Server Role`과 NULL Session 허용 여부.
- `min_password_length`, `password_properties`와 복잡성 플래그.
- `NT_STATUS_ACCESS_DENIED` 또는 연결 실패가 나오면 NULL Session 거부와 SMB·RPC 경로 실패를 분리하고 인증된 조회 경로로 전환한다.

#### 같은 Null Session 범위를 enum4linux-ng로 자동 수집

```bash
test ! -e '<OUTPUT_BASENAME>.json' && test ! -e '<OUTPUT_BASENAME>.yaml'
enum4linux-ng -P <DC> -oA '<OUTPUT_BASENAME>'
```

확인할 출력:

- `min_pw_length`, `pw_history_length`, `min_pw_age`, `max_pw_age`.
- `lockout_threshold`, `lockout_duration`, `lockout_observation_window`.
- `null_session_possible`과 정책이 전체인지 일부인지 여부.
- 일부 값만 나오면 도구 출력의 수집 범위를 확인하고 다른 RPC·LDAP·Windows 경로와 교차 검증한다.

#### 방식 2. LDAP Anonymous Bind로 도메인 객체의 정책 속성 조회

```bash
ldapsearch -x -H ldap://<DC> -b '<BASE_DN>' -s base '(objectClass=*)' minPwdLength pwdHistoryLength pwdProperties minPwdAge maxPwdAge lockoutThreshold lockoutDuration lockOutObservationWindow
```

확인할 출력:

- `minPwdLength`, `pwdProperties`, `lockoutThreshold`.
- `lockoutDuration`과 `lockOutObservationWindow`; AD 시간 값은 100ns 단위 음수이므로 도구의 변환 결과와 교차 확인한다.
- `Insufficient access`와 bind 실패를 구분한다. Anonymous Bind가 되더라도 정책 속성이 반환되지 않으면 익명 정책 읽기가 허용된 것은 아니다.

#### 방식 3. 유효한 도메인 계정으로 원격 조회

```bash
crackmapexec smb <DC> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --pass-pol
```

확인할 출력:

- `[+] <DOMAIN>\<USER>:<PASSWORD>` 인증 성공 뒤 표시되는 도메인 정책.
- 최소 길이, 복잡성, 잠금 임계값, 잠금 지속 시간과 카운터 초기화 시간.
- 인증은 성공했지만 정책이 없으면 `--pass-pol` 지원과 RPC 조회 권한을 확인한다. 인증 실패 시 정책 미존재로 해석하지 않는다.

### Windows 도메인 세션 호스트에서 실행

#### 방식 4. 도메인 Windows 세션에서 기본 명령으로 조회

```cmd
net accounts /domain
```

확인할 출력:

- `Lockout threshold`, `Lockout duration`, `Lockout observation window`.
- 최소·최대 비밀번호 사용 기간, 최소 길이와 기록 길이.
- DC를 찾지 못하거나 access denied가 나오면 현재 계정이 로컬인지 도메인 계정인지, DNS와 DC 경로가 유효한지 확인한다.

#### 방식 5. ActiveDirectory 모듈로 기본 정책과 계정별 FGPP 확인

ActiveDirectory 모듈이 설치되어 있고 현재 도메인 계정으로 정책 객체를 읽을 수 있을 때 사용한다. 먼저 모듈과 기본 정책을 확인한다.

```powershell
Get-Module -ListAvailable ActiveDirectory
Get-ADDefaultDomainPasswordPolicy -Current LoggedOnUser |
  Select-Object LockoutThreshold, LockoutDuration, LockoutObservationWindow, MinPasswordLength, ComplexityEnabled
```

FGPP 객체 전체와 실제 공격 대상 사용자에게 적용되는 resultant PSO를 분리해서 조회한다.

```powershell
Get-ADFineGrainedPasswordPolicy -Filter * |
  Select-Object Name, Precedence, LockoutThreshold, LockoutDuration, LockoutObservationWindow, AppliesTo

Get-ADUserResultantPasswordPolicy -Identity '<USER>' |
  Select-Object Name, Precedence, LockoutThreshold, LockoutDuration, LockoutObservationWindow
```

확인할 출력:

- `Get-ADDefaultDomainPasswordPolicy`는 도메인 기본 정책이고, FGPP가 적용된 사용자의 실제 정책을 대신하지 않는다.
- `Get-ADFineGrainedPasswordPolicy`는 존재하는 PSO와 적용 대상을 열거한다. 목록에서 가장 작은 임계값을 찾는 것만으로 특정 사용자에게 그 PSO가 적용된다고 확정하지 않는다.
- `Get-ADUserResultantPasswordPolicy`가 반환한 객체는 해당 사용자에게 적용되는 하나의 resultant PSO다. 객체가 없거나 오류가 나면 기본 정책 적용을 즉시 단정하지 말고 사용자 식별자, 모듈·DC 연결과 읽기 권한을 확인한다.
- `Precedence`는 값이 작을수록 우선순위가 높다. 직접 적용과 그룹 적용이 겹치면 resultant PSO 출력으로 최종 대상을 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 잠금 임계값·지속 시간·관찰 창이 모두 확인됨 | 조회된 정책이 적용되는 계정의 최대 실패 횟수와 초기화 시점을 계산할 수 있음 | Spraying 안전 기준 확보 | Fine-Grained Password Policy 적용 여부를 확인하고 [[인증 전 AD 사용자 목록 수집]]에서 유효 사용자와 위험 계정을 정리한 뒤 [[내부 AD Password Spraying]] |
| 최소 길이·복잡성만 확인됨 | 비밀번호 후보 단서는 있으나 잠금 위험은 계산 불가 | 부분 정책만 확보 | 잠금 정책을 다른 경로로 재조회하고 확인 전 반복 시도 금지 |
| NULL Session 또는 Anonymous Bind로 정책 조회 성공 | 인증 전 AD 정보 노출 | 익명 정책 조회 가능 | 같은 익명 경로에서 [[인증 전 AD 사용자 목록 수집]] 가능 여부 확인 |
| 익명 조회 거부, credential 조회 성공 | 정상 인증이 필요한 정책 | 인증된 정책 조회 가능 | 확인한 정책과 계정 상태를 기준으로 시도 계획 작성 |
| 모든 조회가 거부됨 | 정책 미확인 | 잠금 위험 불명확 | 정책을 확인할 때까지 반복 Password Spraying을 수행하지 않음 |
| 정책 값이 도구마다 다름 | Fine-Grained Password Policy, DC 차이 또는 변환 오류 가능성 | 적용 정책 미확정 | 가장 작은 잠금 임계값을 우선하고 PDC Emulator·계정별 적용 정책을 재확인 |

## 확인할 출력과 권한

- 도메인 정책과 로컬 호스트 정책을 구분하므로 Windows에서는 `/domain` 결과를 사용한다.
- 조회한 값은 도메인 기본 정책의 현재 반환값을 확정한다. Fine-Grained Password Policy가 적용된 특정 계정의 실제 잠금·비밀번호 정책까지 일괄 확정하지 않는다.
- Password Spraying 전에는 최소한 잠금 임계값, 잠금 지속 시간과 관찰 창을 함께 확인한다.
- `lockoutThreshold: 0`을 무제한 시도 조건으로 해석하지 않고 시도 횟수·간격과 탐지 위험을 별도로 판단한다.
- 익명, 일반 도메인 사용자와 관리자 권한을 구분하며 정책 조회 성공을 다른 AD 객체의 쓰기 권한으로 확대 해석하지 않는다.

## 변경 영향과 로컬 출력 정리

- RPC·LDAP·CrackMapExec·Windows 정책 조회와 AD PowerShell cmdlet은 대상 정책을 읽을 뿐 변경하지 않는다. 다만 인증 경로는 서버 감사 기록을 남길 수 있다.
- `enum4linux-ng -oA`만 Vault 밖의 승인된 작업 디렉터리에 `<OUTPUT_BASENAME>.json`과 `<OUTPUT_BASENAME>.yaml`을 만든다. 실행 전 두 경로가 없음을 확인하고, 기존 파일이 있으면 다른 basename을 선택한다.
- 검토·인계가 끝난 뒤 이번 실행이 만든 두 경로만 삭제하고 부재를 확인한다.

```bash
rm -- '<OUTPUT_BASENAME>.json' '<OUTPUT_BASENAME>.yaml'
test ! -e '<OUTPUT_BASENAME>.json' && test ! -e '<OUTPUT_BASENAME>.yaml'
```

삭제 실패 시 basename 오기, 파일 소유권과 다른 프로세스가 파일을 열고 있는지 먼저 확인한다. 서버 감사 기록은 로컬 출력 삭제로 복원되지 않는다.

## 관련 서비스

- [[SMB 서비스]]
- [[RPC와 NetBIOS 서비스]]
- [[LDAP 서비스]]

## 관련 도구

- [[rpcclient]]
- [[enum4linux-ng]]
- [[crackmapexec]]
- [[ldapsearch]]
- [[PowerView]]

## 관련 상태 라우터

- [[무인증 내부 네트워크에서 AD 단서 확인]]
- [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 참고 링크

- [Microsoft Learn — Get-ADDefaultDomainPasswordPolicy](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-addefaultdomainpasswordpolicy)
- [Microsoft Learn — Get-ADFineGrainedPasswordPolicy](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adfinegrainedpasswordpolicy)
- [Microsoft Learn — Get-ADUserResultantPasswordPolicy](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduserresultantpasswordpolicy)
