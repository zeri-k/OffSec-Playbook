---
tags:
  - 환경/ad
  - 서비스/smb
  - 서비스/kerberos
  - 서비스/ldap
시작조건: ["Domain Controller·도메인·Kerberos realm 식별", "도메인 자격 증명 미확보", "명령 실행 호스트에서 AD 서비스 중 하나 이상 접근 가능"]
필요권한: ["명령 실행 호스트의 일반 사용자 권한과 열거 도구 실행 권한", "대상 도메인 계정은 불필요"]
필요조건: ["SMB NULL Session, LDAP Anonymous Bind 또는 Kerberos 사용자 존재 응답 중 하나", "Kerbrute 사용 시 사용자명 후보 목록"]
결과: ["SMB·LDAP가 반환한 AD 사용자 객체 또는 Kerberos 응답으로 검증한 사용자명 후보", "사용자명 형식", "Password Spraying 대상과 제외 계정 후보"]
---

# 인증 전 AD 사용자 목록 수집

## 한 줄 판단

도메인 자격 증명이 없는 명령 실행 호스트에서 DC의 445/TCP, 389/TCP 또는 88/TCP 중 도달 가능한 서비스에 SMB NULL Session, LDAP Anonymous Bind 또는 Kerberos 사용자 존재 요청을 보내 사용자 객체·사용자명 후보와 제외 계정 단서를 수집한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | `rpcclient`, `ldapsearch` 또는 `kerbrute`를 실행할 수 있는 호스트 | 도구 존재와 현재 네트워크 위치 확인 | 로컬 도구 오류와 DC 응답 실패를 분리 |
| 네트워크 경로 | 명령 실행 호스트에서 DC의 445/TCP, 389/TCP 또는 88/TCP 중 선택한 서비스에 도달 가능 | 서비스별 응답, timeout과 연결 오류 확인 | 한 서비스 실패를 전체 AD 미도달로 일반화하지 말고 다른 식별된 경로 확인 |
| 현재 계정 또는 인증 수단 | 대상 도메인 자격 증명을 사용하지 않는 NULL·Anonymous·Kerberos 사용자 존재 요청 | 명령 옵션과 bind·KDC 응답 확인 | 저장된 세션·ticket 또는 유효 계정이 섞이지 않았는지 확인 |
| 현재 권한 | 명령 실행 호스트의 일반 사용자로 도구 실행 가능, 대상 반환 범위는 익명 정책에 따름 | 로컬 실행 성공과 원격 `ACCESS_DENIED`를 구분 | 로컬 권한 문제와 대상의 익명 열거 거부를 분리 |
| 공격 대상의 조건 | 정확한 `<DC>`, `<DOMAIN>`, Kerberos realm과 LDAP `<BASE_DN>` 식별 | DNS SRV, LDAP RootDSE, SMB·Kerberos 응답 교차 확인 | FQDN·IP, DNS 도메인·realm과 Base DN 형식을 재확인 |
| 필요한 목록 | Kerbrute 경로에만 한 줄당 하나의 사용자명 후보 파일 필요 | `users.txt` 형식과 후보 생성 근거 확인 | 후보 파일이 없으면 SMB·LDAP 반환 경로를 우선하고 무작위 요청을 만들지 않음 |

## 실행

`<DC>`는 선택한 SMB·LDAP·Kerberos 서비스의 DC FQDN 또는 IP, `<DOMAIN>`은 DNS 도메인, `<BASE_DN>`은 RootDSE에서 얻은 DN이다. `<USER_CANDIDATES>`는 Linux 실행 호스트의 한 줄당 사용자명 파일이며 `<USER_ENUM_LOG>`와 `<SPRAY_USER_LIST>`는 이 단계가 새로 만드는 exact 경로다. `<COUNT>`는 출력값이다.

### 열거 경로 선택

| 현재 도달 가능한 서비스와 보유 정보 | 선택할 방식 | 필요한 입력 | 성공 시 얻는 결과 |
|---|---|---|---|
| DC 445/TCP에 도달하고 SMB NULL Session 가능성을 확인할 수 있음 | SMB/RPC 익명 열거 | `<DC>` | 디렉터리가 익명 RPC에 반환한 사용자 객체 이름과 RID |
| DC 389/TCP에 도달하고 LDAP Anonymous Bind가 허용됨 | LDAP 익명 열거 | `<DC>`, `<BASE_DN>` | 익명 bind에 허용된 사용자 객체와 반환 속성 |
| DC 88/TCP에 도달하고 사용자명 후보 파일이 있음 | Kerbrute `userenum` | `<DOMAIN>`, `<DC>`, `users.txt` | KDC 응답 차이로 검증한 Kerberos principal 후보 |

