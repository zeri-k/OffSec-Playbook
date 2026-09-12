---
tags:
  - 환경/cloud
  - 서비스/microsoft365
시작조건: ["대상 도메인과 이메일·UPN 후보 확보", "Microsoft 365 인증 endpoint HTTPS 접근 가능"]
필요조건: ["대상 도메인", "사용자 후보 목록"]
결과: ["유효한 Microsoft 365 사용자 후보", "Password Spraying 입력 목록"]
---

# Microsoft 365 사용자 열거

## 한 줄 판단

대상 도메인이 Microsoft 365를 사용하고 이메일 또는 User Principal Name(UPN) 후보 목록이 있으면, 현재 도구의 `office` 모듈처럼 비밀번호 인증을 시도하지 않는 열거 방식을 명시해 유효한 계정 후보와 결과 파일을 만든다.

## 시작 조건 해석


- SMTP·웹·공개 문서에서 대상 도메인과 이메일 형식 단서를 얻었을 때.
- 비밀번호 검증 전에 존재 가능한 Microsoft 365 계정만 좁혀야 할 때.
- 사용자 존재 확인과 실제 인증 성공을 분리해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| Microsoft 365 사용 여부 | 대상 도메인이 Microsoft 365 인증 흐름을 사용 | `--validate` 결과 | MX·로그인 endpoint와 다른 메일 제공자 확인 |
| 사용자 후보 | 이메일 또는 UPN 형식의 한 줄당 한 계정 목록 | 입력 파일 형식 확인 | [[SMTP 사용자 열거]]와 공개 이메일 단서 재검토 |
| 실행 경로 | 공격 호스트에서 Microsoft 365 HTTPS endpoint 도달 | DNS·프록시·도구 응답 | 인터넷 egress와 프록시 설정 확인 |

## 실행

### 도메인 사용 여부 확인

```bash
python3 o365spray.py --validate --domain <DOMAIN>
```

확인할 출력:

- `[VALID] The following domain is using O365`.

### 사용자 후보 열거

```bash
python3 o365spray.py --version
test ! -e '<O365_OUTPUT_DIR>'
install -d -m 700 '<O365_OUTPUT_DIR>'
python3 o365spray.py --enum --enum-module office -U users.txt --domain <DOMAIN> --output '<O365_OUTPUT_DIR>'
```

확인할 출력:

- `[VALID] user@<DOMAIN>` 형식의 사용자 후보.
- 도구가 표시하는 valid user 결과 파일 경로와 실제 파일 내용.
- 이 결과는 계정 존재 가능성만 의미하며 비밀번호, MFA 통과 또는 서비스 권한을 입증하지 않는다.
- 설치 버전의 `--help`에서 `office` 모듈이 제공되지 않거나 응답 파싱이 달라졌으면 다른 모듈로 자동 대체하지 않는다. 일부 열거 모듈은 사용자마다 인증 시도를 만들 수 있으므로 해당 모듈 설명과 잠금 영향을 먼저 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 도메인 validation 성공 | Microsoft 365 사용자 열거를 진행할 대상 | 클라우드 Identity 대상 확인 | 사용자 후보 형식 정리 |
| 사용자 `VALID`와 결과 파일 확인 | 존재 가능한 Microsoft 365 계정 | 유효 사용자 후보 목록 | 잠금 정책을 확인한 뒤 [[Microsoft 365 Password Spraying]] 검토 |
| 모든 후보가 invalid | 형식·도메인·후보가 맞지 않거나 응답 방식이 달라짐 | 사용자 미확인 | UPN suffix, 이메일 형식과 도구 버전 확인 |
| 도구 오류 또는 일관되지 않은 응답 | 공급자 응답이나 도구 파싱 변화 가능 | 결과 미판정 | 현재 도구 버전과 단일 후보의 원시 응답 확인 |

## 확인할 출력과 권한

- 도메인 사용 여부, 사용자 후보 유효성, 비밀번호 일치와 실제 서비스 로그인을 각각 다른 상태로 기록한다.
- 사용자 열거 결과만으로 계정 접근 권한이나 활성 상태를 확정하지 않는다.

## 후속 공격 연결

- 잠금 정책과 단일 비밀번호 후보를 확보한 경우: [[Microsoft 365 Password Spraying]]
- 이메일 서비스 계정을 별도로 검증하는 경우: [[원격 비밀번호 공격]]

## 변경 영향과 정리

`--output`이 표시한 tested·valid account 파일의 정확한 경로를 기록하고 Vault 밖의 전용 디렉터리에 보관한다. 결과가 더 이상 필요 없을 때만 표시된 `<ENUM_TESTED_FILE>`·`<ENUM_VALID_FILE>`을 `rm --`으로 각각 제거하고 `rmdir -- '<O365_OUTPUT_DIR>'`로 빈 디렉터리를 정리한다. 다른 파일이 있어 `rmdir`가 실패하면 일괄 삭제하지 않고 남은 파일을 확인한다. 원격 열거 요청 로그는 client에서 되돌릴 수 없다.

## 관련 도구

- [[o365spray]]

## 참고 링크

- [o365spray 공식 저장소 — 모듈·버전·잠금 주의 사항](https://github.com/0xZDH/o365spray)
