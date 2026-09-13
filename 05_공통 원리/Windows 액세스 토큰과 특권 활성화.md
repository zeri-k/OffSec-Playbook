# Windows 액세스 토큰과 특권 활성화

## 핵심 개념

Windows에서 계정, 그룹 멤버십과 현재 프로세스의 권한은 같은 값이 아닙니다. 사용자가 로그온하면 LSA가 사용자 SID, 활성화된 그룹 SID, 무결성 수준과 계정·그룹에 할당된 privilege를 포함한 액세스 토큰을 만듭니다. 이후 프로세스는 기본적으로 자신의 primary token을 사용하고, 서버 프로세스의 특정 thread는 클라이언트의 impersonation token을 일시적으로 사용할 수 있습니다.

| 구성 요소 | 역할 | 실전에서 구분할 점 |
|---|---|---|
| 보안 주체와 그룹 SID | 사용자·서비스 계정과 그룹 구성원 자격을 식별 | Administrators 멤버십이 현재 프로세스의 elevated token을 보장하지 않음 |
| Primary token | 프로세스의 기본 보안 컨텍스트 | 새 프로세스를 어떤 사용자 권한으로 만들 수 있는지와 연결됨 |
| Impersonation token | thread가 클라이언트 컨텍스트로 객체에 접근할 때 사용 | token을 얻었다는 사실과 새 primary token으로 프로세스를 만든 결과는 별도임 |
| Privilege | ACL 검사만으로 표현하기 어려운 시스템 작업 권한 | token에 없음, Disabled, Enabled를 구분해야 함 |
| 무결성 수준·제한 SID | UAC·sandbox가 token의 사용 범위를 제한 | 같은 사용자 SID라도 중간·높은 무결성 token의 결과가 다를 수 있음 |
| 객체 security descriptor | owner, DACL, SACL로 파일·레지스트리·프로세스 등 접근을 제어 | token의 SID·privilege와 객체 ACL을 함께 평가해야 실제 접근이 정해짐 |

## 동작 원리

1. 계정 인증이 성공하면 LSA가 그 시점의 사용자·그룹 SID와 privilege 정보를 받아 로그온 세션과 primary token을 만듭니다. 로컬·도메인 그룹과 로컬 보안 정책의 사용자 권한 할당은 token 생성 시 반영됩니다.
2. 그룹의 `member` 속성이나 로컬 그룹 목록은 계정·디렉터리 상태이고, 이미 만들어진 access token은 별도 보안 객체입니다. 멤버십을 추가·제거해도 실행 중인 프로세스 token을 거꾸로 다시 쓰지 않으므로 기존 세션은 변경 전 SID와 privilege를 계속 사용할 수 있습니다. 새 대화형·네트워크 로그온 또는 새 인증 컨텍스트가 새 token을 만들 때 변경된 멤버십이 반영되는지 다시 확인해야 합니다.
3. 프로세스와 thread가 파일·레지스트리·서비스 같은 객체를 열면 Windows는 요청한 access mask를 token의 SID·제한 정보와 객체 DACL에 대조합니다. 따라서 그룹 이름이 보이더라도 deny ACE, UAC 제한 token 또는 부족한 무결성 수준 때문에 접근이 거부될 수 있습니다.
4. 시스템 privilege는 token에 존재하면서 Enabled 상태일 때 privilege-aware API가 사용할 수 있습니다. `AdjustTokenPrivileges`는 token에 이미 있는 privilege를 활성화하거나 비활성화할 수 있을 뿐, 없는 privilege를 새로 부여하지 못합니다.
5. 도구가 privilege를 활성화해도 그 다음 작업이 자동으로 성공하지는 않습니다. 대상 객체·서비스가 존재해야 하고, 도구가 올바른 API와 token 유형을 사용하며, 필요한 추가 권한과 운영체제 조건을 충족해야 합니다.
6. impersonation이 성공한 thread와 그 token으로 생성한 child process는 서로 다른 상태입니다. 서버가 클라이언트를 가장할 수 있어도 token duplicate·primary token 생성·프로세스 생성 단계에서 별도 실패가 발생할 수 있습니다.

대표 privilege는 같은 “Enabled” 상태라도 작동 지점이 다릅니다.

| Privilege | 권한이 적용되는 지점 | 그 결과만으로 확정할 수 없는 것 |
|---|---|---|
| `SeTakeOwnershipPrivilege` | 객체 owner를 현재 주체 또는 허용된 SID로 바꾸는 단계 | 파일 내용 읽기·수정 권한. owner 변경 뒤 DACL 부여가 별도로 필요할 수 있음 |
| `SeBackupPrivilege` | backup semantics를 요청한 파일·레지스트리 읽기 단계 | 일반 `copy` 명령의 ACL 우회. 사용하는 도구가 backup API·flag를 요청해야 함 |
| `SeImpersonatePrivilege` | 인증된 클라이언트의 보안 컨텍스트를 서버 thread가 가장하는 단계 | SYSTEM token 확보와 child process 생성. 가장할 고권한 연결과 후속 token 작업이 필요함 |
| `SeAssignPrimaryTokenPrivilege` | 새 process에 primary token을 지정하는 단계 | 고권한 token의 획득·사용 가능성이나 process 생성 성공. `SeImpersonatePrivilege`와 도구 지원 조건이 같지 않음 |
| `SeDebugPrivilege` | 현재 process가 원래 열 수 없는 다른 process의 handle을 요청하는 단계 | 모든 보호 process 접근과 SYSTEM 실행. privilege 활성화, 요청 access right, PPL 같은 대상 보호와 후속 API 성공을 별도로 확인해야 함 |
| `SeLoadDriverPrivilege` | kernel-mode driver를 load·unload하도록 운영체제에 요청하는 단계 | 임의 driver 허용, kernel exploit 또는 SYSTEM process 생성. signature·architecture·Code Integrity·blocklist와 driver별 취약 조건이 별도로 필요함 |

