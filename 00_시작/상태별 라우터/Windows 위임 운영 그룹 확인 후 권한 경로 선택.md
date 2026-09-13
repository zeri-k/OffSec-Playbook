---
tags:
  - 환경/windows
  - 환경/ad
  - 기능/권한상승
시작상태: ["Windows 현재 token에 Event Log Readers·DnsAdmins·Hyper-V Administrators·Print Operators 중 하나가 반영됨", "Windows 위임 운영 그룹 멤버십 확인"]
목표: ["위임 그룹의 실제 대상 권한 확인", "자격 증명 단서 확보", "서비스 계정 또는 SYSTEM 권한 확인"]
현재계정: ["Windows 로컬 사용자", "AD 도메인 사용자", "AD 서비스 계정"]
현재 가능한 행위: ["Windows 셸에서 명령 실행", "그룹 SID와 대상 channel·DNS server·Hyper-V VM·privilege 조회"]
필요권한: ["현재 Windows 사용자 token에 반영된 위임 그룹 SID와 대상 객체가 허용한 실제 권한"]
필요정보: ["현재 token", "대상 event log channel·Windows DNS server·Hyper-V VM 또는 SeLoadDriverPrivilege 중 해당 항목"]
네트워크위치: ["권한을 평가할 Windows host 또는 관리 RPC가 닿는 Windows server"]
---

# Windows 위임 운영 그룹 확인 후 권한 경로 선택

## 상태 라우터 개요

Event Log Readers, DnsAdmins, Hyper-V Administrators와 Print Operators는 이름만으로 같은 권한 상승을 만들지 않는다. 현재 token의 SID와 각 그룹이 제어하는 event channel·DNS server·VM·driver-load privilege를 실제로 확인한 뒤 현재 목표에 맞는 한 경로를 고른다.

## 적용 조건

| 상태 축 | 조건 |
|---|---|
| 대상 플랫폼 | Windows host, AD-integrated Windows DNS 또는 Hyper-V host |
| 현재 계정 | 위임 그룹이 현재 token에 반영된 로컬·도메인·서비스 계정 |
| 현재 가능한 행위 | Windows 명령 실행과 대상 객체 조회 |
| 현재 권한 | 그룹 디렉터리 상태가 아니라 현재 token SID와 실제 channel·service·VM·privilege 권한 |
| 보유 정보 | 해당 그룹 이름, 대상 host·service·VM 또는 event channel |
| 네트워크 위치 | 로컬 host 또는 관리 RPC·CIM이 대상 server에 닿는 위치 |
| 목표 | 그룹별 읽기·설정·가상화·driver 권한을 재사용 가능한 결과로 검증 |

## 판단 경로

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| 현재 token에 Event Log Readers가 반영되고 읽을 event channel과 시간 범위를 확인함 | [[Windows 이벤트 로그에서 민감 명령줄 검색]] | 계정·시각·process가 연결된 평문 비밀번호·token 후보 또는 channel 가시성 미확정 | 자격 증명 확인 시 [[확보한 자격 증명으로 원격 접근 경로 선택]], 아니면 이 라우터 | Security·PowerShell channel ACL, 4688 명령줄 포함·4104 기록과 retention을 확인하고 그룹 이름만으로 모든 channel read를 확정하지 않음 |
| 현재 token에 DnsAdmins가 반영되고 DNS service account의 controlled code execution이 목표임 | [[DnsAdmins DNS 서버 플러그인 DLL 실행]] | plug-in 설정만 변경된 중간 상태 또는 DNS 서비스 계정의 controlled code execution | SYSTEM 확인 시 [[고권한 세션 확보 후 후속 판단]], 실패·미승인 시 이 라우터 | 기존 `ServerLevelPluginDll`, DLL absolute path·ACL, 별도 DNS stop/start 권한과 서비스 영향 승인을 확인 |
| 현재 token에 DnsAdmins가 반영되고 WPAD client의 NTLM 인증 경로를 시험하는 것이 목표임 | [[DnsAdmins WPAD DNS 레코드로 NTLM 인증 유도]] | wpad DNS 응답, client HTTP 요청 또는 NetNTLM challenge-response | 평문 확인 시 [[확보한 자격 증명으로 원격 접근 경로 선택]], relay는 target service 권한 재평가 | 기존 global query block list·wpad record, 짧은 TTL, listener 도달성과 승인된 client·zone·시간을 확인 |
| 현재 token에 Hyper-V Administrators가 반영되고 승인된 VM과 export 저장 공간을 확인함 | [[Hyper-V VM 내보내기와 가상 디스크 오프라인 수집]] | export VHDX와 NTDS·SYSTEM 또는 SAM·SECURITY·SYSTEM 후보 | credential 자료 확인 시 [[확보한 자격 증명으로 원격 접근 경로 선택]], 아니면 이 라우터 | 실행 중인 원본 VHDX를 직접 조작하지 않고 VM ID·checkpoint chain, 빈 export path와 read-only mount를 확인 |
| 현재 token에 Print Operators가 반영되고 `whoami /priv`에서 `SeLoadDriverPrivilege`가 실제로 확인됨 | [[SeLoadDriverPrivilege로 취약 드라이버 권한 상승]] | driver load 중간 상태 또는 child Identity로 검증한 SYSTEM process | SYSTEM 확인 시 [[고권한 세션 확보 후 후속 판단]], 실패·차단 시 이 라우터 | 그룹 멤버십을 privilege로 추정하지 않는다. signature·architecture, Code Integrity·HVCI·blocklist와 actual loaded driver를 확인 |

## 상태 재평가

- AD 그룹의 `member` 상태와 현재 Windows token의 SID를 분리한다. 그룹 변경 직후 기존 session은 새 권한을 반영하지 않을 수 있다.
- Event Log Readers는 읽기, DnsAdmins는 DNS 설정, Hyper-V Administrators는 VM 관리, Print Operators는 DC의 printer·driver 관련 권한 후보이므로 한 그룹의 성공을 다른 그룹 권한으로 확대하지 않는다.
- 설정 write, service restart, VM export, driver load와 SYSTEM process는 각각 독립적인 성공 단계다.
- 평문 비밀번호·token·NetNTLM·NT hash·NTDS/SYSTEM file은 서로 다른 자료이며 실제 사용처는 [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 다시 고른다.
- 목표가 AD 객체 권한이면 [[AD Identity 확인 후 도메인 컨텍스트 열거]], 현재 host의 일반 권한 상승이면 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]로 돌아간다.
