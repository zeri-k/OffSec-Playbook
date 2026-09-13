---
tags:
  - 기능/권한상승
  - 기능/열거
실행환경: ["Linux", "Windows", "macOS", "Unix"]
필요권한: ["대상 호스트의 로컬 셸"]
필요조건: ["대상 OS와 아키텍처에 맞는 PEASS 하위 도구", "winPEAS C# 실행 파일은 .NET Framework 4.5.2 이상"]
결과: ["정보", "권한 상승 단서", "자격증명"]
---

# PEASS

## 도구 개요

`PEASS`는 운영체제별 권한 상승 열거 도구인 `linPEAS`와 `winPEAS` 등을 묶은 도구 모음이다. 대상 환경에 맞는 하위 도구로 서비스·작업·파일 권한·자격 증명·커널 관련 후보를 빠르게 수집할 때 적합하며, 강조된 항목은 수동 검증이 필요한 후보다.

## 필요한 입력과 실행 환경

- 실행 환경: 로컬 셸을 확보한 Linux, Windows, macOS 또는 Unix 호스트
- 입력: 대상 OS와 아키텍처에 맞는 `linpeas`, `winPEAS` 등 PEASS 하위 도구
- 선택 입력: 긴 출력을 보존할 로컬 파일 경로
- 권한 범위: 관리자·root가 아니라 현재 셸의 권한으로 실행하며, 접근 거부된 검사는 현재 권한에서 수집되지 않은 범위로 남긴다.
- 출력에는 경로·사용자·설정과 자격 증명 후보가 포함될 수 있다. 실제 출력 파일은 Vault 밖의 작업 경로에만 두고 작업 종료 시 별도로 정리한다.

## 표준 사용법

PEASS는 도구 묶음이므로 대상 OS에 맞는 하위 스크립트/바이너리를 받아 실행한다. 아래 Linux 명령은 `linPEAS`, Windows 명령은 `winPEAS`의 문법이며 서로 바꾸어 쓰지 않는다.

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

```powershell
& '<WINPEAS_PATH>' -h
& '<WINPEAS_PATH>' systeminfo userinfo
```

## 대표 예시

