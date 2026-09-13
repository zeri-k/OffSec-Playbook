---
tags:
  - 환경/windows
  - 기능/열거
  - 기능/권한상승
시작상태: ["Windows 셸 확보", "Windows 원격 세션 확보", "Windows 명령 실행 확보"]
목표: ["현재 권한 확인", "현재 계정 확인", "자격 증명 또는 내부망 단서 확보"]
현재계정: ["Windows 로컬 사용자", "AD 도메인 사용자", "미확인"]
현재 가능한 행위: ["셸에서 명령 실행", "RDP GUI 조작", "WinRM 세션에서 명령 실행", "Web Shell에서 명령 실행", "에이전트 세션에서 명령 실행"]
필요권한: ["현재 Windows 사용자 권한"]
필요정보: ["접근한 호스트", "현재 셸 또는 세션"]
네트워크위치: ["접근한 Windows 호스트"]
---

# Windows 셸 또는 세션 확보 후 컨텍스트 열거

## 상태 라우터 개요

Windows에서 명령 실행이나 세션을 확보하면 로컬·도메인 계정, 실제 권한, 지금 할 수 있는 작업, 저장 자격 증명과 네트워크 위치를 분리해 다음 상태 라우터를 고른다.

## 적용 조건

| 상태 축 | 조건 |
|---|---|
| 대상 플랫폼 | Windows |
| 현재 계정 | 로컬 사용자, 도메인 사용자 또는 미확인 상태 |
| 현재 가능한 행위 | 셸·WinRM·Web Shell·에이전트에서 명령 실행 또는 RDP GUI 조작 |
| 현재 권한 | 일반 사용자, 로컬 관리자, SYSTEM, 도메인 권한을 별도 확정 |
| 보유 정보 | 현재 호스트와 세션 |
| 네트워크 위치 | 접근한 Windows 호스트 내부 |
| 목표 | 사용자·권한·도메인 연결·자격 증명·내부망 위치 확정 |

## 판단 경로

현재 사용자·토큰·호스트·셸 제약을 모르면 아래 첫 표부터 확인한다. 확인된 뒤에는 현재 목표에 맞는 표만 사용한다.

### 먼저 확인

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| 대상 Windows 호스트에서 사용자·그룹·무결성 수준과 상승 단서가 미확인 | [[Windows 권한 상승 열거]] | 현재 권한과 상승 후보 | 이 상태 라우터 재평가 | 명령이 실제 대상 호스트에서 실행되는지, 현재 토큰·UAC·세션 유형과 열거 명령 실행 권한을 확인 |

