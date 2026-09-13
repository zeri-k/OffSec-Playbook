# Kerberos 인증 자료와 서비스 접근

## 핵심 개념

Kerberos에서 장기 key, TGT, service ticket과 ticket cache는 서로 다른 단계의 인증 자료입니다. 평문 비밀번호·NT hash/RC4 key·AES key는 계정의 장기 secret에 해당하며 KDC에 사용자의 identity를 증명하거나 응답의 session key를 복호화하는 데 쓰입니다. 반면 ticket은 KDC가 특정 principal·realm·서비스와 유효 기간을 묶어 발급한 단기 자료입니다.

| 구성 요소 | 포함·보호하는 정보 | 사용할 수 있는 범위 |
|---|---|---|
| 평문 비밀번호·NT/RC4·AES key | 사용자 또는 서비스 계정의 장기 key material | 지원되는 암호화 유형으로 새 TGT·ticket 요청. 아직 ticket이나 서비스 접근은 아님 |
| TGT | 사용자 principal, realm, 유효 시간·flag와 KDC용 session 정보 | 같은 realm 또는 trust 경로에서 KDC에 service ticket을 요청 |
| Service ticket(TGS) | client principal과 대상 SPN, 유효 시간·flag, 서비스용 session 정보 | ticket에 적힌 서비스 principal에 제시하며 다른 SPN에 일반화할 수 없음 |
| `.ccache`·`.kirbi`·LSA cache | ticket과 session key를 도구·OS가 보관하는 형식·위치 | 형식 변환이나 주입 뒤에도 ticket 자체의 principal·SPN·시간 제약은 유지됨 |
| SPN | `serviceclass/hostname[:port]` 형태의 서비스 identity | KDC가 어떤 AD 계정의 key로 service ticket을 보호할지 결정 |
| 서비스 ACL·로컬 권한 | 인증 뒤 허용할 share, 명령, 데이터와 역할 | Kerberos 인증 성공과 별도로 서비스가 평가 |

## 동작 원리

1. 사용자는 비밀번호에서 파생된 key 또는 이미 확보한 NT/RC4·AES key로 KDC의 Authentication Service와 교환합니다. 성공하면 KDC는 TGT와 사용자-KDC session key를 반환합니다. TGT 자체는 KDC의 `krbtgt` key로 보호되므로 클라이언트가 내용을 임의 변경할 수 없습니다.
2. 클라이언트는 대상 서비스의 SPN, TGT와 authenticator를 KDC의 Ticket-Granting Service에 보냅니다. KDC는 TGT와 시간·정책을 검증한 뒤 대상 SPN에 등록된 계정의 key로 보호된 service ticket을 발급합니다.
3. 클라이언트는 service ticket과 새 authenticator를 SMB·HTTP·LDAP 같은 서비스에 제시합니다. 서비스는 자신의 key로 ticket을 검증하고 client principal과 authorization data를 얻습니다.
4. 인증이 성공한 뒤 서비스는 share ACL, 로컬 그룹, AD 권한 또는 애플리케이션 역할을 별도로 평가합니다. 그러므로 ticket 발급, ticket cache 적용, 서비스 인증, 원격 명령 실행과 관리자 권한은 각각 다른 상태입니다.

### AS pre-authentication과 AS-REP 자료

일반적인 password 기반 AS 교환에서 pre-authentication은 TGT를 발급하기 전에 client가 KDC와 공유하는 장기 key를 알고 있음을 증명하는 단계입니다. KILE client는 KDC가 알려 준 지원 enctype를 바탕으로 password-derived key로 timestamp를 암호화한 `PA-ENC-TIMESTAMP`를 AS-REQ에 넣고, KDC는 계정의 key로 이를 검증합니다. 최초 AS-REQ에 pre-auth data가 없어 `KDC_ERR_PREAUTH_REQUIRED`와 지원 방식이 반환되는 것은 정상 협상의 일부일 수 있으며, 그 응답만으로 비밀번호 오류나 AS-REP Roasting 성공을 판단하지 않습니다.

계정에 `DONT_REQ_PREAUTH`가 적용되면 KDC는 이 key 보유 증명을 검증하지 않고도 그 principal에 대한 AS-REP와 TGT를 발급할 수 있습니다. AS-REP에서 client가 복호화해야 하는 부분은 대상 계정의 장기 key로 보호되므로, 해당 암호문을 `$krb5asrep$` 형식으로 보존하면 비밀번호 후보를 오프라인으로 검증할 수 있습니다. 그러나 그 문자열은 [[AS-REP Roasting]]의 크래킹 입력이지 현재 cache에 적용된 TGT가 아닙니다. 계정 속성 확인, AS-REP 응답 획득, 평문 후보 복구, 새 인증과 서비스 권한을 각각 분리합니다.

