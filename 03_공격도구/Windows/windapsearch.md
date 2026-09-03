---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/열거
실행환경: ["Linux", "Python 3"]
필요권한: ["LDAP Anonymous Bind 허용 또는 유효한 도메인 자격 증명"]
필요조건: ["DC LDAP 접근", "선택적으로 도메인 사용자와 비밀번호"]
결과: ["AD 사용자", "그룹 및 컴퓨터 정보", "고권한 사용자 목록"]
---

# windapsearch

## 도구 개요

`windapsearch`는 Python으로 LDAP에 bind해 AD 사용자·그룹·컴퓨터와 중첩된 고권한 그룹 구성원을 열거한다. Linux에서 Anonymous Bind 노출을 확인하거나 인증된 LDAP 세션으로 사용자·권한 후보를 빠르게 정리할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: `windapsearch.py`와 Python 3를 실행할 수 있고 DC의 LDAP에 접근 가능한 Linux 호스트
- 입력: DC IP 또는 도메인 FQDN, 선택적인 bind 사용자와 비밀번호, 열거 유형
- 인증 조건: `-u ""`로 Anonymous Bind를 시도할 수 있으며 서버가 차단하면 유효한 도메인 자격 증명이 필요

## 표준 사용법

```bash
python3 windapsearch.py --dc-ip <DC_IP> [-u '<USER>@<DOMAIN>' -p '<PASSWORD>'] <ENUMERATION_OPTION>
```

## 대표 예시

### Anonymous Bind로 전체 AD 사용자 열거

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u "" -U
```

확인할 출력:

- `No username provided. Will try anonymous bind.`
- `success! Binded as: None`
- `Enumerating all AD users`, `Found <COUNT> users`

### 인증 후 Domain Admins 구성원 열거

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' --da
```

확인할 출력:

- `success! Binded as: <DOMAIN>\<USER>`
- `Attempting to enumerate all Domain Admins`
- `Found <COUNT> Domain Admins`

### 중첩 그룹을 포함한 고권한 사용자 열거

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -PU
```

확인할 출력:

- `Attempting to enumerate all AD privileged users`
- 그룹별 `Found <COUNT> nested users`
- 각 객체의 `cn`, `userPrincipalName`

### 인증 후 AD 컴퓨터 열거

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -C
```

확인할 출력:

- 컴퓨터 객체의 `cn`, `distinguishedName`, DNS hostname 등 반환된 속성.
- LDAP bind 성공 뒤 실제 컴퓨터 객체가 반환됐는지 확인한다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `--dc-ip <DC_IP>` | 쿼리할 Domain Controller 지정 | DNS를 사용하지 않고 DC를 직접 선택할 때 |
| `-u`, `--user` | UPN 또는 `DOMAIN\user` 형식의 bind 사용자 | 인증 LDAP 열거 또는 빈 문자열로 Anonymous Bind 시도 |
| `-p`, `--password` | bind 비밀번호 | 인증 사용자로 LDAP bind할 때 |
| `-U`, `--users` | 모든 AD 사용자 열거 | 사용자 목록 생성 |
| `-C`, `--computers` | 모든 AD 컴퓨터 열거 | 후속 DNS·서비스 확인에 사용할 호스트 후보 생성 |
| `--da` | Domain Admins 구성원 열거 | 직접적인 고권한 그룹 구성원 확인 |
| `-PU`, `--privileged-users` | 중첩 그룹을 재귀 조회해 고권한 사용자 열거 | 간접 그룹 구성원까지 포함한 권한 후보 확인 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Getting defaultNamingContext from Root DSE` | Base DN 자동 식별 시도 | 이어지는 `Found: <BASE_DN>` 확인 |
| `success! Binded as: None` | Anonymous Bind 성공 | `-U` 등 익명 조회 가능한 객체 확인 |
| `success! Binded as: <DOMAIN>\<USER>` | 제공한 계정으로 LDAP bind 성공 | 그룹·사용자 열거 결과 분석 |
| `Found <COUNT> users` | AD 사용자 열거 성공 | 컴퓨터 계정과 비활성 계정을 분리 |
| `Found <COUNT> nested users` | 중첩 그룹을 포함한 고권한 사용자 발견 | 실제 그룹 경로와 현재 계정 제어 여부 확인 |
| bind 실패 | Anonymous Bind 차단 또는 자격 증명 불일치 | 사용자 형식, 비밀번호, LDAP 접근성 확인 |

## 관련 공격기법

- [[인증 전 AD 사용자 목록 수집]]
- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
