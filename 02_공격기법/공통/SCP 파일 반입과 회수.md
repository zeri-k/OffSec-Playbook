---
tags:
  - 환경/linux
  - 서비스/ssh
시작조건: ["SSH 서비스 접근 가능", "유효한 SSH 계정 또는 개인키 확보", "파일 반입 또는 회수 필요"]
필요권한: ["원본 파일 읽기 권한", "목적지 디렉터리 쓰기 권한"]
필요조건: ["OpenSSH scp client", "대상 SSH·SFTP 지원", "양쪽 SHA-256 확인 수단"]
결과: ["SSH 연결로 반입하거나 회수한 파일", "송신본과 수신본의 SHA-256 비교 결과"]
---

# SCP 파일 반입과 회수

## 한 줄 판단

공격 호스트에서 대상 SSH 서비스에 인증할 수 있고 송신 파일을 읽으며 수신 경로에 쓸 수 있으면, `scp`로 한 방향의 파일을 복사하고 원격 `ssh` 명령을 포함한 양쪽 SHA-256 비교로 완료 여부를 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | `scp`를 실행할 공격 호스트에서 `<TARGET>:<SSH_PORT>` 접근 가능 | 기존 SSH 로그인 또는 [[SSH credential 및 키 인증 검증]] 결과 | route·피벗·포트와 SSH service 확인 |
| 현재 계정 또는 인증 수단 | `<USER>`의 SSH password·private key 등 실제 인증 성공 자료 | 같은 계정으로 `ssh` 명령 출력 확인 | 계정·키·passphrase·허용 인증 방식 확인 |
| 현재 권한 | 송신 파일 읽기와 수신 디렉터리 쓰기 가능 | 로컬 `test -r`, 원격 `test -r`·`test -w` | 파일 ACL·소유자와 다른 목적지 확인 |
| 공격 대상의 조건 | 대상 SSH 서버가 현재 client의 SFTP 기반 `scp`를 지원 | 전송 결과와 verbose 오류 | 구형 서버에서 SFTP subsystem 오류가 날 때만 legacy SCP 필요 여부 확인 |
| 필요한 파일·목록·주소 | 로컬·원격 source와 destination exact 경로, SSH port, 원본 SHA-256 | 작업 전 목적지 부재와 source hash 확인 | 기존 파일을 덮어쓰지 않을 새 목적지 선택 |

OpenSSH 9.0부터 `scp`는 기본적으로 SFTP 프로토콜을 사용한다. 구형 서버가 SFTP를 제공하지 않는다는 오류를 확인한 경우에만 legacy SCP용 `-O`를 검토하며, 단순 전송 실패에 일괄 적용하지 않는다. `scp`의 SSH port 옵션은 소문자 `-p`가 아니라 대문자 `-P`다.

아래 명령은 password prompt, `ssh-agent` 또는 SSH config로 인증하는 형태다. explicit private key를 사용할 때는 `ssh`와 `scp` 모두에 `-i '<PRIVATE_KEY>'`를 추가하고, 실행 전 키 경로·권한과 대상 계정이 일치하는지 확인한다.

## 실행

아래 반입과 회수는 서로 다른 방향의 대체 절차다. 필요한 한 방향만 수행한다.

### 공격 호스트에서 대상으로 파일 반입

공격 호스트에서 원본과 원격 목적지의 기존 상태를 먼저 확인한다.

```bash
test -r '<LOCAL_SOURCE_FILE>'
sha256sum '<LOCAL_SOURCE_FILE>'
ssh -p <SSH_PORT> <USER>@<TARGET> "test ! -e '<REMOTE_DESTINATION>' && test -w '<REMOTE_DIRECTORY>'"
```

목적지가 없고 디렉터리에 쓸 수 있을 때 전송한다.

```bash
scp -P <SSH_PORT> '<LOCAL_SOURCE_FILE>' '<USER>@<TARGET>:<REMOTE_DESTINATION>'
ssh -p <SSH_PORT> <USER>@<TARGET> "ls -l '<REMOTE_DESTINATION>'; sha256sum '<REMOTE_DESTINATION>'"
```