### 도메인 신뢰를 지나는 경우

도메인 trust는 계정과 권한을 합치는 기능이 아니라, 서로 다른 realm의 KDC가 referral ticket을 처리할 수 있게 하는 관계입니다. 사용자의 account domain KDC는 대상 SPN이 다른 realm에 있음을 확인하면 trust object에 보관된 inter-realm key로 referral TGT를 만들고, 신뢰 경로의 다음 KDC는 이를 검증해 최종 서비스 ticket 발급 여부를 판단합니다. 따라서 trust object 존재, referral TGT 발급, 대상 realm의 service ticket 발급, 서비스 인증과 서비스 ACL 통과는 각각 다른 성공 단계입니다.

PAC의 group SID와 `ExtraSids`는 서비스가 권한을 평가할 때 쓰는 authorization data입니다. 그러나 신뢰 경계를 통과할 때 KDC는 trust type과 `trustAttributes`에 따라 SID filtering을 적용할 수 있고, selective authentication이 설정된 경계에서는 대상 컴퓨터의 `Allowed to authenticate` 권한도 별도로 필요합니다. 같은 포리스트의 parent-child 경로에서 동작한 `ExtraSids` 절차를 forest trust나 external trust에 그대로 일반화할 수 없습니다. [[AD 도메인 트러스트 열거와 공격 경로 식별]]에서 실제 경계와 방향을 확인하고, [[자식 도메인 ExtraSids Golden Ticket]]은 `WITHIN_FOREST` 경로의 조건을 충족할 때만 선택합니다.

이 과정에서 자료의 방향도 다릅니다. [[OverPass the Hash]]는 장기 key material로 새 TGT를 요청하는 경로이고, [[Pass the Ticket]]은 이미 발급된 TGT 또는 service ticket을 현재 로그온 세션·cache에서 사용하는 경로입니다. [[Kerberoasting]]은 인증된 요청자가 SPN 계정의 service ticket을 요청해 그 암호문을 오프라인 분석 대상으로 가져오는 것이며, 요청자에게 그 서비스 계정의 권한이나 평문 비밀번호가 이미 생긴 것은 아닙니다.

## 조건이 결과에 미치는 영향

- Principal·realm: 같은 사용자 이름이어도 realm이 다르면 다른 principal입니다. 로컬 계정의 NT hash는 AD KDC가 관리하는 도메인 계정의 TGT 요청 자료가 아닙니다.
- SPN·hostname: 클라이언트가 요청한 SPN과 서비스가 자신의 계정에 등록한 SPN이 맞아야 합니다. IP 주소나 alias로 접속하면 기대한 SPN을 만들지 못해 Kerberos 대신 다른 인증으로 fallback하거나 오류가 날 수 있습니다.
- 시간: ticket의 시작·종료 시각과 authenticator 재사용 방지 때문에 client·KDC·service의 시간이 허용 오차 안에 있어야 합니다.
- 암호화 유형: key material의 종류와 계정·도메인 정책이 허용하는 enctype가 맞아야 합니다. RC4 거부는 AES key가 필요하다는 분기이지 비밀번호 자체가 틀렸다는 단독 증거가 아닙니다.
- 로그온 세션·cache: ticket을 한 세션에 주입해도 다른 사용자·프로세스의 cache에서 자동으로 보이지 않을 수 있습니다. `klist`는 실제 명령을 실행할 같은 컨텍스트에서 확인해야 합니다.
- Delegation과 두 번째 홉: 첫 서비스에 인증한 사실만으로 그 서비스가 사용자를 대신해 두 번째 서비스의 ticket을 얻을 수 없습니다. forwardable flag, 위임 설정과 실행 컨텍스트가 별도로 필요합니다.
- 신뢰 방향과 SID filtering: `Direction`은 trust object를 보유한 도메인 관점의 값입니다. 인증 가능 방향이 맞아도 SID filtering·selective authentication·대상 서비스 ACL 때문에 실제 접근은 거부될 수 있습니다.

## 실전에서의 해석

