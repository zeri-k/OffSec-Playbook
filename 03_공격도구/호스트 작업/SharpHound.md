---
tags:
  - 환경/ad
  - 기능/열거
실행환경: ["Windows"]
필요권한: ["도메인 사용자 세션 또는 동등한 AD 읽기 권한"]
필요조건: ["도메인 연결", "SharpHound 실행 가능"]
결과: ["BloodHound ZIP 수집 결과"]
---

# SharpHound

## 도구 개요

SharpHound는 Windows에서 AD 객체·ACL·세션·원격 접근 관계를 수집해 BloodHound용 ZIP 파일을 만든다. 수집 방법과 대상 컴퓨터를 조절하며 Windows 도메인 내부에서 그래프 분석 자료를 준비할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: DC와 도메인 호스트에 접근 가능한 Windows 호스트
- 필요한 입력: 현재 도메인 사용자 세션 또는 명시한 domain
- 수집 영향: collection method에 따라 LDAP 조회와 다수 호스트 접속이 발생하므로 필요한 method와 대상 목록을 먼저 정한다.
- collection method와 output ZIP basename은 Windows collector host의 값이고 domain은 AD DNS domain(예: `directory.example.invalid`)이다. graph edge·finding은 collection 시점의 후보이며 실제 ACL/권한은 별도 확인한다.

## 표준 사용법

`<COLLECTION_METHODS>`는 current collector가 지원하는 method 목록, `<OUTPUT_PREFIX>`는 Windows collector host의 새 ZIP basename, `<DOMAIN>`은 AD DNS domain이다. collection output의 graph edge·finding은 수집 시점 후보이며 실제 ACL·object READ/WRITE 권한은 해당 object에서 별도로 확인한다.

```powershell
.\SharpHound.exe -c <COLLECTION_METHODS> --zipfilename <OUTPUT_NAME>
```

## 대표 예시

### 전체 관계 수집

```powershell
.\SharpHound.exe -c All --zipfilename <OUTPUT_NAME>
```

확인할 출력:

- `Resolved Collection Methods`, 처리한 객체 수, `Enumeration Completed`, ZIP 파일.

### DC 중심 수집

```powershell
.\SharpHound.exe -c DCOnly --zipfilename <OUTPUT_NAME>
```

확인할 출력:

- LDAP 중심 수집 완료와 컴퓨터별 원격 접속 축소 여부.

### 특정 도메인 지정

```powershell
.\SharpHound.exe -d <DOMAIN> -c Group,ACL,Trusts --zipfilename <OUTPUT_NAME>
```

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-c` | 수집 방법 | `All`, `DCOnly`, `Group`, `ACL`, `Session` 등 |
| `-d` | 대상 domain | 현재 domain 외 대상을 명시 |
| `--zipfilename` | 결과 ZIP 이름 | 수집 결과 관리 |
| `--computerfile` | 컴퓨터 목록 제한 | 특정 컴퓨터만 수집할 때 |
| `--stealth` | 원격 접속을 줄이는 수집 모드 | 수집 부하와 노출 감소가 필요할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Enumeration Completed` | 선택한 수집 방법 완료 | ZIP 존재와 크기 확인 후 [[BloodHound]] 업로드 |
| 객체 수가 예상보다 적음 | scope, domain 또는 권한 제한 가능 | domain, collection method와 LDAP 접근 확인 |
| 호스트 접속 오류 다수 | 방화벽·오프라인·권한 문제 | DCOnly와 호스트별 검증으로 구분 |
| ZIP 생성 실패 | 출력 경로·잠금·권한 문제 | 쓰기 가능한 경로와 파일 사용 여부 확인 |

## 관련 공격기법

- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[AD ACL 권한 열거와 공격 경로 식별]]
