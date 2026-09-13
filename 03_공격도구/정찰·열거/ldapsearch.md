---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/열거
실행환경: ["Linux"]
필요권한: ["LDAP Anonymous Bind 허용 또는 유효한 bind 자격 증명"]
필요조건: ["LDAP에 접근 가능한 DC", "Base DN", "LDAP 검색 필터"]
결과: ["AD 객체 정보", "사용자 목록", "비밀번호 정책"]
---

# ldapsearch

## 도구 개요

`ldapsearch`는 LDAP 서버에 bind하여 Base DN과 filter에 맞는 디렉터리 객체와 속성을 조회하는 명령줄 클라이언트다. AD에서 사용자 계정·비밀번호 정책처럼 필요한 속성을 정확한 검색 범위로 추출할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: DC의 LDAP에 접근 가능하고 OpenLDAP client가 설치된 Linux 호스트
- 입력: DC 주소, Base DN, 검색 scope, LDAP filter와 필요한 attribute
- 인증 조건: 서버가 허용하면 `-x` simple anonymous bind를 사용할 수 있으며, 그렇지 않으면 유효한 bind 자격 증명이 필요

## 표준 사용법

`<DC_IP>`는 LDAP를 제공하는 DC 주소(예: `directory.example.test`)이며, `<BASE_DN>`은 RootDSE에서 얻은 DN(예: `DC=corp,DC=example,DC=test`)이다. `<SCOPE>`는 `base`·`one`·`sub`, `<LDAP_FILTER>`는 RFC 4515 filter, `[ATTRIBUTES]`는 반환 속성 목록이다. 명령은 LDAP에 도달 가능한 Linux 호스트에서 실행한다.

```bash
ldapsearch -H ldap://<DC_IP> -x -b "<BASE_DN>" -s <SCOPE> "<LDAP_FILTER>" [ATTRIBUTES]
```

## 대표 예시

### Anonymous Bind로 비밀번호 정책 조회

```bash
ldapsearch -H ldap://<DC_IP> -x -b "<BASE_DN>" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

확인할 출력:

- `lockoutDuration`, `lockOutObservationWindow`, `lockoutThreshold`
- `minPwdLength`, `pwdProperties`, `pwdHistoryLength`

### Anonymous Bind로 사용자 이름 수집

```bash
ldapsearch -H ldap://<DC_IP> -x -b "<BASE_DN>" -s sub "(&(objectclass=user))" | grep 'sAMAccountName:' | cut -f2 -d' '
```

확인할 출력:

- 사용자와 컴퓨터 계정의 `sAMAccountName`
- Password Spraying 대상 목록에서 제외해야 할 `$` 접미사 컴퓨터 계정

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-H ldap://<HOST>` | LDAP URI 지정 | 최신 ldapsearch에서 DC와 protocol 지정 |
| `-x` | SASL 대신 simple authentication 사용 | Anonymous Bind 또는 simple bind 쿼리 |
| `-b "<BASE_DN>"` | 검색 시작 DN 지정 | 도메인 또는 특정 OU 범위 제한 |
| `-s sub` | Base DN 아래 subtree 전체 검색 | 사용자·정책 등 하위 객체 열거 |
| `"<LDAP_FILTER>"` | 반환할 객체 조건 지정 | 사용자 객체나 특정 attribute 보유 객체 제한 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `lockoutThreshold` | 계정 잠금 전 실패 횟수 | Password Spraying 횟수와 간격 결정 |
| `lockOutObservationWindow` | 실패 횟수 관찰 window | 다음 시도 전 대기 시간 계산 |
| `minPwdLength`, `pwdProperties` | 최소 길이와 복잡성 설정 | 후보 비밀번호가 정책을 만족하는지 확인 |
| `sAMAccountName` | AD 사용자 또는 컴퓨터 계정 이름 | 사용자 계정만 정제하고 중복 제거 |
| `result: 0 Success` 또는 객체 반환 | bind와 LDAP 검색 성공 | 필요한 attribute와 후속 filter 확장 |
| `Invalid credentials` / bind 오류 | Anonymous Bind 차단 또는 자격 증명 불일치 | bind 방식과 인증 정보 확인 |
| `No such object` | Base DN이 잘못되었거나 검색 범위 밖 | RootDSE의 naming context와 Base DN 확인 |

## 버전과 환경 차이

- 이전 버전에서 사용하던 `-h <HOST>`는 최신 ldapsearch에서 폐기되었으므로 `-H ldap://<HOST>` 형식을 우선한다.
- LDAPS를 사용해야 하면 URI scheme과 포트를 환경에 맞게 변경하고 certificate 검증 조건을 별도로 확인한다.

## 관련 공격기법

- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 전 AD 사용자 목록 수집]]
