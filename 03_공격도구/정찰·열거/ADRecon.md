---
tags:
  - 환경/ad
  - 기능/열거
실행환경: ["Windows PowerShell"]
필요권한: ["현재 AD 계정에 허용된 디렉터리·정책·비밀 속성 읽기 권한"]
필요조건: ["도메인 사용자 컨텍스트", "대상 AD와 DNS 접근", "ADRecon.ps1"]
결과: ["AD 인벤토리 CSV", "GPO XML·HTML", "선택적 Excel 보고서", "사용자·그룹·컴퓨터·trust·DNS·LAPS 단서"]
---

# ADRecon

## 도구 개요

ADRecon은 도메인·포리스트·trust·사용자·그룹·컴퓨터·GPO·DNS와 계정 보안 속성을 PowerShell로 폭넓게 수집하여 CSV와 보고서로 정리하는 도구다. 기존 열거에서 빠진 항목을 보완하고 도메인 전체 인벤토리와 보고 자료를 한 번에 만들 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 AD와 DNS에 접근 가능한 Windows PowerShell
- 계정 조건: 도메인 사용자 컨텍스트로 시작하며 LAPS·BitLocker Recovery Key 같은 항목은 별도 읽기 권한 필요
- 선택 구성: Excel 보고서 생성에는 Excel, GPO 출력에는 GroupPolicy PowerShell 모듈 필요
- 출력 위치: 실행 디렉터리 아래 생성되는 `ADRecon-Report-<TIMESTAMP>` 폴더

## 표준 사용법

```powershell
.\ADRecon.ps1
```

## 대표 예시

### 전체 기본 수집

```powershell
.\ADRecon.ps1
```

확인할 출력:

- `Domain`, `Forest`, `Trusts`, `Users and SPNs`, `Groups`, `GPOs`, `DNS Zones and Records` 등 완료된 항목.
- `Total Execution Time`과 `Output Directory`가 표시되고 실제 결과 폴더가 생성되어야 한다.

### 기존 결과에서 Excel 보고서 생성

```powershell
.\ADRecon.ps1 -GenExcel <REPORT_DIRECTORY>
```

확인할 출력:

- 지정한 보고서 디렉터리의 CSV를 바탕으로 생성된 Excel 결과.
- Excel이 없으면 CSV만 생성될 수 있으므로 스크립트 완료와 Excel 파일 생성을 구분한다.

## 주요 수집 범위와 조건

| 항목 | 의미 | 추가 조건 |
|---|---|---|
| Domain·Forest·Trusts | 도메인 구조와 신뢰 관계 | 현재 계정의 디렉터리 읽기 범위 |
| Users and SPNs·Groups | 계정·서비스·멤버십 인벤토리 | 고권한 여부와 비밀번호 유효성은 별도 확인 |
| GPOs·gPLinks·GPOReport | 정책과 적용 범위 | GroupPolicy PowerShell 모듈 필요 |
| DNS Zones and Records | AD DNS 정보 | DNS 객체 읽기와 이름 해석 확인 |
| LAPS·BitLocker Recovery Keys | 관리 비밀번호·복구 키 속성 | 해당 비밀 속성의 실제 읽기 권한 필요 |
| `-GenExcel` | 기존 CSV에서 Excel 보고서 생성 | Excel 설치 필요 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `[-] <SECTION>` 진행 항목 | 해당 수집 단계 실행 | 완료 여부와 결과 CSV 확인 |
| `Output Directory` | 결과 폴더 생성 위치 | CSV·GPO XML·HTML 파일 존재 확인 |
| `GPO-Report.html`, `GPO-Report.xml` | GPO 보고서 생성 | Group3r·직접 GPO 조회와 교차 확인 |
| LAPS·BitLocker 항목 빈 결과 | 객체 부재 또는 현재 계정 읽기 제한 | 배포 여부와 비밀 속성 읽기 권한을 분리해 확인 |
| Excel 보고서 없음 | Excel 미설치 또는 보고서 생성 단계 실패 | CSV 생성 여부와 `-GenExcel` 실행 조건 확인 |

## 관련 공격기법

- [[AD 보안 구성과 GPO 감사]]