### Linux 권한 상승 자동 열거

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee linpeas.out
```

대상 Linux 호스트에서 실행해 의심 지점과 권한 상승 후보를 확인한다.

### 결과를 나중에 정리하기

```bash
/tmp/linpeas.sh | tee /tmp/linpeas_$(hostname).txt
```

셸이 불안정하거나 출력이 길 때 결과 파일을 남겨 둔다.

### Windows에서 winPEAS 실행

대상 Windows PowerShell에서 OS 아키텍처와 C# 실행 파일의 .NET Framework 조건을 먼저 확인한다. `<WINPEAS_PATH>`는 대상 Windows 호스트에 있는 공식 release asset의 절대 경로이며, 가상 예시는 `C:\\Temp\\winPEASx64.exe`다.

```powershell
[Environment]::Is64BitOperatingSystem
(Get-ItemProperty -LiteralPath 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full' -Name Release -ErrorAction SilentlyContinue).Release
Get-Item -LiteralPath '<WINPEAS_PATH>' | Select-Object FullName
& '<WINPEAS_PATH>' -h
```

공식 C# build는 .NET Framework 4.5.2 이상이 필요하며 v4 Full의 `Release` 값은 4.5.2 기준인 `379893` 이상이어야 한다. 값이 없거나 더 작거나 실행 시 CLR 오류가 나면 C# 실행 파일을 계속 시도하지 말고, 대상 환경에서 허용되는 공식 PowerShell·BAT build 또는 수동 열거를 선택한다. x64 Windows에는 x64 build를 우선하고 release asset의 아키텍처와 실제 호스트를 대조한다.

먼저 현재 시스템과 사용자·Windows access token(권한 privilege, UAC filtered/elevated 상태) 범위를 짧게 확인한다.

```powershell
& '<WINPEAS_PATH>' systeminfo userinfo
```

확인할 출력:

- `System Information`에서 OS·build·UAC·보안 설정, `Users Information`에서 현재 사용자·그룹·현재 process access token의 privilege가 출력되어야 한다.
- 이 출력은 현재 실행 프로세스가 조회한 상태다. Administrators 그룹 이름이나 `SeImpersonatePrivilege`가 보이는 것만으로 UAC filtered token이 아닌 elevated access token 또는 권한 상승 성공을 의미하지 않는다.
- `Bad image`, CLR 초기화 오류 또는 즉시 종료가 나오면 아키텍처·.NET 조건·파일 무결성을 먼저 확인한다. 일부 검사만 `Access is denied`이면 전체 실행 실패로 일반화하지 않는다.

기본 실행은 느린 추가 검사 일부를 제외한 전체 카테고리를 순회하므로, 허용된 평가 범위와 실행 시간을 확인한 뒤 사용한다. `notcolor`는 리다이렉션하거나 색상 없는 셸에서 항목을 읽을 때 선택한다.

```powershell
& '<WINPEAS_PATH>' notcolor
```

확인할 출력:

- 서비스·예약 작업·파일 ACL·설치 소프트웨어·저장 자격 증명 카테고리의 후보를 확인한다.
- 빨간색 또는 강조 표시는 도구가 흥미롭다고 분류한 후보이지 악용 성공이 아니다. 실행 주체, 실제 변경 가능한 객체, trigger와 현재 ACL을 기본 명령으로 다시 확인한다.
- 녹색은 일반적으로 보호 설정이 적용된 항목이다. 색상이 없으면 섹션 제목과 원시 값으로 판단한다.
- credential 문자열은 소유 계정·대상 서비스·유효성이 아직 미확인인 자료이며, 운영체제·패키지 취약 표시는 build·patch·구성 조건을 별도로 확인할 후보다.

### Windows 결과에서 다음 절차 선택

| winPEAS 관찰 | 출력으로 확정되는 범위 | 다음 확인 |
|---|---|---|
| 현재 token에 privilege가 표시됨 | 해당 privilege의 보유·활성 상태 | `whoami /all`로 재확인하고 [[Windows 권한 상승 열거]]의 해당 기법 조건 확인 |
| 고권한 서비스·작업과 수정 가능한 경로가 함께 표시됨 | 현재 계정이 제어할 수 있을 가능성이 있는 객체 | 실행 계정, 정확한 binary/script 경로, ACL과 trigger를 각각 수동 확인 |
| 저장 비밀번호·key·ticket 후보가 표시됨 | 문자열 또는 파일을 읽을 수 있음 | 소유 주체·형식·대상 서비스를 식별하고 실제 인증은 별도 검증 |
| OS·소프트웨어 취약 후보가 표시됨 | 버전 기반 후보 | 설치 build·patch·아키텍처와 해당 취약점의 구체적 전제 확인 |
| 특정 카테고리에서 접근 거부 | 현재 token으로 그 검사를 완료하지 못함 | 다른 카테고리 결과는 유지하고 객체별 조회 권한 또는 수동 명령 확인 |


## 주요 하위 도구

| 도구 | 대상 환경 | 사용하는 상황 |
|---|---|---|
| `linpeas.sh` | Linux, Unix, macOS | 셸에서 스크립트 실행이 가능할 때 |
| `winPEASx86.exe` | 32-bit Windows | 32-bit Windows 또는 x86 build만 실행 가능한 환경 |
| `winPEASx64.exe` | 64-bit Windows | 일반적인 64-bit Windows 환경 |
| `winPEASany_ofs.exe` | Windows/.NET | 공식 release의 단일 C# build를 사용할 수 있고 .NET Framework 4.5.2 이상일 때 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 실행할 하위 도구 선택 | PEASS 자체는 묶음이며 단일 검사 결과를 내지 않음 | Linux는 [[linpeas]], Windows는 해당 winPEAS binary의 도움말과 결과로 이동 |
| release asset 이름 | 대상 OS/아키텍처에 맞는 하위 도구 식별 | `linpeas.sh`, `winPEASx86.exe`, `winPEASx64.exe`, `winPEASany_ofs.exe` 중 현재 release와 환경에 맞는 파일 선택 |
| 하위 도구별 색상/카테고리 출력 | 각 OS 열거기가 만든 후보이며 PEASS 공통 성공 신호는 아님 | Linux는 [[linpeas]], Windows는 [[Windows 권한 상승 열거]]에서 작은 기본 명령으로 재확인 |

## 관련 공격기법

- [[Linux 권한 상승 열거]]
- [[Windows 권한 상승 열거]]

## 참고 링크

- [PEASS-ng GitHub](https://github.com/peass-ng/PEASS-ng)
- [winPEAS 공식 README와 실행 인자](https://github.com/peass-ng/PEASS-ng/tree/master/winPEAS/winPEASexe)
- [Microsoft: 설치된 .NET Framework 버전 확인](https://learn.microsoft.com/dotnet/framework/install/how-to-determine-which-versions-are-installed)
