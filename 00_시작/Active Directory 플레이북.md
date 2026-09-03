---
tags:
  - 환경/ad
---

# Active Directory 플레이북

Active Directory(AD) 학습과 시험에서 현재 상태를 먼저 고르고, 확인한 단서를 다음 세부 기법으로 넘기는 전용 진입점이다. `02_공격기법`의 물리 구조는 MITRE ATT&CK 전술을 유지하며, 이 문서는 전술 폴더를 가로지르는 실행 순서만 연결한다.

시작할 때는 명령 실행 호스트의 네트워크 위치, 그 호스트에서 도달하는 대상 주소·포트·서비스, 현재 운영체제 계정과 AD 계정, 보유한 비밀번호·NT hash·AES key·ticket, 이미 검증한 로컬·도메인 권한을 따로 기록한다. 사용자 이름을 아는 것, 인증 자료를 보유한 것, 특정 서비스 인증이 성공한 것과 그 서비스에서 관리 권한을 얻은 것은 서로 다른 상태다.

## 현재 상태에서 시작

| 현재 관찰한 위치·계정·권한 | 먼저 열 문서 | 성공하면 얻는 상태 | 실패 시 다음 확인 |
|---|---|---|---|
| 내부 네트워크의 실행 호스트에서 DNS·Kerberos·LDAP·SMB 후보에는 도달하지만 사용할 AD 계정이 없음 | [[무인증 내부 네트워크에서 AD 단서 확인]] | 호스트·도메인 컨트롤러(DC)·도메인·사용자 이름 후보 | DNS 이름 해석, TCP·UDP별 도달성, 실행 호스트의 네트워크 위치를 다시 확인 |
| AD 사용자·서비스·머신 계정의 비밀번호·NT hash·AES key 또는 ticket이 있고 대상 인증 서비스에 도달함 | [[AD Identity 확인 후 도메인 컨텍스트 열거]] | 인증에 사용한 AD 계정과 그 계정이 읽을 수 있는 도메인·객체 관계 | 인증 자료 종류, 도메인·사용자 형식, DC 시간과 대상 서비스 지원 방식을 다시 확인 |
| 다른 도메인 또는 포리스트 trust를 확인했지만 인증 방향과 후속 경로가 불명확함 | [[AD 도메인 트러스트 열거와 공격 경로 식별]] | trust 유형·방향·경계와 대상 도메인의 SPN·외부 그룹 멤버십 후보 | Source·Target, 선택적 인증, SID filtering과 대상 LDAP·Kerberos 접근을 다시 확인 |
| Windows 호스트에 제한된 셸이 있지만 현재 계정의 로컬·도메인 소속과 권한이 불명확함 | [[제한된 Windows 셸에서 AD와 호스트 열거]] | 현재 계정, DC, 라우팅 경로와 세션 단서 | 셸 실행 사용자와 네트워크·DNS 구성을 먼저 확인 |
| BloodHound 또는 접근 제어 목록(ACL)에서 현재 사용 중인 AD 계정이 특정 객체를 제어할 수 있음 | [[AD 객체 제어권 확보 후 악용 경로 선택]] | 확인한 권한명·대상 객체·상속 범위에 직접 대응하는 기법 | 수집 시점, 현재 계정의 보안 식별자(SID), 대상 객체와 실제 권한 상속을 재확인 |
| 비밀번호·NT hash·AES key·SSH key 또는 ticket을 얻었지만 어느 원격 서비스에서 어떤 권한으로 통하는지 모름 | [[확보한 자격 증명으로 원격 접근 경로 선택]] | 서비스별 인증 성공과 일반 사용자·원격 관리·DB 권한 범위 | 인증 자료의 주체·대상·형식과 해당 서비스 도달성을 다시 확인 |
| 한 Windows 호스트에서 로컬 관리자 또는 SYSTEM 세션을 실제로 검증함 | [[고권한 세션 확보 후 후속 판단]] | 해당 호스트에서 수집 가능한 인증 자료와 새 내부망 경로 | 현재 토큰, UAC 원격 제한, 보호 프로세스와 도메인 권한을 별도로 확인 |
| 제어 중인 AD 사용자·그룹 이름은 있지만 그 SID의 도메인 복제 권한 여부를 확인하지 못함 | [[AD 계정의 디렉터리 복제 권한 확인]] | 같은 SID의 두 필수 복제 권한 충족 또는 미충족 판정 | 확인 대상 계정명·SID, 도메인 DN과 ACL 읽기 결과를 재확인 |
| 현재 AD 계정에 디렉터리 복제 권한이 있음 | [[DCSync]] | 도메인 계정의 NT hash·Kerberos key 등 인증 자료 | 그룹 이름이 아니라 복제 권한 GUID와 대상 도메인 명명 컨텍스트를 재확인 |
| 같은 포리스트의 자식 도메인 관리자·복제 권한 또는 `krbtgt` key와 부모 trust 단서가 있음 | [[자식 도메인 장악 후 부모 도메인 경로 선택]] | 부모 도메인 대상 계정의 NT hash·Kerberos key로 이어질 입력·경로 또는 실제 자격 증명 | 실제 자식 principal·RID, trust direction, SID filtering, 양쪽 realm DNS·KDC와 부모 RPC 경로를 재확인 |