### 권한과 실행 컨텍스트

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| `whoami /priv`에서 `SeBackupPrivilege`가 현재 token에 할당되고 일반 읽기가 거부된 보호 파일과 쓰기 경로가 확인됨 | [[SeBackupPrivilege로 보호된 파일과 hive 복사]] | backup semantics로 만든 보호 파일 사본 | 자격 증명이면 [[확보한 자격 증명으로 원격 접근 경로 선택]], 그 외에는 이 라우터 | privilege 활성 상태, 원본·출력 경로 ACL과 사본 내용이 실제 자격 증명인지 확인 |
| `SeBackupPrivilege`가 활성화되고 SAM·SECURITY·SYSTEM hive의 저장 경로를 확인함 | [[SeBackupPrivilege로 보호된 파일과 hive 복사]] | 같은 시점의 오프라인 hive 세트 | 이 라우터에서 [[Windows SAM SECURITY SYSTEM 덤프]]로 분석 | 세 파일의 생성·크기·hash와 동일 호스트·시점의 세트인지 확인 |
| DC에서 `SeBackupPrivilege`가 활성화되고 섀도 복사본의 NTDS.dit와 같은 시점의 SYSTEM hive를 확보할 수 있음 | [[SeBackupPrivilege로 보호된 파일과 hive 복사]] | NTDS.dit·SYSTEM 오프라인 분석 입력 | [[impacket-secretsdump]] 결과에서 계정별 hash를 확인한 뒤 [[확보한 자격 증명으로 원격 접근 경로 선택]] | 실제 DC인지, 두 파일의 시점·무결성과 사본만으로 Domain Admin 세션이나 DCSync 권한을 얻은 것은 아님을 확인 |
| `whoami /priv`에서 `SeTakeOwnershipPrivilege`가 현재 token에 할당되고 원래 owner·ACL을 기록할 수 있는 보호 파일이 확인됨 | [[SeTakeOwnershipPrivilege로 보호 파일 ACL 변경]] | 최소 ACL로 읽은 파일 내용 또는 자격 증명 후보 | 자격 증명이면 [[확보한 자격 증명으로 원격 접근 경로 선택]], 그 외에는 이 라우터 | 파일의 업무 영향, privilege 활성 가능 여부와 원래 owner·ACL 복구 가능 여부를 확인 |
| SQL Server·IIS 같은 서비스 계정 셸의 `whoami /priv`에서 `SeImpersonatePrivilege Enabled`가 확인됨 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]] | 검증에 성공하면 SYSTEM 명령 실행 | SYSTEM 확인 시 [[고권한 세션 확보 후 후속 판단]], 실패 시 이 라우터 | `Enabled`는 후보 조건일 뿐 성공이 아니다. 대상 build·arch에 맞는 실행 파일과 무결성, 쓰기·실행 경로, AppLocker·AV를 확인하고 `CreateProcessAsUser() OK`와 자식 명령의 `nt authority\system` 출력을 함께 검증 |
| `whoami /priv`에서 `SeAssignPrimaryTokenPrivilege`가 확인되고 classic JuicyPotato가 동작하는 legacy Windows build·edition임 | [[JuicyPotato로 SeAssignPrimaryTokenPrivilege 권한 상승]] | 검증에 성공하면 SYSTEM 명령 실행 | SYSTEM 확인 시 [[고권한 세션 확보 후 후속 판단]], 실패 시 이 라우터 | `Disabled`를 미보유로 보지 않되 성공으로도 보지 않는다. `SeIncreaseQuotaPrivilege`, exact OS별 CLSID, 빈 로컬 COM 포트와 `CreateProcessAsUser OK`·자식 명령의 `nt authority\system`을 확인한다. Windows 10 1809+·Server 2019+에는 이 classic 경로를 일반 적용하지 않는다. |
| 현재 token에 `SeLoadDriverPrivilege`가 있고 승인된 취약 driver·loader·driver별 exploit가 있음 | [[SeLoadDriverPrivilege로 취약 드라이버 권한 상승]] | 실제 child Identity로 검증한 SYSTEM 명령 실행 또는 driver load만 된 중간 상태 | SYSTEM 확인 시 [[고권한 세션 확보 후 후속 판단]], 실패·차단 시 이 라우터 | `Disabled`를 미보유로 보지 않되 driver load 성공으로도 보지 않는다. signature·architecture, Code Integrity·HVCI·blocklist, actual loaded driver와 child `nt authority\system`을 각각 확인한다. |
| 현재 token에 Event Log Readers·DnsAdmins·Hyper-V Administrators·Print Operators 중 하나가 반영됐지만 그룹별 실제 대상 권한과 목표를 아직 고르지 않음 | 수동 확인: 현재 token SID와 대상 channel·DNS server·VM·`SeLoadDriverPrivilege`를 그룹별로 확인 | 위임 그룹과 실제 대상 권한의 대응 관계 | [[Windows 위임 운영 그룹 확인 후 권한 경로 선택]] | 그룹 디렉터리 상태와 현재 token, 읽기·설정·service control·VM 관리·driver load 권한을 서로 분리 |

### 셸 제약과 세션 기능

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| 대상 Windows 호스트에서 `cmd.exe`, 제한된 PowerShell 또는 기본 LOLBin만 사용할 수 있음 | [[제한된 Windows 셸에서 AD와 호스트 열거]] | 현재 계정·호스트·도메인·네트워크 단서 | 로컬 호스트 목표면 이 라우터, AD 객체·권한 목표면 [[AD Identity 확인 후 도메인 컨텍스트 열거]] | 실행 가능한 기본 명령, PowerShell 언어 모드, 현재 계정의 명령 실행 범위와 대상 호스트 응답을 확인 |
| 대상 Windows Web Shell이나 제한된 셸에서 PowerShell 명령을 실행할 수 있고 Meterpreter 세션 관리 기능이 필요함 | [[Metasploit Web Delivery로 Meterpreter 세션 획득]] | Windows 대상의 Meterpreter 세션 | [[Meterpreter 세션 후속 행동]] | 대상에서 공격 호스트의 HTTP·reverse handler 포트 연결, payload architecture와 PowerShell 실행 오류 확인 |
| 대상 Windows 호스트에서 명령·스크립트·payload가 차단됐거나 Defender·AppLocker·PowerShell 제한 상태를 아직 확인하지 않음 | [[Windows 방어 제어 상태 확인]] | 실행·차단 경계 | 이 상태 라우터에서 실행 가능한 수단으로 재평가 | 차단이 정책·권한·경로·파일 평판 중 무엇인지, 현재 사용자와 대상 호스트의 실제 적용 정책을 확인 |

