---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/열거
실행환경: ["Windows GUI"]
필요권한: ["현재 AD 계정에 허용된 디렉터리 객체·속성·권한 읽기 권한"]
필요조건: ["대상 AD 연결 정보와 도메인 계정 또는 기존 snapshot"]
결과: ["AD 객체·속성·권한 조회 결과", "오프라인 분석용 AD snapshot", "snapshot 간 변경 사항"]
---

# AD Explorer

## 도구 개요

AD Explorer는 Active Directory 객체·속성·스키마와 권한을 GUI에서 탐색하고 검색하는 Sysinternals 도구다. 현재 AD 상태를 snapshot으로 저장해 오프라인에서 조사하거나 서로 다른 시점의 객체·속성·권한 변화를 비교할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 AD에 연결할 수 있는 Windows GUI 호스트
- 연결 입력: 대상 도메인 또는 DC, 도메인 사용자와 비밀번호
- 오프라인 입력: 이전에 생성한 AD Explorer snapshot 파일
- 권한 조건: 반환되는 객체와 속성은 연결에 사용한 계정의 디렉터리 읽기 범위에 따라 달라짐

## 표준 사용법

```cmd
ADExplorer.exe
```

처음 실행할 때 대상과 자격 증명으로 연결하거나 기존 snapshot을 연다.

## 대표 예시

### 현재 AD 탐색

연결 창에서 대상 도메인·DC와 계정을 입력한 뒤 트리에서 객체를 선택하여 속성과 보안 권한을 확인한다.

확인할 출력:

- 연결한 도메인의 Distinguished Name(DN) 트리와 선택한 객체의 속성·스키마·권한.
- 객체가 보이지 않으면 연결 대상, 현재 계정과 해당 객체의 읽기 권한을 구분해 확인한다.

### 오프라인 분석용 snapshot 생성

`File -> Create Snapshot`에서 설명과 기존에 없던 exact 저장 경로 `<AD_EXPLORER_SNAPSHOT_PATH>`를 지정한다. 생성된 snapshot을 다시 열어 현재 AD 연결 없이 객체와 속성을 탐색한다.

확인할 출력:

- 지정한 경로의 snapshot 파일과 생성 시점.
- snapshot은 저장 시점의 상태이며 현재 그룹·권한·객체 상태를 보장하지 않는다.

## 주요 기능

| 기능 | 의미 | 사용하는 상황 |
|---|---|---|
| 객체 탐색 | AD 트리와 객체 속성·스키마 확인 | 특정 사용자·그룹·컴퓨터·정책 조사 |
| Search | 조건에 맞는 객체 검색 | 넓은 디렉터리에서 속성 후보 축소 |
| Create Snapshot | 현재 AD 데이터베이스 상태 저장 | 오프라인 분석과 보고 근거 보존 |
| Snapshot 비교 | 서로 다른 시점의 객체·속성·권한 차이 확인 | 변경 전후 감사 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| AD 트리와 객체 속성 표시 | 현재 계정으로 디렉터리 조회 성공 | 중요한 속성·권한을 별도 명령으로 재검증 |
| snapshot 파일 생성 | 시점별 AD 상태 저장 성공 | 설명·수집 계정·시점과 대상 도메인 기록 |
| snapshot 간 차이 | 객체·속성·권한 변경 후보 | 현재 AD에서 실제 변경 상태 확인 |
| 연결 또는 객체 조회 실패 | 대상·인증·네트워크 또는 읽기 권한 문제 | DNS, LDAP 연결, 계정 형식과 객체 ACL 확인 |

## 변경 영향과 복구

snapshot은 연결 계정이 읽을 수 있던 AD 객체·속성·권한을 포함할 수 있는 민감한 로컬 파일이다. 실행 전 `Test-Path -LiteralPath '<AD_EXPLORER_SNAPSHOT_PATH>'`가 `False`인지 확인하고, 생성한 exact 경로와 수집 시점을 Vault 밖의 승인된 작업 기록에 남긴다.

AD Explorer에서 snapshot을 닫고 후속 분석이 끝난 뒤 다음처럼 이번 작업 파일만 제거한다.

```powershell
Remove-Item -LiteralPath '<AD_EXPLORER_SNAPSHOT_PATH>'
Test-Path -LiteralPath '<AD_EXPLORER_SNAPSHOT_PATH>'
```

마지막 출력이 `False`여야 로컬 파일 정리가 끝난 것이다. 이미 복사·업로드한 snapshot과 디렉터리 조회 감사 기록은 이 삭제로 제거되지 않는다.

## 관련 공격기법

- [[AD 보안 구성과 GPO 감사]]

## 참고 링크

- [Microsoft Sysinternals: AD Explorer](https://learn.microsoft.com/sysinternals/downloads/adexplorer)
