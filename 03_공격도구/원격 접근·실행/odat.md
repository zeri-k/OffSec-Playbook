---
tags:
  - 서비스/oracle
  - 기능/열거
  - 기능/인증검증
실행환경: ["Linux"]
필요조건: ["Oracle TNS 접근", "모듈에 따라 SID, Service Name 또는 Oracle 인증 정보"]
결과: ["Oracle listener·SID·Service Name 정보", "유효 자격증명 후보", "인증 뒤 추가 점검할 기능 후보"]
---

# odat

## 도구 개요

ODAT는 Oracle TNS의 SID·Service Name과 계정을 열거하고 Oracle 기능을 module별로 점검하는 도구 모음이다. 이 문서는 listener·접속 식별자와 credential 후보를 찾는 대표 진입 절차를 다룬다. 파일 접근·명령 실행·네트워크 기능은 `odat <module> --help`에서 현재 버전의 입력과 권한을 확인한 뒤 별도 절차로 검증한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 공통 입력: Oracle TNS 호스트와 포트
- 열거 입력: SID 또는 Service Name 후보
- 인증·기능 점검 입력: Oracle 계정, 비밀번호 또는 계정 목록
- 버전 조건: 설치된 `odat --version` 또는 package 버전과 `odat <module> --help`를 먼저 확인한다. 현재 upstream `master-python3`의 `utlfile` 행동은 아래 상태 변경 주의 사항을 따른다.

## 표준 사용법

```bash
odat <module> -s <host> -p <port> [options]
```

## 대표 예시

### TNS listener와 SID·Service Name 확인

```bash
odat tnscmd -s <TARGET> -p 1521 --ping
odat sidguesser -s <TARGET> -p 1521
odat snguesser -s <TARGET> -p 1521
```

`tnscmd` 응답은 listener 도달성을, `sidguesser`와 `snguesser` 출력은 각각 유효한 SID·Service Name 후보를 뜻한다. 이 단계는 계정 인증이나 database 권한을 확인하지 않는다. `all`은 인증 뒤 공격 module까지 순서대로 점검할 수 있어 이 열거 전용 대표 절차에서는 사용하지 않는다.

### 계정 추측

```bash
odat passwordguesser -s <TARGET> -p 1521 -d <SID> --accounts-file <ACCOUNTS_FILE>
```

실행 전에 Oracle profile의 `FAILED_LOGIN_ATTEMPTS`·`PASSWORD_LOCK_TIME`과 승인된 시도 범위를 확인한다. `valid credential`은 Oracle 인증 성공이지 DBA·SYSDBA·파일 기능 권한을 의미하지 않는다.

### TNS 응답 확인

```bash
odat tnscmd -s <TARGET> -p 1521 --ping
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `sidguesser`, `snguesser` | SID와 Service Name 후보를 각각 확인 |
| `passwordguesser` | Oracle 계정 추측 |
| `-s`, `-p` | 서버 주소와 포트 |
| `-d` | SID 지정 |
| `-n` | Service Name 지정; 현 upstream `master-python3` 문법 |
| `-U`, `-P` | 사용자명과 비밀번호 |
| `--sysdba` | SYSDBA 권한으로 접속 시도 |
| `--putFile <REMOTE_PATH> <REMOTE_NAME> <LOCAL_FILE>` | `utlfile`로 서버 파일 생성; 현 upstream에서 임시 DIRECTORY object를 생성·권한 부여·drop함 |
| `--removeFile <REMOTE_PATH> <REMOTE_NAME>` | `utlfile`이 같은 방식으로 exact 서버 파일을 제거 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SID/service 또는 listener 정보 확인 | Oracle 접속 대상 식별 성공 | 유효 계정 확인과 권한/기능 열거로 진행 |
| valid credential 출력 | Oracle 계정 확인 | `sqlplus`로 직접 접속해 권한과 schema 확인 |
| SID·Service Name 후보 출력 | Oracle 접속 식별자 후보 확인 | `passwordguesser` 또는 승인된 계정으로 인증을 별도 확인 |
| `ORA-*` 오류 | 인증, SID/service, 권한, 기능 제한 문제 | 오류 코드 기준으로 접속 정보와 권한 재확인 |
| SID/service 탐지 실패 | listener 응답 제한 또는 이름 불일치 | 수동 `tnsping`, 서비스명 후보, 포트 확인 |
| 기능 모듈 실패 | package 권한 부족 또는 기능 비활성화 | 모듈별 요구 권한과 Oracle 버전 확인 |
| 연결 불안정 | 방화벽, latency, TLS/버전 문제 | 타임아웃, 재시도 횟수, 터널 안정성 확인 |

## 상태 변경 주의

현 upstream `master-python3`의 `utlfile --putFile`·`--removeFile`은 지정한 절대 경로를 가리키는 임시 `ODATPREFIX<RANDOM>` DIRECTORY object를 생성하고 `PUBLIC`에 `READ,WRITE`를 부여한 뒤 동작 후 object를 drop한다. 따라서 이 모듈은 단순 파일 I/O가 아니며 `CREATE ANY DIRECTORY`와 권한 부여 가능성을 포함한 DB 상태 변경이다.

- 실행하기 전 `odat.py`와 `DirectoryManagement.py`의 해당 버전 구현을 확인하고 SQL audit에서 식별할 작업 시간·계정·원격 파일 경로를 Vault 밖에 기록한다.
- 중단되면 `ALL_DIRECTORIES` 조회에서 해당 작업 시간에 생성된 exact `ODATPREFIX...` object와 원격 파일을 별도로 확인한다. prefix만으로 모든 object를 삭제하지 않는다.
- 임시 object drop·파일 제거를 확인하지 못했으면 정리 완료로 기록하지 않는다. 이미 존재하는 DIRECTORY object로 파일 하나만 검증할 수 있으면 [[Oracle TNS 서비스#선택적 UTL_FILE 텍스트 쓰기 proof|UTL_FILE 텍스트 쓰기 proof]]를 우선한다.

## 관련 공격기법

- [[Oracle TNS SID 열거]]

## 참고 링크

- [ODAT 공식 저장소](https://github.com/quentinhardy/odat)
- [ODAT `utlfile` 구현](https://github.com/quentinhardy/odat/blob/master-python3/UtlFile.py)
- [ODAT DIRECTORY 관리 구현](https://github.com/quentinhardy/odat/blob/master-python3/DirectoryManagement.py)