| 관찰한 상태 | 확정할 수 있는 것 | 아직 확인할 것 |
|---|---|---|
| 계정 비밀번호·NT/RC4·AES key 보유 | 특정 principal 후보의 장기 인증 자료 보유 | KDC가 받아들이는지, realm·enctype가 맞는지 |
| TGT 또는 service ticket이 `klist`·cache에 표시됨 | 해당 로그온 세션의 cache에 표시된 principal·server용 ticket 자료가 존재함 | Start Time·End Time·Renew Time·flag와 현재 시각, 해당 자료의 발급 요청 성공 여부와 실제 서비스 사용 성공 |
| TGT 발급 요청이 성공 응답을 반환함 | 해당 AS 요청에서 KDC가 principal용 TGT를 발급함 | ticket의 유효 시간·cache 적용 여부와 대상 SPN의 service ticket 요청 성공 |
| 특정 SPN의 service ticket 발급 요청이 성공 응답을 반환함 | 해당 TGS 요청에서 KDC가 그 SPN용 service ticket을 발급함 | ticket의 유효 시간·현재 로그온 세션 또는 cache 적용 여부와 서비스 수락 여부 |
| Kerberoasting hash 추출 | service ticket의 오프라인 분석 자료를 얻음 | 비밀번호 복구 가능성, 복구된 credential 유효성과 서비스 권한 |
| SMB·WinRM·LDAP Kerberos 인증 성공 | 서비스가 제시된 Kerberos 인증 자료를 수락하고 해당 principal의 인증 상태를 성립함 | share 읽기·쓰기, 원격 shell, 디렉터리 권한과 관리자 여부 |

`klist`는 현재 로그온 세션의 ticket cache를 조회합니다. cache에 보이는 ticket은 존재 자체만 확정하며, 시간 필드와 현재 시각을 따로 확인해야 합니다. KDC 발급 성공은 해당 AS·TGS 요청의 성공 응답으로, 실제 사용 성공은 대상 서비스의 인증 응답으로 각각 확인합니다.

`KDC_ERR_S_PRINCIPAL_UNKNOWN`은 SPN 누락·불일치, `KRB_AP_ERR_SKEW`는 시간 차이, `KRB_AP_ERR_MODIFIED`는 서비스가 ticket을 올바른 key로 해독하지 못한 상황을 우선 가리킵니다. 하나의 오류를 계정 비밀번호 실패로 뭉뚱그리지 않고 AS 요청, TGS 요청, AP 교환 중 처음 실패한 경계를 확인합니다.

## 관련 기법

- [[Pass the Ticket]]
- [[OverPass the Hash]]
- [[Kerberoasting]]
- [[AS-REP Roasting]]
- [[WinRM 두 번째 홉 명시적 자격 증명 재인증]]

## 참고 링크

- [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/windows-server/security/kerberos/kerberos-authentication-overview)
- [Microsoft: klist](https://learn.microsoft.com/windows-server/administration/windows-commands/klist)
- [Microsoft: Ticket-Granting Tickets](https://learn.microsoft.com/windows/win32/secauthn/ticket-granting-tickets)
- [Microsoft: Service principal names](https://learn.microsoft.com/windows/win32/ad/service-principal-names)
- [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance)
- [MS-KILE: Pre-authentication](https://learn.microsoft.com/openspecs/windows_protocols/ms-kile/4ce3ddc0-aaaa-4a1b-b48b-62a07e906926)
- [MS-KILE: Pre-authentication data](https://learn.microsoft.com/openspecs/windows_protocols/ms-kile/ae60c948-fda8-45c2-b1d1-a71b484dd1f7)
- [MS-KILE: AS exchange](https://learn.microsoft.com/openspecs/windows_protocols/ms-kile/a076994a-47a9-4d16-af77-d4abb523afe4)
- [MS-KILE: Cross-Domain Referrals](https://learn.microsoft.com/openspecs/windows_protocols/ms-kile/bac4dc69-352d-416c-a9f4-730b81ababb3)
- [MS-ADTS: trustDirection](https://learn.microsoft.com/openspecs/windows_protocols/ms-adts/5026a939-44ba-47b2-99cf-386a9e674b04)
- [MS-ADTS: trustAttributes](https://learn.microsoft.com/openspecs/windows_protocols/ms-adts/e9a2d23c-c31e-4a6f-88a0-6646fdb51a3c)
- [MS-PAC: SID Filtering and Claims Transformation](https://learn.microsoft.com/openspecs/windows_protocols/ms-pac/55fc19f2-55ba-4251-8a6a-103dd7c66280)
