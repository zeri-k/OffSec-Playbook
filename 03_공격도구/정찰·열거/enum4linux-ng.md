---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["대상 SMB 호스트"]
결과: ["정보"]
---

# enum4linux-ng

## 도구 개요

`enum4linux-ng`는 SMB/RPC에서 사용자·그룹·공유·비밀번호 정책과 RID 정보를 자동 수집하는 `enum4linux`의 재구현 도구다. 익명 또는 계정 기반 열거를 한 번에 수행하고 YAML·JSON 결과로 저장해 후속 분석할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 SMB/RPC/NetBIOS에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 SMB 호스트 IP/FQDN(예: `smb.example.test`)과 필요하면 도메인·사용자·비밀번호. `<TARGET>`은 SMB 호스트, `<USER>`는 SMB에 전달할 계정명, `<PASSWORD>`는 그 계정의 평문 값이며, `-oA enum4linux-ng-policy`는 Linux 실행 호스트의 새 출력 접두사다.
- 인증 조건: null session이 차단된 대상은 share와 RPC를 읽을 수 있는 유효 credential이 필요하다.


## 표준 사용법

```bash
enum4linux-ng.py [options] <target>
```

## 대표 예시

### SMB 기본 정보 전체 열거

```bash
./enum4linux-ng.py <TARGET> -A
```

### credential을 사용한 SMB 열거

```bash
./enum4linux-ng.py <TARGET> -A -u '<USER>' -p '<PASSWORD>'
```

### 비밀번호 정책만 조회하고 결과 저장

```bash
./enum4linux-ng.py -P -oA enum4linux-ng-policy <TARGET>
```

저장된 YAML·JSON 결과에서 최소 길이, 복잡성, 이력, 잠금 임계값과 지속 시간을 확인한다. 값이 비어 있으면 정책 부재로 단정하지 않고 RPC 권한과 다른 조회 방법을 확인한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-A` | 가능한 주요 열거를 한 번에 수행 |
| `-u`, `-p` | 인증 사용자와 비밀번호 지정 |
| `-d` | 도메인 지정 |
| `-R` | RID cycling 수행 |
| `-P` | 비밀번호 정책 조회 |
| `-oA <prefix>` | YAML·JSON 등 지원 형식으로 결과 저장 |
| `-w` | workgroup/domain 지정 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 사용자/그룹/share/policy 출력 | SMB/RPC 열거 성공 | 접근 가능한 share, 사용자 목록, 비밀번호 정책을 후속 공격에 반영 |
| null/guest 결과 일부 확인 | 익명 또는 guest 접근 가능 | 민감 share와 RID/user 열거 가능 범위 확인 |
| `ACCESS_DENIED` | 인증 필요 또는 권한 제한 | 확보 credential, guest/null 차이, 다른 열거 방식 확인 |
| SMB 연결 실패 | 포트, SMB 버전, signing, 방화벽 문제 | SMB 클라이언트 옵션과 NetExec/smbclient 결과 비교 |

## 관련 공격기법

- [[SMB 익명 열거와 공유 권한 확인]]
- [[AD 비밀번호 정책 열거 및 조회]]
- [[인증 전 AD 사용자 목록 수집]]
