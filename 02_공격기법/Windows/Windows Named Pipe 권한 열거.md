---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 사용자 셸 또는 세션 확보", "현재 process token 확인 가능"]
필요권한: ["현재 Windows token으로 Named Pipe 목록과 대상 pipe security descriptor를 조회할 권한"]
필요조건: ["PowerShell 또는 Sysinternals PipeList", "대상 architecture에서 실행 가능한 Sysinternals AccessChk"]
결과: ["현재 token이 읽거나 쓸 수 있는 Named Pipe 후보", "pipe server와 protocol을 추가 확인할 대상"]
문서역할: 수동절차
---

# Windows Named Pipe 권한 열거

## 한 줄 판단

대상 Windows 호스트의 현재 사용자 셸에서 Named Pipe 목록과 DACL을 조회하여 현재 token에 write가 허용된 pipe를 찾고, server process·권한·protocol을 확인할 후속 후보로 남긴다.

Named Pipe는 server process가 만든 이름 있는 IPC 객체이며 client가 연결할 때 요청한 access right를 client token과 pipe security descriptor의 DACL에 대조한다. 따라서 pipe 이름이나 `RW` 표시는 단서일 뿐이며, token·DACL 관계는 [[Windows 액세스 토큰과 특권 활성화]]와 함께 해석한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 조사할 Named Pipe가 존재하는 대상 Windows 호스트의 로컬 셸 | `hostname`, `whoami` | 다른 호스트의 pipe namespace와 혼동하지 않음 |
| 현재 계정과 token | 사용자 SID·활성 그룹 SID·무결성 수준을 식별 가능 | `whoami /all` | 계정 이름만으로 ACL 일치를 판단하지 않음 |
| 목록 조회 경로 | PowerShell `\\.\pipe\` 또는 대상 OS를 지원하는 PipeList 실행 가능 | 각 명령의 실제 목록·오류 확인 | 명령 부재, AppLocker·AV 차단과 조회 권한 부족을 구분 |
| DACL 조회 도구 | Microsoft Sysinternals AccessChk의 exact 실행 경로와 version 확인 | `"<ACCESSCHK_PATH>" -accepteula -nobanner -?` | 다른 version이면 해당 help의 `-w`·`-v`와 `\pipe\` 문법 확인 |
| 공격 대상 조건 | 관심 pipe의 이름과 write 권한 후보가 식별됨 | PipeList 또는 PowerShell 목록과 AccessChk 결과 대조 | 이름만 있고 DACL을 읽지 못하면 미확인으로 유지 |

## 실행

`<ACCESSCHK_PATH>`와 `<PIPELIST_PATH>`는 대상 Windows 호스트의 도구 절대 경로이며, `<PIPE_NAME>`은 앞 단계 목록에서 찾은 pipe 이름이다. pipe 이름·`RW` 표시는 입력 단서일 뿐 실제 write 권한은 현재 token과 DACL 결과로만 판단한다.

### 1. 현재 token과 pipe 목록 확인

대상 Windows PowerShell에서 현재 token을 확인하고 pipe 이름을 나열한다.

```powershell
whoami /all
Get-ChildItem \\.\pipe\ | Select-Object Name
```

Sysinternals PipeList를 사용할 수 있으면 instance 수도 함께 확인한다.

```cmd
"<PIPELIST_PATH>" /accepteula
```

확인할 출력:

- 현재 사용자·그룹 SID·무결성 수준과 pipe 이름.
- PipeList의 `Instances`·`Max Instances`는 현재 instance 상태이며 server process의 identity나 client write 권한을 보여 주지는 않는다.
- 빈 목록이나 접근 오류가 나오면 pipe 부재로 단정하지 말고 현재 셸·도구 차단과 namespace 조회 권한을 확인한다.

### 2. write 후보와 대상 pipe DACL 확인

AccessChk v6.15 기준으로 `\pipe\` prefix는 Named Pipe 경로를 뜻하고, `-w`는 write access가 있는 항목만, `-v`는 구체적인 granted access를 표시한다.

```cmd
"<ACCESSCHK_PATH>" -accepteula -nobanner -w -v "\pipe\*"
"<ACCESSCHK_PATH>" -accepteula -nobanner -v "<DOMAIN_OR_HOST>\<USER>" "\pipe\<PIPE_NAME>"
```

확인할 출력:

- `<PIPE_NAME>`에 대해 현재 사용자 또는 현재 token의 활성 그룹 SID에 허용된 `FILE_WRITE_DATA`, `FILE_APPEND_DATA`·`FILE_CREATE_PIPE_INSTANCE`, `GENERIC_WRITE` 또는 `FILE_ALL_ACCESS` 같은 권리.
- `Everyone`·`Authenticated Users` ACE가 보이면 현재 token이 실제 그 SID를 활성 상태로 포함하는지 확인한다. deny ACE·무결성 수준과 실제 client open 결과가 다를 수 있으므로 목록만으로 write 성공을 확정하지 않는다.
- `Access is denied`는 security descriptor 조회가 불완전한 상태다. pipe가 안전하거나 존재하지 않는다는 결과로 바꾸지 않는다.

### 3. pipe server와 protocol 범위 확인

pipe DACL이 넓어도 server가 client 데이터를 어떻게 검증하고 어떤 계정으로 동작하는지가 확인되어야 권한 상승 후보다. 다음 항목을 해당 제품·service 문서와 현재 process 상태에서 대조한다.

- pipe를 생성한 service·process와 실행 계정
- 설치된 제품과 exact version
- client가 보낼 수 있는 message 형식과 server가 수행하는 동작
- 현재 token으로 요청한 read·write access가 실제 허용되는지
- 알려진 취약점이라면 대상 version·구성과 crash·service 영향

server process를 신뢰성 있게 식별하지 못하거나 protocol을 확인하지 못하면 `write 가능한 Named Pipe 후보`로만 남긴다. 임의 데이터를 쓰거나 공개 exploit를 실행하는 단계는 이 조회 문서의 범위가 아니다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| pipe 이름만 나열됨 | IPC 객체 존재만 확인 | Named Pipe 목록 | 관심 제품·service와 이름을 대응하고 DACL 조회 |
| 현재 사용자·활성 그룹에 read만 허용됨 | 현재 요청 범위에서 읽기 권한만 확인 | write 권한 미확인 | write 후보에서 제외하고 다른 pipe·서비스 제어 지점 확인 |
| 현재 사용자·활성 그룹에 write 계열 권리가 표시됨 | DACL상 write 후보 | Named Pipe 권한 단서 | server process·실행 계정·protocol과 실제 client open을 확인 |
| write 후보와 고권한 server의 취약한 message 처리가 모두 확인됨 | 제품·version별 공격 전제 후보 | 별도 기법 검토 상태 | 해당 취약점의 공식 원출처·영향·복구 절차가 있는 기법으로 분리 |
| AccessChk 차단 또는 DACL 접근 거부 | 해당 pipe의 권한 미확정 | 부분 열거 | [[Windows 방어 제어 상태 확인]]과 현재 token·도구 version 확인 |

## 확인할 출력과 권한

- pipe 목록, DACL상 write 권리, 실제 client handle 획득, server의 고권한 동작은 서로 다른 단계다.
- write 권한이 있어도 server가 입력을 무시하거나 안전하게 검증하면 권한 상승으로 이어지지 않는다.
- 이 절차는 pipe 목록과 security descriptor를 조회할 뿐 pipe를 만들거나 ACL·service 상태를 변경하지 않는다.

## 관련 공격기법

- [[Windows 권한 상승 열거]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]
- [[Windows 방어 제어 상태 확인]]

## 관련 도구

- [[powershell]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 참고 링크

- [Microsoft: Named Pipe Security and Access Rights](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights)
- [Microsoft Sysinternals: AccessChk](https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk)
- [Microsoft Sysinternals: PipeList](https://learn.microsoft.com/en-us/sysinternals/downloads/pipelist)
