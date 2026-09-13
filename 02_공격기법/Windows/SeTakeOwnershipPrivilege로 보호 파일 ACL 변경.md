---
tags:
  - 환경/windows
문서역할: 수동절차
시작조건: ["대상 Windows 호스트에서 명령 실행", "현재 token에 SeTakeOwnershipPrivilege가 존재"]
필요권한: ["현재 token에 할당된 SeTakeOwnershipPrivilege", "대상 파일의 ACL 변경에 필요한 권한"]
필요조건: ["대상 파일 경로와 원래 owner·ACL 기록"]
결과: ["대상 파일의 소유권과 읽기 권한", "파일에서 확인한 정보·자격 증명 후보"]
---

# SeTakeOwnershipPrivilege로 보호 파일 ACL 변경

## 한 줄 판단

현재 token에 `SeTakeOwnershipPrivilege`가 있고 변경할 대상 파일이 있으면, 필요한 경우 그 privilege 하나만 현재 PowerShell process에서 활성화하고 소유권·최소 ACL을 변경해 파일을 읽은 뒤 원래 보안 설명자와 privilege 상태로 복구한다.

## 사용할 때

- `whoami /priv`에서 `SeTakeOwnershipPrivilege`가 `Enabled` 또는 `Disabled`로 존재하고, 파일은 나열할 수 있지만 내용 읽기가 거부될 때.
- 파일·폴더·레지스트리 같은 securable object의 owner·ACL 변경이 실제 대상의 동작에 영향을 줄 수 있을 때.
- 이미 읽기 가능한 다른 정보 수집 경로가 없고, 파일 경로·원래 owner·ACL을 기록할 수 있을 때.

## 전제 조건

특권의 할당·활성화와 ACL 변경은 [[Windows 액세스 토큰과 특권 활성화]]의 순서로 먼저 구분한다.

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 token 권한 | `SeTakeOwnershipPrivilege`가 현재 token에 존재 | `whoami /priv` | `Disabled`면 아래 선택적 활성화 후 재확인하고, 목록에 없으면 현재 token에서 활성화할 수 없으므로 다른 기법 선택 |
| 대상 | 파일 경로·현재 owner·ACL과 업무 영향 파악 | `Get-Acl`, `icacls` | 민감 파일·실행 중 설정 파일이면 변경 영향을 먼저 확인 |
| 복구 정보 | 변경 전 owner와 DACL, 덮어쓰지 않을 ACL backup 경로가 기록됨 | `Get-Acl`, `icacls /save`, `Test-Path` | 기록 없이 소유권·ACL 변경을 시작하지 않음 |
| 실행 위치 | 파일이 존재하는 대상 Windows 호스트 | `hostname`, `Test-Path` | 대상 세션·경로를 재확인 |

## 실행

먼저 privilege와 원래 보안 설명자를 기록한다. 이 문서는 교육 원천의 모든 token privilege를 한꺼번에 활성화하는 script를 사용하지 않는다. `AdjustTokenPrivileges`는 현재 token에 이미 있는 privilege만 조정하며, API 자체가 성공을 반환해도 `GetLastError()==ERROR_NOT_ALL_ASSIGNED(1300)`이면 요청한 privilege가 token에 없었던 상태다.

```powershell
whoami /priv
Get-Acl '<TARGET_FILE>' | Format-List Owner,Access
icacls '<TARGET_FILE>'
Test-Path -LiteralPath '<ACL_BACKUP_FILE>'
icacls '<TARGET_FILE>' /save '<ACL_BACKUP_FILE>'
```

`Test-Path`가 `False`인 고유한 `<ACL_BACKUP_FILE>`을 사용한다. `icacls /save`는 DACL 복구 입력이고 owner는 포함하지 않으므로 `Get-Acl`의 정확한 `<ORIGINAL_OWNER>`를 별도로 기록한다.

### Disabled privilege 하나만 현재 process에서 활성화

`whoami /priv`에서 `Disabled`로 확인한 경우에만 같은 PowerShell process에서 아래 P/Invoke를 정의한다. Windows PowerShell 5.1과 PowerShell 7 모두 Win32 `advapi32.dll`을 호출하며, 별도 script 파일이나 system-wide 설정을 만들지 않는다.

