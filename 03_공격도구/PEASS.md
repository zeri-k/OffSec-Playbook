---
tags:
  - 기능/권한상승
  - 기능/열거
실행환경: ["Linux", "Windows", "macOS", "Unix"]
필요권한: ["대상 호스트의 로컬 셸"]
필요조건: ["대상 OS와 아키텍처에 맞는 PEASS 하위 도구"]
결과: ["정보", "권한 상승 단서", "자격증명"]
---

# PEASS

## 도구 개요

`PEASS`는 운영체제별 권한 상승 열거 도구인 `linPEAS`와 `winPEAS` 등을 묶은 도구 모음이다. 대상 환경에 맞는 하위 도구로 서비스·작업·파일 권한·자격 증명·커널 관련 후보를 빠르게 수집할 때 적합하며, 강조된 항목은 수동 검증이 필요한 후보다.

## 필요한 입력과 실행 환경

- 실행 환경: 로컬 셸을 확보한 Linux, Windows, macOS 또는 Unix 호스트
- 입력: 대상 OS와 아키텍처에 맞는 `linpeas`, `winPEAS` 등 PEASS 하위 도구
- 선택 입력: 긴 출력을 보존할 로컬 파일 경로

## 표준 사용법

PEASS는 도구 묶음이므로 대상 OS에 맞는 하위 스크립트/바이너리를 받아 실행한다.

```shell
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

## 대표 예시

### Linux 권한 상승 자동 열거

```shell
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee linpeas.out
```

대상 Linux 호스트에서 실행해 의심 지점과 권한 상승 후보를 확인한다.

### 결과를 나중에 정리하기

```shell
/tmp/linpeas.sh | tee /tmp/linpeas_$(hostname).txt
```

셸이 불안정하거나 출력이 길 때 결과 파일을 남겨 둔다.


## 주요 하위 도구

| 도구 | 대상 환경 | 사용하는 상황 |
|---|---|---|
| `linpeas.sh` | Linux, Unix, macOS | 셸에서 스크립트 실행이 가능할 때 |
| `winPEASx86.exe` | 32-bit Windows | x86 프로세스 또는 OS 환경 |
| `winPEASx64.exe` | 64-bit Windows | 일반적인 64-bit Windows 환경 |
| `winPEASany.exe` | Windows/.NET | 단일 .NET 빌드가 필요한 환경 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 실행할 하위 도구 선택 | PEASS 자체는 묶음이며 단일 검사 결과를 내지 않음 | Linux는 [[linpeas]], Windows는 해당 winPEAS binary의 도움말과 결과로 이동 |
| release asset 이름 | 대상 OS/아키텍처에 맞는 하위 도구 식별 | `linpeas.sh`, `winPEASx86.exe`, `winPEASx64.exe`, `winPEASany.exe` 중 환경에 맞는 파일 선택 |
| 하위 도구별 색상/카테고리 출력 | 각 OS 열거기가 만든 후보이며 PEASS 공통 성공 신호는 아님 | 해당 하위 도구 문서와 수동 검증 절차에서 재현 |

## 관련 공격기법

- [[Linux 권한 상승 열거]]
- [[Windows 권한 상승 열거]]

## 참고 링크

- [PEASS-ng GitHub](https://github.com/peass-ng/PEASS-ng)
