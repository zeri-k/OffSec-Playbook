---
tags:
  - 환경/ad
  - 기능/복호화
실행환경: ["Linux"]
필요조건: ["GPP XML에서 추출한 cpassword 값"]
결과: ["복호화된 GPP 비밀번호 후보"]
---

# gpp-decrypt

## 도구 개요

`gpp-decrypt`는 Group Policy Preferences XML의 `cpassword` 값을 공개된 AES key로 복호화해 평문 비밀번호 후보를 반환한다. 이 도구는 `cpassword`만 처리하므로 연결된 사용자명과 정책 적용 대상은 원본 XML에서 확인해야 한다.

## 필요한 입력과 실행 환경

- 실행 위치: `gpp-decrypt`가 설치된 Linux 호스트
- 필요한 입력: `Groups.xml` 등에서 직접 확인한 `cpassword` 값
- 연결 정보: XML의 사용자명과 정책 적용 대상을 함께 보존한다.

`<CPASSWORD>`는 `Groups.xml` 등에서 얻은 XML attribute의 암호문 문자열이며, 예시는 실제 값을 쓰지 않고 `<CPASSWORD>` 전체를 해당 attribute 값으로 치환한다.

명령은 XML을 확보한 Linux 분석 호스트에서 실행하며, 복호화 출력은 같은 XML의 사용자명·정책 적용 대상과 함께 해석한다.

## 표준 사용법

```bash
gpp-decrypt '<CPASSWORD>'
```

## 대표 예시

```bash
gpp-decrypt '<CPASSWORD>'
```

확인할 출력:

- 한 줄로 반환되는 복호화된 비밀번호 후보.

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| 평문 문자열 반환 | `cpassword` 복호화 성공 | XML의 사용자명·적용 대상과 함께 최소 인증 검증 |
| 출력 없음 또는 오류 | 입력값 형식·복사 오류 가능성 | 원본 XML의 `cpassword` 값을 다시 확인 |
| 인증 실패 | 오래된 GPP 값 또는 계정 상태·범위 불일치 | 현재 계정 상태와 적용 호스트 확인 |

## 관련 공격기법

- [[SYSVOL GPP 자격 증명 수집]]
