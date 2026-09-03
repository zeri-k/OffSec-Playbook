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

## 사용할 때

- Kerberos KDC에 접근할 수 있고 도메인명과 사용자 후보를 알고 있을 때.
- PowerView에서 `DONT_REQ_PREAUTH`가 설정된 계정을 확인했을 때.
- Kerbrute 사용자 열거 중 `has no pre auth required`와 AS-REP hash가 함께 반환됐을 때.

## 전제 조건

먼저 **명령을 실행할 위치에서 DC의 88번 포트에 접근할 수 있는지** 확인한다.

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
2. Rubeus, Kerbrute 또는 GetNPUsers.py로 AS-REP를 요청한다.
3. `$krb5asrep$` hash가 반환된 경우에만 Hashcat으로 오프라인 크래킹한다.
4. 평문이 복구되면 자격 증명 상태로 전환하고 실제 사용 범위는 별도 검증한다.

### Windows 공격 호스트에서 실행

#### PowerView로 대상 계정 확인

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

확인할 출력:

- `useraccountcontrol`의 `DONT_REQ_PREAUTH`.
- 이 결과는 취약 설정 후보이며, AS-REP hash 반환과 비밀번호 복구 성공을 각각 따로 확인한다.

#### Rubeus로 AS-REP 요청

```powershell
.\Rubeus.exe asreproast /user:<USER> /nowrap /format:hashcat
```

확인할 출력:

- `AS-REQ w/o preauth successful!`.
- Hashcat에 입력할 수 있는 `$krb5asrep$` 형식 hash.

### Linux 공격 호스트에서 실행

#### Kerbrute로 사용자 열거와 AS-REP 확인

```bash
kerbrute userenum -d <DOMAIN> --dc <DC_IP> <USER_LIST>
```

확인할 출력:

- `VALID USERNAME`은 사용자명 확인 결과일 뿐 AS-REP 수집 성공이 아니다.
- `has no pre auth required. Dumping hash to crack offline`와 이어지는 `$krb5asrep$` hash를 함께 확인한다.

#### GetNPUsers.py로 사용자 목록 확인

```bash
GetNPUsers.py <DOMAIN>/ -dc-ip <DC_IP> -no-pass -usersfile <USER_LIST>
```

확인할 출력:

- `$krb5asrep$` 형식 hash가 반환된 사용자.
- `doesn't have UF_DONT_REQUIRE_PREAUTH set`은 해당 사용자가 이 공격 조건을 충족하지 않는다는 뜻이다.

#### Hashcat으로 오프라인 크래킹

```bash
hashcat -m 18200 <ASREP_HASH_FILE> <WORDLIST> --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- `Status...........: Cracked`와 `Recovered........: 1/1`.
- hash 뒤에 표시되는 복구 비밀번호.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `DONT_REQ_PREAUTH`만 확인됨 | 취약 설정 후보이나 hash는 아직 없음 | AS-REP 요청 후보 | Rubeus 또는 GetNPUsers.py로 실제 hash 반환 확인 |
| Kerbrute의 `VALID USERNAME`만 확인됨 | 유효 사용자명만 확인됨 | 사용자 후보 | AS-REP hash가 함께 반환되는지 확인 |
| `$krb5asrep$` hash 반환 | pre-authentication 미요구 계정의 AS-REP 수집 성공 | 오프라인 크래킹 가능한 hash | [[오프라인 해시 크래킹]] |
| `doesn't have UF_DONT_REQUIRE_PREAUTH set` | 해당 사용자는 취약 조건을 충족하지 않음 | hash 미획득 | 다른 사용자 후보를 확인하거나 이 기법을 종료 |
| Hashcat `Status: Cracked`와 평문 출력 | 비밀번호 복구 성공 | 도메인 자격 증명 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| Hashcat 크래킹 미완료 | AS-REP hash는 있으나 평문을 복구하지 못함 | hash만 확보 | wordlist 적합성을 재평가하고 취약 설정을 발견 사항으로 기록 |

## 확인할 출력과 권한

- `$krb5asrep$` 반환은 AS-REP 수집 성공이며 평문 비밀번호나 서비스 접근 성공을 의미하지 않는다.
- `VALID USERNAME`과 `DONT_REQ_PREAUTH`는 서로 다른 상태다.
- 공격자가 사용할 유효한 도메인 계정은 필요하지 않지만, 공격 대상은 AD에 존재하는 도메인 사용자여야 한다.
- 공격 호스트의 도메인 가입 여부와 요청자에게 도메인 인증 수단이 필요한지는 별개의 조건이다.
- Hashcat의 `Cracked`와 복구 평문이 확인된 뒤에도 실제 서비스 접근과 권한은 별도로 검증한다.
- `GenericWrite` 또는 `GenericAll`로 pre-authentication 설정을 변경하는 경로는 대상 계정 상태를 바꾸므로 이 조회·수집 절차에 포함하지 않는다.

## 관련 서비스

- [[88_Kerberos]]
- [[389_636_LDAP]]

## 관련 도구

- [[PowerView]]
- [[rubeus]]
- [[kerbrute]]
- [[impacket-GetNPUsers]]
- [[hashcat]]

## 비교 기법

- 유효한 도메인 계정 또는 TGT로 SPN 계정의 서비스 티켓을 요청할 때: [[Kerberoasting]]

## 관련 상태 라우터

- 평문 비밀번호를 복구했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- hash를 복구하지 못했거나 다른 AD 후보를 확인할 때: [[AD Identity 확인 후 도메인 컨텍스트 열거]]
