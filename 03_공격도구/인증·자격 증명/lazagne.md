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

- 실행 위치: 초기 접근을 얻은 Windows, Linux 또는 macOS 호스트의 로컬 세션
- 필요한 입력: 대상 OS에 맞는 LaZagne 실행 파일/스크립트와 검사할 module
- 권한 조건: 현재 사용자 profile을 읽을 수 있어야 하며, 다른 사용자나 시스템 저장소는 추가 권한이 필요하다.


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
.\LaZagne.exe browsers -oN
```

### Linux에서 JSON 결과 저장

```bash
python3 laZagne.py all -oJ
```

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

## 관련 공격기법

- [[Windows 저장 자격증명 수집]]
- [[Windows 파일 자격증명 검색]]
- [[Linux 파일 자격증명 검색]]
