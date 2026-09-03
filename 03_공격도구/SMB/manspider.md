---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/자격증명수집
실행환경: ["Linux"]
필요조건: ["SMB 인증 정보", "공유 읽기 권한"]
결과: ["파일", "자격증명", "정보"]
---

# manspider

## 도구 개요

`manspider`는 원격 SMB 공유의 파일명·확장자·내용을 검색해 비밀번호·키·설정이 포함된 파일 후보를 찾는 도구다. 여러 호스트와 공유에서 정해진 keyword를 일괄 검색하고 매칭 파일을 로컬에 보존해야 할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 SMB share에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 host/대역, 도메인·사용자·비밀번호와 검색할 keyword
- 범위 입력: 검색할 share, 확장자, 최대 파일 크기와 콘텐츠/파일명 검색 방식


## 표준 사용법

```bash
manspider <target> [options]
```

## 대표 예시

### SMB share에서 password 문자열 검색

```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <TARGET> -c 'passw' -u '<USER>' -p '<PASSWORD>'
```

### 검색 결과를 로컬 loot 디렉터리에 보존

```text
-v ./manspider:/root/.manspider 로 Docker volume을 연결하면 다운로드된 매칭 파일과 로그를 로컬에 남길 수 있다.
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-c` | 파일 내용에서 찾을 문자열 지정 |
| `-u`, `-p` | SMB 인증 사용자와 비밀번호 |
| `-d` | 도메인 지정 |
| target | 단일 IP, 호스트명, 대역 지정 |
| Docker volume | loot와 결과를 호스트에 보존 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| keyword 매칭 파일 출력 | share 안에서 민감 파일 후보 발견 | 파일을 내려받아 credential, key, 설정값 여부 확인 |
| 일부 share만 검색 | 현재 credential로 해당 share를 읽을 수 있음 | 검색하지 못한 share의 접근 권한을 별도로 확인 |
| 결과 없음 | 키워드/확장자/권한 범위 제한 가능 | 검색어, 확장자, share 목록, 계정 권한 보강 |
| timeout/접근 거부 | 네트워크 지연 또는 share 권한 부족 | 특정 share로 범위 축소, credential 재확인 |

## 관련 공격기법

- [[SMB 공유 자격증명 수집]]
