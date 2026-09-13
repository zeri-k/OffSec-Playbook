---
tags:
  - 기능/열거
실행환경: ["Linux", "Windows", "macOS"]
필요권한: ["Shodan API 계정과 현재 plan에서 허용된 조회 권한"]
필요조건: ["현재 관계를 확인할 public IP 또는 도메인", "API key가 별도 local profile에 구성된 Shodan CLI"]
결과: ["제3자 관측 시점의 IP·port·service banner 후보", "공개 도메인의 subdomain·DNS 후보"]
---

# shodan

## 도구 개요

`shodan` CLI는 Shodan이 이전에 관측한 Internet-facing host·service 자료를 IP 또는 domain 기준으로 조회한다. 직접 대상에 연결하지 않고 공개 노출 후보를 좁힐 때 사용하며, 결과의 port·banner·취약점 표시는 현재 대상 상태나 실제 취약성을 보장하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치와 경로: Shodan API에 HTTPS로 연결할 수 있는 분석 호스트
- 계정·권한: local profile에 API key가 이미 구성되어 있고 `host` 또는 `domain` 조회를 허용하는 현재 account plan
- 대상 입력: 현재 관계를 확인할 public `<IP>` 또는 `<DOMAIN>`; `<PUBLIC_IP>`는 DNS·등록 정보와 대조할 공개 IP(예: `198.51.100.20`), `<DOMAIN>`은 공개 FQDN(예: `www.example.test`)이다.
- 사전 확인: 현재 DNS 해석·등록 정보와 범위 자료로 IP·domain의 대상 관계를 확인

API key를 command line, shell history나 Vault에 넣지 않는다. `shodan init`은 local 설정 파일에 credential을 기록하므로 이 문서의 조회 절차에 포함하지 않는다.

## 표준 사용법

```bash
shodan host <PUBLIC_IP>
```

## 대표 예시

### 계정 plan과 남은 조회 범위 확인

```bash
shodan version
shodan info
```

확인할 출력:

- 설치된 CLI version과 API plan·표시되는 사용 가능 기능 및 credit 정보.
- 인증·plan 오류는 대상 정보 부재가 아니며, credential을 다시 명령행에 입력하지 않고 local profile·계정 권한을 확인한다.

### 현재 DNS에서 확인한 public IP 조회

```bash
shodan host <PUBLIC_IP>
```

확인할 출력:

- IP, organization·ASN 후보, 마지막 관측 시각과 관측된 port·service banner.
- 출력은 Shodan의 관측 시점과 scan 범위에 한정된다. 현재 port open, 제품 version 또는 대상 조직 소유를 확정하지 않는다.

### 공개 domain 후보 조회

현재 CLI help에 `domain` subcommand가 있고 account plan에서 허용될 때 사용한다.

```bash
shodan domain <DOMAIN>
```

확인할 출력:

- Shodan이 연결한 subdomain·DNS 정보 후보.
- wildcard, CDN·공유 hosting과 과거 레코드가 포함될 수 있으므로 현재 권한 DNS와 개별 host 응답으로 재검증한다.

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Last Update` 또는 관측 시각과 port·banner | 해당 시점에 Shodan이 수집한 서비스 후보 | 현재 DNS 관계와 직접 서비스 확인 |
| organization·ASN | IP 등록·관측 metadata 후보 | WHOIS·RIR의 현재 등록 정보와 교차 확인 |
| 결과 없음 | Shodan dataset에 현재 표시할 관측 자료가 없음 | 현재 서비스 부재로 단정하지 말고 DNS·직접 확인 |
| 인증·membership·credit 오류 | API profile 또는 account plan 조건 미충족 | local profile·plan·quota 확인 후 이 분기 생략 |

## 버전과 환경 차이

subcommand와 출력 필드는 CLI·API version 및 account plan에 따라 달라질 수 있다. 실행 전 `shodan version`, `shodan info`, `shodan host -h` 또는 `shodan domain -h`로 설치본의 지원 범위를 확인한다. `scan submit`은 제3자 조회가 아니라 Shodan에 새 능동 scan을 요청하고 scan credit을 사용하므로 이 대표 절차에 포함하지 않는다.

## 관련 공격기법

- [[DNS 열거와 Zone Transfer]]
- [[웹 정찰과 경로 열거]]

## 참고 링크

- [Shodan CLI 설치와 API profile](https://help.shodan.io/command-line-interface/0-installation)
- [Shodan CLI 시작과 출력 필드](https://help.shodan.io/command-line-interface/1-getting-started)
- [Shodan IP 정보 조회](https://help.shodan.io/developer-fundamentals/looking-up-ip-info)
- [Shodan domain 조회 안내](https://help.shodan.io/command-line-interface/4-network-monitoring)
