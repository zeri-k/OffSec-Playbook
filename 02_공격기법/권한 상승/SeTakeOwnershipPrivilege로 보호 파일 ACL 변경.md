---
tags:
  - 환경/windows
문서역할: 수동절차
시작조건: ["대상 Windows 호스트에서 명령 실행", "현재 token에 SeTakeOwnershipPrivilege가 할당됨"]
필요권한: ["SeTakeOwnershipPrivilege", "대상 파일의 ACL 변경에 필요한 권한"]
필요조건: ["대상 파일 경로와 원래 owner·ACL 기록"]
결과: ["대상 파일의 소유권과 읽기 권한", "파일에서 확인한 정보·자격 증명 후보"]
---

# SeTakeOwnershipPrivilege로 보호 파일 ACL 변경

## 한 줄 판단

현재 token에 `SeTakeOwnershipPrivilege`가 있고 변경할 대상 파일이 있으면, 소유권을 현재 사용자로 바꾼 뒤 필요한 최소 ACL을 부여하여 파일을 읽고 원래 보안 설명자로 복구한다.

## 사용할 때

- `whoami /priv`에서 `SeTakeOwnershipPrivilege`가 확인되고, 파일은 나열할 수 있지만 내용 읽기가 거부될 때.
- 파일·폴더·레지스트리 같은 securable object의 owner·ACL 변경이 실제 대상의 동작에 영향을 줄 수 있을 때.
- 이미 읽기 가능한 다른 정보 수집 경로가 없고, 파일 경로·원래 owner·ACL을 기록할 수 있을 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 token 권한 | `SeTakeOwnershipPrivilege`가 존재하고 활성화 가능 | `whoami /priv` | 권한이 없으면 일반 ACL 또는 다른 수집 기법 선택 |
| 대상 | 파일 경로·현재 owner·ACL과 업무 영향 파악 | `Get-Acl`, `icacls` | 민감 파일·실행 중 설정 파일이면 변경 영향을 먼저 확인 |
| 복구 정보 | 변경 전 owner와 ACL이 기록됨 | `Get-Acl <FILE> | Format-List` | 기록 없이 소유권·ACL 변경을 시작하지 않음 |
| 실행 위치 | 파일이 존재하는 대상 Windows 호스트 | `hostname`, `Test-Path` | 대상 세션·경로를 재확인 |

## 실행

먼저 privilege와 원래 보안 설명자를 기록한다. privilege가 disabled라면 현재 환경에서 허용된 token privilege 활성화 방법을 사용한 뒤 다시 확인한다.

```powershell
whoami /priv
Get-Acl '<TARGET_FILE>' | Format-List Owner,Access
icacls '<TARGET_FILE>'
```

파일 소유권을 가져오고 현재 사용자에게 필요한 범위만 부여한 뒤 내용을 확인한다.

```cmd
takeown /f "<TARGET_FILE>"
icacls "<TARGET_FILE>" /grant "<CURRENT_USER>:R"
type "<TARGET_FILE>"
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

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