### 자격 증명과 인증 컨텍스트

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| 현재 Windows 사용자가 프로필·설정·history 경로를 읽을 수 있지만 파일 기반 자격 증명 자료를 아직 열거하지 않음 | [[Windows 파일 자격증명 검색]] | 계정명이 확인된 평문 비밀번호·NT hash·API token·개인키 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] | 파일 읽기 권한·검색 범위·최신성, 값에 연결된 계정과 대상 서비스 포트 도달성을 확인 |
| 현재 Windows 사용자가 Credential Manager와 저장된 로그온 항목을 조회할 수 있지만 저장 자격 증명을 아직 열거하지 않음 | [[Windows 저장 자격증명 수집]] | 저장된 대상·사용자명과 재사용 가능한 인증 자료 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] | 현재 사용자 프로필, 저장 항목 유형, 대상 서비스와 값의 실제 사용 가능 범위를 확인 |
| `cmdkey /list`에서 `Domain:interactive` 저장 자격 증명과 사용자명이 확인됨 | [[저장된 자격 증명으로 runas 프로세스 실행]] | 저장된 계정으로 실행되는 새 Windows 프로세스와 실제 token | 이 상태 라우터에서 새 프로세스의 계정·그룹·무결성 수준 재평가 | 저장 항목 유형, 정확한 사용자명과 `runas` 오류 코드 확인 |
| 현재 계정과 다른 로컬·도메인 계정의 사용자명·평문 비밀번호가 있고 현재 호스트에서 그 계정의 로컬 프로세스가 필요함 | [[확보한 평문 비밀번호로 runas 사용자 프로세스 실행]] | 입력한 계정의 로컬 Windows 프로세스와 실제 token | 이 상태 라우터에서 새 프로세스의 계정·그룹·무결성 수준 재평가 | 계정의 로컬·도메인 표기, 평문 비밀번호와 현재 호스트의 로컬 로그온 권한을 확인 |
| 현재 계정과 다른 AD 계정의 사용자명·평문 비밀번호가 있고 현재 로컬 로그인 계정을 유지한 채 원격 통합 인증을 사용해야 함 | [[확보한 AD 비밀번호로 runas netonly 네트워크 인증 컨텍스트 생성]] | 지정 AD 계정으로 인증된 원격 서비스 접근 | AD 객체·권한 목표면 [[AD Identity 확인 후 도메인 컨텍스트 열거]], 서비스 선택 목표면 [[확보한 자격 증명으로 원격 접근 경로 선택]] | 대상 서비스의 Windows 통합 인증 지원, FQDN·DNS·시간·포트와 지정 계정의 서비스 권한을 확인 |

### 파일 반입과 회수

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| Windows 대상에서 공격 호스트의 SMB 445/TCP에 연결할 수 있고 파일 반입이 필요함 | [[SMB 공유로 Windows 파일 반입]] | Windows 대상의 파일과 무결성 확인 결과 | 이 상태 라우터에서 파일 실행 조건과 현재 권한 재평가 | 통신 방향, Guest 차단, SMB 공유명과 대상 저장 경로 ACL 확인 |
| Windows 대상에서 공격 호스트의 HTTP 포트에 연결할 수 있고 `certutil.exe`를 사용할 수 있음 | [[Certutil로 Windows HTTP 파일 반입]] | Windows 대상의 파일과 무결성 확인 결과 | 이 상태 라우터에서 파일을 사용할 기법 재선택 | HTTP listener 주소·포트, 대상 저장 경로 ACL과 송수신 SHA-256 확인 |
| Windows 대상에서 파일을 읽을 수 있고 공격 호스트의 HTTP 수신 포트에 연결 가능함 | [[Windows HTTP 파일 회수]] | 공격 호스트로 회수한 파일과 무결성 확인 결과 | 이 상태 라우터에서 확보한 파일의 후속 분석 선택 | 대상 파일 ACL, HTTP 요청 수신과 multipart·raw POST 형식 확인 |

