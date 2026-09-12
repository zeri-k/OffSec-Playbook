---
tags:
  - 환경/windows
  - 기능/프로세스실행
실행환경: ["Windows"]
필요권한: ["Windows 셸"]
필요조건: ["사용할 Windows 계정의 인증 정보"]
결과: ["프로세스", "로그온 세션", "네트워크 인증 컨텍스트"]
---

# runas

## 도구 개요

`runas`는 Windows에서 다른 사용자 자격 증명으로 새 프로세스를 시작한다. 일반 실행은 해당 사용자의 로컬 token을 사용하지만 `/netonly`는 로컬 identity를 유지한 채 원격 서비스 인증에만 지정 계정을 사용하므로 두 모드를 구분해야 한다.

## 필요한 입력과 실행 환경

- 실행 환경: Windows
- 입력: 사용할 로컬 또는 도메인 계정과 실행할 프로그램
- 인증 입력: 대화형으로 입력할 계정 비밀번호
- `/netonly` 조건: 지정 계정을 사용할 네트워크 서비스와 해당 서비스로의 도달성

## 표준 사용법

```cmd
runas /user:<domain\user> <program>
```

## 대표 예시

### 다른 도메인 사용자로 cmd 실행

```cmd
runas /user:<DOMAIN>\<USER> cmd.exe
```

### 네트워크 인증에만 다른 credential 사용

```cmd
runas /netonly /user:<DOMAIN>\<USER> "cmd.exe"
```

### PowerShell을 netonly 세션으로 실행

```cmd
runas /netonly /user:<DOMAIN>\<USER> "powershell.exe"
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `/user` | 실행에 사용할 사용자 지정 |
| `/netonly` | 네트워크 인증에만 지정 credential 사용 |
| `/savecred` | Windows Credential Manager에 저장된 credential 사용 |
| `/profile` | 사용자 profile 로드 |
| `/env` | 현재 환경 변수 사용 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 새 프로세스 실행 | 다른 사용자 credential로 프로세스 생성 성공 | 새 프로세스에서 `whoami`, 네트워크 접근 여부 확인 |
| `/netonly` 실행 | 로컬은 현재 사용자, 네트워크는 지정 계정 사용 | SMB/LDAP/MSSQL 같은 네트워크 인증으로 검증 |
| password incorrect | credential 오류 | 도메인/로컬 계정 형식과 비밀번호 재확인 |
| logon type 제한 | 정책 또는 권한 문제 | interactive/network logon 권한과 UAC 상태 확인 |
| 프로세스는 뜨지만 접근 실패 | `/netonly` 동작 오해 또는 권한 부족 | 로컬/네트워크 인증 차이와 대상 서비스 권한 확인 |
| UAC 영향 | 관리자 토큰 미상승 | 관리자 권한 실행 여부와 UAC 상태 확인 |

## 관련 공격기법

- 확보한 평문 비밀번호로 로컬 사용자 프로세스 생성: [[확보한 평문 비밀번호로 runas 사용자 프로세스 실행]]
- 확보한 AD 비밀번호로 네트워크 인증 컨텍스트 생성: [[확보한 AD 비밀번호로 runas netonly 네트워크 인증 컨텍스트 생성]]
- `/savecred`로 저장된 계정의 프로세스 생성: [[저장된 자격 증명으로 runas 프로세스 실행]]
- `/netonly`로 다른 AD 계정의 MSSQL 인증: [[DB 인증과 데이터 열거]]