SMB와 LDAP는 서버가 반환한 사용자 객체를 수집한다. Kerbrute는 이미 가진 사용자명 후보를 KDC 응답으로 검증하므로 후보 파일 없이 전체 사용자 목록을 만들어 주지 않는다. 한 경로의 거부를 다른 서비스의 거부로 확대 해석하지 않는다.

### 1. SMB NULL Session으로 도메인 사용자 열거

명령 실행 호스트에서 credential을 보내지 않고 DC의 SMB/RPC가 도메인 사용자 목록을 반환하는지 확인한다.

```bash
rpcclient -U "" -N <DC> -c 'enumdomusers'
```

확인할 출력:

- `user:[<USER>] rid:[<RID>]` 형식의 도메인 사용자.
- `NT_STATUS_ACCESS_DENIED`가 아닌 실제 사용자 목록 반환 여부.

보조 열거와 계정 상태 단서 확인:

```bash
enum4linux -U <DC>
crackmapexec smb <DC> --users
impacket-samrdump <DC>
```

확인할 출력:

- `Enumerated domain user(s)`와 도메인 사용자 이름.
- 제공되는 경우 `badpwdcount`, `baddpwdtime`, disabled·locked 상태.
- `impacket-samrdump`가 반환한 계정 이름, RID와 계정 설명. 연결 성공 뒤 사용자 항목이 없으면 익명 SAMR 조회가 제한된 상태인지 확인한다.

### 2. LDAP Anonymous Bind로 사용자 객체 조회

LDAP 익명 bind가 허용되면 식별한 Base DN 아래에서 반환되는 사용자 객체와 속성만 수집한다.

```bash
ldapsearch -x -H ldap://<DC> -b '<BASE_DN>' -s sub '(&(objectCategory=person)(objectClass=user))' sAMAccountName userPrincipalName userAccountControl
```

확인할 출력:

- `sAMAccountName`과 `userPrincipalName`으로 확인되는 계정 형식.
- `userAccountControl`과 객체 속성에서 식별 가능한 비활성 계정 단서.
- 컴퓨터 계정처럼 이름이 `$`로 끝나는 객체가 사용자 목록에 섞였는지 여부.
- bind 성공 뒤 결과가 비어 있으면 Base DN·검색 필터·익명 속성 공개 범위를 확인한다. 연결 성공이나 bind 성공만으로 사용자 객체 읽기 권한을 확정하지 않는다.

### 3. Kerbrute로 사용자명 후보 검증

SMB·LDAP에서 목록을 받지 못했거나 별도 후보가 있으면 KDC 응답 차이로 후보를 검증한다.

`<USER_ENUM_LOG>`와 `<SPRAY_USER_LIST>`는 Vault 밖의 작업 디렉터리에 새 경로로 정하고, 기존 파일이 없음을 먼저 확인한다.

```bash
test ! -e '<USER_ENUM_LOG>' && test ! -e '<SPRAY_USER_LIST>'
```

```bash
kerbrute userenum -d <DOMAIN> --dc <DC> '<USER_CANDIDATES>' -o '<USER_ENUM_LOG>'
```

확인할 출력:

- `[+] VALID USERNAME: <USER>@<DOMAIN>`.
- `Done! Tested <COUNT> usernames (<VALID_COUNT> valid)` 요약.
- KDC·realm·시간 오류와 실제 invalid 응답을 구분한다.

### 4. Spraying 입력 목록 정리

- Kerbrute 로그를 사용한다면 양성 행의 마지막 필드만 추출해 별도 목록으로 만들고, 결과를 직접 확인한다.

```bash
awk '/VALID USERNAME:/ {print $NF}' '<USER_ENUM_LOG>' | sort -u > '<SPRAY_USER_LIST>'
```

- 한 줄에 사용자명 하나만 남기고 중복을 제거한다.
- `$`로 끝나는 컴퓨터 계정, `krbtgt`, 비활성·잠긴 계정과 잠금 임계값에 가까운 계정은 제외한다.
- 서비스 계정과 고권한 계정은 잠금 영향과 계정 중요도를 별도로 확인한다.
- 원본 열거 결과와 정리한 대상 목록을 구분한다.

## 변경 영향과 로컬 파일 정리

- SMB·LDAP 조회는 디렉터리 객체를 변경하지 않는다. Kerbrute `userenum`도 비밀번호를 제출하지 않지만 DC에 TGT 요청과 이벤트 ID 4768 흔적을 남길 수 있으며, 이 원격 감사 기록은 클라이언트에서 되돌릴 수 없다.
- 이 절차가 새로 만든 파일은 정확히 기록한 `<USER_ENUM_LOG>`와 `<SPRAY_USER_LIST>`뿐이다. 사용 후 해당 경로만 삭제하고, 후보 원본이나 같은 이름의 다른 파일은 제거하지 않는다.

