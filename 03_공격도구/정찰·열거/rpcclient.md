---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/열거
  - 기능/프로토콜접근
실행환경: ["Linux"]
필요조건: ["SMB 또는 MSRPC 접근", "익명 세션 또는 계정 인증 조건"]
결과: ["사용자", "그룹", "공유", "정책 정보"]
---

# rpcclient

## 도구 개요

`rpcclient`는 SMB를 통해 MSRPC 명령을 대화형 또는 단일 명령 방식으로 실행해 사용자·그룹·공유·도메인 정보와 비밀번호 정책을 조회한다. 자동 열거 결과를 특정 RPC 호출로 교차 확인하거나 RID별 정보를 세밀하게 조회할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 입력: SMB/MSRPC 대상 호스트
- 인증 입력: null session 또는 사용자명과 비밀번호
- 자동화 입력: `-c`로 실행할 RPC 명령과 RID 범위

## 표준 사용법

```bash
rpcclient -U '<user>%<password>' <target>
```

## 대표 예시

### null session 접속 확인

```bash
rpcclient -U '%' <TARGET>
```

### 사용자 목록 한 번에 열거

```bash
rpcclient -N -U '' <TARGET> -c 'enumdomusers'
```

### 도메인 정보와 비밀번호 정책 조회

```bash
rpcclient -N -U '' <DC> -c 'querydominfo'
rpcclient -N -U '' <DC> -c 'getdompwinfo'
```

`min_password_length`, `password_properties`, 잠금 관련 값이 반환되면 사용자 목록과 함께 저장하고 [[AD 비밀번호 정책 열거 및 조회]]에서 시도 횟수와 대기 시간을 결정한다.

### RID brute force로 사용자 정보 조회

```bash
for i in $(seq 500 1100); do rpcclient -N -U '' <TARGET> -c "queryuser 0x$(printf '%x' $i)"; done
```


## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-U '<USER>%<PASSWORD>'` | SMB 인증 정보 지정 | 계정 기반 RPC 열거 |
| `-N` | 비밀번호 프롬프트 생략 | null 또는 빈 비밀번호 세션 |
| `-c '<COMMAND>'` | 명령 실행 후 종료 | `enumdomusers`, `queryuser` 자동화 |
| `enumdomusers` | 도메인 사용자 열거 | 사용자 목록 수집 |
| `queryuser <RID>` | RID의 사용자 정보 조회 | RID cycling |
| `querydominfo` | 도메인 기본 정보 조회 | 도메인명과 정책 범위 확인 |
| `getdompwinfo` | 도메인 비밀번호 정책 조회 | 최소 길이와 정책 속성 확인 |
| `netshareenum` | 공유 목록 열거 | SMB 공유 후보 확인 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `enumdomusers`, `queryuser`, `netshareenum` 결과 | RPC 기반 사용자/share 열거 성공 | 사용자 목록, 그룹, share 권한을 공격 후보에 반영 |
| null session 허용 | 인증 없이 일부 RPC 정보 접근 가능 | RID cycling, password policy, share 열거 범위 확인 |
| `NT_STATUS_ACCESS_DENIED` | 권한 부족 또는 익명 접근 제한 | guest/credential 사용, 다른 RPC 명령 확인 |
| 연결/프로토콜 오류 | SMB/RPC 접근 문제 | SMB 포트, 도메인명, 인증 형식, signing 여부 확인 |

## 관련 공격기법

- [[SMB 익명 열거와 공유 권한 확인]]
- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 전 AD 사용자 목록 수집]]
