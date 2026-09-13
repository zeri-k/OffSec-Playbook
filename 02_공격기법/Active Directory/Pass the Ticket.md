---
tags:
  - 환경/ad
  - 서비스/kerberos
시작조건: ["Kerberos TGT 또는 특정 SPN의 TGS 확보", "ticket을 적용할 Windows 로그온 세션 또는 Linux 셀 확보"]
필요권한: ["Windows ticket 주입 또는 Linux ccache 읽기 권한", "ticket 주체의 대상 서비스 권한"]
필요조건: ["ticket principal·TGT/TGS·SPN·만료 시각 확인", "DNS·FQDN·realm·시간 정합", "TGT 사용 시 <DC_FQDN>:88 도달 가능", "실행 호스트에서 <HOST_FQDN>:<SERVICE_PORT> 도달 가능"]
결과: ["현재 실행 세션의 Kerberos ticket cache 적용 상태", "ticket 주체 권한 범위의 서비스 인증", "허용된 share·원격 명령·세션 접근"]
---

# Pass the Ticket

## 한 줄 판단

Kerberos TGT 또는 특정 SPN의 TGS를 보유하고 ticket을 적용할 Windows·Linux 호스트에서 `<HOST_FQDN>:<SERVICE_PORT>`에 도달할 수 있으면, 현재 실행 세션에 ticket을 주입·지정해 ticket 주체가 허용된 SMB·WinRM·WMI·LDAP 접근 범위를 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | ticket을 적용할 Windows·Linux 호스트에서 DNS, 필요 시 `<DC_FQDN>:88`, `<HOST_FQDN>:<SERVICE_PORT>` 도달 가능 | FQDN 해석, KDC·서비스 포트 연결과 시간 정합 확인 | DNS·route·방화벽·realm·DC 시간 확인 |
| 현재 계정 또는 인증 수단 | `.kirbi`·`.ccache`·base64 ticket의 principal·TGT/TGS·SPN·만료 시각 확인 | 주입·지정 후 같은 세션의 `klist` | 만료·SPN 불일치·잘못된 principal이면 적합한 ticket 확보 |
| 현재 권한 | Windows ticket 주입 또는 Linux ccache 읽기·환경 지정 가능 | Rubeus·Mimikatz 주입 출력, `export KRB5CCNAME` 후 `klist` | 주입 권한, ccache 파일 권한과 실행 세션 재확인 |
| 공격 대상의 조건 | TGS의 SPN 또는 TGT로 요청할 SPN이 `<HOST_FQDN>` 서비스와 일치하고 ticket 주체에게 해당 서비스 권한 존재 | `klist`의 SPN과 SMB·WinRM·WMI·LDAP 실제 접속 결과 대조 | IP 대신 FQDN, 올바른 SPN·realm, 계정 그룹·ACL 확인 |
| 필요한 파일·목록·주소 | `<TICKET.kirbi>` 또는 `/tmp/user.ccache`, `<DC_FQDN>`, `<HOST_FQDN>`, `<REALM>`, `<SHARE>` | 파일 경로, realm·FQDN·SPN 대응 확인 | ticket 파일·realm·FQDN·share 수정 |

## 실행

`<TICKET.kirbi>`와 `<CCACHE_FILE>`은 ticket을 적용할 호스트의 파일이며, `<DC_FQDN>`·`<HOST_FQDN>`·`<REALM>`·`<SHARE>`는 TGT/TGS에 표시된 서비스 경계와 일치해야 한다. PID·LUID는 주입 출력에서, principal·SPN·만료 시각은 `klist`에서 다시 확인한다.

### Ticket 종류 선택

| 보유 ticket | 사용할 때 | 필요한 네트워크 경로 | 사용 범위 |
|---|---|---|---|
| TGT | 여러 서비스의 TGS를 새로 요청하거나 아직 최종 SPN을 정하지 않았을 때 | 실행 호스트→`<DC_FQDN>:88`, 실행 호스트→`<HOST_FQDN>:<SERVICE_PORT>` | TGT 주체로 새 TGS를 요청할 수 있지만 서비스 권한은 별도 확인 |
| 특정 서비스용 TGS | `klist`의 `Server`에 표시된 SPN과 최종 서비스가 정확히 일치할 때 | 실행 호스트→`<HOST_FQDN>:<SERVICE_PORT>` | 표시된 SPN에만 제시할 수 있으며 다른 서비스·호스트에는 재사용 불가 |

