---
tags:
  - 환경/ad
  - 기능/열거
실행환경: ["Windows CMD", "Windows PowerShell"]
필요권한: ["현재 AD 계정에 허용된 디렉터리·호스트 정보 읽기 권한"]
필요조건: ["대상 도메인 또는 서버", "LDAP 또는 ADWS 접근", "도메인 계정 또는 통합 인증 세션"]
결과: ["AD healthcheck 보고서", "도메인 위험 점수", "연결된 도메인 지도", "보안 설정·호스트 scanner 결과"]
---

# PingCastle

## 도구 개요

PingCastle은 AD 환경의 계정·컴퓨터·trust·권한 위임과 보안 설정을 검사해 위험 점수, 도메인 지도와 보고서를 만드는 감사 도구다. 도메인 전체의 보안 상태를 빠르게 분류하고 추가로 직접 확인할 설정·호스트·관계 후보를 찾을 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 도메인의 DNS와 LDAP 389번 또는 ADWS 9389번에 접근 가능한 Windows 호스트
- 기본 입력: 현재 통합 인증 세션 또는 `--user`와 비밀번호, 대상 서버·프로토콜
- 범위 입력: 현재 도메인, 연결된 모든 도메인 또는 특정 scanner
- 출력 해석: 위험 점수와 finding은 우선순위 단서이며 실제 취약점·권한을 별도로 검증함

## 표준 사용법

```cmd
PingCastle.exe --help
PingCastle.exe
```

인자 없이 실행하면 대화형 Terminal User Interface(TUI)에서 healthcheck·consolidation·domain map·scanner·export 등을 선택한다.

## 대표 예시

### 기본 healthcheck 실행

```cmd
PingCastle.exe
```

대화형 메뉴에서 `healthcheck`를 선택한다.

확인할 출력:

- 도메인 개요, 사용자·그룹·trust와 이상 징후, 전체 위험 점수가 포함된 보고서.
- 보고서 파일이 실제 생성됐는지와 대상 도메인·수집 시점을 확인한다.

### 대상 서버와 프로토콜 입력 확인

```cmd
PingCastle.exe --help
```

확인할 출력:

- `--server`, `--port`, `--user`, `--password`, `--protocol`의 현재 버전 문법.
- 연결 오류가 나면 서버 이름 해석, LDAP·ADWS 포트, 인증 단계를 나눠 확인한다.

## 주요 옵션과 모드

| 옵션·모드 | 의미 | 사용하는 상황 |
|---|---|---|
| `healthcheck` | 도메인의 보안 위험과 기준 상태 평가 | 전체 상태와 우선순위 확인 |
| `conso` | 여러 보고서 집계 | 다중 도메인 결과 통합 |
| `carto` | 연결된 도메인 지도 작성 | trust 구조 확인 |
| `scanner` | ACL·로컬 관리자·LAPS·공유·Spooler·SMB 등 특정 검사 | healthcheck 결과의 범위 보강 |
| `export` | 사용자 또는 컴퓨터 내보내기 | 인벤토리 결과 확보 |
| `--server <SERVER>` | 연결할 서버 지정 | 기본 DC 대신 특정 대상 사용 |
| `--port <PORT>` | ADWS 또는 LDAP 포트 지정 | 연결 경로 명시 |
| `--protocol <PROTOCOL>` | ADWS·LDAP 사용 순서 선택 | 한 프로토콜이 제한된 환경 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| 위험 점수와 규칙별 finding | 보안 구성 문제 후보 발견 | 각 객체·서비스·정책에서 현재 상태 직접 확인 |
| domain map | 수집된 trust 관계 시각화 | [[AD 도메인 트러스트 열거와 공격 경로 식별]]로 방향·인증 확인 |
| scanner 결과 | 특정 호스트·설정의 노출 후보 | 서비스별 세부 기법으로 재검증 |
| HTML 보고서 생성 | 감사 결과 파일 생성 성공 | 대상·수집 시점·도구 버전 기록 |
| 연결 실패 | DNS·포트·프로토콜·인증 단계 문제 | `--server`, `--port`, `--protocol`과 계정 확인 |

## 버전과 환경 차이

- 이 문서에서 확인한 예시는 PingCastle `2.10.1.0` 출력이다. 현재 버전의 메뉴·scanner·옵션은 `--help`에서 다시 확인한다.

## 관련 공격기법

- [[AD 보안 구성과 GPO 감사]]
