---
tags:
  - 환경/ad
  - 기능/열거
실행환경: ["Linux"]
필요권한: ["현재 사용 중인 계정의 AD 객체·관계 읽기 권한"]
필요조건: ["도메인 credential 또는 NTLM hash", "DC·DNS 접근"]
결과: ["BloodHound JSON 또는 ZIP 수집 결과"]
---

# bloodhound-python

## 도구 개요

`bloodhound-python`은 Linux에서 AD 객체·ACL·세션 관계를 수집해 BloodHound가 읽는 JSON 또는 ZIP 파일을 만든다. 도메인에 가입되지 않은 Linux 호스트에서 수집 범위를 조절하며 그래프 분석 자료를 준비할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: 대상 DC와 DNS에 접근 가능한 Linux 호스트
- 필요한 입력: 도메인명, 사용자와 비밀번호 또는 NTLM hash, DNS 서버/DC
- 권한 조건: 일반 도메인 사용자의 읽기 권한으로 시작하며 수집 방법별 원격 접근 제한을 구분한다.

## 표준 사용법

```bash
bloodhound-python -u <USER> -p '<PASSWORD>' -d <DOMAIN> -ns <DC_IP> -c <COLLECTION_METHOD>
```

## 대표 예시

### 전체 수집

```bash
bloodhound-python -u <USER> -p '<PASSWORD>' -d <DOMAIN> -ns <DC_IP> -c all --zip
```

확인할 출력:

- 발견한 domain, computer, user, group, trust 수와 생성된 ZIP/JSON 파일.

### DC 중심 수집으로 범위 축소

```bash
bloodhound-python -u <USER> -p '<PASSWORD>' -d <DOMAIN> -ns <DC_IP> -c DCOnly --zip
```

확인할 출력:

- LDAP 기반 객체·그룹·ACL·trust 수집 결과와 컴퓨터 접속 생략 여부.

### NTLM hash로 인증

```bash
bloodhound-python -u <USER> --hashes ':<NTLM_HASH>' -d <DOMAIN> -ns <DC_IP> -c DCOnly --zip
```

### 신뢰 관계가 있는 두 도메인 수집

```bash
bloodhound-python -d <SOURCE_DOMAIN> -dc <SOURCE_DC_FQDN> -c All -u <USER> -p '<PASSWORD>'
bloodhound-python -d <TARGET_TRUST_DOMAIN> -dc <TARGET_DC_FQDN> -c All -u <USER>@<SOURCE_DOMAIN> -p '<PASSWORD>'
zip -r trusted-forest-bloodhound.zip *.json
```

확인할 출력:

- 각 대상의 domain·forest·computer·user·group·trust 수와 생성된 JSON 파일.
- 대상 DC FQDN이 해석되지 않으면 실행 호스트가 사용하는 DNS 서버와 도메인 검색 suffix를 먼저 확인한다.
- 양쪽 데이터를 함께 분석해야 외부 도메인 그룹 멤버십을 한 그래프에서 확인할 수 있다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-u`, `-p` | 사용자와 비밀번호 | 평문 credential 사용 |
| `--hashes` | LM:NTLM hash | NTLM hash로 인증 |
| `-d` | 대상 AD domain | 조회 기준 지정 |
| `-ns` | DNS server | DC 이름 해석 고정 |
| `-dc` | 수집에 사용할 DC의 FQDN | 특정 현재·신뢰 대상 도메인의 DC 지정 |
| `-c` | 수집 방법 | `all`, `DCOnly`, `Session`, `ACL` 등 범위 조절 |
| `--zip` | 결과 압축 | BloodHound 업로드 단순화 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Found ... users/groups/computers` | LDAP 객체 수집 성공 | 생성 파일과 수집 범위 확인 |
| trust·ACL·session 파일 생성 | 관계 분석 자료 확보 | [[BloodHound]]에 업로드 |
| DNS timeout·DC 해석 실패 | 이름 해석 또는 도달성 문제 | `-ns`, 도메인 FQDN, 시간과 DC 접근 확인 |
| 일부 컴퓨터 수집 실패 | 방화벽·권한·호스트 오프라인 가능 | DCOnly 결과와 호스트별 도달성 비교 |

## 관련 공격기법

- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[AD 도메인 트러스트 열거와 공격 경로 식별]]
