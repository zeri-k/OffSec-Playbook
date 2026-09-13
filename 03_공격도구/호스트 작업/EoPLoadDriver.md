---
tags:
  - 환경/windows
  - 기능/권한상승
실행환경: ["Windows x64 CMD 또는 PowerShell"]
필요권한: ["현재 token의 SeLoadDriverPrivilege"]
필요조건: ["registry service path", "대상 OS·architecture와 호환되는 driver image", "Code Integrity 정책이 driver load를 허용"]
결과: ["SeLoadDriverPrivilege 활성화 시도", "per-user registry service key", "driver load NTSTATUS"]
---

# EoPLoadDriver

## 도구 개요

`EoPLoadDriver`는 현재 token의 `SeLoadDriverPrivilege`를 활성화하고 지정한 registry service path와 driver image로 `NtLoadDriver`를 호출하는 PoC다. 이 도구의 성공 범위는 driver load까지이며, SYSTEM process 생성은 driver별 exploit와 새 process Identity로 별도 확인한다.

## 필요한 입력과 실행 환경

- 실행 위치: driver를 load할 Windows host의 x64 process.
- 첫 입력: `System\CurrentControlSet\<DRIVER_SERVICE_NAME>` 형식의 service registry path.
- 둘째 입력: target Windows host에서 읽을 수 있는 exact `.sys` path.
- 권한·정책: 현재 token의 `SeLoadDriverPrivilege`, driver signature·architecture와 Code Integrity·HVCI·blocklist 허용.
- `<DRIVER_SERVICE_NAME>`은 현재 사용자 hive 아래의 고유 service key 이름이고 `.sys`는 대상 Windows host의 절대 경로(예: `C:\\Temp\\driver.sys`)다. registry path와 파일 path는 같은 문자열이 아니며 기존 key·driver와 충돌하지 않아야 한다.

## 표준 사용법

`<DRIVER_SERVICE_NAME>`은 현재 user hive의 새 service key 이름, `<DRIVER_PATH>`는 target Windows host의 exact `.sys` path다. privilege·signature·CI 조건은 load 이전 조건이며 registry key 생성과 driver load·exploit process 결과를 분리해 확인한다. 복구는 이 같은 service name·path만 대상으로 한다.

```cmd
EoPLoadDriver.exe <REGISTRY_SERVICE_PATH> <DRIVER_IMAGE_PATH>
```

## 대표 예시

```cmd
EoPLoadDriver.exe System\CurrentControlSet\<DRIVER_SERVICE_NAME> <VULNERABLE_DRIVER_PATH>
```

확인할 출력:

- `SeLoadDriverPrivilege Enabled`, `Loading Driver`와 호출 결과 NTSTATUS.
- NTSTATUS 숫자를 추측으로 성공 처리하지 말고 `driverquery`, CodeIntegrity event와 driver별 device handle 접근으로 load 여부를 교차 확인한다.
- 이 출력만으로 kernel exploit 또는 SYSTEM child process가 성공했다고 확정할 수 없다.

## 주요 옵션

이 PoC는 두 위치 인자를 사용한다.

| 인자 | 의미 | 사용하는 상황 |
|---|---|---|
| `<REGISTRY_SERVICE_PATH>` | 현재 사용자 hive 아래에 만들 driver service key의 상대 경로 | exact key를 사전 확인하고 이후 정리할 때 |
| `<DRIVER_IMAGE_PATH>` | load할 `.sys`의 absolute path | 식별한 build의 hash·signature·architecture를 확인한 driver에만 사용 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `SeLoadDriverPrivilege Enabled` | 현재 process token에서 privilege 활성화 호출 성공 | driver load 결과와 `whoami /priv` 확인 |
| `Loading Driver: ...` | registry path를 사용해 load 호출 시도 | NTSTATUS와 actual loaded driver 확인 |
| privilege 활성화 실패 | 현재 token에 privilege가 없거나 활성화 불가 | 그룹 상태·새 token과 현재 Identity 확인 |
| Code Integrity block | driver policy·signature·blocklist 조건 미충족 | 정책을 비활성화하지 말고 현재 경로 중단 |

도구가 만든 registry key를 삭제해도 이미 loaded된 driver가 unload되지는 않는다. driver별 unload routine이나 재부팅 전까지 host 영향이 남을 수 있다.

## 관련 공격기법

- [[SeLoadDriverPrivilege로 취약 드라이버 권한 상승]]

## 참고 링크

- [Tarlogic Security: EoPLoadDriver](https://github.com/TarlogicSecurity/EoPLoadDriver)
- [Microsoft Learn: Privilege Constants](https://learn.microsoft.com/en-us/windows/win32/secauthz/privilege-constants)