### 실행 환경 선택

| ticket 형식·환경 | 적용 방식 | 명령 실행 위치 | 성공 기준 |
|---|---|---|---|
| `.kirbi` 또는 Rubeus base64 ticket을 사용하는 Windows 세션 | Rubeus·Mimikatz로 현재 로그온 세션에 주입 | 최종 서비스에 도달하는 Windows 호스트의 같은 로그온 세션 | import 성공과 `klist`의 cache 항목 확인. 실제 사용은 서비스 응답으로 별도 확인 |
| `.ccache`를 사용하는 Linux 셀 | `KRB5CCNAME`으로 현재 shell과 자식 프로세스에 지정 | 최종 서비스에 도달하고 ccache를 읽을 수 있는 Linux 호스트 | `klist`의 cache 항목 확인 뒤 `-k -no-pass` 또는 Kerberos 지원 클라이언트의 서비스 응답으로 실제 사용 확인 |

1. ticket의 principal, TGT·TGS 종류, TGS라면 `Server` SPN과 만료 시각을 확인한다.
2. TGT를 사용할 때는 `<DC_FQDN>:88`, 모든 방식에서 `<HOST_FQDN>:<SERVICE_PORT>` 도달성을 확인한다.
3. Windows는 가능하면 ticket 전용 `/netonly` 로그온 세션에 주입하고, Linux는 원본 ccache를 작업용 cache로 복사해 서비스 클라이언트에만 지정한다.
4. `klist`로 현재 실행 컨텍스트의 ticket을 확인한다.
5. FQDN을 사용해 SMB·WinRM·WMI·LDAP 서비스 인증과 실제 허용 행동을 각각 검증한다.

### 명령과 확인할 출력

#### Windows에서 ticket 주입

Rubeus 2.0.3 이상이면 현재 업무 로그온 세션을 오염시키지 않도록 ticket이 들어간 전용 process를 만든다. 출력된 PID와 LUID를 각각 `<PTT_PROCESS_PID>`와 `<PTT_LOGON_LUID>`로 기록하고, 생성된 `cmd.exe` 안에서 서비스 client를 실행한다.

```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show /ticket:<TICKET.kirbi>
REM 새로 열린 cmd.exe에서 실행
klist
```

