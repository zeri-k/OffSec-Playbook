---
tags:
  - 서비스/oracle
  - 기능/열거
  - 기능/인증검증
실행환경: ["Linux"]
필요조건: ["Oracle TNS 접근", "모듈에 따라 SID, Service Name 또는 Oracle 인증 정보"]
결과: ["Oracle 정보", "유효 자격증명", "파일 접근", "명령 출력"]
---

# odat

## 도구 개요

ODAT는 Oracle TNS의 SID·Service Name과 계정을 열거하고 Oracle 기능을 module별로 점검하는 도구 모음이다. 초기 접속 정보 확인부터 계정이 사용할 수 있는 파일, 명령, 네트워크 기능의 범위를 한 도구에서 넓게 살펴볼 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 공통 입력: Oracle TNS 호스트와 포트
- 열거 입력: SID 또는 Service Name 후보
- 인증·기능 점검 입력: Oracle 계정, 비밀번호 또는 계정 목록

## 표준 사용법

```bash
odat <module> -s <host> -p <port> [options]
```

## 대표 예시

### Oracle 전체 기본 점검

```bash
odat all -s <TARGET> -p 1521
```

### 계정 추측

```bash
odat passwordguesser -s <TARGET> -p 1521 -d XE --accounts-file accounts.txt
```

### TNS 응답 확인

```bash
odat tnscmd -s <TARGET> -p 1521 --ping
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `all` | 여러 점검 모듈을 한 번에 실행 |
| `passwordguesser` | Oracle 계정 추측 |
| `-s`, `-p` | 서버 주소와 포트 |
| `-d` | SID 지정 |
| `--serviceName` | Service Name 지정 |
| `-U`, `-P` | 사용자명과 비밀번호 |
| `--sysdba` | SYSDBA 권한으로 접속 시도 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SID/service 또는 listener 정보 확인 | Oracle 접속 대상 식별 성공 | 유효 계정 확인과 권한/기능 열거로 진행 |
| valid credential 출력 | Oracle 계정 확인 | `sqlplus`로 직접 접속해 권한과 schema 확인 |
| `UTL_FILE`, Java, external table 관련 성공 | 파일 쓰기 또는 명령 실행 가능성 존재 | 기능별 전제 조건과 권한을 좁혀 검증 |
| `ORA-*` 오류 | 인증, SID/service, 권한, 기능 제한 문제 | 오류 코드 기준으로 접속 정보와 권한 재확인 |
| SID/service 탐지 실패 | listener 응답 제한 또는 이름 불일치 | 수동 `tnsping`, 서비스명 후보, 포트 확인 |
| 기능 모듈 실패 | package 권한 부족 또는 기능 비활성화 | 모듈별 요구 권한과 Oracle 버전 확인 |
| 연결 불안정 | 방화벽, latency, TLS/버전 문제 | 타임아웃, 재시도 횟수, 터널 안정성 확인 |

## 관련 공격기법

- [[Oracle TNS SID 열거]]