확인할 출력:

- `scp` 종료 상태가 0이고 원격 파일 크기와 SHA-256이 로컬 원본과 같다.
- progress meter의 `100%`는 송신 종료를 나타내지만 원격 파일의 hash·실행 권한까지 입증하지 않는다.

### 대상에서 공격 호스트로 파일 회수

공격 호스트의 로컬 목적지가 없고 원격 원본을 현재 SSH 계정으로 읽을 수 있는지 확인한다.

```bash
test ! -e '<LOCAL_DESTINATION>'
ssh -p <SSH_PORT> <USER>@<TARGET> "test -r '<REMOTE_SOURCE_FILE>' && sha256sum '<REMOTE_SOURCE_FILE>'"
scp -P <SSH_PORT> '<USER>@<TARGET>:<REMOTE_SOURCE_FILE>' '<LOCAL_DESTINATION>'
ls -l '<LOCAL_DESTINATION>'
sha256sum '<LOCAL_DESTINATION>'
```

확인할 출력:

- 원격 원본과 로컬 회수본의 SHA-256이 같다.
- `Permission denied`가 SSH 인증 단계인지 원격 원본 읽기·목적지 쓰기 단계인지 `scp -v`와 선행 `ssh` 명령으로 구분한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 인증 성공, 전송 종료 상태 0, 양쪽 SHA-256 일치 | 선택한 방향의 파일 복사 완료 | 파일 반입 또는 회수 | 파일 사용 조건 또는 분석 범위를 별도 확인 |
| SSH 로그인은 되지만 `scp`에 SFTP subsystem 오류 | 인증은 성공했으나 기본 전송 프로토콜 미지원 | 전송 미완료 | 서버 버전·SFTP 설정을 확인하고 필요한 경우에만 `-O` 검토 |
| 원격 `test -r`·`test -w` 실패 | 현재 계정의 파일 권한이 부족함 | SSH 접근만 확보 | source ACL 또는 별도 writable destination 확인 |
| progress가 끝났지만 hash 불일치 | 다른 source·destination, 중단 또는 변형 가능 | 손상된 파일 | 파일을 사용하지 말고 exact 경로와 크기를 대조해 새 목적지로 재전송 |
| SSH 인증 실패 | 파일 권한 확인 전 연결 실패 | 전송 경로 미확보 | 계정 범위·키·passphrase·허용 인증 방식 확인 |

## 변경 영향과 복구

반입은 대상에 `<REMOTE_DESTINATION>`을 만들고, 회수는 공격 호스트에 `<LOCAL_DESTINATION>`을 만든다. SSH 서비스·계정·authorized key는 이 절차에서 변경하지 않는다.

| 생성 항목 | 기존 상태와 식별값 | 정리 명령 | 완료 확인 |
|---|---|---|---|
| 원격 반입 파일 | 전송 전 `test ! -e`, exact 경로와 반입 뒤 SHA-256 | 파일을 사용하는 원격 process를 먼저 종료한 뒤 `ssh -p <SSH_PORT> <USER>@<TARGET> "rm -- '<REMOTE_DESTINATION>'; test ! -e '<REMOTE_DESTINATION>'"` | 원격 `test` 종료 상태 0 |
| 로컬 회수본 | 회수 전 `test ! -e`, exact 경로와 원격 원본·회수본 SHA-256 | 분석·인계가 끝난 뒤 `rm -- '<LOCAL_DESTINATION>'` | `test ! -e '<LOCAL_DESTINATION>'` 성공 |

작업 전 존재했거나 기준 hash를 기록하지 않은 파일은 이 절차로 안전하게 원상복구할 수 없다. SSH 연결이 끊겨 원격 파일 부재를 확인할 수 없으면 원격 정리 완료로 기록하지 않는다.

## 관련 도구

- [[ssh]]

## 관련 상태 라우터

- [[상황별 파일 전송]]

## 참고 링크

- [OpenSSH `scp(1)`](https://man.openbsd.org/scp.1)
- [OpenSSH `ssh(1)`](https://man.openbsd.org/ssh.1)