```powershell
$PRIVILEGE_WAS_DISABLED = $true
$PRIVILEGE_PROCESS_PID = $PID
Get-Process -Id $PRIVILEGE_PROCESS_PID | Select-Object Id,StartTime,Path

Add-Type -TypeDefinition @'
using System;
using System.ComponentModel;
using System.Runtime.InteropServices;

public static class TokenPrivilege
{
    private const UInt32 TOKEN_QUERY = 0x0008;
    private const UInt32 TOKEN_ADJUST_PRIVILEGES = 0x0020;
    private const UInt32 SE_PRIVILEGE_ENABLED = 0x00000002;

    [StructLayout(LayoutKind.Sequential)]
    private struct LUID
    {
        public UInt32 LowPart;
        public Int32 HighPart;
    }

    [StructLayout(LayoutKind.Sequential)]
    private struct TOKEN_PRIVILEGES
    {
        public UInt32 PrivilegeCount;
        public LUID Luid;
        public UInt32 Attributes;
    }

    [DllImport("kernel32.dll")]
    private static extern IntPtr GetCurrentProcess();

    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern bool CloseHandle(IntPtr handle);

    [DllImport("advapi32.dll", SetLastError = true)]
    private static extern bool OpenProcessToken(IntPtr processHandle, UInt32 desiredAccess, out IntPtr tokenHandle);

    [DllImport("advapi32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    private static extern bool LookupPrivilegeValue(string systemName, string name, out LUID luid);

    [DllImport("advapi32.dll", SetLastError = true)]
    private static extern bool AdjustTokenPrivileges(
        IntPtr tokenHandle,
        bool disableAllPrivileges,
        ref TOKEN_PRIVILEGES newState,
        UInt32 bufferLength,
        IntPtr previousState,
        IntPtr returnLength);

    public static void Set(string privilegeName, bool enabled)
    {
        IntPtr tokenHandle;
        if (!OpenProcessToken(GetCurrentProcess(), TOKEN_QUERY | TOKEN_ADJUST_PRIVILEGES, out tokenHandle))
            throw new Win32Exception(Marshal.GetLastWin32Error());

        try
        {
            LUID luid;
            if (!LookupPrivilegeValue(null, privilegeName, out luid))
                throw new Win32Exception(Marshal.GetLastWin32Error());

            TOKEN_PRIVILEGES state = new TOKEN_PRIVILEGES();
            state.PrivilegeCount = 1;
            state.Luid = luid;
            state.Attributes = enabled ? SE_PRIVILEGE_ENABLED : 0;

            bool adjusted = AdjustTokenPrivileges(tokenHandle, false, ref state, 0, IntPtr.Zero, IntPtr.Zero);
            int error = Marshal.GetLastWin32Error();
            if (!adjusted || error != 0)
                throw new Win32Exception(error);
        }
        finally
        {
            CloseHandle(tokenHandle);
        }
    }
}
'@

[TokenPrivilege]::Set('SeTakeOwnershipPrivilege', $true)
whoami /priv | Select-String 'SeTakeOwnershipPrivilege'
```

확인할 출력:

- `SeTakeOwnershipPrivilege ... Enabled`가 보여야 이 PowerShell process와 이후 생성하는 child command에서 owner 변경을 시도한다.
- Win32 error 1300이면 현재 token에 privilege가 없다. 사용자 권한 할당을 바꾼 직후라면 새 로그온 token을 만들고 다시 확인하며, 그렇지 않으면 다른 기법을 선택하고 owner 변경을 실행하지 않는다.
- access denied나 다른 Win32 error면 현재 process token handle 접근, 실행 계정과 token 제한을 먼저 확인한다.
- 처음부터 `Enabled`였다면 `$PRIVILEGE_WAS_DISABLED = $false`로 기록하고 위 활성화 코드는 실행하지 않는다.

파일 소유권을 가져오고 현재 사용자에게 필요한 범위만 부여한 뒤 내용을 확인한다. owner 변경과 실제 read ACE가 별도 단계인 이유는 [[Windows 파일 소유권과 ACL]]의 access check 경계를 따른다.

```powershell
takeown.exe /f '<TARGET_FILE>'
icacls.exe '<TARGET_FILE>' /grant '<CURRENT_USER>:R'
Get-Content -LiteralPath '<TARGET_FILE>'
```

확인할 출력:

- `takeown`의 `now owned by user`와 `icacls`의 `Successfully processed`.
- `type` 또는 `Get-Content`에서 실제 파일을 읽을 수 있는지 확인한다. 소유권을 얻은 것만으로 자동으로 read access가 생기지 않을 수 있다.
- `Access is denied`가 계속되면 privilege 활성 상태, 대상 파일이 아닌 상위 경로 ACL, 사용자·도메인 표기와 명시적 deny ACE를 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| owner 변경과 읽기 성공 | 보호 파일 접근 성공 | 파일 내용과 자격 증명 후보 확보 | [[Windows 파일 자격증명 검색]] 또는 [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| owner 변경 후에도 읽기 실패 | 소유권과 DACL 권한이 별개 | 파일 접근 미완료 | 최소 ACL 부여 조건과 deny ACE를 검토 |
| owner·ACL 변경이 업무 영향 우려 | 파괴적 변경 가능성 | 변경 중단 또는 증거 수준 확인 | 복구 계획을 재확인 |

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| `<TARGET_FILE>` owner와 DACL | 애플리케이션·원래 사용자 접근이 바뀔 수 있음 | 변경 전 기록과 `Get-Acl`, `icacls` 비교 | 기록한 원래 owner와 ACL을 복원하고 원래 접근을 검증 |

현재 owner를 유지한 상태에서 저장한 DACL을 먼저 적용하고, 마지막에 원래 owner를 복원한다. `/restore`의 첫 인수는 ACL backup에 기록된 상대 경로의 기준이 되는 대상 파일의 부모 디렉터리다.

```cmd
icacls "<TARGET_PARENT_DIRECTORY>" /restore "<ACL_BACKUP_FILE>"
icacls "<TARGET_FILE>" /setowner "<ORIGINAL_OWNER>"
icacls "<TARGET_FILE>"
```

```powershell
Get-Acl '<TARGET_FILE>' | Format-List Owner,Access
```

owner, ACE와 상속 상태가 작업 전 기록과 일치하고 원래 사용 주체의 접근이 정상인지 확인해야 복구 완료다. `/restore` 또는 `/setowner`가 실패하면 backup을 삭제하지 말고 현재 owner·DACL과 실패 출력을 보존한다. 복구 확인 뒤, 작업 전 존재하지 않았고 이번 절차에서 만든 ACL backup만 제거한다.

```powershell
Remove-Item -LiteralPath '<ACL_BACKUP_FILE>' -Force
Test-Path -LiteralPath '<ACL_BACKUP_FILE>'
```

마지막 출력이 `False`여야 backup 정리까지 끝난 것이다. 이름이 비슷한 ACL 파일이나 대상 디렉터리 전체를 삭제하지 않는다.

처음 `Disabled`였던 privilege를 이 PowerShell process에서 활성화했다면 owner·DACL 복구가 끝난 뒤 같은 process에서 다시 비활성화한다. 처음부터 `Enabled`였던 상태는 변경하지 않는다.

```powershell
if ($PRIVILEGE_WAS_DISABLED) {
    [TokenPrivilege]::Set('SeTakeOwnershipPrivilege', $false)
    whoami /priv | Select-String 'SeTakeOwnershipPrivilege'
}
```

마지막 출력이 `Disabled`여야 process token 상태 복구까지 끝난 것이다. ACL 복구 전에 privilege를 끄지 않는다. 비활성화가 실패하면 별도 PowerShell에서 시작 시각·path가 기록과 같은 process만 종료하고 PID 부재를 확인한다.

```powershell
Get-Process -Id <PRIVILEGE_PROCESS_PID> | Select-Object Id,StartTime,Path
Stop-Process -Id <PRIVILEGE_PROCESS_PID>
Get-Process -Id <PRIVILEGE_PROCESS_PID> -ErrorAction SilentlyContinue
```

process 종료는 token 변경을 없애지만 owner·DACL 변경을 복원하지 않으므로 파일 복구와 별도로 판정한다.

## 관련 공통 원리

- [[Windows 파일 소유권과 ACL]]


## 참고 링크
- [Microsoft: SeTakeOwnershipPrivilege](https://learn.microsoft.com/windows/security/threat-protection/security-policy-settings/take-ownership-of-files-or-other-objects), [Microsoft: AdjustTokenPrivileges](https://learn.microsoft.com/windows/win32/api/securitybaseapi/nf-securitybaseapi-adjusttokenprivileges), [Microsoft: OpenProcessToken](https://learn.microsoft.com/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocesstoken), [Microsoft: LookupPrivilegeValue](https://learn.microsoft.com/windows/win32/api/winbase/nf-winbase-lookupprivilegevaluew), [Microsoft: takeown](https://learn.microsoft.com/windows-server/administration/windows-commands/takeown), [Microsoft: icacls](https://learn.microsoft.com/windows-server/administration/windows-commands/icacls)
## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
