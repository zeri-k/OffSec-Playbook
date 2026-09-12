---
tags:
  - 환경/linux
  - 기능/권한상승
  - 기능/열거
실행환경: ["Linux"]
필요권한: ["로컬 쉘"]
필요조건: ["linpeas.sh"]
결과: ["정보", "권한 상승 단서", "자격증명"]
---

# linpeas

## 도구 개요

linPEAS는 Linux 호스트의 권한, 파일, 서비스, 자격 증명, 컨테이너와 네트워크 구성을 폭넓게 검사해 권한 상승 후보를 강조하는 열거 스크립트다. 수동 점검 범위를 빠르게 좁히기 좋지만 강조된 항목은 취약점이나 root 획득을 확정하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: 초기 접근을 얻은 Linux 호스트의 shell
- 필요한 입력: 대상 환경에 맞는 `linpeas.sh`와 필요하면 결과 저장 경로
- 환경 조건: 스크립트 전송·실행 권한이 필요하며, 현재 사용자 권한과 설치 명령에 따라 수집 범위가 달라진다.


## 표준 사용법

```bash
chmod +x linpeas.sh
./linpeas.sh
```

## 대표 예시

### 대상에서 기본 권한 상승 점검

```bash
chmod +x linpeas.sh
./linpeas.sh
```

### 다운로드 후 결과 저장

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee linpeas.out
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-a` | 더 많은 검사를 포함해 실행 |
| `-s` | stealth/quiet 성격의 검사 모드 |
| `-q` | 출력량 감소 |
| `-h` | 도움말 출력 |
| `tee` 활용 | 결과를 파일로 보존하며 동시에 화면 출력 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `╔══╣`로 시작하는 검사 카테고리 | linPEAS가 권한, 파일, 서비스, 네트워크 검사를 구분해 출력 | 관심 카테고리의 실제 경로와 권한을 수동 명령으로 재현 |
| 강조된 `PEASS` finding | 스크립트 규칙에 걸린 권한 상승 또는 자격 증명 후보 | 파일 소유자, 쓰기 권한, 실행 조건을 개별 확인 |
| `linpeas.out`에 저장된 전체 출력 | `tee`로 현재 세션 결과를 보존 | 현재 사용자와 hostname을 기록하고 관련 finding만 추려 재검증 |
| `WARNING: ...` 또는 권한 거부 | 현재 사용자에게 읽기/실행 권한이 없는 검사 항목 | 권한 우회로 단정하지 말고 현재 권한 범위를 기준으로 판단 |

## 관련 공격기법

- [[Linux 권한 상승 열거]]
- [[Linux 파일 자격증명 검색]], [[Linux Shell History 자격증명 검색]], [[Linux 개인키 검색]]
