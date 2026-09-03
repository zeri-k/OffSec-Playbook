---
tags:
  - 환경/ad
  - 서비스/kerberos
시작조건: ["Kerberos TGT 또는 특정 SPN의 TGS 확보", "ticket을 적용할 Windows 로그온 세션 또는 Linux 셀 확보"]
필요권한: ["Windows ticket 주입 또는 Linux ccache 읽기 권한", "ticket 주체의 대상 서비스 권한"]
필요조건: ["ticket principal·TGT/TGS·SPN·만료 시각 확인", "DNS·FQDN·realm·시간 정합", "TGT 사용 시 <DC_FQDN>:88 도달 가능", "실행 호스트에서 <HOST_FQDN>:<SERVICE_PORT> 도달 가능"]
결과: ["현재 실행 세션의 Kerberos ticket 사용 상태", "ticket 주체 권한 범위의 서비스 인증", "허용된 share·원격 명령·세션 접근"]
---

# Pass the Ticket

## 한 줄 판단

Kerberos TGT 또는 특정 SPN의 TGS를 보유하고 ticket을 적용할 Windows·Linux 호스트에서 `<HOST_FQDN>:<SERVICE_PORT>`에 도달할 수 있으면, 현재 실행 세션에 ticket을 주입·지정해 ticket 주체가 허용된 SMB·WinRM·WMI·LDAP 접근 범위를 확인한다.

## 사용할 때

- 현재 보유 정보: `.kirbi`·`.ccache`·Rubeus base64 ticket의 principal, TGT/TGS 종류, SPN과 만료 시각을 알고 있다.
- 명령 실행 위치: Windows ticket은 적용할 로그온 세션에서 Rubeus·Mimikatz로 주입하고, Linux ticket은 ccache를 읽을 수 있는 셀에서 `KRB5CCNAME`으로 지정한 뒤 같은 환경에서 서비스 클라이언트를 실행한다.
- 도달해야 하는 대상: TGT로 service ticket을 요청하려면 `<DC_FQDN>:88`, 서비스를 검증하려면 `<HOST_FQDN>:445`·`5985/5986`·WMI/LDAP 등 선택한 포트에 도달해야 한다.
- 현재 계정·권한: 명령을 실행하는 로컬 계정과 ticket principal은 서로 다를 수 있다. ticket 주체의 서비스 권한이 실제 행동 범위를 결정한다.
- 지금 가능한 행동·성공 범위: TGT는 추가 service ticket 요청, TGS는 표시된 SPN에 사용한다. ticket 주입·`klist` 표시, 서비스 인증, share·세션·원격 명령, 관리자·복제 권한을 각각 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | ticket을 적용할 Windows·Linux 호스트에서 DNS, 필요 시 `<DC_FQDN>:88`, `<HOST_FQDN>:<SERVICE_PORT>` 도달 가능 | FQDN 해석, KDC·서비스 포트 연결과 시간 정합 확인 | DNS·route·방화벽·realm·DC 시간 확인 |
| 현재 계정 또는 인증 수단 | `.kirbi`·`.ccache`·base64 ticket의 principal·TGT/TGS·SPN·만료 시각 확인 | 주입·지정 후 같은 세션의 `klist` | 만료·SPN 불일치·잘못된 principal이면 적합한 ticket 확보 |
| 현재 권한 | Windows ticket 주입 또는 Linux ccache 읽기·환경 지정 가능 | Rubeus·Mimikatz 주입 출력, `export KRB5CCNAME` 후 `klist` | 주입 권한, ccache 파일 권한과 실행 세션 재확인 |
| 공격 대상의 조건 | TGS의 SPN 또는 TGT로 요청할 SPN이 `<HOST_FQDN>` 서비스와 일치하고 ticket 주체에게 해당 서비스 권한 존재 | `klist`의 SPN과 SMB·WinRM·WMI·LDAP 실제 접속 결과 대조 | IP 대신 FQDN, 올바른 SPN·realm, 계정 그룹·ACL 확인 |
| 필요한 파일·목록·주소 | `<TICKET.kirbi>` 또는 `/tmp/user.ccache`, `<DC_FQDN>`, `<HOST_FQDN>`, `<REALM>`, `<SHARE>` | 파일 경로, realm·FQDN·SPN 대응 확인 | ticket 파일·realm·FQDN·share 수정 |

## 실행

### Ticket 종류 선택

| 보유 ticket | 사용할 때 | 필요한 네트워크 경로 | 사용 범위 |
|---|---|---|---|
| TGT | 여러 서비스의 TGS를 새로 요청하거나 아직 최종 SPN을 정하지 않았을 때 | 실행 호스트→`<DC_FQDN>:88`, 실행 호스트→`<HOST_FQDN>:<SERVICE_PORT>` | TGT 주체로 새 TGS를 요청할 수 있지만 서비스 권한은 별도 확인 |
| 특정 서비스용 TGS | `klist`의 `Server`에 표시된 SPN과 최종 서비스가 정확히 일치할 때 | 실행 호스트→`<HOST_FQDN>:<SERVICE_PORT>` | 표시된 SPN에만 제시할 수 있으며 다른 서비스·호스트에는 재사용 불가 |

### 실행 환경 선택