## 실전에서의 해석

`whoami /groups`는 현재 token에 들어온 SID와 속성을, `whoami /priv`는 privilege의 존재와 현재 상태를 보여 줍니다. 출력에 privilege 이름이 없으면 현재 token에는 없으며, `Disabled`이면 존재하지만 현재 명령이 곧바로 사용할 수 있다는 뜻은 아닙니다. 해당 privilege를 요청·활성화하도록 구현된 도구가 성공했는지 그 도구의 출력으로 다시 확인해야 합니다.

그룹 변경 명령의 성공과 새 token 생성도 분리합니다. [[로컬 관리자 그룹 구성원 추가]]나 [[AD 그룹 구성원 추가로 권한 확대]]에서 멤버십이 보이면 디렉터리 상태 변경만 확인된 것입니다. 기존 로그온에서 `shutdown /l`을 실행하고 다시 로그인한 뒤 `whoami /all`과 logon ID를 확인하고, 실제 파일·서비스·DRSUAPI처럼 목표 객체에서 필요한 access가 허용되는지 다시 검증합니다. 반대로 멤버십을 원복해도 변경 중 만들어진 token과 session이 자동 폐기되지는 않으므로 다시 `shutdown /l` 후 로그인해 확대된 token이 폐기됐는지 확인합니다.

이 구분 때문에 [[SeTakeOwnershipPrivilege로 보호 파일 ACL 변경]]에서는 privilege 확인, owner 변경, DACL 변경과 파일 접근을 각각 검증합니다. [[SeBackupPrivilege로 보호된 파일과 hive 복사]]에서는 일반 파일 읽기와 backup semantics를 사용하는 복사를 구분하고, [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]에서는 `Enabled` 출력과 SYSTEM child process 생성을 별도 상태로 판단합니다. [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]도 SYSTEM token 표시, `CreateProcessAsUser` 성공과 자식 Identity를 나눠 판정합니다. `SeDebugPrivilege`도 process handle을 여는 데 영향을 줄 뿐입니다. [[LSASS 메모리 덤프]]에서는 dump 생성과 credential 자료 추출을, [[SeDebugPrivilege로 SYSTEM 자식 프로세스 생성]]에서는 SYSTEM parent handle·process 생성·자식 Identity를 각각 확인합니다. [[SeLoadDriverPrivilege로 취약 드라이버 권한 상승]]도 privilege 활성화, driver load, driver별 kernel exploit와 SYSTEM child process를 차례로 확인하며, Code Integrity가 load를 차단하면 privilege 보유만으로 우회할 수 없습니다.

결과는 다음 경계를 넘을 때마다 다시 확정합니다.

- 그룹 멤버십 확인 → 현재 token에 SID가 어떤 속성으로 들어왔는지 확인
- 그룹 멤버십 변경 → 기존 token과 새 로그온·인증 컨텍스트를 구분
- privilege 보유 → Disabled/Enabled와 도구의 활성화 성공 확인
- privilege 사용 성공 → owner 변경·backup read·impersonation처럼 실제 작업 결과 확인
- impersonation token 확보 → 새 프로세스의 `whoami`와 무결성 수준 확인
- 고권한 identity 확인 → 대상 파일·서비스에서 허용되는 구체적 행동 확인

## 관련 기법

- [[SeTakeOwnershipPrivilege로 보호 파일 ACL 변경]]
- [[SeBackupPrivilege로 보호된 파일과 hive 복사]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]
- [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]
- [[SeDebugPrivilege로 SYSTEM 자식 프로세스 생성]]
- [[SeLoadDriverPrivilege로 취약 드라이버 권한 상승]]
- [[LSASS 메모리 덤프]]
- [[로컬 관리자 그룹 구성원 추가]]
- [[AD 그룹 구성원 추가로 권한 확대]]

## 참고 링크

- [Microsoft: Access Tokens](https://learn.microsoft.com/windows/win32/secauthz/access-tokens)
- [Microsoft: Security Principals와 token 생성](https://learn.microsoft.com/windows-server/identity/ad-ds/manage/understand-security-principals)
- [Microsoft: whoami](https://learn.microsoft.com/windows-server/administration/windows-commands/whoami)
- [Microsoft: How User Account Control Works](https://learn.microsoft.com/windows-server/security/user-account-control/how-user-account-control-works)
- [Microsoft: AdjustTokenPrivileges](https://learn.microsoft.com/windows/win32/api/securitybaseapi/nf-securitybaseapi-adjusttokenprivileges)
- [Microsoft: Privilege Constants](https://learn.microsoft.com/windows/win32/secauthz/privilege-constants)
- [Microsoft: Debug Privilege](https://learn.microsoft.com/windows-hardware/drivers/debugger/debug-privilege)
- [Microsoft: UpdateProcThreadAttribute](https://learn.microsoft.com/windows/win32/api/processthreadsapi/nf-processthreadsapi-updateprocthreadattribute)
