---
tags:
  - 환경/linux
  - 기능/권한상승
  - 기능/열거
실행환경: ["Linux"]
필요권한: ["로컬 쉘"]
필요조건: ["LinEnum.sh"]
결과: ["정보", "권한 상승 단서", "자격증명"]
---

# LinEnum

## 도구 개요

LinEnum은 Linux 시스템의 sudo 설정, SUID 파일, cron, 서비스, 파일 권한과 자격 증명 흔적을 한 번에 열거하는 셸 스크립트다. 초기 로컬 열거에서 권한 상승과 자격 증명 검색 후보를 빠르게 모을 때 유용하며, 출력은 검증이 필요한 단서 목록이다.

## 필요한 입력과 실행 환경

- 실행 위치: 초기 접근을 얻은 Linux 호스트의 Bash shell
- 필요한 입력: `LinEnum.sh`와 선택할 검사/출력 옵션
- 환경 조건: 스크립트 전송·실행 권한이 필요하며, 현재 사용자 권한에 따라 읽을 수 있는 설정과 credential 범위가 달라진다.


## 표준 사용법

```shell
chmod +x LinEnum.sh
./LinEnum.sh
```

옵션 없이 실행하면 제한적인 기본 스캔을 수행한다. 결과 저장이나 thorough scan이 필요하면 옵션을 추가한다.

## 대표 예시

### 기본 열거

```shell
chmod +x LinEnum.sh
./LinEnum.sh
```

현재 사용자 권한으로 기본 권한 상승 체크를 수행한다.

### 자세한 검사와 리포트 저장

```shell
./LinEnum.sh -t -r linenum-report
```

느리지만 더 많은 검사를 수행하고 리포트 이름을 지정한다.

### 키워드 기반 파일 검색

```shell
./LinEnum.sh -k password -t
```

파일 내용에서 특정 키워드를 찾고 싶을 때 사용한다.

### 결과 내보내기 위치 지정

```shell
./LinEnum.sh -e /tmp/linenum-export -r report
```

export 기능을 사용할 때 저장 위치와 리포트 이름을 지정한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-k <keyword>` | 여러 파일에서 특정 키워드 검색 |
| `-e <path>` | export 위치 지정 |
| `-t` | thorough scan 수행 |
| `-s` | 현재 사용자 비밀번호를 받아 `sudo` 권한을 확인. 비밀번호가 process 입력·터미널 기록에 노출될 수 있고 `sudo` 인증 시도가 남을 수 있으므로, 현재 계정의 비밀번호를 입력하는 경우에만 사용 |
| `-r <name>` | 리포트 이름 지정 |
| `-h` | 도움말 출력 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `[-]`, `[+]` 검사 섹션 | LinEnum의 시스템/권한/파일 검색 구간 | 관련 섹션의 원본 명령과 파일 경로를 수동으로 확인 |
| `-r <name>` report 파일 | 지정한 이름으로 저장된 LinEnum 결과 | 셸 출력과 report가 같은 실행 시점의 것인지 확인 |
| `-k <keyword>` 일치 경로 | 키워드 검색이 찾은 후보 파일 | 비밀값 노출 여부와 현재 사용자 읽기 권한을 재확인 |
| `Permission denied` | 현재 계정으로 검사할 수 없는 경로 | 실패를 취약점으로 해석하지 않고 권한 경계를 기록 |

## 관련 공격기법

- [[Linux 권한 상승 열거]]
- [[Linux 파일 자격증명 검색]], [[Linux Shell History 자격증명 검색]], [[Linux 개인키 검색]]

## 참고 링크

- [LinEnum GitHub](https://github.com/rebootuser/LinEnum)
