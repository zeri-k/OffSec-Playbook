---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/자격증명수집
실행환경: ["Linux"]
필요조건: ["도메인명", "사용자 목록"]
결과: ["Kerberos hash"]
---

# impacket-GetNPUsers

## 도구 개요

`impacket-GetNPUsers`는 Kerberos 사전 인증이 비활성화된 AD 사용자를 식별하고 AS-REP hash를 요청한다. 사용자 목록을 이용한 비인증 확인과 도메인 계정을 이용한 조회를 모두 지원해 AS-REP Roasting 자료를 수집할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: DC의 Kerberos 88번에 접근 가능한 Linux 호스트. 인증된 도메인 전체 조회 방식은 LDAP 389/636 경로도 필요하다.
- 필요한 입력: 비인증 방식은 도메인명·DC 주소·사용자 목록, LDAP 조회 방식은 도메인 credential 또는 Kerberos cache
- 출력 조건: 후속 cracker에 맞춰 `-format hashcat` 또는 John 형식을 선택한다.
- 비인증 목록 파일은 Linux 실행 host의 한 줄 한 사용자 파일이고 `<DC_IP>`는 Kerberos endpoint다. LDAP credential을 쓰는 경우 목록에 없는 domain objects만 추가로 발견할 수 있으며 hash 출력은 인증 성공이 아니다.


## 표준 사용법

`<DOMAIN>`·`<DC_IP>`는 Kerberos domain/KDC address, `<USER_LIST>`은 Linux host의 one-user-per-line input, `<OUTPUT_FILE>`은 새 hash output path다. credential block의 requester와 unauthenticated list block을 혼동하지 않으며 AS-REP hash output은 password recovery나 AD session이 아니다.

```bash
impacket-GetNPUsers '<DOMAIN>/' -usersfile '<USER_LIST>' -dc-ip <DC_IP> -no-pass
```

## 대표 예시

### 사용자 목록으로 AS-REP hash 수집

```bash
impacket-GetNPUsers '<DOMAIN>/' -usersfile '<USER_LIST>' -dc-ip <DC_IP> -no-pass -format hashcat -outputfile '<ASREP_HASH_FILE>'
```

### credential로 roastable 계정 요청

```bash
impacket-GetNPUsers '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -request -format hashcat -outputfile '<ASREP_HASH_FILE>'
```

- 첫 방식은 `-usersfile`의 각 principal에 AS-REQ를 보내므로 도메인 credential이 필요 없다.
- 둘째 방식은 제공한 credential로 LDAP에서 `UF_DONT_REQUIRE_PREAUTH`가 설정된 활성 사용자를 조회한 뒤 `-request`로 AS-REP hash를 요청한다.
- `-outputfile`이 생성하는 파일의 기준선·정리는 [[AS-REP Roasting]]에서 수행한다.

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `-usersfile` | 확인할 사용자 목록 |
| `-no-pass` | 비밀번호 없이 AS-REQ 시도 |
| `-request` | roastable 계정의 AS-REP hash 요청 |
| `-format hashcat` | Hashcat 형식으로 출력 |
| `-outputfile` | 결과 파일 저장 |
| `-dc-ip` | 도메인 컨트롤러 IP 지정 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `$krb5asrep$` hash | AS-REP Roasting 가능 | Hashcat/John으로 cracking |
| `KDC_ERR_PREAUTH_REQUIRED` | pre-auth가 켜진 정상 계정 | Kerberoasting 또는 password spraying 검토 |
| user unknown | 사용자 목록 부정확 | kerbrute/LDAP/SMB로 사용자 후보 재정리 |
| KDC/realm 오류 | 도메인/DC/시간 문제 | realm, DC IP, DNS, 시간 확인 |
| hash 없음 | roastable 계정 없음 | LDAP에서 `DONT_REQ_PREAUTH` 여부 확인 |
| 출력 형식 오류 | cracking mode 불일치 | `-format hashcat`, hash prefix, 도구 버전 확인 |

## 관련 공격기법

- [[AS-REP Roasting]]

## 참고 링크

- [Impacket GetNPUsers](https://github.com/fortra/impacket/blob/master/examples/GetNPUsers.py)
