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

PingCastle은 command line 실행 시 현재 디렉터리에 HTML·XML 보고서를 생성한다. 기존 파일과 섞이지 않는 전용 디렉터리를 만든 뒤 현재 지원 버전의 명시적 healthcheck 명령을 사용하고, 출력된 exact 파일명을 기록한다.

`<PINGCASTLE_RUN_DIRECTORY>`는 실행 Windows 호스트의 새 절대 작업 디렉터리(예: `C:\\Temp\\pingcastle-20260914`)다. `<PINGCASTLE_EXE_PATH>`는 같은 호스트의 실행 파일 절대 경로, `<DOMAIN_FQDN>`은 조회할 AD DNS 도메인(예: `corp.example.test`)이다. `<PINGCASTLE_HTML_PATH>`·`<PINGCASTLE_XML_PATH>`·`<PINGCASTLE_LOG_PATH>`는 이 실행이 해당 작업 디렉터리에 만든 정확한 보고서 경로다.

```powershell
if (Test-Path -LiteralPath '<PINGCASTLE_RUN_DIRECTORY>') { throw 'run directory already exists' }
New-Item -ItemType Directory -Path '<PINGCASTLE_RUN_DIRECTORY>' | Out-Null
Push-Location '<PINGCASTLE_RUN_DIRECTORY>'
try {
    & '<PINGCASTLE_EXE_PATH>' --healthcheck --server <DOMAIN_FQDN>
} finally {
    Pop-Location
}
```

확인할 출력:

- 도메인 개요, 사용자·그룹·trust와 이상 징후, 전체 위험 점수가 포함된 보고서.
- HTML·XML 보고서의 exact 경로, 대상 도메인·수집 시점과 실행 버전을 확인한다.

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

- 교육 원천의 `2.10.1.0` TUI와 지원 종료 시각은 역사적 출력이다. 시스템 시간을 과거로 바꾸지 말고 현재 지원 release와 그 `--help`를 사용한다.
- 현재 문서의 명령은 `--healthcheck --server`를 지원하는 release 기준이다. scanner 이름·report level·runtime 요구 사항은 설치한 release 문서에서 확인한다.

## 변경 영향과 복구

보고서에는 계정·그룹·trust·GPO와 보안 finding이 포함될 수 있다. 후속 분석이 끝나면 실행 출력에서 기록한 HTML·XML·log exact 경로만 제거하고, 전용 디렉터리가 비었을 때만 삭제한다.

```powershell
foreach ($path in @('<PINGCASTLE_HTML_PATH>', '<PINGCASTLE_XML_PATH>', '<PINGCASTLE_LOG_PATH>')) {
    if (Test-Path -LiteralPath $path) { Remove-Item -LiteralPath $path }
}
Remove-Item -LiteralPath '<PINGCASTLE_RUN_DIRECTORY>'
Test-Path -LiteralPath '<PINGCASTLE_RUN_DIRECTORY>'
```

마지막 출력이 `False`여야 로컬 정리가 끝난 것이다. 보고서를 다른 분석 system에 업로드했다면 그 사본과 AD·host 조회 기록은 별도 잔여 영향으로 관리한다.

## 관련 공격기법

- [[AD 보안 구성과 GPO 감사]]

## 참고 링크

- [PingCastle: Healthcheck](https://pingcastle.com/documentation/healthcheck/)
- [Netwrix: PingCastle Standard and Basic User Guide](https://docs.netwrix.com/docs/pingcastle/4_0)