| ticket 형식·환경 | 적용 방식 | 명령 실행 위치 | 성공 기준 |
|---|---|---|---|
| `.kirbi` 또는 Rubeus base64 ticket을 사용하는 Windows 세션 | Rubeus·Mimikatz로 현재 로그온 세션에 주입 | 최종 서비스에 도달하는 Windows 호스트의 같은 로그온 세션 | import 성공과 `klist`의 principal·Server·만료 시각 확인 |
| `.ccache`를 사용하는 Linux 셀 | `KRB5CCNAME`으로 현재 shell과 자식 프로세스에 지정 | 최종 서비스에 도달하고 ccache를 읽을 수 있는 Linux 호스트 | `klist` 확인 뒤 `-k -no-pass` 또는 Kerberos 지원 클라이언트의 서비스 응답 |

1. ticket의 principal, TGT·TGS 종류, TGS라면 `Server` SPN과 만료 시각을 확인한다.
2. TGT를 사용할 때는 `<DC_FQDN>:88`, 모든 방식에서 `<HOST_FQDN>:<SERVICE_PORT>` 도달성을 확인한다.
3. Windows는 사용할 로그온 세션에 주입하고, Linux는 서비스 클라이언트를 실행할 같은 shell에서 `KRB5CCNAME`으로 지정한다.
4. `klist`로 현재 실행 컨텍스트의 ticket을 확인한다.
5. FQDN을 사용해 SMB·WinRM·WMI·LDAP 서비스 인증과 실제 허용 행동을 각각 검증한다.

### 명령과 확인할 출력

#### Windows에서 ticket 주입

```cmd
Rubeus.exe ptt /ticket:<TICKET.kirbi>
mimikatz.exe "kerberos::ptt <TICKET.kirbi>" exit
klist
```

확인할 출력:

- `ticket successfully imported`와 `klist`의 ticket principal, `Server` SPN, 시작·만료 시각.
- import 성공은 현재 Windows 로그온 세션이 ticket을 보유한다는 뜻이며 해당 서비스 인증이나 관리자 권한을 입증하지 않는다.

#### Linux에서 ccache 사용

```bash
export KRB5CCNAME=/tmp/user.ccache
klist
smbclient -k //<HOST>/<SHARE>
evil-winrm -i <HOST_FQDN> -r <REALM>
```

확인할 출력:

- `klist`의 ticket principal·종류·SPN·만료 시각과 Kerberos 기반 서비스 응답.
- share 목록이나 WinRM prompt가 반환되어야 서비스 접근이 확인되며, `KRB5CCNAME` 지정만으로는 인증 성공을 판정하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `klist`에 주입한 ticket이 표시된다. | 현재 세션의 ticket 사용 가능 | Kerberos ticket | ticket에 표시된 계정, SPN과 만료 시각을 확인한 뒤 해당 서비스에 연결 |
| Kerberos 옵션으로 SMB share, WinRM, WMI 접속이 성공한다. | ticket 주체의 서비스 권한 확인 | 서비스 접근 또는 세션 | WinRM은 [[WinRM 원격 PowerShell 세션]], WMI는 [[WMI 원격 명령 실행]]으로 전환 |
| ticket 주체에게 복제 권한이 있고 DC 복제 요청이 성공한다. | 복제 권한 확인 | 도메인 복제 권한 | [[DCSync]]에서 요청 가능한 naming context와 권한 확인 |
| KRB_AP_ERR_SKEW | 시간 불일치 | 시작 상태 유지 | DC 시간 동기화 |
| TGT는 보이지만 새 TGS 요청에서 KDC/SRV 오류 | DC 도달성, DNS·FQDN·realm 또는 시간 오류 | TGT 보유, 새 TGS 미발급 | `<DC_FQDN>:88`, `/etc/hosts`, `krb5.conf`, realm과 시간 확인 |
| 특정 TGS가 보이지만 서비스에서 SPN 오류 | ticket의 `Server`와 접속한 FQDN·서비스가 일치하지 않음 | 특정 TGS 보유, 대상 서비스 인증 미확인 | IP 대신 ticket에 맞는 FQDN을 사용하고 SPN·서비스 종류 확인 |
| 접근 거부 | ticket은 유효하지만 ticket 주체의 대상 서비스 권한 부족 | 유효 ticket 보유, 대상 서비스 권한 미확보 | ticket 주체의 그룹·서비스 ACL과 로그온 권한 확인 |

## 확인할 출력과 권한

- ticket 보유: `klist`에 principal·TGT/TGS·SPN·만료 시각이 보이는 상태다.
- ticket 적용: Windows import 또는 Linux `KRB5CCNAME` 지정 뒤 같은 실행 컨텍스트의 `klist`에서 확인한다.
- 서비스 인증: ticket에 맞는 FQDN으로 접속해 SMB 목록·WinRM prompt·WMI 명령 응답처럼 서비스 고유 결과를 확인한다.
- 실제 권한: 인증 성공 뒤 읽기·쓰기·원격 명령·관리자·복제 권한을 각각 별도로 검증한다.

## 후속 공격 연결

- Kerberos key에서 TGT 생성: [[OverPass the Hash]]
- AD CS/Shadow Credentials: [[AD CS ESC8 NTLM Relay]], [[Shadow Credentials]]
- 복제 권한이 있으면: [[DCSync]]

## 관련 상태 라우터

- ticket 주체와 서비스 접근이 확인됐으면: [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- 새 세션을 아직 확보하지 못했고 ticket의 지원 서비스를 선택해야 하면: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[rubeus]]
- [[mimikatz]]
- [[klist]]
- [[impacket-ticketConverter]]
- [[evil-winrm]]
- [[impacket-wmiexec]]
