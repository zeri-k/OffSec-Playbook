---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 SMB 또는 RPC 접근"]
결과: ["사용자", "그룹", "공유", "비밀번호 정책"]
---

# enum4linux

## 도구 개요

`enum4linux`는 여러 Samba 명령을 묶어 SMB/RPC의 사용자·그룹·공유와 비밀번호 정책을 열거하는 레거시 Perl 도구다. 기존 CPTS 명령을 재현하거나 대상의 기본 SMB 정보를 빠르게 모을 때 사용할 수 있으며, 새 환경에서 구조화된 결과가 필요하면 `enum4linux-ng`와 구분한다.

## 필요한 입력과 실행 환경

- 실행 환경: 대상 SMB/RPC에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 IP 또는 호스트명
- 인증 조건: NULL session이 거부되면 유효 credential을 사용하거나 [[enum4linux-ng]]로 결과 형식을 교차 확인한다.

## 표준 사용법

```bash
enum4linux <OPTIONS> <TARGET>
```

## 대표 예시

### 비밀번호 정책 조회

```bash
enum4linux -P <DC_IP>
```

확인할 출력:

- 최소 비밀번호 길이, 복잡성, 잠금 임계값과 잠금 지속 시간.

### 도메인 사용자 수집

```bash
enum4linux -U <DC_IP>
```

확인할 출력:

- `user:[<NAME>]` 형식의 사용자 목록.

### 전체 기본 열거

```bash
enum4linux -a <TARGET>
```

확인할 출력:

- domain/workgroup, 사용자, 그룹, 공유와 정책 정보.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-P` | 비밀번호 정책 | Password Spraying 전 정책 확인 |
| `-U` | 사용자 목록 | 인증 전 AD 사용자 후보 수집 |
| `-S` | 공유 목록 | 익명 접근 가능한 share 확인 |
| `-G` | 그룹과 구성원 | 고권한 그룹 후보 확인 |
| `-a` | 기본 전체 열거 | 초기 SMB/RPC 범위 파악 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Password Policy Information` | 정책 조회 성공 | [[AD 비밀번호 정책 열거 및 조회]]에서 잠금 안전 한계 계산 |
| 사용자·그룹·share 출력 | NULL 또는 인증 세션 열거 성공 | [[인증 전 AD 사용자 목록 수집]], share별 실제 권한 확인 |
| `Cannot request session` | 해당 포트·세션 방식 실패 | 445 결과, NULL/Guest 차이와 [[enum4linux-ng]] 비교 |
| 부분 정책만 반환 | 일부 RPC 호출만 허용 | rpcclient·NetExec 결과로 보완 |

## 버전과 환경 차이

- `enum4linux`는 레거시 Perl 도구이고 `enum4linux-ng`와 옵션·출력 형식이 다르다. 새 환경에서는 [[enum4linux-ng]]를 우선하되 CPTS 자료의 기존 명령을 재현할 때 이 문서를 사용한다.

## 관련 공격기법

- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 전 AD 사용자 목록 수집]]
- [[SMB 익명 열거와 공유 권한 확인]]
