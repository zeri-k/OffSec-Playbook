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
- 선택 구성: 현재 upstream에서 Excel application은 선택 사항이며, GPO 출력에는 GroupPolicy PowerShell 모듈·RSAT가 필요
- 출력 위치: `-OutputDir`로 지정하고 실행 전에 존재하지 않음을 확인한 전용 폴더

## 표준 사용법

```powershell
.\ADRecon.ps1
```

## 대표 예시

### 전체 기본 수집

기존 결과와 섞이지 않는 전용 경로를 지정한다. 현재 upstream의 기본 수집은 모든 optional module을 의미하지 않으므로 `-Collect`·`-OutputType`을 생략한 결과 범위를 실행 출력에서 확인한다.

```powershell
if (Test-Path -LiteralPath '<ADRECON_RUN_DIRECTORY>') { throw 'run directory already exists' }
.\ADRecon.ps1 -OutputDir '<ADRECON_RUN_DIRECTORY>'
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
- 지정한 output folder의 CSV를 읽어 Excel 파일이 생성됐는지 확인한다. 현재 upstream은 Excel application을 필수로 요구하지 않지만, 역사적 교육자료나 구버전 script의 요구 사항과 혼용하지 않는다.

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
| Excel 보고서 없음 | 현재 output type·version에서 생성을 생략했거나 보고서 생성 단계 실패 | CSV 생성 여부, `-GenExcel` 입력 경로와 현재 release 요구 조건 확인 |

## 변경 영향과 복구

보고서 directory에는 사용자·SPN·group·trust·DNS·GPO와 권한이 허용된 경우 LAPS·BitLocker 같은 민감 속성이 포함될 수 있다. 후속 분석과 필요한 사본 처리가 끝난 뒤, 실행 전 존재하지 않았던 exact `<ADRECON_RUN_DIRECTORY>`만 제거한다.

```powershell
Remove-Item -LiteralPath '<ADRECON_RUN_DIRECTORY>' -Recurse
Test-Path -LiteralPath '<ADRECON_RUN_DIRECTORY>'
```

마지막 출력이 `False`여야 로컬 수집물 정리가 끝난 것이다. 디렉터리 존재 기준선을 확인하지 못했거나 다른 작업 파일이 섞였다면 재귀 삭제하지 않고 생성 manifest와 exact 파일을 먼저 대조한다. LDAP·ADWS·SYSVOL 조회 기록과 이미 복사한 보고서는 이 정리로 제거되지 않는다.

## 관련 공격기법

- [[AD 보안 구성과 GPO 감사]]

## 참고 링크

- [ADRecon 공식 저장소와 현재 사용법](https://github.com/adrecon/ADRecon)