Rubeus 2.0.2 이하에서 현재 로그온 세션에 직접 주입하는 아래 방식은 해당 세션의 기존 ticket과 분리해 제거하기 어렵다. 격리된 일회성 로그온 세션에서만 fallback으로 사용한다.

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
test ! -e '<PTT_WORK_CCACHE>'
cp -- '<CCACHE_FILE>' '<PTT_WORK_CCACHE>'
chmod 600 '<PTT_WORK_CCACHE>'
KRB5CCNAME='<PTT_WORK_CCACHE>' klist
KRB5CCNAME='<PTT_WORK_CCACHE>' smbclient -k //<HOST>/<SHARE>
KRB5CCNAME='<PTT_WORK_CCACHE>' evil-winrm -i <HOST_FQDN> -r <REALM>
```

확인할 출력:

- `klist`의 ticket principal·종류·SPN·만료 시각과 Kerberos 기반 서비스 응답.
- share 목록이나 WinRM prompt가 반환되어야 서비스 접근이 확인되며, `KRB5CCNAME` 지정만으로는 인증 성공을 판정하지 않는다.

## 변경 영향과 복구

서비스 client를 먼저 정상 종료한 뒤 ticket cache와 전용 로그온 process를 정리한다. 원본 `<TICKET.kirbi>`·`<CCACHE_FILE>`은 이 문서가 만든 항목이 아니므로 증거 보존 정책에 따라 별도 처리하고 복구 명령에서 삭제하지 않는다.

| 생성·변경 항목 | 기록할 기존 값·식별값 | 종료·정리 명령 | 완료 확인 |
|---|---|---|---|
| Windows Rubeus 전용 로그온 세션 | Rubeus `createnetonly`가 출력한 `<PTT_PROCESS_PID>`·`<PTT_LOGON_LUID>` | 서비스 client 종료 뒤 관리자 셸이면 `Rubeus.exe purge /luid:<PTT_LOGON_LUID>`, 이어서 `taskkill /PID <PTT_PROCESS_PID> /T` | `tasklist /FI "PID eq <PTT_PROCESS_PID>"`에 process가 없고 해당 전용 세션이 종료됨 |
| Windows 현재 로그온 세션 직접 주입 | 실행 전·후 `klist`, 세션 ID와 기존 ticket 목록 | 격리된 일회성 로그온 세션에서만 `klist purge` 후 로그오프 | 세션 종료. 공유 세션이면 다른 ticket을 보존하면서 주입 ticket만 되돌릴 수 없어 복구 제한으로 기록 |
| Linux 작업용 ccache | 기존에 없던 `<PTT_WORK_CCACHE>` exact 경로 | SMB·WinRM 등 client를 먼저 `exit`한 뒤 `kdestroy -c 'FILE:<PTT_WORK_CCACHE>'` | `test ! -e '<PTT_WORK_CCACHE>'`가 성공하고 원본 `<CCACHE_FILE>`은 그대로 존재 |

LUID 대상 `purge`는 상승된 권한이 필요하다. 권한이 없으면 다른 session의 ticket을 건드리지 말고 기록한 process tree를 종료해 전용 로그온 세션을 해제한다. 종료 후에도 세션이 남으면 exact PID·LUID와 하위 process를 다시 확인하며 process 이름 전체를 종료하지 않는다. KDC·서비스 감사 기록과 이미 수행한 원격 접근은 되돌릴 수 없다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `klist`에 주입·지정한 ticket이 표시된다. | 현재 세션의 cache에 해당 ticket 자료가 존재함 | Kerberos ticket cache 적용 | 시간 필드·principal·SPN을 확인한 뒤 해당 서비스에 연결해 실제 사용 검증 |
| Kerberos 옵션으로 SMB share, WinRM, WMI 접속이 성공한다. | ticket 주체의 서비스 권한 확인 | 서비스 접근 또는 세션 | WinRM은 [[WinRM 원격 PowerShell 세션]], WMI는 [[WMI 원격 명령 실행]]으로 전환 |
| ticket 주체에게 복제 권한이 있고 DC 복제 요청이 성공한다. | 복제 권한 확인 | 도메인 복제 권한 | [[DCSync]]에서 요청 가능한 naming context와 권한 확인 |
| KRB_AP_ERR_SKEW | 시간 불일치 | 시작 상태 유지 | DC 시간 동기화 |
| TGT는 보이지만 새 TGS 요청에서 KDC/SRV 오류 | DC 도달성, DNS·FQDN·realm 또는 시간 오류 | TGT 보유, 새 TGS 미발급 | `<DC_FQDN>:88`, `/etc/hosts`, `krb5.conf`, realm과 시간 확인 |
| 특정 TGS가 보이지만 서비스에서 SPN 오류 | ticket의 `Server`와 접속한 FQDN·서비스가 일치하지 않음 | 특정 TGS 보유, 대상 서비스 인증 미확인 | IP 대신 ticket에 맞는 FQDN을 사용하고 SPN·서비스 종류 확인 |
| 서비스가 Kerberos 인증 성공을 먼저 표시한 뒤 객체·share 접근 거부 | 인증 자료는 서비스에 수락됐지만 요청한 동작 권한 부족 | 인증된 서비스, 대상 동작 권한 미확보 | ticket 주체의 그룹·서비스 ACL과 로그온 권한 확인 |
| 인증 성공 여부를 확인하기 전 접근 거부 | cache 존재만 확인됐고 인증·인가 중 첫 실패 경계가 불명확함 | ticket 자료 보유, 서비스 접근 미확정 | 서비스의 Kerberos 오류·로그온 결과를 먼저 확인한 뒤 ACL 판단 |

## 확인할 출력과 권한

- ticket 보유: `klist`에 principal·TGT/TGS·SPN·만료 시각이 보이는 상태다.
- ticket cache 적용: Windows import 또는 Linux `KRB5CCNAME` 지정 뒤 같은 실행 컨텍스트의 `klist`에 자료가 보이는지 확인한다. 이 단계는 KDC 재검증이나 서비스 수락을 뜻하지 않는다.
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

## 참고 링크

- [Rubeus](https://github.com/GhostPack/Rubeus)
- [Microsoft klist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist)
- [MIT Kerberos kdestroy](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html)
