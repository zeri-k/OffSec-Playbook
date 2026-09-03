---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/자격증명수집
실행환경: ["Windows"]
필요권한: ["도메인 사용자 세션과 해당 공유의 읽기 권한"]
필요조건: ["도메인 연결", "접근 가능한 SMB 공유"]
결과: ["민감 파일 후보", "자격증명 단서"]
---

# Snaffler

## 도구 개요

`Snaffler`는 AD 도메인의 SMB 공유를 순회하며 규칙에 맞는 자격 증명·설정·키와 민감 문서 후보를 찾는 Windows 도구다. 도메인 환경에서 접근 가능한 공유가 많아 수동 검색이 어려울 때 민감 파일 후보를 우선순위화하는 데 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: 도메인에 연결된 Windows 호스트
- 입력: `Snaffler.exe`와 도메인 사용자 세션
- 네트워크 조건: SMB 공유 열거 및 읽기 가능
- 선택 입력: 검색할 특정 UNC share


## 표준 사용법

```cmd
Snaffler.exe -s -d <DOMAIN> -o <OUTPUT_FILE> -v data
```

## 대표 예시

### 도메인 share 기본 credential hunting

```cmd
C:\Users\Public>Snaffler.exe -s -d <DOMAIN> -o snaffler.log -v data
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-s` | 도메인에서 접근 가능한 share 검색 실행 |
| `-d <DOMAIN>` | 검색할 AD 도메인 지정 |
| `-o <FILE>` | 결과를 파일로 저장 |
| `-v data` | 민감 데이터 후보를 포함하는 상세 출력 수준 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| rule hit 또는 민감 파일 후보 출력 | network share에서 후속 공격 단서 발견 | 파일 내용 확인 후 credential/키/설정값 재사용 가능성 검토 |
| share 목록 순회 | 도메인 share 검색 진행 | 우선순위 높은 hit부터 수동 검증 |
| 결과 없음 | 권한 범위 또는 rule 한계 | 다른 계정, 검색 rule, 특정 share 지정 확인 |
| 접근 오류 | share 권한 또는 네트워크 문제 | 계정 권한, UNC 경로, 도메인 연결 상태 확인 |

## 관련 공격기법

- [[Snaffler로 도메인 SMB 공유 민감 파일 탐색]]
- [[SMB 공유 자격증명 수집]]
- [[Windows 파일 자격증명 검색]]
