---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["유효한 도메인 credential", "DC LDAP 접근", "쓰기 가능한 출력 디렉터리"]
결과: ["AD DNS 레코드", "records.csv", "내부 호스트명과 IP"]
---

# adidnsdump

## 도구 개요

`adidnsdump`는 LDAP을 통해 AD 통합 DNS 객체를 읽고 이름, record type, 값을 `records.csv`로 정리하는 도구다. 일반 DNS 질의에서 드러나지 않는 AD DNS 레코드를 인증된 계정의 읽기 범위 안에서 모으고, 필요하면 알 수 없는 값을 DNS로 해석할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: DC LDAP와 DNS에 접근 가능한 Linux 호스트
- 필요한 입력: `<DOMAIN>\<USER>`, password, `ldap://<DC>` 또는 `ldaps://<DC>` URL
- 출력: 현재 작업 디렉터리에 생성되는 `records.csv`
- `-r` 사용 조건: DC 또는 지정 DNS를 통해 알 수 없는 레코드를 resolve할 수 있어야 한다.

## 표준 사용법

```bash
adidnsdump -u '<DOMAIN>\<USER>' 'ldap://<DC>' [options]
```

## 대표 예시

### AD DNS zone dump

```bash
adidnsdump -u '<DOMAIN>\<USER>' 'ldap://<DC>'
head records.csv
```

확인할 출력:

- `Bind OK`, `Found <N> records`.
- `records.csv`의 `type,name,value`.

### 알 수 없는 레코드 resolve

```bash
adidnsdump -u '<DOMAIN>\<USER>' 'ldap://<DC>' -r
rg '^A,' records.csv
```

확인할 출력:

- 이전에 `?`였던 이름과 해석된 A record.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-u` | 도메인과 사용자 지정 | 인증된 LDAP bind |
| `ldap://<DC>` | LDAP 대상 URL | AD DNS zone 조회 |
| `ldaps://<DC>` | LDAPS 대상 URL | TLS가 필요한 환경 |
| `-r` | 알 수 없는 레코드 resolve 시도 | `?` 값을 IP로 보강 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Bind OK` | LDAP 인증 성공 | zone query와 레코드 수 확인 |
| `Found <N> records` | DNS 객체 열거 성공 | `records.csv` 내용 검토 |
| `?,<NAME>,?` | 레코드 값 미해석 | `-r`, DNS 경로와 이름 해석 확인 |
| `A,<NAME>,<IP>` | 이름과 IPv4 대응 확인 | 호스트 활성 상태와 서비스 직접 확인 |
| bind 오류 | LDAP 계정 인증, 도메인 형식 또는 연결 문제 | 사용자·도메인 표기, LDAP URL, TLS와 DC 389·636번 연결 확인 |

## 관련 공격기법

- [[AD DNS 레코드 열거]]
