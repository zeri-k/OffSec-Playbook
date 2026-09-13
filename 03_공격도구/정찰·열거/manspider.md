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
manspider <TARGET> -u '<USER>' -p '<PASSWORD>' -l '<MANSPIDER_OUTPUT_DIR>' [options]
```

현재 MANSPIDER 2.x는 match 파일을 기본적으로 loot 경로에 다운로드하고 log를 남긴다. 로컬 사본이 필요 없으면 `-n`으로 download를 끄고, 필요하면 이번 작업의 고유 `-l` 경로를 지정한다.

## 대표 예시

### SMB share에서 password 문자열 검색

```bash
MANSPIDER_OUTPUT_DIR="$(mktemp -d "${PWD}/manspider.XXXXXX")"
printf 'MANSPIDER output: %s\n' "$MANSPIDER_OUTPUT_DIR"
docker run --rm -v "$MANSPIDER_OUTPUT_DIR:/root/.manspider" blacklanternsecurity/manspider <TARGET> --sharenames '<SHARE>' -c 'passw' -u '<USER>' -p '<PASSWORD>'
```

### 검색 결과를 로컬 loot 디렉터리에 보존

```text
-v <MANSPIDER_OUTPUT_DIR>:/root/.manspider 로 Docker volume을 연결하면 다운로드된 매칭 파일과 로그를 이번 작업의 고유 로컬 경로에 남길 수 있다.
```

### 파일명·content match만 보고 download하지 않기

```bash
manspider <TARGET> --sharenames '<SHARE>' -f '<FILENAME_REGEX>' -c '<CONTENT_REGEX>' -n -l "$MANSPIDER_OUTPUT_DIR" -u '<USER>' -p '<PASSWORD>'
```

`-n`은 match 파일 download를 끄지만 log 생성과 원격 SMB 조회 흔적까지 없애지 않는다. 출력 경로·share·match를 확인한 뒤 필요한 파일만 별도 수집한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-c` | 파일 내용에서 찾을 문자열 지정 |
| `-f`, `-e` | 파일명 regex 또는 확장자로 범위 축소 |
| `-u`, `-p` | SMB 인증 사용자와 비밀번호 |
| `-d` | 도메인 지정 |
| `-l` | loot·log용 로컬 출력 디렉터리 |
| `-n` | match 파일을 다운로드하지 않음 |
| `--sharenames` | 지정한 공유로 검색 범위 제한 |
| target | 단일 IP, 호스트명, 대역 지정 |
| Docker volume | loot와 결과를 호스트에 보존 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| keyword 매칭 파일 출력 | share 안에서 민감 파일 후보 발견 | 파일을 내려받아 credential, key, 설정값 여부 확인 |
| 일부 share만 검색 | 현재 credential로 해당 share를 읽을 수 있음 | 검색하지 못한 share의 접근 권한을 별도로 확인 |
| 결과 없음 | 키워드/확장자/권한 범위 제한 가능 | 검색어, 확장자, share 목록, 계정 권한 보강 |
| timeout/접근 거부 | 네트워크 지연 또는 share 권한 부족 | 특정 share로 범위 축소, credential 재확인 |

## 변경 영향과 정리

MANSPIDER는 원격 공유를 읽고 `<MANSPIDER_OUTPUT_DIR>`에 log와, `-n`이 없으면 match 파일 사본을 만든다. `<MANSPIDER_OUTPUT_DIR>`은 Linux 실행 호스트의 새 고유 디렉터리이고, `<TARGET>`·`<SHARE>`는 각각 SMB 호스트와 검색할 공유(예: `fileserver.example.test`, `Public`)다. `<USER>`·`<PASSWORD>`는 SMB 인증 입력, `<FILENAME_REGEX>`·`<CONTENT_REGEX>`는 도구가 적용할 정규식이다. 원격 파일을 수정하지는 않지만 SMB·AD 감사 흔적은 남을 수 있다. 필요한 분석 뒤 이번 실행이 만든 고유 디렉터리 안의 파일·빈 디렉터리만 정리한다.

```bash
find "$MANSPIDER_OUTPUT_DIR" -xdev -depth -type f -delete
find "$MANSPIDER_OUTPUT_DIR" -xdev -depth -type d -empty -delete
test ! -e "$MANSPIDER_OUTPUT_DIR"
```

경로가 남으면 예상하지 않은 파일 형식·소유자·열린 process를 확인하고 정리 완료로 표시하지 않는다. 다른 `~/.manspider`나 이름 pattern으로 출력을 일괄 삭제하지 않는다.

## 관련 공격기법

- [[SMB 공유 자격증명 수집]]

## 참고 링크

- [MANSPIDER 공식 저장소 — usage·loot·filter](https://github.com/blacklanternsecurity/MANSPIDER)
