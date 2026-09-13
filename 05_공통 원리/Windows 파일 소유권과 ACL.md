# Windows 파일 소유권과 ACL

## 핵심 개념

Windows 파일의 security descriptor에는 owner와 DACL이 별도 필드로 들어 있습니다. DACL은 SID별 허용·거부 ACE와 상속 정보를 가지며, 프로세스가 요청한 읽기·쓰기·삭제 같은 access mask를 현재 access token의 SID와 대조할 때 사용됩니다. owner는 파일 내용에 대한 읽기 권한 그 자체가 아니라 DACL을 변경할 수 있는 주체입니다.

| 구성 요소 | 역할 | 같은 것으로 해석하면 안 되는 상태 |
|---|---|---|
| Owner | security descriptor의 소유 주체이며 DACL 변경 통제권을 가짐 | owner가 됐다는 사실과 파일 내용 읽기·수정 성공 |
| DACL | 객체 접근을 허용·거부하는 ACE 목록 | DACL 조회 가능과 원하는 access mask 허용 |
| ACE | 특정 SID에 권리를 허용하거나 거부하고 상속 범위를 지정 | 그룹 allow 하나와 최종 접근 결과 |
| 액세스 토큰 | 사용자·그룹 SID, 무결성 수준과 privilege를 제공 | 계정의 그룹 구성원 자격과 현재 프로세스의 primary 액세스 토큰 상태 |
| 요청한 access mask | 프로세스가 이번 handle에서 요구한 구체적 권리 | 파일을 열었다는 사실과 모든 읽기·쓰기·삭제 권리 |

## 동작 원리

1. 프로세스가 파일을 열 때 읽기·쓰기·삭제처럼 필요한 access mask를 요청합니다. Windows는 현재 token의 사용자·그룹 SID와 파일 DACL의 ACE를 순서대로 평가해 요청 권리를 허용하거나 거부합니다.
2. 명시적 ACE는 상속 ACE보다 먼저 평가되는 canonical 순서를 사용하고, 같은 그룹에서는 deny가 allow보다 앞섭니다. 여러 그룹의 allow 권리는 합쳐질 수 있지만 먼저 적용되는 deny나 제한된 액세스 토큰 때문에 최종 결과가 달라질 수 있습니다.
3. 파일 owner는 DACL을 바꿀 수 있지만 owner가 되는 순간 데이터 읽기 권리가 자동 추가되지는 않습니다. `SeTakeOwnershipPrivilege`는 먼저 owner를 바꿀 수 있게 하고, 실제 읽기가 필요하면 새 owner가 DACL에 필요한 최소 ACE를 추가하는 별도 단계가 이어집니다.
4. 현재 token에서 `SeBackupPrivilege`가 활성화되고 API·도구가 backup semantics로 읽기 권리를 요청하면 Windows는 파일 ACL과 무관하게 backup용 읽기 접근을 부여할 수 있습니다. 이 우회는 backup에 필요한 읽기 권리에 한정되며 함께 요청한 비읽기 권리는 여전히 DACL 평가를 받습니다. 또한 owner나 DACL을 바꾸는 동작이 아니고, 일반 `copy`가 같은 결과를 낸다는 뜻도 아닙니다. 사본을 만들 출력 경로는 별도 handle이므로 그 디렉터리와 새 파일의 일반 쓰기 권한도 따로 필요합니다.
5. 파일이나 상위 디렉터리의 DACL을 수정하면 상속 설정에 따라 하위 객체와 서비스 계정의 접근도 달라질 수 있습니다. 따라서 기존 owner, 전체 DACL과 상속 상태가 복구 입력이며 새 allow ACE 한 줄만 기억해서는 원래 상태를 재구성할 수 없습니다.

## 실전에서의 해석

| 관찰한 상태 | 확정할 수 있는 것 | 아직 확인할 것 |
|---|---|---|
| `SeTakeOwnershipPrivilege`가 현재 token에 존재함 | privilege-aware 도구가 활성화를 요청할 후보 | 활성화 성공, owner 변경과 대상 파일 접근 |
| owner가 현재 사용자로 변경됨 | 현재 주체가 DACL을 변경할 수 있는 상태 | 필요한 ACE 추가와 실제 파일 읽기 |
| 읽기 ACE 추가 뒤 파일 내용이 반환됨 | 그 시점의 token과 DACL로 해당 파일 읽기 성공 | 다른 파일·쓰기·실행 권한과 기존 DACL 복구 |
| backup semantics를 사용한 복사 성공 | 활성화된 `SeBackupPrivilege`와 해당 도구 경로로 데이터 읽기 성공 | 일반 파일 API의 읽기 권한과 원본 ACL 변경 여부 |
| owner·DACL 복원 뒤 기준 출력과 일치함 | 기록한 security descriptor 범위가 원래 값과 일치 | 서비스·애플리케이션의 실제 접근 정상 여부 |

현재 프로세스의 액세스 토큰과 privilege가 만들어지는 과정은 [[Windows 액세스 토큰과 특권 활성화]]에서 설명합니다. owner·ACL을 실제로 바꾸는 명령과 복구는 [[SeTakeOwnershipPrivilege로 보호 파일 ACL 변경]]에, DACL을 바꾸지 않고 backup semantics로 읽는 절차는 [[SeBackupPrivilege로 보호된 파일과 hive 복사]]에 남깁니다.

## 관련 기법

- [[SeTakeOwnershipPrivilege로 보호 파일 ACL 변경]]
- [[SeBackupPrivilege로 보호된 파일과 hive 복사]]

## 참고 링크

- [Microsoft: Access Control](https://learn.microsoft.com/windows/win32/secauthz/access-control)
- [Microsoft: DACLs and ACEs](https://learn.microsoft.com/windows/win32/secauthz/dacls-and-aces)
- [Microsoft: Order of ACEs in a DACL](https://learn.microsoft.com/windows/win32/secauthz/order-of-aces-in-a-dacl)
- [Microsoft: File Security and Access Rights](https://learn.microsoft.com/windows/win32/fileio/file-security-and-access-rights)
- [Microsoft: Privilege Constants](https://learn.microsoft.com/windows/win32/secauthz/privilege-constants)
- [Microsoft: SeTakeOwnershipPrivilege](https://learn.microsoft.com/windows/security/threat-protection/security-policy-settings/take-ownership-of-files-or-other-objects)
