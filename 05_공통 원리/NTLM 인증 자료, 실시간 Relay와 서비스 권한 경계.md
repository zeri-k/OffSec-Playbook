# NTLM 인증 자료, 실시간 Relay와 서비스 권한 경계

## 핵심 개념

NTLM 관련 결과는 모두 `hash`라고 부르면 사용할 수 있는 위치를 잘못 판단하기 쉽습니다. 계정의 장기 secret, 한 번의 challenge-response 교환을 기록한 자료, relay로 성립한 서비스 연결과 그 연결에서 얻은 새 인증 자료는 서로 다른 상태입니다.

| 자료·상태 | 의미 | 직접 할 수 없는 것 |
|---|---|---|
| 평문 비밀번호 | 계정의 password 후보 | 계정 상태·지원 서비스·실제 권한을 확인하지 않은 로그인 성공 |
| NT hash | Windows 계정 비밀번호에서 파생된 장기 key | 모든 서비스 인증과 관리자 권한. 대상이 NTLM을 허용하고 계정 범위·권한이 맞아야 함 |
| NetNTLMv1/v2 challenge-response | 특정 server challenge에 대한 client의 인증 응답과 계정·도메인 단서 | 계정 NT hash로 취급한 Pass the Hash, 다른 challenge에 대한 저장 응답 재사용 |
| relay된 서비스 인증 연결 | 원래 client의 실시간 NTLM 교환을 최종 target service가 수락한 상태 | 평문·NT hash 획득, 다른 서비스 세션이나 관리자 권한 |
| relay 후 PFX·ticket·dump·객체 변경 | relay된 계정의 대상 서비스 권한으로 수행한 별도 작업 결과 | 원래 인증 성공만으로 이 후속 결과가 자동 발생했다는 해석 |

참여 주체도 분리합니다.

| 주체 | 역할 | 확인할 경계 |
|---|---|---|
| 원래 client | 사용자·서비스·머신 계정의 NTLM 메시지를 생성 | 어떤 Identity와 host가 인증을 시작했는지 |
| 인증 수신 endpoint | client가 연결한다고 믿거나 강제로 연결된 서비스 | 이름 해석·UNC·HTTP·RPC 등 인증이 시작된 이유와 listener 위치 |
| relay listener | 원래 client와 최종 target 사이에서 NTLM 메시지를 전달 | 두 네트워크 경로, 동시 교환과 protocol 변환 지원 |
| 최종 target service | 자신의 challenge를 보내고 response를 검증한 뒤 서비스 session 생성 | NTLM 수락, signing·EPA·CBT와 relay 계정의 서비스 권한 |
| 검증 주체 | 로컬 계정이면 target의 계정 DB, 도메인 계정이면 DC가 검증에 참여 | 인증 검증 성공과 application authorization의 차이 |

## 동작 원리

연결 지향 NTLM 인증에서는 client가 `NEGOTIATE_MESSAGE`를 보내고 server가 자신의 정책·대상 정보를 담은 `CHALLENGE_MESSAGE`와 8-byte server challenge를 반환합니다. client는 계정 secret에서 계산한 response, 계정·도메인과 협상 정보를 `AUTHENTICATE_MESSAGE`로 돌려보냅니다. server는 response를 검증하고 성공하면 application protocol에 그 Identity의 security context를 설정합니다. 도메인 계정은 server가 challenge·response와 계정 정보를 DC에 보내 검증할 수 있지만, 파일·LDAP·HTTP 같은 실제 허용 작업은 최종 service가 별도로 판정합니다.

### 캡처와 오프라인 복구

