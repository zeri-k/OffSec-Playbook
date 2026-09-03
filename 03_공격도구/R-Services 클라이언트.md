---
tags:
  - 환경/unix
  - 서비스/r-services
  - 기능/프로토콜접근
  - 기능/원격실행
실행환경: ["Linux", "Unix"]
필요조건: ["R-Services 접근 가능", "사용자명", "trust 관계 또는 rexec 비밀번호"]
결과: ["사용자 정보", "명령 출력", "셸"]
---

# R-Services 클라이언트

## 도구 개요

R-Services 클라이언트는 `rlogin`, `rsh`, `rexec`, `rusers`를 이용해 Unix R-Services의 원격 로그인·명령 실행과 노출된 사용자 정보를 확인하는 도구 묶음이다. 호스트 신뢰에 따른 비밀번호 없는 접근과 `rexec` 인증, `rusersd` 정보 노출을 각각 실제 연결로 구분해 검증할 때 적합하다.

## 필요한 입력과 실행 환경

- 입력: 대상 호스트, 사용자명, trust 조건 또는 `rexec` 비밀번호
- 실행 환경: `rlogin`, `rsh`, `rexec`, `rusers` 클라이언트가 설치된 Linux/Unix 호스트
- `rusers` 조회에는 대상의 `rusersd` RPC 서비스가 별도로 필요하다.

## 표준 사용법

```bash
rlogin -l <USER> <TARGET>
rsh -l <USER> <TARGET> <COMMAND>
rexec -l <USER> -p '<PASSWORD>' <TARGET> <COMMAND>
rusers -al <TARGET>
```

## 대표 예시

### 원격 로그인과 명령 실행

```bash
rlogin -l <USER> <TARGET>
rsh -l <USER> <TARGET> id
rexec -l <USER> -p '<PASSWORD>' <TARGET> id
```

확인할 출력:

- 비밀번호 요구 여부, trust 거부, 원격 사용자 context, 명령 출력

### 로그인 사용자 정보

```bash
rusers -al <TARGET>
```

`rusers`는 대상의 `rusersd` RPC 서비스가 있어야 한다. `rwho`는 현재 호스트가 수신한 `rwhod` 정보를 표시하는 명령이므로 임의 대상 질의 명령으로 사용하지 않는다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-l <USER>` | 원격 사용자명 지정 | `rlogin`, `rsh`, `rexec`의 사용자 컨텍스트 지정 |
| `-p <PASSWORD>` | `rexec` 비밀번호 지정 | trust 대신 비밀번호 인증을 확인할 때 |
| `rusers -a` | 빈 응답을 포함한 호스트 정보 요청 | `rusersd` 노출 범위를 확인할 때 |
| `rusers -l` | 긴 형식으로 사용자 정보 표시 | 사용자명과 로그인 단서를 함께 볼 때 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 비밀번호 없이 셸 또는 명령 출력 | `.rhosts`/`hosts.equiv` trust 가능성 | 현재 사용자와 신뢰 범위 확인 |
| `Permission denied` | 사용자명, source host, trust 조건 불일치 | 계정명과 신뢰 파일 단서 재확인 |
| `Connection refused` | 해당 R-Service 비활성 | 512/513/514 서비스별 구분 |
| 사용자 목록 반환 | `rusersd` 정보 노출 | 계정 후보와 호스트 관계 정리 |

## 관련 공격기법

- [[R-Services trust 기반 원격 접근]]

## 관련 서비스

- [[512_513_514_R-Services]]

## 실전 진입

- 포트 512/513/514가 식별되면 [[512_513_514_R-Services]]에서 service별 노출을 먼저 구분하고, trust 조건이 보일 때만 [[R-Services trust 기반 원격 접근]]으로 이어간다.
- `nmap`으로 포트와 서비스 단서를 다시 확인할 때는 [[nmap]]의 service scan 결과를 이 문서의 실제 client 연결로 재검증한다.
