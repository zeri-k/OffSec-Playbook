---
tags:
  - 환경/windows
  - 기능/복호화
실행환경: ["Linux", "macOS"]
필요권한: ["BitLocker 볼륨 또는 이미지 읽기 권한", "FUSE 마운트 권한"]
필요조건: ["BitLocker 볼륨 또는 이미지", "BitLocker 복구키, 암호 또는 BEK 파일"]
결과: ["파일", "정보"]
---

# dislocker

## 도구 개요

`dislocker`는 Linux 또는 macOS에서 BitLocker 볼륨과 디스크 이미지를 복호화해 마운트 가능한 `dislocker-file`을 만든다. 암호화된 Windows 볼륨의 파일 시스템을 별도 분석 환경에서 열어볼 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: `dislocker`와 FUSE를 사용할 수 있는 Linux 또는 macOS 분석 호스트
- 필요한 입력: BitLocker 볼륨/VHD/loop device와 password, recovery key 또는 BEK 파일
- 마운트 입력: `dislocker-file`을 둘 디렉터리와 복호화된 파일시스템을 마운트할 별도 디렉터리


## 표준 사용법

```bash
dislocker -r -V <BITLOCKER_DEVICE> -u -- <DISLOCKER_FUSE_DIR>
```

`-u` 뒤에 password를 붙이지 않으면 prompt로 입력해 shell history·process arguments 노출을 피한다. 48자리 recovery password는 `-p`를 값 없이 사용한다. `-r`은 BitLocker 원본에 대한 쓰기를 막는다.

## 대표 예시

### 기존 상태와 작업 경로 확인

```bash
DISLOCKER_ORIGINAL_DIR="$PWD"
DISLOCKER_ROOT="$(mktemp -d "${PWD}/dislocker.XXXXXX")"
mkdir -- "$DISLOCKER_ROOT/fuse" "$DISLOCKER_ROOT/mount"
sudo losetup -j <BITLOCKER_IMAGE>
findmnt --target "$DISLOCKER_ROOT/fuse"
findmnt --target "$DISLOCKER_ROOT/mount"
```

두 mountpoint와 해당 image의 loop mapping이 기존에 없어야 이번 작업 자원을 구분할 수 있다.

### VHD를 읽기 전용 loop device로 연결

```bash
LOOP_DEV="$(sudo losetup --find --show --partscan --read-only <BITLOCKER_IMAGE>)"
sudo lsblk -o NAME,TYPE,FSTYPE,SIZE,MOUNTPOINTS "$LOOP_DEV"
```

`lsblk`에서 실제 BitLocker partition 번호를 확인한 뒤 `<BITLOCKER_PARTITION>`에 `${LOOP_DEV}p<PARTITION_NUMBER>`를 기록한다. raw volume을 직접 받은 경우에는 loop device를 만들지 않고 그 승인된 read-only device를 사용한다.

### BitLocker password로 복호화 파일 생성

```bash
sudo dislocker -r -V <BITLOCKER_PARTITION> -u -- "$DISLOCKER_ROOT/fuse"
findmnt --target "$DISLOCKER_ROOT/fuse"
sudo test -r "$DISLOCKER_ROOT/fuse/dislocker-file"
```

### 복호화된 볼륨 마운트

```bash
sudo mount -o loop,ro "$DISLOCKER_ROOT/fuse/dislocker-file" "$DISLOCKER_ROOT/mount"
findmnt --target "$DISLOCKER_ROOT/mount"
sudo ls -la "$DISLOCKER_ROOT/mount"
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-u[password]` | 사용자 password/PIN. 값을 생략하면 prompt 입력 |
| `-p[recovery_password]` | recovery password. 값을 생략하면 prompt 입력 |
| `-k <bek_file>` | BEK file 사용 |
| `-r` | 원본 BitLocker volume을 read-only로 취급 |
| `-V <volume>` | BitLocker volume 또는 partition 지정 |
| `-- <dir>` | 복호화된 `dislocker-file`을 생성할 출력 디렉터리 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `dislocker-file` 생성 | BitLocker 볼륨 복호화 성공 | loop mount 후 파일 시스템과 민감 파일 확인 |
| recovery key/password 오류 | 복호화 키 불일치 | recovery key 형식, password, BEK 파일 여부 확인 |
| mount 실패 | FUSE/loop/권한 문제 | mount point, root 권한, 파일 시스템 타입 확인 |
| 파일 접근 가능 | 암호화 볼륨 내부 데이터 확인 가능 | 사용자 파일, credential, 설정 파일을 우선 검색 |

## 변경 영향과 복구

정리는 연결의 하위 계층부터 수행한다. 먼저 복호화된 NTFS mount를 해제하고, 그다음 dislocker FUSE mount, 마지막으로 이번에 만든 loop device를 제거한다.

```bash
cd -- "$DISLOCKER_ORIGINAL_DIR"
sudo umount "$DISLOCKER_ROOT/mount"
sudo umount "$DISLOCKER_ROOT/fuse"
sudo losetup --detach "$LOOP_DEV"
findmnt --target "$DISLOCKER_ROOT/mount"
findmnt --target "$DISLOCKER_ROOT/fuse"
sudo losetup -j <BITLOCKER_IMAGE>
rmdir -- "$DISLOCKER_ROOT/mount" "$DISLOCKER_ROOT/fuse" "$DISLOCKER_ROOT"
test ! -e "$DISLOCKER_ROOT"
```

두 `findmnt`와 `losetup -j`가 이번 자원을 반환하지 않아야 정리가 확인된다. `target is busy`면 해당 mount 아래 현재 shell·열린 file/process를 먼저 확인하고 강제·lazy unmount로 완료를 가장하지 않는다. 연결이 이미 끊겼다면 `findmnt`와 `losetup`에서 현재 소유자를 다시 확인하고 재사용된 loop device를 detach하지 않는다. raw volume을 직접 사용했다면 loop detach 단계는 생략한다.

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]

## 참고 링크

- [Dislocker upstream](https://github.com/Aorimn/dislocker)
- [dislocker-fuse options](https://github.com/Aorimn/dislocker/blob/master/man/linux/dislocker-fuse.1)