## AD 계정 확보 전

[[dehashed.py]]에서 얻은 이메일·사용자 이름·유출 비밀번호는 현재 유효한 AD 계정이나 비밀번호로 단정하지 않고 후보로만 다룬다.

1. [[내부망 수동 호스트 식별]]로 현재 브로드캐스트 도메인의 IP·이름 후보를 만든다.
2. 같은 링크에서 LLMNR·NBT-NS 요청이 반복되면 [[무인증 내부망에서 Responder로 AD 자격 증명 확보]]에서 수동 관찰, 포이즈닝, NetNTLMv2 수집, 오프라인 크래킹과 인증 검증을 순서대로 수행한다.
3. 대상 Classless Inter-Domain Routing(CIDR) 대역에서 [[ICMP 기반 내부 호스트 확인]]으로 응답 호스트를 보강한다. Internet Control Message Protocol(ICMP) 무응답은 호스트 부재로 단정하지 않는다.
4. Domain Name System(DNS)·Kerberos·Lightweight Directory Access Protocol(LDAP)·Server Message Block(SMB) 단서가 보이면 [[AD 도메인 컨텍스트 기본 확인]]으로 도메인과 DC를 확정한다.
5. [[인증 전 AD 사용자 목록 수집]]과 [[AD 비밀번호 정책 열거 및 조회]]로 유효 사용자와 잠금 위험을 먼저 분리한다.
6. 도메인명과 사용자 후보가 있고 현재 명령 실행 위치에서 DC의 Kerberos 88번 포트에 연결할 수 있으면, 별도 도메인 계정 없이 [[AS-REP Roasting]]을 요청한다. `$krb5asrep$`이 반환된 사용자만 사전 인증이 꺼진 대상이며, `KDC_ERR_PREAUTH_REQUIRED` 사용자는 이 기법의 대상이 아니다.
7. 사용자 목록, 잠금 정책, 시도 횟수와 간격이 정해진 경우에만 [[내부 AD Password Spraying]]을 수행한다.

## 인증 후 열거

