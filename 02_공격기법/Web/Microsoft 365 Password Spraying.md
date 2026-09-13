---
tags:
  - 환경/cloud
  - 서비스/microsoft365
시작조건: ["Microsoft 365 유효 사용자 후보 목록 확보", "명령 실행 호스트에서 Microsoft 365 인증 endpoint HTTPS 접근 가능"]
필요조건: ["대상 도메인", "유효 사용자 후보 목록", "단일 비밀번호 후보", "잠금 정책과 시도 간격"]
결과: ["평문 비밀번호가 일치하는 Microsoft 365 계정 후보", "MFA·Conditional Access를 포함한 서비스별 접근 검증 필요 상태"]
---

# Microsoft 365 Password Spraying

## 한 줄 판단

Microsoft 365 유효 사용자 후보와 단일 비밀번호 후보가 있고 인증 endpoint에 접근할 수 있으면 계정별 시도 횟수와 간격을 제한해 비밀번호 일치 후보를 확인한다.

## 시작 조건 해석


- 현재 보유 정보: 대상 도메인, [[Microsoft 365 사용자 열거]]에서 확인한 이메일·UPN 후보 목록과 한 번에 검증할 단일 평문 비밀번호 후보.
- 명령 실행 위치: Microsoft 365 로그인·API HTTPS endpoint에 접근 가능한 공격 호스트. 내부 SMTP·IMAP·POP3 도달성은 전제가 아니다.
- 현재 가능한 행동과 결과: 각 사용자에게 같은 비밀번호를 한 번씩 시도해 `VALID user:password` 후보를 얻을 수 있지만 Exchange·SharePoint·Teams·VPN 권한이나 MFA 통과는 별도 확인 대상이다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 HTTPS 경로 | 공격 호스트에서 Microsoft 365 인증 endpoint 접근 가능 | `--validate` 응답과 프록시·DNS 확인 | 인터넷 egress, 프록시와 도메인 해석 확인 |
| 대상 도메인 | 대상 도메인이 Microsoft 365를 사용함 | `python3 o365spray.py --validate --domain <DOMAIN>` | MX 레코드, 로그인 포털과 다른 메일 제공자 확인 |
| 사용자 목록 | Microsoft 365 유효 사용자 후보가 이메일·UPN 형식으로 정리됨 | [[Microsoft 365 사용자 열거]] 결과와 파일 형식 확인 | 사용자명 형식과 열거 결과 재검토 |
| 비밀번호 후보 | 단일 평문 후보를 제한 횟수로 검증 가능 | 실습 단서와 계정 정책 확인 | 후보 수를 늘리기 전에 잠금 위험 재검토 |
| 잠금·접근 정책 | 잠금 임계치, 시도 간격, MFA·Conditional Access 영향을 판단 가능 | 정책 정보와 인증 응답 확인 | 중지 기준과 다음 시도 시점 확인 |

## 실행

### Microsoft 365 사용 여부 확인

```bash
python3 o365spray.py --validate --domain <DOMAIN>
```

확인할 출력:

- `[VALID] The following domain is using O365`.

### 단일 비밀번호 Password Spraying

```bash
python3 o365spray.py --version
test ! -e '<O365_OUTPUT_DIR>'
install -d -m 700 '<O365_OUTPUT_DIR>'
python3 o365spray.py --spray -U usersfound.txt -p '<PASSWORD>' --count <ATTEMPTS_PER_WINDOW> --lockout <RESET_MINUTES> --domain <DOMAIN> --output '<O365_OUTPUT_DIR>'
```

`<DOMAIN>`은 Microsoft 365 tenant와 연결된 도메인(가상 예: `example.test`)이고, `usersfound.txt`는 앞선 사용자 열거에서 만든 이메일·UPN 한 줄 목록이다. `<ATTEMPTS_PER_WINDOW>`와 `<RESET_MINUTES>`는 확인한 잠금 정책의 계정별 시도 수와 reset 시간이며, `<O365_OUTPUT_DIR>`은 공격 호스트의 새 전용 결과 디렉터리다.

확인할 출력:

- `[VALID] user@<DOMAIN>:<PASSWORD>`와 valid credential 저장 경로.
- 이 출력이 없으면 평문 비밀번호를 확보하지 못한 상태다.
- 성공 항목도 MFA와 실제 대상 서비스 권한을 별도로 확인한다.
- `--count`는 reset window마다 사용자별로 시도할 비밀번호 수이고 `--lockout`은 확인한 reset 시간(분)이다. 두 값을 임의의 `1`로 고정하지 않고 실제 정책보다 보수적으로 정한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| O365 validation `VALID` | 대상 도메인이 Microsoft 365 사용 | 클라우드 인증 대상 확인 | 사용자 후보와 잠금 정책 확인 |
| spray 결과 `VALID user:password` | 해당 사용자와 비밀번호 조합이 유효함 | Microsoft 365 credential 확보 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 실제 서비스 접근과 MFA 분리 검증 |
| MFA 또는 Conditional Access 오류 | 비밀번호는 맞지만 추가 인증 조건이 존재할 수 있음 | credential 유효, 대화형 로그인 미확인 | 오류 의미와 실제 적용 조건 확인 |
| spray 전부 실패 | 비밀번호 후보 부적합 또는 인증 제한 | 유효 credential 미확인 | 잠금 위험을 넘지 않는 범위에서 후보와 정책 재검토 |
| 도구 오류 | 도구 또는 공급자 응답 변화 | 공격 결과 미판정 | 도구 버전과 현재 로그인 응답 형식 확인 |

## 확인할 출력과 권한

- 도메인 확인, 사용자 유효성, 비밀번호 유효성과 실제 서비스 로그인을 각각 다른 상태로 판정한다.
- 유효 비밀번호가 확인되어도 MFA와 Conditional Access를 통과했다는 뜻은 아니다.
- 메일함, SharePoint, Teams와 VPN 권한은 동일하지 않으므로 서비스별 접근 권한을 따로 확인한다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상 Microsoft 365 계정의 인증 실패 카운터 | 반복 실패 시 계정 잠금·경고·Conditional Access 이벤트 발생 가능 | 계정별 시도 수·간격과 인증 응답 확인 | 정한 횟수에 도달하거나 잠금·경고 신호가 보이면 즉시 중지하고 정책상 재시도 가능 시점까지 기다림 |
| 로컬 valid credential·tested account 결과 파일 | 평문 credential과 사용자 시도 결과가 전용 디렉터리에 생성됨 | 도구가 표시한 각 결과 파일의 정확한 경로와 권한 확인 | 접근 제한된 경로에 두고, 폐기할 때만 기록한 `<SPRAY_RESULT_FILE>` 등 정확한 파일을 `rm --`으로 제거한 뒤 빈 `<O365_OUTPUT_DIR>`을 `rmdir --`로 정리 |

인증 성공·실패 이벤트와 탐지 기록은 client에서 되돌릴 수 없다. 대기 뒤 다시 시도할 수 있다는 사실을 원상복구로 기록하지 않으며, 잠긴 계정의 해제는 권한 있는 운영 절차에 맡긴다.

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[o365spray]]

## 관련 공격기법

- [[Microsoft 365 사용자 열거]]
- [[원격 비밀번호 공격]]

## 참고 링크

- [o365spray 공식 저장소 — `--count`, `--lockout`, module 동작](https://github.com/0xZDH/o365spray)
