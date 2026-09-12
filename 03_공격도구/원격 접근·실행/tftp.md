---
tags:
  - 서비스/tftp
  - 기능/프로토콜접근
  - 기능/파일전송
실행환경: ["Linux"]
필요조건: ["TFTP 접근 가능", "파일명 후보"]
결과: ["파일", "읽기 또는 쓰기 결과"]
---

# tftp

## 도구 개요

`tftp`는 TFTP 서버와 파일을 `get`·`put`하는 단순 파일 전송 클라이언트다. 알려진 설정·백업 파일명을 직접 요청하거나 제한적으로 업로드 허용 여부를 확인할 때 적합하며, 일반적인 디렉터리 목록 기능은 제공하지 않는다.

## 필요한 입력과 실행 환경

- 입력: 대상 TFTP 서버, 추정한 원격 파일명, 다운로드 경로 또는 업로드할 로컬 파일
- 실행 환경: TFTP 클라이언트가 설치된 Linux 호스트
- TFTP에는 디렉터리 목록 명령이 없으므로 원격 파일명을 미리 알아야 한다.

## 표준 사용법

```bash
tftp <TARGET>
tftp <TARGET> -m binary -c get <REMOTE_FILE> <LOCAL_FILE>
```

Linux/Unix 구현체의 대화형 명령과 `-c` 지원 여부는 배포판별로 다르므로 `tftp --help`로 확인한다.

## 대표 예시

### 대화형 파일 다운로드

```bash
tftp <TARGET>
tftp> binary
tftp> get <REMOTE_FILE> <LOCAL_FILE>
tftp> quit
```

### 대화형 파일 업로드

```bash
tftp <TARGET>
tftp> binary
tftp> put <LOCAL_FILE> <REMOTE_FILE>
tftp> quit
```

### 비대화형 파일 다운로드

```bash
tftp <TARGET> -m binary -c get <REMOTE_FILE> <LOCAL_FILE>
```

### Windows 기본 클라이언트

```cmd
tftp [-i] <TARGET> get <REMOTE_FILE> [<LOCAL_FILE>]
tftp [-i] <TARGET> put <LOCAL_FILE> [<REMOTE_FILE>]
```

## 주요 옵션과 명령

| 옵션·명령 | 의미 | 사용하는 상황 |
|---|---|---|
| `-m binary` / `binary` | octet 전송 모드 사용 | 설정, 펌웨어, 압축 파일 전송 |
| `-c <COMMAND>` | 접속 후 명령을 바로 실행 | 구현체가 지원할 때 비대화형 작업 |
| `get <REMOTE> <LOCAL>` | 원격 파일 다운로드 | 추정한 설정·백업 파일 확인 |
| `put <LOCAL> <REMOTE>` | 로컬 파일 업로드 | 쓰기 허용 여부를 제한적으로 검증 |
| `status` | 현재 전송 모드와 서버 상태 확인 | 대화형 세션 점검 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 파일 수신 및 로컬 파일 생성 | 원격 파일 읽기 가능 | 설정·계정·내부 주소 확인 |
| 업로드 후 같은 파일 재수신 | 파일 쓰기 확인 | 쓰기 경로의 실제 영향 검증 |
| `File not found` | 서버 응답은 있으나 파일명 불일치 가능 | 다른 파일명 후보 시도 |
| timeout | UDP 필터링, 서버 미응답, 응답 경로 문제 | 패킷 캡처와 UDP 접근성 확인 |
| `Access violation` | 읽기 또는 쓰기 정책 제한 | 반대 작업과 다른 파일명 확인 |

## 버전과 환경 차이

- Linux/Unix `tftp`는 구현체에 따라 `-m binary -c` 또는 대화형 `binary`, `get`, `put` 흐름을 지원한다. 지원하지 않는 옵션을 쓰기 전에 `tftp --help`를 확인한다.
- Windows 기본 `tftp`는 `tftp [-i] <host> {get|put} <source> [destination]` 문법을 쓴다. binary/octet 전송이 필요하면 `-i`를 사용한다.

## 관련 공격기법

- [[TFTP 설정 파일 수집]]

## 관련 서비스

- [[TFTP 서비스]]

## 참고 링크

- [GNU Inetutils tftp](https://www.gnu.org/software/inetutils/manual/inetutils.html#tftp-invocation)
- [Windows tftp 명령](https://learn.microsoft.com/en-ie/windows-server/administration/windows-commands/tftp)
