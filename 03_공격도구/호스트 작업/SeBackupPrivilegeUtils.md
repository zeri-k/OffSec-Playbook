---
tags:
  - 환경/windows
  - 기능/파일수집
실행환경: ["대상 Windows PowerShell"]
필요권한: ["SeBackupPrivilege가 현재 token에 할당됨"]
필요조건: ["SeBackupPrivilegeUtils.dll와 SeBackupPrivilegeCmdLets.dll", "대상에서 쓰기 가능한 출력 경로"]
결과: ["backup semantics로 만든 보호 파일 사본"]
---

# SeBackupPrivilegeUtils

## 도구 개요

`SeBackupPrivilegeUtils`와 `SeBackupPrivilegeCmdLets`는 `SeBackupPrivilege`를 활성화하고 backup semantics로 보호 파일을 복사하는 PowerShell 모듈 조합이다. 일반 파일 읽기가 거부되지만 현재 token에 backup 권한이 있는 Windows 세션에서 선택한다.

## 필요한 입력과 실행 환경

- 실행 위치: 파일이 있는 대상 Windows 호스트의 PowerShell 세션.
- 필요한 권한: `whoami /priv`에 `SeBackupPrivilege`가 있어야 하며, 명시적 deny ACE는 별도로 실패할 수 있다.
- 필요한 파일: 대상 환경에 맞는 `SeBackupPrivilegeUtils.dll`, `SeBackupPrivilegeCmdLets.dll`.
- 대상 조건: 원본 파일 경로와 현재 계정이 쓸 수 있는 사본 저장 경로.

## 표준 사용법

```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege
Copy-FileSeBackupPrivilege '<SOURCE_FILE>' '<DESTINATION_FILE>'
```

## 대표 예시

### privilege 상태 확인과 활성화

```powershell
Get-SeBackupPrivilege
Set-SeBackupPrivilege
Get-SeBackupPrivilege
```

확인할 출력:

- 마지막 출력이 `SeBackupPrivilege is enabled`여야 한다.
- privilege 활성화는 대상 파일을 읽거나 복사한 성공을 뜻하지 않는다.

### 보호 파일 사본 생성

```powershell
Copy-FileSeBackupPrivilege '<PROTECTED_FILE>' '<WRITABLE_PATH>\copied-file'
Get-FileHash '<WRITABLE_PATH>\copied-file'
```

확인할 출력:

- `Copied <SIZE> bytes`와 생성된 사본의 크기·hash.
- 이 출력만으로 원본의 최신성, 사본 내용의 유용성이나 고권한 세션을 확정할 수는 없다.
- access denied이면 privilege 활성 상태, 원본의 명시적 deny ACE, 출력 경로 ACL을 순서대로 확인한다.

## 주요 명령

| 명령 | 의미 | 사용하는 상황 |
|---|---|---|
| `Get-SeBackupPrivilege` | 현재 세션의 backup privilege 활성 상태 확인 | 복사 전 조건 확인 |
| `Set-SeBackupPrivilege` | 현재 token의 backup privilege 활성화 시도 | privilege가 disabled일 때 |
| `Copy-FileSeBackupPrivilege` | backup semantics로 원본 파일을 사본으로 생성 | 일반 읽기가 거부된 파일 수집 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `SeBackupPrivilege is enabled` | 현재 세션에서 privilege 활성화 | 보호 파일 복사 시도 |
| `Copied <SIZE> bytes` | 사본 생성 성공 | 크기·hash와 내용 확인 |
| access denied | token·ACE·원본 또는 출력 경로 조건 실패 | `whoami /priv`, 원본 DACL, 대상 경로 ACL 확인 |

## 관련 공격기법

- [[SeBackupPrivilege로 보호된 파일과 hive 복사]]