### AD와 Kerberos 작업

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| 현재 Windows 세션에서 Kerberos ticket을 확인했지만 사용할 서비스나 목표를 고르지 않음 | 수동 확인: ticket의 사용자·realm·만료와 현재 목표를 확인 | 계정명이 연결된 ticket과 서비스 후보 | AD 객체·권한 목표면 [[AD Identity 확인 후 도메인 컨텍스트 열거]], ticket 사용처 확인이면 [[확보한 자격 증명으로 원격 접근 경로 선택]] | 대상 FQDN·SPN, DNS·시간 정합성과 Kerberos·대상 서비스 도달성을 확인 |
| 현재 세션이 도메인 사용자이거나 대상 호스트가 AD에 연결됨 | [[AD 도메인 컨텍스트 기본 확인]] | AD 계정과 DC 경로 | AD 객체·권한 목표면 [[AD Identity 확인 후 도메인 컨텍스트 열거]], 로컬 호스트 목표면 이 라우터 | 도메인 조인과 현재 로그온 계정을 분리하고 DC의 DNS·LDAP·Kerberos 도달성을 확인 |
| Windows 도메인 사용자 세션과 도메인 DNS·SMB 경로가 있고 접근 가능한 공유가 많아 민감 파일 후보를 자동 선별해야 함 | [[Snaffler로 도메인 SMB 공유 민감 파일 탐색]] | 읽기 가능한 도메인 공유와 자격 증명·키·설정 파일 후보 | 자격 증명 확인 시 [[확보한 자격 증명으로 원격 접근 경로 선택]] | 현재 Windows 로그인 계정, 도메인명·DC, Snaffler 실행 가능 여부와 대상 호스트 445/TCP 도달성을 확인 |

### 네트워크 경로와 확인된 고권한

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| 대상 Windows 호스트에서 추가 인터페이스·route·내부 연결을 확인함 | [[피벗팅 경로 식별과 내부망 열거]] | 새 네트워크 위치와 피벗 후보 | [[내부망 경로 확보 후 피벗 구성]] | 추가 대역 route, 중간 호스트에서의 명령 실행·포트 연결, 새 대역의 대상 서비스 도달성을 확인 |
| 대상 Windows 호스트에서 상승된 로컬 관리자 또는 SYSTEM 실행 컨텍스트를 확인함 | 수동 확인: 현재 사용자·token·호스트를 기록하고 목표에 맞는 고권한 작업으로 전환 | 확인된 Windows 고권한 세션 | [[고권한 세션 확보 후 후속 판단]] | 관리자 그룹 표시와 실제 상승 token, UAC와 현재 프로세스의 실행 주체를 확인 |

## 상태 재평가

- 로컬 Administrators, SYSTEM, 도메인 그룹 멤버십과 AD 객체 권한을 같은 신호로 취급하지 않는다.
- 인증 성공, 대화형 세션, 관리 공유 접근과 원격 명령 실행 가능성을 각각 확인한다.
- 현재 계정, 무결성 수준, 할 수 있는 작업, 보유한 평문 비밀번호·NT hash·Kerberos ticket·개인키 또는 네트워크 위치가 바뀌면 다시 분기한다.
- 도메인 조인 호스트라도 현재 세션은 로컬 사용자일 수 있으므로 플랫폼과 현재 계정을 분리한다.
- 동시에 성립하는 상태를 버리지 않는다. 현재 호스트의 실행 환경·로컬 권한이 목표면 이 라우터를, AD 객체·권한 관계가 목표면 [[AD Identity 확인 후 도메인 컨텍스트 열거]]를, 인증 자료의 사용처가 목표면 [[확보한 자격 증명으로 원격 접근 경로 선택]]을, 다른 대역 도달성이 목표면 [[내부망 경로 확보 후 피벗 구성]]을 선택한다.
- 같은 목표·상태의 명령·기법 실패는 연결된 기법 문서의 실패 분기와 이 라우터에서 처리한다. 목표가 바뀌거나 계정·권한·가능한 행위·네트워크 위치가 바뀌어 다음 라우터가 불명확하면 [[Playbook|홈]]으로 돌아가 상태와 목표를 함께 다시 고른다.

## 관련 실전 시나리오

- Web Shell에서 Meterpreter와 내부망 SMB 실행까지 이어갈 때: [[Windows Web Shell에서 내부망 SMB 관리자 명령 실행까지]]
