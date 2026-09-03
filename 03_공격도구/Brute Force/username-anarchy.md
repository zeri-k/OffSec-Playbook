---
tags:
  - 기능/워드리스트
실행환경: ["Linux"]
필요조건: ["실명 또는 이름 목록"]
결과: ["사용자명 목록"]
---

# username-anarchy

## 도구 개요

`username-anarchy`는 실명을 여러 계정명 표기 규칙으로 변환해 사용자명 후보 목록을 만드는 도구다. 조직의 계정명 형식을 아직 확정하지 못했을 때 가능한 표기를 넓게 생성하는 데 유용하다. 출력만으로 계정 존재를 판단할 수는 없다.

## 필요한 입력과 실행 환경

- 입력: 이름과 성이 포함된 목록 또는 실명 목록
- 실행 환경: `username-anarchy` 스크립트를 실행할 수 있는 Linux 호스트

## 표준 사용법

```bash
./username-anarchy <First> <Last>
./username-anarchy -i <names.txt> > users.txt
```

이름과 성이 포함된 목록을 입력하면 여러 규칙의 사용자명 후보를 출력한다.

## 대표 예시

### 한 사람의 이름으로 사용자명 생성

```bash
./username-anarchy Betty Jayde > user.list
```

확인할 출력:

- `alice`, `alice.smith`, `a.smith`, `asmith` 같은 후보

### 이름 목록으로 사용자명 생성

```bash
./username-anarchy -i <OPERATOR_HOME>/names.txt > users.txt
```

확인할 출력:

- 입력한 이름마다 여러 로그인 ID 후보가 생성된다.

### Hydra와 연결

```bash
hydra -L user.list -p '<PASSWORD>' ssh://<TARGET>
```

확인할 출력:

- `[22][ssh] host: <TARGET> login: <USER> password: <PASSWORD>`

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-i <FILE>` | 이름 목록 입력 | 직원명 목록을 한 번에 변환 |
| `> users.txt` | 결과 저장 | 인증 검증 도구에 넘길 목록 생성 |
| `<First> <Last>` | 단일 이름 입력 | 빠른 후보 생성 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 여러 사용자명 후보 출력 | 조직 ID 규칙 후보 확보 | `hydra`, `netexec`, `kerbrute`로 검증 |
| 결과가 맞지 않음 | 조직 규칙이 다름 | 이메일 주소, 문서 작성자, SMB/LDAP 단서로 규칙 보정 |

## 관련 공격기법

- [[원격 비밀번호 공격]]

## 관련 도구

- [[hydra]]
- [[netexec]]
- [[kerbrute]]