| 확인할 축 | 실행 문서 | 판단 포인트 |
|---|---|---|
| 사용자·컴퓨터 객체 | [[인증 후 AD 사용자와 컴퓨터 객체 열거]] | 계정 속성과 컴퓨터 FQDN을 수집하고 현재성을 확인해 후속 열거 입력을 만든다. |
| 고권한·업무 그룹과 중첩 구성원 | [[AD 고권한 그룹과 중첩 구성원 열거]] | 직접·간접 구성원 경로를 확인하고 실제 권한 검증 대상을 좁힌다. |
| Service Principal Name(SPN) 계정 | [[SPN 계정 열거]] | 사용자 기반 서비스 계정과 컴퓨터·관리형 계정을 구분한 뒤 Kerberoasting 대상을 고른다. |
| 원격 Windows 로그온 사용자·로컬 관리자 단서 | [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]] | 세션 후보와 현재 계정의 로컬 관리자 후보를 구분해 다음 호스트를 고른다. |
| 사용자·그룹·컴퓨터·ACL·GPO·trust·세션 관계 | [[AD 관계 그래프 수집과 공격 경로 식별]] | BloodHound edge를 조사 순서로 사용하고 중요한 관계를 원본 조회로 재검증한다. |
| 도메인·포리스트 신뢰 관계 | [[AD 도메인 트러스트 열거와 공격 경로 식별]] | Source·Target, 방향, 전이성, 포리스트 경계와 실제 대상 도메인 조회 가능 여부를 구분한다. |
| DNS 레코드 | [[AD DNS 레코드 열거]] | hostname과 IP를 연결하고 `unknown` 레코드는 재조회한다. |
| Description·User Account Control(UAC) 위험 속성 | [[AD 계정 위험 속성과 Description 열거]] | 평문 단서와 `PASSWD_NOTREQD`를 인증 성공으로 오인하지 않는다. |
| SMB·SYSVOL | [[Snaffler로 도메인 SMB 공유 민감 파일 탐색]], [[SMB 공유 자격증명 수집]], [[SYSVOL GPP 자격 증명 수집]] | 도메인 전체 공유 후보를 선별한 뒤 파일 READ와 오래된 값·현재 서비스에서 검증한 비밀번호·key를 분리한다. |
| 객체 ACL | [[AD ACL 권한 열거와 공격 경로 식별]] | 현재 계정 SID, 대상 Distinguished Name(DN), 권한 종류와 상속 범위를 함께 확인한다. |
| AD 사용자·그룹의 디렉터리 복제 권한 | [[AD 계정의 디렉터리 복제 권한 확인]] | 도메인 루트에서 같은 SID의 `Get-Changes`와 `Get-Changes-All`을 확인하고 DCSync 실행 전제를 판정한다. |
| Group Policy Object(GPO) 쓰기 단서 | [[AD GPO 쓰기 권한과 영향 범위 열거]] | 표시 이름, 링크 대상 Organizational Unit(OU)과 영향 호스트를 확인한 뒤 변경 여부를 판단한다. |
| 도메인 전체 보안 구성·GPO·trust·계정 상태 감사 | [[AD 보안 구성과 GPO 감사]] | snapshot·healthcheck·GPO·인벤토리 결과를 분리하고 각 finding을 실제 객체·서비스에서 재검증한다. |
| 원격 접근 그룹·로컬 그룹 | [[AD 원격 접근 권한 열거]] | 그룹 멤버십과 실제 Remote Desktop Protocol(RDP)·Windows Remote Management(WinRM)·Microsoft SQL Server(MSSQL) 접근 성공을 구분한다. |
| Spooler 원격 인터페이스 | [[Print Spooler 원격 인터페이스 노출 확인]] | `True`는 인증 강제 후보이며 relay 성공이나 PrintNightmare 취약성 증명이 아니다. |

## 자격 증명 공격

| 단서 | 실행 문서 | 성공 뒤 상태 |
|---|---|---|
| 도메인명·사용자 후보와 DC Kerberos 88번 접근, 공격자 도메인 계정은 불필요 | [[AS-REP Roasting]] | 사전 인증이 꺼진 사용자의 AS-REP hash, 복구 시 해당 사용자의 평문 비밀번호 후보 |
| 유효한 도메인 계정 또는 TGT와 DC LDAP 389/636번 접근 | [[SPN 계정 열거]] | SPN이 설정된 사용자 기반 서비스 계정 후보 |
| SPN 서비스 계정과 DC Kerberos 88번 접근 | [[Kerberoasting]] | 서비스 계정의 Ticket Granting Service(TGS) hash, 복구 시 해당 서비스 계정의 평문 비밀번호 후보 |
| 사용자 SPN 쓰기 권한 | [[임시 SPN 설정]]과 [[표적 Kerberoasting]] | 변경한 SPN 원복 후 대상 사용자 비밀번호 후보 |
| Local Administrator Password Solution(LAPS) 읽기 권한 | [[LAPS 비밀번호 읽기 권한과 자격 증명 수집]] | LAPS가 관리하는 특정 호스트의 로컬 관리자 비밀번호 |
| SMB/SYSVOL의 비밀값 | [[SYSVOL GPP 자격 증명 수집]] | 로컬 또는 도메인 계정의 비밀번호·key 후보 |
| Link-Local Multicast Name Resolution(LLMNR)·NetBIOS Name Service(NBT-NS) 요청 | [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]] | NetNTLMv2 challenge-response, relay·offline cracking 분기 |
| 디렉터리 복제 권한 후보 계정 | [[AD 계정의 디렉터리 복제 권한 확인]] | 확인 대상 SID의 두 필수 복제 권한 충족 또는 미충족 판정 |
| 같은 계정 SID의 두 필수 디렉터리 복제 권한 확인 | [[DCSync]] | 도메인 계정의 NT hash·Kerberos key 등 인증 자료 |

