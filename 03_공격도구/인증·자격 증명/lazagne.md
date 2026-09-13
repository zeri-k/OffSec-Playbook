---
tags:
  - 기능/자격증명수집
실행환경: ["Linux", "Windows", "macOS"]
필요권한: ["로컬 쉘"]
필요조건: ["대상 OS용 LaZagne 실행 파일 또는 script"]
결과: ["자격증명"]
---

# lazagne

## 도구 개요

`LaZagne`은 현재 권한으로 읽을 수 있는 브라우저·메일·데이터베이스·애플리케이션 저장 자격 증명을 모듈별로 수집하는 도구다. 초기 접근한 호스트에서 저장된 자격 증명 후보를 빠르게 점검할 때 적합하다.

## 필요한 입력과 실행 환경

`<LAZAGNE_OUTPUT_DIRECTORY>`는 LaZagne를 실행하는 호스트의 새 출력 디렉터리(예: `/tmp/lazagne-output`)이며 `<LAZAGNE_OUTPUT_FILE>`은 바로 앞 `find` 출력에서 얻은 생성 파일명이다. `<UPLOADED_LAZAGNE_PATH>`는 대상 Windows 호스트에 올린 실행 파일의 정확한 경로다.

출력 경로와 파일명은 LaZagne를 실행하는 Linux 또는 Windows 호스트 기준이다. 전송해 분석하는 파일은 생성 파일과 구분하고, 새 결과 파일만 정리 대상으로 기록한다.

- 실행 위치: 초기 접근을 얻은 Windows, Linux 또는 macOS 호스트의 로컬 세션
- 필요한 입력: 대상 OS에 맞는 LaZagne 실행 파일/스크립트와 검사할 module
- 권한 조건: 현재 사용자 profile을 읽을 수 있어야 하며, 다른 사용자나 시스템 저장소는 추가 권한이 필요하다.
- 파일 조건: 실행 파일을 대상에 반입했다면 실행 전 부재를 확인한 exact 경로를 기록한다. `-oN`·`-oJ`·`-oA`는 민감한 결과 파일을 만들므로 콘솔 출력만 필요한 경우 지정하지 않는다.


## 표준 사용법

```bash
laZagne.exe all
python3 laZagne.py all
```

## 대표 예시

### Windows 전체 credential 수집

```powershell
.\LaZagne.exe all
```

### 브라우저 저장 비밀번호만 확인

```powershell
.\LaZagne.exe browsers
```

### Linux에서 JSON 결과 저장

```bash
test ! -e '<LAZAGNE_OUTPUT_DIRECTORY>'
mkdir '<LAZAGNE_OUTPUT_DIRECTORY>'
python3 laZagne.py all -oJ -output '<LAZAGNE_OUTPUT_DIRECTORY>'
find '<LAZAGNE_OUTPUT_DIRECTORY>' -maxdepth 1 -type f -print
```

마지막 출력의 exact 파일명을 `<LAZAGNE_OUTPUT_FILE>`로 기록한다. 출력 directory가 이미 있으면 사용하지 말고 이번 실행 전용 경로를 다시 정한다.

## 주요 옵션

| 옵션/모듈 | 설명 |
| --- | --- |
| `all` | 모든 지원 모듈 실행 |
| `browsers` | 브라우저 저장 credential 검색 |
| `-oN` | 일반 텍스트 결과 저장 |
| `-oJ` | JSON 결과 저장 |
| `-vv` | 상세 출력 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 저장된 credential 출력 | 브라우저/클라이언트/시스템 저장 자격 증명 확보 | 서비스별 로그인과 권한 범위 확인 |
| 일부 모듈 access denied | 현재 권한으로 접근 불가 | 관리자 권한, 사용자 컨텍스트, 대상 프로필 확인 |
| 결과 없음 | 저장 credential이 없거나 보호됨 | 다른 사용자 프로필, DPAPI, 브라우저 세션 파일 확인 |
| AV/EDR 차단 | 도구 실행 탐지 | 수동 파일 수집 또는 오프라인 분석 방식 검토 |

## 변경 영향과 복구

콘솔 출력만 사용하면 LaZagne 자체가 결과 파일을 만들지는 않는다. 실행 파일을 이번 작업에서 반입했거나 `-oN`·`-oJ`·`-oA`를 사용했다면, 후속 검증이 끝난 뒤 기록한 exact 파일만 제거한다.

```powershell
Remove-Item -LiteralPath '<UPLOADED_LAZAGNE_PATH>' -Force
Test-Path -LiteralPath '<UPLOADED_LAZAGNE_PATH>'
```

```bash
rm -- '<LAZAGNE_OUTPUT_FILE>'
rmdir '<LAZAGNE_OUTPUT_DIRECTORY>'
test ! -e '<LAZAGNE_OUTPUT_DIRECTORY>'
```

실행 파일이 원래 존재했다면 첫 두 명령을 적용하지 않는다. 출력 파일이 여러 개면 실행 직후 기록한 exact 경로마다 `rm --`을 반복하고, directory가 비어 있을 때만 `rmdir`로 이번에 만든 directory를 제거한다. 마지막 확인이 성공해야 로컬 결과 정리가 확인된다. 화면·terminal log와 이미 사용한 credential은 파일 삭제로 되돌릴 수 없다.

## 관련 공격기법

- [[Windows 저장 자격증명 수집]]
- [[Windows 파일 자격증명 검색]]
- [[Linux 파일 자격증명 검색]]

## 참고 링크

- [AlessandroZ LaZagne: Usage](https://github.com/AlessandroZ/LaZagne#usage)
