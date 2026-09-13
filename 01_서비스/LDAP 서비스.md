---
tags:
  - 환경/ad
  - 서비스/ldap
대표포트:
  - "T:389"
  - "T:636"
  - "T:3268"
  - "T:3269"
서비스:
  - LDAP
  - LDAPS
  - Global Catalog
---

# LDAP 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Lightweight Directory Access Protocol(LDAP), LDAP over TLS(LDAPS) 또는 Global Catalog 포트에 연결할 수 있고, 아직 디렉터리에 bind할 AD 계정이나 객체 권한은 확인하지 않은 상태에서 시작한다. Root Directory Service Entry(RootDSE)로 도메인 Distinguished Name(DN), DNS 호스트명과 Active Directory Domain Services(AD DS) 구현 단서를 확정한 뒤 익명 bind, 인증된 읽기, 객체별 쓰기와 디렉터리 복제 권한을 구분한다.

**첫 화면 상태:** 지금 가능한 일은 RootDSE와 서버가 허용한 범위의 디렉터리 조회다. 성공하면 도메인·도메인 컨트롤러와 현재 bind 주체가 읽을 수 있는 객체·속성을 얻는다. 연결 실패 시 389의 StartTLS와 636/3269의 TLS·인증서·이름 해석을, `invalidCredentials`는 계정 형식과 인증 자료를, `strongerAuthRequired`는 signing·TLS를, `insufficientAccessRights`는 대상 객체와 현재 계정의 접근 제어 목록(ACL)을 다시 확인한다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | RootDSE bootstrap | `ldapsearch -x -H ldap://<TARGET> -s base -b "" defaultNamingContext dnsHostName supportedLDAPVersion supportedSASLMechanisms supportedCapabilities` | base DN, DC FQDN, 지원 LDAP/SASL 기능과 AD DS 구현 단서를 확인한다. |
| 2 | realm·이름 해석 | `defaultNamingContext`를 DNS 이름으로 변환하고 `dig <DC_FQDN>`으로 확인 | `<DOMAIN>`/Kerberos realm, DC FQDN과 DNS 해석을 맞춘다. |
| 3 | 익명 bind 범위 | `ldapsearch -x -H ldap://<TARGET> -b "<BASE_DN>"` | 인증 없이 조회되는 객체·속성·정책 범위를 확인한다. |
| 4 | 인증된 사용자 열거 | `netexec ldap <TARGET> -u <USER> -p <PASSWORD> --users` | 계정 인증 성공과 그 계정의 사용자·도메인 정보 읽기 범위를 확인한다. |
| 5 | SPN과 pre-auth 후보 | `impacket-GetUserSPNs '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <TARGET>`, `impacket-GetNPUsers <DOMAIN>/ -usersfile users.txt -dc-ip <TARGET> -no-pass` | SPN 계정과 `$krb5asrep$` 출력을 확인한다. |
| 6 | LDAP 기능·정책 단서 | `nmap --script ldap-rootdse,ldap-search -p389,636 <TARGET>` | RootDSE 결과와 signing·channel binding 등 relay 보호 조건의 추가 확인 필요성을 판단한다. |

**출력 해석 경계:** `defaultNamingContext`와 `dnsHostName`은 조회한 디렉터리의 도메인 컨텍스트를 확정하지만, 공격 호스트의 DNS 해석이나 Kerberos 인증 성공은 확정하지 않는다. `--users` 성공은 해당 계정의 LDAP 읽기 범위만 확정한다. SPN, AS-REP hash, ACL 또는 복제 관련 ACE가 보여도 hash 크래킹, 객체 변경, DCSync 성공은 각 기법에서 별도로 검증한다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| RootDSE에서 도메인 DN·DC FQDN·AD DS 단서 | [[AD 도메인 컨텍스트 기본 확인]] | `ldapsearch`, `klist` | 도메인·DC·현재 사용 중인 AD 계정과 서비스 도달성 |
| 사용자·그룹·컴퓨터 열거 | [[원격 비밀번호 공격]] | `netexec`, `kerbrute` | SMB/WinRM/RDP/SSH/메일처럼 실제 지원되는 서비스에서 검증할 사용자 이름·비밀번호 후보 |
| SPN 사용자 계정 | [[Kerberoasting]] | `impacket-GetUserSPNs`, `rubeus` | offline cracking 대상 TGS hash |
| pre-auth 미요구 계정 | [[AS-REP Roasting]] | `impacket-GetNPUsers`, `rubeus` | offline cracking 대상 AS-REP hash |
| 프린터·애플리케이션 관리 화면의 LDAP `Test Connection`과 TLS 없는 simple bind 설정 | [[LDAP Test Connection 자격 증명 노출 검증]] | `tcpdump`, `netcat` | 장비의 bind 전송·평문 노출 여부와 설정 복구 상태 |
| LDAP signing·channel binding 조건이 relay를 허용할 가능성 | [[NTLM Relay 조건 검토]] | `impacket-ntlmrelayx` | relay된 계정 권한으로 가능한 LDAP action 후보 |
| 사용자·컴퓨터 객체의 `msDS-KeyCredentialLink` 쓰기 권한 | [[Shadow Credentials]] | `pywhisker`, `certipy` | 대상 객체에 KeyCredential을 추가해 certificate와 Kerberos ticket을 발급받을 가능성 |
| `DS-Replication-Get-Changes` 계열 복제 권한 | [[DCSync]] | `impacket-secretsdump`, `mimikatz` | 검증된 디렉터리 복제 가능성 |
| Kerberos ticket 기반 LDAP 접근 | [[Pass the Ticket]] | `klist`, `netexec` | ticket 주체의 LDAP 읽기·쓰기 범위 |

## 서비스 고유 주의 사항

- LDAPS만 허용되면 636/3269 연결과 인증서 신뢰 문제를 확인한다.
- 익명 bind 실패는 정상 설정일 수 있으며 유효한 AD 계정 비밀번호·NT hash 또는 Kerberos ticket으로 조회 범위를 다시 확인한다.
- LDAP 읽기, 특정 객체 쓰기, relay action과 DCSync 복제 권한은 각각 다른 상태다.
- relay는 signing, channel binding, EPA와 relay된 계정의 실제 객체 권한에 영향을 받는다.
