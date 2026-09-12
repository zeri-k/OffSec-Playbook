---
tags:
  - 환경/ad
  - 서비스/kerberos
대표포트:
  - "T:88"
  - "U:88"
서비스:
  - Kerberos
  - KDC
---

# Kerberos 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Kerberos Key Distribution Center(KDC) TCP/UDP 88에 도달할 수 있고, 도메인 realm·사용자 이름·비밀번호·NT hash·AES key·ticket 중 하나 이상의 후보가 있지만 유효한 AD 계정이나 서비스 권한은 아직 확인하지 않은 상태에서 시작한다. 도메인 컨트롤러(DC) 이름 해석과 시간을 맞춘 뒤 사용자 존재, 계정 인증, Ticket-Granting Ticket(TGT), service ticket과 Kerberos key를 서로 다른 상태로 판단한다.

**첫 화면 상태:** 지금 가능한 일은 사용자 이름·ticket·Kerberos key의 유효성을 나누어 확인하는 것이다. 성공하면 오프라인 크래킹 대상 hash 또는 ticket 기반 서비스 인증 후보를 얻는다. `KRB_AP_ERR_SKEW`는 시간, `KDC_ERR_C_PRINCIPAL_UNKNOWN`은 사용자·realm, `KDC_ERR_PREAUTH_FAILED`는 비밀번호·Kerberos key, `KDC_ERR_S_PRINCIPAL_UNKNOWN`은 대상 Service Principal Name(SPN)과 DNS를 확인한다. 유효 사용자·TGT 보유가 SMB·LDAP·WinRM 권한을 보장하지 않는다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | realm·DC FQDN bootstrap | [[AD 도메인 컨텍스트 기본 확인]]의 RootDSE·DNS·현재 사용 중인 AD 계정 확인 | `<DOMAIN>`/realm, DC FQDN과 이름 해석을 먼저 맞춘다. |
| 2 | TCP·UDP KDC 응답 | `nmap -sS -sU -sV -p T:88,U:88 <TARGET>` | KDC 후보와 TCP/UDP 접근 차이를 확인한다. |
| 3 | 도메인 시간 | `net time /domain`, `w32tm /stripchart /computer:<DC>` | clock skew가 있으면 동기화 후 인증을 재시도한다. |
| 4 | 유효 사용자 | `kerbrute userenum --dc <TARGET> -d <DOMAIN> users.txt` | `valid user`를 인증 성공과 구분해 저장한다. |
| 5 | pre-auth 미요구 계정 | `impacket-GetNPUsers <DOMAIN>/ -usersfile users.txt -dc-ip <TARGET> -no-pass` | `$krb5asrep$` 출력 여부를 확인한다. |
| 6 | SPN과 TGS hash | `impacket-GetUserSPNs '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <TARGET> -request` | SPN 목록과 `$krb5tgs$` 출력을 확인한다. |
| 7 | ticket cache와 실제 인증 | `klist`, `netexec smb <TARGET> -k --use-kcache` | TGT/TGS의 realm·만료 시간과 Kerberos 기반 서비스 접근을 연결한다. |

**출력 해석 경계:** TCP/UDP 88 응답은 KDC 후보로의 도달성만 확정하며 realm 설정·유효 계정·취약점은 확정하지 않는다. `valid user`는 사용자명 존재 단서이고 비밀번호 인증이나 권한을 증명하지 않는다. `$krb5asrep$`·`$krb5tgs$` 출력은 오프라인 분석 자료이고, `klist`의 TGT/TGS 표시는 현재 cache에 ticket 자료가 있다는 뜻이다. KDC 발급 요청 성공과 SMB·LDAP·WinRM의 서비스 인증·권한은 각각 해당 요청의 응답으로 별도 검증한다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| realm·DC·현재 사용 중인 AD 계정이 불명확함 | [[AD 도메인 컨텍스트 기본 확인]] | `klist`, `ldapsearch`, `nslookup` | 도메인·DC와 실제 AD Identity 컨텍스트 |
| 유효 사용자 이름 | [[원격 비밀번호 공격]] | `kerbrute`, `netexec` | SMB/WinRM/RDP/SSH/메일처럼 실제 지원되는 인증 서비스에서 검증할 사용자 이름·비밀번호 후보 |
| pre-auth 미요구 계정 또는 `$krb5asrep$` | [[AS-REP Roasting]] | `impacket-GetNPUsers`, `rubeus` | offline cracking 대상 AS-REP hash |
| SPN 사용자 계정 또는 `$krb5tgs$` | [[Kerberoasting]] | `impacket-GetUserSPNs`, `rubeus` | offline cracking 대상 TGS hash |
| TGT/TGS 또는 `.kirbi` | [[Pass the Ticket]] | `klist`, `rubeus`, `impacket-ticketConverter` | cache 적용 뒤 별도 검증할 ticket 주체의 서비스 인증 후보 |
| NTLM/AES key | [[OverPass the Hash]] | `rubeus`, `mimikatz` | 새 TGT 발급 가능성 |
| Linux ccache/keytab | [[Linux Kerberos keytab ccache 악용]] | `klist`, `impacket-ticketConverter` | Linux에서 재사용 가능한 Kerberos identity |
| Kerberos 기반 WinRM 접근 | [[WinRM 원격 PowerShell 세션]] | `evil-winrm`, `netexec` | ticket 주체의 원격 PowerShell 권한 |
| DC에 도달하며 복제 권한이 있는 AD 계정의 비밀번호·NT hash·Kerberos ticket | [[DCSync]] | `impacket-secretsdump`, `mimikatz` | 검증된 디렉터리 복제 가능성 |
| 사용자·컴퓨터 객체의 `msDS-KeyCredentialLink` 쓰기 권한 | [[Shadow Credentials]] | `certipy`, `pywhisker` | 대상 객체에 KeyCredential을 추가해 certificate와 Kerberos ticket을 발급받을 가능성 |

## 서비스 고유 주의 사항

- FQDN, realm 대소문자, `/etc/hosts`, DNS 해석과 DC 시간을 일치시킨다.
- 사용자 열거 성공, 인증 성공, 일반 사용자 권한, 관리자·복제 권한은 서로 다르다.
- NTLM 차단 환경에서는 Kerberos ticket 기반 접근 가능성을 별도로 확인한다.
