---
tags:
  - 환경/unix
  - 서비스/nfs
시작조건: ["NFS export 확인 또는 NFSv4 서비스 식별", "클라이언트 허용 대역에서 접근"]
필요권한: ["export 읽기 권한", "명령 실행 호스트의 mount 권한"]
필요조건: ["NFS 접근", "클라이언트 허용 대역", "로컬 mountpoint"]
결과: ["NFS 파일 접근", "파일·자격 증명 후보", "숫자 UID·GID 권한 매핑", "선택적으로 확인한 export 쓰기 권한"]
---

# NFS export 마운트와 권한 매핑 검증

## 한 줄 판단

현재 명령 실행 위치에서 대상 NFS 포트와 export에 접근할 수 있으면 읽기 전용으로 마운트하여 파일을 수집하고, 이름 해석과 분리된 숫자 UID·GID와 파일 mode로 실제 읽기 범위를 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 버전·export | v3 export 또는 v4 pseudo-root 후보 | `showmount`, NFS NSE와 mount 오류 | v4-only와 v3 mountd 실패를 구분 |
| source 허용 | 현재 명령 실행 호스트의 IP가 export 허용 범위 | mount 응답 | VPN·피벗 위치와 export 정책 확인 |
| 읽기 전용 mount | `ro` option | `findmnt --target` | 쓰기 mount로 진행하지 않음 |
| 로컬 mountpoint | 이번 실행에서 새로 만든 빈 디렉터리 | `mktemp -d` 결과와 `mountpoint` | 기존 디렉터리를 mountpoint나 정리 대상으로 재사용하지 않음 |

## 실행

`<TARGET>`은 NFS 서버 주소(가상 예시 `192.0.2.60`)이고, `<EXPORT>`는 `showmount` 또는 NFSv4 pseudo-root에서 확인한 export 경로(예: `exports/app`)다. `<MOUNTPOINT>`는 공격 호스트에서 `mktemp`로 만든 새 디렉터리, `<FILE>`은 목록에서 선택한 export 내부 상대 경로다. 아래 mount·UID/GID 확인은 NFS client 권한이 있는 공격 호스트에서 실행하며, 쓰기 검증의 `$PROOF`는 생성·조회·삭제에 같은 값으로 재사용한다.

### export와 버전 확인

```bash
showmount -e <TARGET>
nmap -sV -p111,2049 --script nfs-showmount,nfs-ls,nfs-statfs <TARGET>
```

### NFSv3 읽기 전용 마운트

```bash
MOUNTPOINT="$(mktemp -d ./target-nfs.XXXXXX)"
sudo mount -t nfs -o vers=3,ro,nosuid,nodev,nolock <TARGET>:/<EXPORT> "$MOUNTPOINT"
findmnt --target "$MOUNTPOINT"
```

### NFSv4 읽기 전용 마운트

```bash
MOUNTPOINT="$(mktemp -d ./target-nfs.XXXXXX)"
sudo mount -t nfs -o vers=4,ro,nosuid,nodev <TARGET>:/ "$MOUNTPOINT"
findmnt --target "$MOUNTPOINT"
```

NFSv4 pseudo-root에서는 export가 하위 경로로 보일 수 있다. 루트에서 실제 경로를 확인한 뒤 필요한 경우 해당 경로만 다시 mount한다.

```bash
find "$MOUNTPOINT" -maxdepth 2 -mindepth 1 -printf '%M %u:%g %p\n'
sudo umount "$MOUNTPOINT"
sudo mount -t nfs -o vers=4,ro,nosuid,nodev <TARGET>:/<EXPORT> "$MOUNTPOINT"
findmnt --target "$MOUNTPOINT"
```

### 파일과 UID·GID 확인

```bash
find "$MOUNTPOINT" -maxdepth 3 -type f \( -name 'id_*' -o -name '*.bak' -o -name '*.conf' \) -print
ls -lan "$MOUNTPOINT"
stat -c '%n %s bytes uid=%u gid=%g mode=%a' "$MOUNTPOINT/<FILE>"
sudo umount "$MOUNTPOINT"
```

### 쓰기와 UID·GID 매핑 확인

읽기 전용 mount를 해제한 뒤, 쓰기 검증이 필요한 export만 읽기·쓰기 옵션으로 다시 mount한다.

