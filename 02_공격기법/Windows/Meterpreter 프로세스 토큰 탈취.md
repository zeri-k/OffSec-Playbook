---
tags:
  - 환경/windows
시작조건: ["Windows Meterpreter 세션 확보", "다른 사용자로 실행 중인 접근 가능한 프로세스 식별"]
필요권한: ["대상 프로세스 token을 열고 복제할 수 있는 현재 Meterpreter 세션 권한"]
필요조건: ["대상 프로세스 PID", "대상 프로세스의 사용자와 현재성 확인"]
결과: ["Meterpreter 세션에 적용된 다른 Windows 사용자 token", "해당 token으로 접근 가능한 파일·서비스·네트워크 자원"]
---

# Meterpreter 프로세스 토큰 탈취

## 한 줄 판단

Windows Meterpreter 세션에서 다른 사용자로 실행 중인 프로세스의 PID와 접근 권한을 확인했으면 `steal_token`으로 그 프로세스 token을 현재 세션에 적용하고 `getuid`와 실제 자원 접근으로 권한 변화를 확인한다.

## 사용할 때

- `ps`에 현재 사용자와 다른 계정으로 실행 중인 프로세스가 보일 때.
- 현재 Meterpreter 세션이 대상 프로세스 token에 접근할 수 있을 때.
- 파일·서비스·네트워크 접근이 대상 사용자의 token에서 달라지는지 확인하려 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 현재 세션 | Windows Meterpreter와 현재 사용자 확인 | `getuid`, `sysinfo` | 일반 셸과 Meterpreter 명령을 구분 |
| 대상 프로세스 | PID, 프로세스명, 사용자와 architecture 확인 | `ps` | 사용자 열이 비어 있거나 종료된 PID 제외 |
| token 접근 | 현재 세션에서 대상 PID의 token 복제 가능 | `steal_token` 결과 | 현재 권한과 보호 프로세스 여부 확인 |

## 실행

```text
meterpreter > getuid
meterpreter > getpid
meterpreter > ps
meterpreter > steal_token <TARGET_PID>
meterpreter > getuid
```

확인할 출력:

- `ps`의 대상 PID·프로세스명·사용자.
- 첫 `getuid`의 원래 사용자와 현재 Meterpreter `getpid`.
- `Stolen token with username: <DOMAIN_OR_HOST>\<USER>` 메시지.
- 두 번째 `getuid`가 탈취한 token의 사용자로 바뀌는지.

token 적용 뒤 필요한 자원에서 실제 권한을 확인한다.

```text
meterpreter > shell
C:\> whoami /all
C:\> dir <TARGET_PATH>
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Stolen token`과 `getuid` 사용자 변경 | 대상 프로세스 token이 현재 Meterpreter 세션에 적용됨 | 다른 Windows 사용자 token | 대상 파일·공유·서비스에서 실제 접근 확인 |
| token 사용 후 이전에 거부된 자원 접근 성공 | 대상 token의 권한 행사 확인 | 확장된 파일·서비스 접근 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]에서 권한 범위 재평가 |
| 고권한 사용자 token과 관리자 작업 성공 | 고권한 token 확인 | 고권한 Meterpreter 세션 | [[고권한 세션 확보 후 후속 판단]] |
| `Access is denied` 또는 token 열기 실패 | 현재 세션 권한 부족 또는 보호 프로세스 | token 미획득 | 대상 PID·architecture·현재 privilege 확인 |
| PID가 사라짐 | 프로세스 종료 또는 PID 변경 | 대상 token 미확정 | `ps`를 다시 실행해 현재 프로세스 확인 |

## 확인할 출력과 권한

- `steal_token` 성공 메시지와 `getuid` 변경을 모두 확인한다.
- token 사용자명, 로컬 관리자 그룹, 상승된 관리자 token과 도메인 권한을 각각 다른 상태로 기록한다.
- token 적용은 현재 Meterpreter 세션의 실행 상태이며 대상 계정의 비밀번호나 hash를 얻은 결과가 아니다.

## 변경 영향과 복구

`steal_token`은 대상 계정·process 자체를 변경하지 않지만 현재 Meterpreter session의 impersonation token을 바꾼다. 필요한 접근 확인을 마친 뒤 같은 session에서 원래 process token으로 되돌린다.

```text
meterpreter > rev2self
meterpreter > getuid
```

확인할 출력:

- `rev2self`가 오류 없이 끝나고 두 번째 `getuid`가 실행 전 기록한 사용자로 돌아온다.
- 탈취 token에서만 가능했던 자원 접근을 다시 시도할 필요가 있다면, 원래 token의 접근 결과와 비교한다.
- `rev2self`가 실패하거나 session이 먼저 끊겼으면 원래 token 복구를 확인할 수 없으므로 복구 완료로 기록하지 않는다. 다른 process나 계정 객체를 이름으로 종료·변경하지 않는다.

## 관련 도구

- [[meterpreter]]

## 관련 상태 라우터

- [[Meterpreter 세션 후속 행동]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Rapid7 Metasploit Framework: Meterpreter stdapi system commands](https://github.com/rapid7/metasploit-framework/blob/master/lib/rex/post/meterpreter/ui/console/command_dispatcher/stdapi/sys.rb)