[[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]이나 [[MSSQL 서비스 Hash 캡처]]에서 저장하는 NetNTLMv2 자료는 그 수신 endpoint가 보낸 challenge와 client response의 기록입니다. 비밀번호 후보로 response를 다시 계산해 일치 여부를 확인하는 오프라인 입력으로는 사용할 수 있지만, 계정의 NT hash 자체는 아닙니다. 평문 후보를 복구한 뒤에도 현재 계정 상태와 대상 서비스 인증은 별도로 확인합니다.

### 실시간 relay

[[NTLM Relay 조건 검토]]의 relay는 저장 응답을 나중에 재생하는 절차가 아닙니다. relay listener가 원래 client의 negotiate를 최종 target으로 보내고, target이 만든 challenge를 원래 client에게 돌려준 뒤, 그 challenge에 대한 authenticate를 target으로 전달하는 동안 두 연결이 함께 살아 있어야 합니다. NetNTLMv2 response에는 server challenge와 target·channel 관련 정보가 결합될 수 있으므로 다른 target이 새로 만든 challenge에 과거 응답을 임의로 대입하지 않습니다.

target이 인증을 수락해도 확보한 것은 그 service 안의 relay 계정 security context입니다. SMB share 읽기·원격 실행, LDAP 객체 쓰기, HTTP enrollment는 각각 별도 권한입니다. [[AD CS ESC8 NTLM Relay]]에서 PFX를 얻거나 이후 PKINIT으로 TGT를 발급받는 것은 relay 인증 다음의 독립된 서비스 작업 결과입니다.

### relay를 제한하는 결합

- application protocol이 인증 뒤 메시지를 session key로 서명하도록 강제하면 relay listener가 그 service 통신을 임의로 계속하기 어렵습니다. SMB signing required 여부는 SMB target에 대한 조건이며 다른 protocol의 보호 상태를 대신하지 않습니다.
- EPA는 service binding으로 의도한 SPN을, TLS에서는 CBT로 바깥 channel과 Windows 인증을 결합할 수 있습니다. server가 이를 실제로 강제해야 relay 방어로 판정하며 audit·지원 상태만으로 보호를 확정하지 않습니다.
- NTLM 수신 차단, 지원 protocol 불일치, client→listener 또는 listener→target 경로 부재는 인증 교환 전 단계에서 relay를 막습니다.
- signing·EPA 조건이 약해도 relay 계정에 대상 action 권한이 없으면 인증 연결만 성립하고 원하는 작업은 거부됩니다.

## 실전에서의 해석

| 관찰한 상태 | 확정할 수 있는 것 | 다음에 확인할 것 |
|---|---|---|
| `USER::DOMAIN:...` NetNTLMv2 line 저장 | 특정 NTLM 교환의 challenge-response 확보 | 전체 형식·계정 출처와 오프라인 복구 후보 |
| Hashcat에서 평문 후보 복구 | 저장 response와 일치하는 password 후보 | 현재 계정 상태와 승인된 서비스의 실제 인증 |
| 계정 NT hash 확보 | NTLM 장기 key 후보 | [[Pass the Hash]]가 지원하는 대상 서비스·계정 범위와 권한 |
| relay listener에 target 인증 성공 표시 | target service가 relay된 Identity를 수락 | service 안의 READ·WRITE·명령·enrollment 권한 |
| relay 인증 뒤 작업 거부 | 인증과 target action 권한이 다름 | relay 계정 ACL·role과 target별 보호 설정 |
| PFX·TGT·dump·객체 변경 출력 | 해당 후속 action의 개별 성공 후보 | 산출물·Identity·실제 서비스 사용과 복구 상태 |

`NetNTLM hash`, `NTLM hash`, `relay 성공` 같은 짧은 표현만 기록하지 않고 자료 형식, 인증 주체, challenge를 만든 target, 수신 시각과 실제 성공 단계를 함께 구분합니다. listener·coercion·크래킹·서비스 action의 실행 명령과 포트 충돌·산출물 정리는 연결된 실전 문서에서 확인합니다.

## 참고 링크

- [Microsoft MS-NLMP: Message Syntax](https://learn.microsoft.com/openspecs/windows_protocols/ms-nlmp/907f519d-6217-45b1-b421-dca10fc8af0d)
- [Microsoft MS-NLMP: NTLM over SMB](https://learn.microsoft.com/openspecs/windows_protocols/ms-nlmp/c083583f-1a8f-4afe-a742-6ee08ffeb8cf)
- [Microsoft MS-NLMP: Client Receives a CHALLENGE_MESSAGE](https://learn.microsoft.com/openspecs/windows_protocols/ms-nlmp/8bbf686d-06d1-4511-9502-b0c071f9d6d7)
- [Microsoft: SMB signing](https://learn.microsoft.com/windows-server/storage/file-server/smb-signing)
- [Microsoft: Extended Protection](https://learn.microsoft.com/windows/win32/wsw/extended-protection)
- [Microsoft KB5005413: Mitigating NTLM Relay Attacks on AD CS](https://support.microsoft.com/help/5005413)
