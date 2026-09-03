---
tags:
  - 기능/취약점검증
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["제품명, 버전, CVE, EDB-ID 또는 Nmap XML"]
결과: ["취약점 후보", "PoC"]
---

# searchsploit

## 도구 개요

`searchsploit`은 Exploit-DB 로컬 미러에서 제품명·버전·CVE·EDB-ID에 맞는 공개 exploit과 PoC를 검색하고 파일 경로·내용을 확인하거나 복사한다. 서비스 단서에서 검토할 코드를 빠르게 좁힐 때 적합하지만, 검색 일치만으로 대상의 취약 여부가 확정되지는 않는다.

## 필요한 입력과 실행 환경

- 실행 환경: Exploit-DB 로컬 데이터베이스가 설치된 Linux 호스트
- 입력: 제품명, 버전, CVE 또는 EDB-ID
- 선택 입력: Nmap XML 결과 파일

## 표준 사용법

```shell
searchsploit <검색어>
searchsploit [옵션] <검색어>
```

검색 결과만 신뢰하지 말고 exploit 파일 내부의 대상 버전, 전제 조건, 파라미터, 인증 필요 여부를 반드시 확인한다.

## 대표 예시

### 키워드로 exploit 검색

```shell
searchsploit "Simple Backup"
```

제품명이나 플러그인명으로 관련 exploit을 찾는다.

### 버전까지 포함해 검색

```shell
searchsploit "OpenSSH 7.2"
```

버전을 포함하면 후보가 줄어들지만, 백포팅된 배포판 패키지는 단순 버전 매칭이 틀릴 수 있다.

### exploit 경로 확인

```shell
searchsploit -p 51937
```

EDB-ID의 로컬 파일 경로와 Exploit-DB URL을 확인한다.

### exploit 파일 복사

```shell
searchsploit -m 51937
```

현재 디렉터리로 exploit 파일을 복사한다. 복사한 뒤 코드를 읽고 필요한 URL, 포트, 파일명, 인증값을 수정한다.

### exploit 내용 바로 확인

```shell
searchsploit -x 51937
```

복사하기 전에 파일 내용을 빠르게 확인한다.

### Exploit-DB 웹 링크 확인

```shell
searchsploit -w 51937
```

웹 페이지 링크를 확인한다. 로컬 파일보다 최신 설명이나 댓글이 필요한 경우 유용하다.

### Nmap XML 결과로 검색

```shell
nmap -sV -oX target.xml <TARGET>
searchsploit --nmap target.xml
```

Nmap 서비스 탐지 결과를 바탕으로 exploit 후보를 자동 검색한다.

### 데이터베이스 업데이트

```shell
searchsploit -u
```

로컬 exploitdb 패키지의 데이터베이스를 갱신한다.

## 주요 옵션

| 옵션 | 의미 |
| --- | --- |
| `-u` | Exploit-DB 로컬 데이터베이스 업데이트 |
| `-m <EDB-ID>` | exploit 파일을 현재 디렉터리로 복사 |
| `-p <EDB-ID>` | exploit 경로와 URL 정보 출력 |
| `-x <EDB-ID>` | exploit 파일 내용 확인 |
| `-w` | Exploit-DB 웹 링크 출력 |
| `--nmap <xml>` | Nmap XML 결과 기반 검색 |
| `--exclude <term>` | 특정 단어가 포함된 결과 제외 |
| `-j` | JSON 형식 출력 |
| `-t` | 제목만 대상으로 검색 |
| `-c` | 대소문자 구분 검색 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 제품/버전과 맞는 exploit 후보 출력 | 공개 exploit 검토 대상 존재 | exploit 조건, 인증 필요 여부, 영향 버전 확인 |
| 여러 exploit 후보 | 모듈/플러그인/버전 범위가 넓음 | 대상 배너와 파일 경로, 플랫폼을 기준으로 좁힘 |
| 결과 없음 | Exploit-DB에 직접 매칭 없음 | 제품명 변형, CVE, GitHub/벤더 advisory로 재검색 |
| exploit 복사 후 실행 전 | 코드 검토 필요 | 하드코딩 주소, payload, Python 버전, 의존성 확인 |

## 관련 공격기법

- [[Public Exploit 검토와 검증]]
