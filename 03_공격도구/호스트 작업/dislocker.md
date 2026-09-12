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
dislocker <bitlocker_device> -u<password> -- <output_mount_dir>
```

## 대표 예시

### VHD를 loop device로 연결

```bash
sudo losetup -fP Backup.vhd
```

### BitLocker password로 복호화 파일 생성

```bash
sudo dislocker /dev/loop0p1 -u1234qwer -- /media/bitlocker
```

### 복호화된 볼륨 마운트

```bash
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-u<password>` | 사용자 비밀번호/PIN 지정 |
| `-p<recovery_password>` | recovery password 지정 |
| `-k <bek_file>` | BEK file 사용 |
| `-- <dir>` | 복호화된 `dislocker-file`을 생성할 출력 디렉터리 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `dislocker-file` 생성 | BitLocker 볼륨 복호화 성공 | loop mount 후 파일 시스템과 민감 파일 확인 |
| recovery key/password 오류 | 복호화 키 불일치 | recovery key 형식, password, BEK 파일 여부 확인 |
| mount 실패 | FUSE/loop/권한 문제 | mount point, root 권한, 파일 시스템 타입 확인 |
| 파일 접근 가능 | 암호화 볼륨 내부 데이터 확인 가능 | 사용자 파일, credential, 설정 파일을 우선 검색 |

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