## ACL 제어권

ACL은 권한명만 보고 실행하지 않는다. [[AD 객체 제어권 확보 후 악용 경로 선택]]에서 대상 객체와 현재 사용 중인 계정을 다시 맞춘 뒤 아래 세부 기법으로 이동한다.

- 사용자 비밀번호 제어: [[AD 사용자 비밀번호 강제 재설정]]
- 그룹 구성원 제어: [[AD 그룹 구성원 추가로 권한 확대]]
- 사용자 SPN 제어: [[임시 SPN 설정]]과 [[표적 Kerberoasting]]
- `msDS-KeyCredentialLink` 제어: [[Shadow Credentials]]
- 디렉터리 복제 권한 후보 확인: [[AD 계정의 디렉터리 복제 권한 확인]]
- 같은 SID의 두 필수 복제 권한 확인 후 실행: [[DCSync]]

## 고영향 경로

- NoPac 영향 조건: [[NoPac sAMAccountName 스푸핑 권한 상승]]
- PrintNightmare 영향 조건: [[PrintNightmare 원격 코드 실행]]
- Active Directory Certificate Services(AD CS) Web Enrollment와 relay 조건: [[AD CS ESC8 NTLM Relay]]
- 같은 포리스트의 자식 도메인 장악 후 부모 권한 확장: [[자식 도메인 ExtraSids Golden Ticket]]

## 도메인 트러스트

| 확인한 trust 상태 | 다음 문서 | 성공 기준 |
|---|---|---|
| trust가 보이지만 유형·방향·인증 가능 범위가 불명확함 | [[AD 도메인 트러스트 열거와 공격 경로 식별]] | Source·Target, 방향, 포리스트 경계와 대상 도메인 객체 조회 결과 확인 |
| 같은 포리스트의 자식 도메인을 장악했고 자식 `krbtgt` key를 얻을 수 있음 | [[자식 도메인 장악 후 부모 도메인 경로 선택]] | ExtraSids ticket 생성과 부모 서비스 권한 또는 부모 계정 DCSync를 단계별로 확인 |
| forest trust를 통해 대상 도메인의 SPN 계정이 조회됨 | [[Kerberoasting]] | 대상 도메인이 표시된 `$krb5tgs$` hash 획득, 복구 후 대상 도메인 인증 성공 |
| Linux에서 양쪽 포리스트의 BloodHound 데이터를 수집할 수 있음 | [[AD 도메인 트러스트 열거와 공격 경로 식별]] | 양쪽 도메인의 JSON과 `Users with Foreign Domain Group Membership` 관계 확인 |
| 대상 도메인 로컬 그룹에 현재 포리스트 계정이 외부 구성원으로 포함됨 | [[WinRM 원격 PowerShell 세션]] | 대상 호스트에 해당 계정으로 원격 로그인하고 실제 계정·권한 확인 |
| 다른 포리스트에 동일하거나 대응되는 운영 계정이 있고 현재 비밀번호·hash를 보유함 | [[확보한 자격 증명으로 원격 접근 경로 선택]] | 대상 forest의 서비스 인증 성공과 실제 권한 확인 |

trust 존재, 대상 도메인 객체 조회, TGS hash 획득, 원격 로그인과 관리자 권한은 각각 다른 상태다. forest 간 SID History 악용은 검증된 실행 명령과 확인 절차가 이 Playbook에 없으므로 별도 공격기법으로 만들지 않는다.

취약점 스캐너의 가능성 표시, ticket 생성, SYSTEM 셸과 DCSync 성공을 각각 다른 단계로 기록한다. 객체·서비스·파일을 변경한 기법은 해당 문서의 `변경 영향과 복구`까지 완료한다.

## 아직 확장하지 않는 범위

- forest 간 SID History 변경·악용은 검증된 실행 명령과 확인 결과가 없어 별도 기법으로 확장하지 않는다. Linux forest trust 흐름은 신뢰 대상 Kerberoasting과 양쪽 도메인의 BloodHound 수집·외부 그룹 멤버십 확인까지 연결한다.
- Windows와 Linux 로컬 권한 상승은 각 HTB 모듈 학습 후 별도 개념 디렉터리와 실행 흐름으로 확장한다.
