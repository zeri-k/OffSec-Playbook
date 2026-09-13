---
tags:
  - 환경/windows
시작조건: ["Windows 명령 실행", "SeImpersonatePrivilege Enabled"]
필요권한: ["현재 token의 SeImpersonatePrivilege"]
필요조건: ["대상 Windows build와 arch에 맞는 PrintSpoofer 실행 파일", "기존 파일과 충돌하지 않는 쓰기·실행 경로"]
결과: ["SYSTEM 명령 실행", "고권한 세션"]
---

# PrintSpoofer로 SeImpersonatePrivilege 권한 상승

## 한 줄 판단

Windows 서비스 계정 셸에서 `SeImpersonatePrivilege`가 활성화되어 있고 실행 파일을 둘 수 있다면, 대상 호스트에서 PrintSpoofer로 SYSTEM token을 가장해 `nt authority\system` 명령 실행을 확인한다.

## 사용할 때

- SQL Server·IIS 같은 서비스 계정으로 Windows 명령을 실행할 수 있을 때.
- `whoami /priv`에서 `SeImpersonatePrivilege`가 `Enabled`로 확인될 때.
- 먼저 짧은 `whoami` 명령으로 권한 상승 성공 여부를 확인한 뒤 후속 세션을 열 때.

## 전제 조건

현재 token에 privilege가 할당·활성화된 상태와 impersonation token·SYSTEM child process 생성은 [[Windows 액세스 토큰과 특권 활성화]]처럼 서로 다른 단계다. 아래 조건을 모두 확인하며 `Enabled`만으로 성공을 단정하지 않는다.

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 명령 실행 위치 | 대상 Windows 호스트의 셸 또는 명령 실행 채널 | `hostname & whoami` | 목표 호스트와 현재 실행 계정을 먼저 확정 |
| 현재 token 특권 | `SeImpersonatePrivilege Enabled` | `whoami /priv` | Disabled·Absent이면 다른 권한 상승 기법 선택 |
| 실행 파일 | 대상 arch와 맞는 PrintSpoofer | 파일 크기와 SHA-256 비교 | [[Certutil로 Windows HTTP 파일 반입]] 등으로 다시 반입 |
| 저장·실행 경로 | 현재 계정이 쓰고 실행할 수 있고 기존 파일과 충돌하지 않는 `<PRINTSPOOFER_PATH>` | `Test-Path -LiteralPath '<PRINTSPOOFER_PATH>'`가 반입 전에 `False`인지 확인 | 고유 경로를 다시 정하고 ACL·AppLocker·AV 차단 단계 확인 |

## 실행

### 일반 Windows 셸에서 실행

```cmd
whoami /priv
<PRINTSPOOFER_PATH> -c "cmd /c whoami"
```

### impacket-mssqlclient의 xp_cmdshell에서 실행

```text
SQL> xp_cmdshell whoami /priv
SQL> xp_cmdshell <PRINTSPOOFER_PATH> -c "cmd /c whoami"
```

확인할 출력:

- `SeImpersonatePrivilege`가 `Enabled`여야 한다.
- PrintSpoofer의 `Found privilege: SeImpersonatePrivilege`와 `CreateProcessAsUser() OK`를 확인한다.
- 실행한 명령의 출력이 `nt authority\system`이어야 SYSTEM 권한 상승이 완료된 것이다.
- `CreateProcessAsUser()` 실패이면 특권 표시만 보고 성공으로 판단하지 말고 Windows build·token·서비스 격리와 보안 제품 차단을 확인한다.

SYSTEM 컨텍스트로 후속 명령을 실행해야 할 때는 검증한 같은 `-c` 위치에 명령을 넣는다.

```cmd
<PRINTSPOOFER_PATH> -c "cmd /c <SYSTEM_COMMAND>"
```

[[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]과 RoguePotato는 별도 privilege·버전·CLSID·RPC 통신 조건을 가지는 경로다. 이 문서는 PrintSpoofer의 named pipe impersonation 경로만 지원하므로, PrintSpoofer가 실패했다는 이유만으로 다른 Potato 계열 도구의 가능성이나 성공을 판정하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `CreateProcessAsUser() OK`와 `nt authority\system` 출력 | SYSTEM token 가장과 명령 실행 성공 | SYSTEM 명령 실행 | [[고권한 세션 확보 후 후속 판단]] |
| 특권은 `Enabled`지만 프로세스 생성 실패 | 현재 환경에서 PrintSpoofer 실행 실패 | 일반 서비스 계정 명령 실행 유지 | Windows build·arch, token 유형과 실행 차단 원인을 확인 |
| 파일 실행 자체가 차단됨 | 반입 파일·경로·정책 문제 | 파일 반입만 완료 | 파일 hash, MOTW·AppLocker·AV와 쓰기·실행 ACL 확인 |

## 확인할 출력과 권한

- `SeImpersonatePrivilege Enabled`는 전제 조건이다.
- `CreateProcessAsUser() OK`와 자식 명령의 `nt authority\system` 출력이 함께 있어야 결과 권한을 확정한다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상에 반입한 PrintSpoofer 파일 | 디스크에 도구 파일이 남음 | 반입 전 부재를 확인한 exact `<PRINTSPOOFER_PATH>`·hash·수정 시각 기록 | 후속 작업 완료 후 아래 명령으로 이번 작업에서 반입한 exact 파일만 제거 |
| SYSTEM으로 실행한 후속 명령 | 명령 내용에 따라 계정·파일·서비스가 변경될 수 있음 | 각 후속 기법의 변경 전후 상태 확인 | 변경을 수행한 기법의 복구 절차 적용 |

```powershell
Get-Item -LiteralPath '<PRINTSPOOFER_PATH>' | Select-Object FullName,Length,LastWriteTime
Remove-Item -LiteralPath '<PRINTSPOOFER_PATH>' -Force
Test-Path -LiteralPath '<PRINTSPOOFER_PATH>'
```

마지막 출력이 `False`여야 반입 파일 정리를 확인한 것이다. 삭제가 거부되면 정확한 경로, 현재 token의 삭제 권한과 해당 파일을 사용 중인 process를 확인한다. 기존 파일이 있었거나 이번 반입 여부를 입증할 수 없으면 이름만 보고 삭제하지 않는다.

## 관련 공격기법

- [[MSSQL xp_cmdshell 명령 실행]]
- [[Certutil로 Windows HTTP 파일 반입]]
- [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]]

## 관련 도구

- [[PrintSpoofer]]

## 관련 상태 라우터

- [[MSSQL 인증 세션 확보 후 권한과 실행 경로 선택]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [itm4n: PrintSpoofer](https://github.com/itm4n/PrintSpoofer)
