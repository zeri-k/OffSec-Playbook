---
tags:
  - 환경/ad
  - 기능/열거
실행환경: ["Windows CMD", "Windows PowerShell"]
필요권한: ["현재 Windows 세션에서 대상 도메인 정보를 조회할 수 있는 계정"]
필요조건: ["netdom 실행 파일", "대상 도메인 이름", "대상 도메인 서비스 접근"]
결과: ["직접 도메인 트러스트", "도메인 컨트롤러 목록", "워크스테이션·서버 계정 목록"]
---

# netdom

## 도구 개요

`netdom`은 Windows에서 도메인의 직접 trust, Domain Controller(DC), 워크스테이션·서버 계정 목록을 조회하는 관리 명령이다. 추가 스크립트 없이 현재 도메인의 기본 구조와 직접 신뢰 관계를 빠르게 파악할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: `netdom.exe`를 사용할 수 있고 대상 도메인에 연결할 수 있는 Windows CMD 또는 PowerShell.
- 입력: 조회할 도메인의 FQDN.
- 확인 범위: `trust`는 직접 신뢰 관계, `dc`는 DC 계정, `workstation`은 워크스테이션과 서버 계정 목록을 반환한다.

## 표준 사용법

```cmd
netdom query /domain:<DOMAIN_FQDN> <QUERY_TYPE>
```

## 대표 예시

### 직접 도메인 트러스트 조회

```cmd
netdom query /domain:<DOMAIN_FQDN> trust
```

확인할 출력:

- `Trusted\Trusting domain`에 표시된 상대 도메인.
- `<->` 같은 방향 표시와 `Direct` 관계.

### Domain Controller 조회

```cmd
netdom query /domain:<DOMAIN_FQDN> dc
```

확인할 출력:

- `List of domain controllers with accounts in the domain` 아래의 DC 이름.

### 워크스테이션과 서버 조회

```cmd
netdom query /domain:<DOMAIN_FQDN> workstation
```

확인할 출력:

- 워크스테이션 이름과 `( Workstation or Server )`로 표시된 컴퓨터 계정.

## 주요 옵션

| 옵션·값 | 의미 | 사용하는 상황 |
|---|---|---|
| `query` | 지정한 도메인의 객체 범주 조회 | trust·DC·컴퓨터 목록을 확인할 때 |
| `/domain:<DOMAIN_FQDN>` | 조회 대상 도메인 지정 | 현재 도메인이 아닌 도메인을 명시할 때 |
| `trust` | 직접 신뢰 관계 목록 반환 | 상대 도메인과 방향 단서 확인 |
| `dc` | DC 계정 목록 반환 | 대상 도메인의 DC 이름 확인 |
| `workstation` | 워크스테이션·서버 계정 목록 반환 | 대상 컴퓨터 이름 후보 수집 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `The command completed successfully.` | 요청한 목록 조회 완료 | 반환된 이름을 DNS와 서비스 조회로 확인 |
| trust 상대 도메인과 방향 표시 | 직접 trust 후보 확인 | [[AD 도메인 트러스트 열거와 공격 경로 식별]]에서 유형·전이성·인증 방향 확인 |
| DC 이름 목록 | 도메인의 DC 계정 확인 | FQDN·IP와 LDAP·Kerberos·SMB 도달성 확인 |
| 워크스테이션·서버 이름 목록 | AD 컴퓨터 계정 후보 확인 | DNS·호스트 활성 상태를 별도로 확인 |
| 빈 목록 또는 오류 | 조회 권한·도메인 이름·연결 문제 또는 객체 부재 가능 | 현재 계정, `/domain` 값, DNS와 대상 도메인 접근 확인 |

## 관련 공격기법

- [[AD 도메인 트러스트 열거와 공격 경로 식별]]

## 참고 링크

- [Microsoft: netdom query](https://learn.microsoft.com/windows-server/administration/windows-commands/netdom-query)
