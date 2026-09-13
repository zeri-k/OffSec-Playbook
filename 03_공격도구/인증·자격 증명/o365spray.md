---
tags:
  - 환경/cloud
  - 서비스/microsoft365
  - 기능/열거
  - 기능/인증검증
실행환경: ["Linux"]
필요조건: ["Python 3 실행 환경", "대상 도메인", "열거 또는 spraying 시 사용자 목록", "spraying 시 비밀번호 후보와 잠금 정책"]
결과: ["유효 계정 후보", "유효 자격증명"]
---

# o365spray

## 도구 개요

`o365spray`는 Microsoft 365 도메인 검증, 사용자 열거, Password Spraying을 단계별로 수행하는 도구다. 같은 도구 안에서 tenant 확인부터 유효 계정 후보 축소와 단일 비밀번호 검증까지 이어서 다룰 때 유용하다.

## 필요한 입력과 실행 환경

`<DOMAIN>`은 tenant의 DNS 도메인(예: `example.onmicrosoft.com` 또는 `corp.example.test`)이고, `users.txt`·`usersfound.txt`는 UPN 한 줄씩인 Linux 호스트 파일이다. `<PASSWORD>`는 한 round의 단일 평문 값, `<ATTEMPTS_PER_WINDOW>`와 `<RESET_MINUTES>`는 관찰한 정책의 양의 정수, `<O365_OUTPUT_DIR>`은 새 로컬 출력 디렉터리, `<O365_RESULT_FILE>`은 그 안에서 도구가 만든 정확한 결과 파일이다. 평문 credential 결과는 Vault에 저장하지 않는다.

- 실행 환경: Python 3와 도구 의존성이 설치된 Linux 호스트
- 공통 입력: Microsoft 365 도메인
- 열거 입력: UPN 또는 이메일 형식의 사용자 목록
- spraying 입력: 유효 사용자 목록, 비밀번호 후보, 잠금 정책의 시도 횟수와 reset window

## 표준 사용법

```bash
python3 o365spray.py --validate --domain <DOMAIN>
python3 o365spray.py --enum --enum-module office -U users.txt --domain <DOMAIN> --output '<O365_OUTPUT_DIR>'
python3 o365spray.py --spray -U usersfound.txt -p '<PASSWORD>' --count <ATTEMPTS_PER_WINDOW> --lockout <RESET_MINUTES> --domain <DOMAIN> --output '<O365_OUTPUT_DIR>'
```

검증, 사용자 열거, spraying 순서로 진행하고 `--count`와 `--lockout`을 확인한 잠금 정책에 맞춘다.

## 대표 예시

### Microsoft 365 도메인 검증

```bash
python3 o365spray.py --validate --domain <DOMAIN>
```

확인할 출력:

- `[VALID] The following domain is using O365`.

### 사용자 열거

```bash
python3 o365spray.py --version
python3 o365spray.py --enum --enum-module office -U users.txt --domain <DOMAIN> --output '<O365_OUTPUT_DIR>'
```

확인할 출력:

- `[VALID] user@<DOMAIN>`.
- `enum_valid_accounts` 결과 파일.
- `office`가 현재 설치 버전에서 비밀번호 인증을 제출하지 않는 모듈인지 `--help`와 upstream module 설명으로 확인한다. 인증 시도형 열거 모듈은 별도 잠금 영향에 포함한다.

### Password Spraying

```bash
python3 o365spray.py --spray -U usersfound.txt -p '<PASSWORD>' --count <ATTEMPTS_PER_WINDOW> --lockout <RESET_MINUTES> --domain <DOMAIN> --output '<O365_OUTPUT_DIR>'
```

확인할 출력:

- `[VALID] user@<DOMAIN>:<PASSWORD>`.
- `spray_valid_credentials` 결과 파일.
- `<ATTEMPTS_PER_WINDOW>`와 `<RESET_MINUTES>`가 각각 실제 잠금 정책의 window당 시도 수와 reset 시간(분)에 맞는지 확인한다.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `--validate` | 대상 도메인의 O365 사용 여부 확인 | spraying 전 대상 서비스 확인 |
| `--domain` | 대상 도메인 지정 | 모든 모드 |
| `--enum` | 사용자 열거 모드 | 유효 계정 후보 축소 |
| `-U` | 사용자 목록 파일 | enum/spray 대상 지정 |
| `--spray` | Password Spraying 모드 | 단일 또는 제한된 비밀번호 검증 |
| `-p` | 단일 비밀번호 지정 | lockout 위험을 낮춘 spraying |
| `--count` | 한 lockout window에서 사용할 비밀번호 수 | 잠금 임계값에 맞춘 시도 수 지정 |
| `--lockout` | 잠금 정책의 reset 시간(분) | spray batch 사이 대기 시간 반영 |
| `--enum-module office` | GetCredentialType 기반 사용자 후보 판정 모듈 | 비밀번호 인증을 제출하지 않는 열거 경로를 명시할 때. 지원 여부·응답 방식은 현재 버전 확인 |
| `--output` | 결과 파일을 저장할 디렉터리 | 계정·credential 결과를 Vault 밖의 전용 경로로 분리 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| O365 validation `VALID` | Microsoft 365 사용 가능성 높음 | 사용자 열거 또는 spraying 준비 |
| enum `VALID` | 유효 사용자 후보 | 중복을 제거해 spraying 입력 목록으로 사용 |
| spray `VALID user:password` | credential 유효 가능성 | MFA/Conditional Access와 실제 접근 권한 분리 확인 |
| validation 실패 | O365 미사용, 탐지 방식 변경 | MX 레코드, 로그인 포털, 제공자 확인 |
| enum 오류 | 사용자 형식 또는 모듈 문제 | 전체 이메일 주소, UPN, 도구 버전 확인 |
| spray 차단 | rate limit, 잠금 정책, Conditional Access | 추가 시도를 멈추고 응답 코드와 정책 조건 확인 |
| 도구가 동작하지 않음 | Microsoft 응답 변경 또는 도구 노후화 | 최신 버전과 모듈 상태 확인 |

실행 전 `test ! -e '<O365_OUTPUT_DIR>'`와 `install -d -m 700 '<O365_OUTPUT_DIR>'`로 전용 경로를 만든다. 도구가 출력한 tested·valid 파일의 정확한 경로를 기록하고, 평문 credential 결과를 Vault에 저장하지 않는다. 폐기 시 기록한 파일만 `rm -- '<O365_RESULT_FILE>'`로 제거하고 빈 디렉터리만 `rmdir -- '<O365_OUTPUT_DIR>'`로 정리한다. 파일명 패턴으로 결과 디렉터리 전체를 삭제하지 않는다.

## 관련 공격기법

- [[Microsoft 365 사용자 열거]], [[Microsoft 365 Password Spraying]]

## 참고 링크

- [o365spray 공식 저장소와 CLI·모듈 설명](https://github.com/0xZDH/o365spray)