```bash
sudo mount -t nfs -o vers=<3_OR_4>,rw,nosuid,nodev <TARGET>:/<EXPORT> "$MOUNTPOINT"
findmnt --target "$MOUNTPOINT"
PROOF="nfs-proof-$(date -u +%Y%m%dT%H%M%SZ)-$$.txt"
printf 'NFS write verification: %s\n' "$PROOF" > "$MOUNTPOINT/$PROOF"
stat -c '%n %s bytes uid=%u gid=%g mode=%a' "$MOUNTPOINT/$PROOF"
```

확인할 출력:

- mount 옵션에 `rw`가 표시되고 고유 파일이 생성되는지.
- 서버가 파일에 적용한 숫자 UID·GID와 mode. 로컬 사용자명과 서버의 사용자명이 같다고 가정하지 않는다.
- UID/GID 0 요청만 anonymous identity로 바뀌면 `root_squash`, 모든 요청이 같은 anonymous UID/GID로 바뀌면 `all_squash` 후보로 판정한다. `anonuid`·`anongid`가 기본값과 다를 수 있으므로 숫자 하나만 보고 계정명을 단정하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `ro` mount와 파일 목록 확인 | 현재 source에서 export 읽기 가능 | NFS 파일 접근 | 필요한 파일의 내용·UID/GID 확인 |
| 파일 내용과 UID·GID 확인 | 해당 파일 READ 권한 | 파일·자격 증명 후보 수집 | 파일 유형별 후속 인증·크래킹 선택 |
| 특정 UID·GID에서만 읽힘 | 서버가 숫자 identity로 권한 적용 | UID·GID 매핑 확인 | 서버 파일 mode·ACL과 필요한 identity 구분 |
| 검증 파일 생성과 숫자 UID·GID 확인 | 현재 source와 로컬 UID·GID가 export에 매핑됨 | NFS 쓰기·권한 매핑 | 서버 파일의 실제 소유권과 실행 경로를 별도로 확인 |
| 로컬 root의 생성 파일이 anonymous UID/GID로 표시 | `root_squash` 또는 `all_squash`가 적용된 후보 | root 요청의 서버 측 매핑 확인 | 일반 UID 요청과 비교해 두 정책을 구분 |
| mount 거부 | source 대역·export·버전 불일치 가능 | NFS 접근 미확보 | 네트워크 위치·버전·경로 확인 |

## 확인할 출력과 권한

- mount 성공, 파일 목록과 정상 파일 읽기를 각각 구분한다.
- 로컬 `sudo`는 mount 권한일 뿐 서버 root 권한이 아니다.

## 변경 영향과 복구

이번 실행에서 만든 정확한 검증 파일만 제거하고 mount를 해제한다.

```bash
rm -f -- "$MOUNTPOINT/$PROOF"
test ! -e "$MOUNTPOINT/$PROOF" && echo 'proof removed'
sudo umount "$MOUNTPOINT"
findmnt --target "$MOUNTPOINT" || echo 'NFS unmounted'
rmdir -- "$MOUNTPOINT"
test ! -e "$MOUNTPOINT" && echo 'mountpoint removed'
```

- proof 삭제가 거부되면 export를 끊기 전에 정확한 `$MOUNTPOINT/$PROOF`와 숫자 UID/GID를 확인하고, 같은 identity 또는 서버 관리 경로로 제거한다. 확인할 수 없으면 원격 파일 정리 완료로 기록하지 않는다.
- mount 명령이 실패했으면 `mountpoint -q "$MOUNTPOINT"`로 실제 mount 여부를 확인한 뒤, mount되지 않은 이번 빈 디렉터리만 `rmdir`한다.

## 참고 링크

- [nfs(5) — Linux NFS client options](https://man7.org/linux/man-pages/man5/nfs.5.html)
- [exports(5) — root_squash, all_squash and anonymous UID/GID](https://man7.org/linux/man-pages/man5/exports.5.html)
- [showmount(8) — MNT service and NFSv4-only limitation](https://man7.org/linux/man-pages/man8/showmount.8.html)
- [Nmap nfs-ls NSE documentation](https://nmap.org/nsedoc/scripts/nfs-ls.html)

## 후속 공격 연결

- SSH key: [[SSH credential 및 키 인증 검증]]
- 암호화 파일: [[보호된 파일 및 아카이브 크래킹]]

## 관련 서비스

- [[NFS 서비스]]

## 관련 상태 라우터

- [[파일 서비스 접근 후 자격 증명과 초기 접근 연결]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[showmount]]
- [[nmap]]