```bash
rm -- '<USER_ENUM_LOG>' '<SPRAY_USER_LIST>'
test ! -e '<USER_ENUM_LOG>' && test ! -e '<SPRAY_USER_LIST>'
```

- 삭제가 실패하면 먼저 파일 경로, 현재 사용자 소유권과 실행 중인 후속 도구가 파일을 열고 있는지 확인한다. 로그·목록에 실제 계정명이 포함되므로 Vault나 일반 메모에 복사하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| NULL Session에서 사용자 목록 반환 | SMB/RPC 익명 컨텍스트에 도메인 사용자 목록 조회가 허용됨 | 디렉터리가 반환한 AD 사용자 목록 | [[AD 비밀번호 정책 열거 및 조회]]로 잠금 정책과 계정 상태 확인 |
| LDAP Anonymous Bind에서 사용자 객체 반환 | 익명 bind에 허용된 범위에서 sAMAccountName·UPN과 상태 속성을 수집 가능 | 사용자명 형식과 제외 계정 후보 | 컴퓨터·비활성 계정을 제외하고 정책 확인 |
| Kerbrute `VALID USERNAME` | KDC가 해당 principal과 임의 후보를 다르게 처리함 | 검증된 사용자명 후보 | 잠금 정책과 인증 실패 이벤트 기록 여부를 확인한 뒤 [[내부 AD Password Spraying]] |
| Kerbrute에서 모든 후보가 invalid | 이름 규칙, 도메인·realm 또는 후보 목록이 부적합할 수 있음 | 유효 사용자 미확인 | UPN 형식, DC·시간과 사용자명 생성 규칙 재검토 |
| `badpwdcount`가 임계값에 가깝거나 locked·disabled 표시 | 추가 인증 시 계정 잠금 또는 운영 영향 위험 | Spraying 제외 계정 | 대상 목록에서 즉시 제외하고 해당 계정에 적용되는 잠금 정책과 현재 상태 확인 |
| 익명 SMB·LDAP 모두 거부 | 기본 익명 열거 차단 | 사용자 목록 미확인 | Kerbrute 후보 검증 또는 외부 사용자명 수집으로 전환 |
| timeout, 이름 해석 또는 realm 오류 | 사용자 존재 여부를 비교하기 전에 네트워크·대상 컨텍스트 확인 실패 | 사용자 상태 미확인 | DC FQDN·IP, DNS, 445·389·88/TCP 경로와 시간 동기화 확인 |
| bind·세션은 성립하지만 사용자 조회가 접근 거부 | 서비스 접근과 디렉터리 객체 조회 권한이 분리됨 | 익명 열거 범위 미확인 | 다른 익명 프로토콜 경로와 Kerberos 후보 검증을 분리해 시도 |

## 확인할 출력과 권한

- Kerbrute 사용자 열거는 비밀번호를 제출하지 않지만 다량의 TGT 요청이 이벤트 ID 4768로 관찰될 수 있다.
- 사용자 열거와 `passwordspray`는 다르다. 여기서는 `userenum`만 실행하며 실패한 비밀번호 시도를 만들지 않는다.
- SMB·LDAP의 디렉터리 결과를 우선하고, OSINT나 이름 규칙으로 만든 값은 Kerbrute에서 `VALID USERNAME`이 확인되기 전까지 후보로 유지한다.
- SMB/RPC 목록은 그 익명 조회에서 반환된 객체, LDAP 결과는 bind와 Base DN에 허용된 속성, Kerbrute 결과는 KDC 응답으로 구분된 principal만 확정한다.
- 사용자 존재 확인은 자격 증명, 비밀번호 유효성, 로그인 권한 또는 계정 활성 상태의 증명이 아니다. disabled·locked 단서도 현재 상태와 적용 정책을 별도 확인한다.

## 관련 서비스

- [[SMB 서비스]]
- [[RPC와 NetBIOS 서비스]]
- [[LDAP 서비스]]
- [[Kerberos 서비스]]

## 관련 도구

- [[rpcclient]]
- [[enum4linux]]
- [[enum4linux-ng]]
- [[crackmapexec]]
- [[kerbrute]]
- [[ldapsearch]]
- [[windapsearch]]
- [[impacket-samrdump]]

## 관련 상태 라우터

- [[무인증 내부 네트워크에서 AD 단서 확인]]
- [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 참고 링크

- [Kerbrute 공식 저장소 — Usage와 User Enumeration](https://github.com/ropnop/kerbrute)
