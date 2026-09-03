---
tags:
  - 환경/windows
  - 기능/형식변환
실행환경: ["Linux"]
필요권한: ["BitLocker 이미지 또는 볼륨 읽기 권한"]
필요조건: ["BitLocker 이미지 또는 볼륨"]
결과: ["해시"]
---

# bitlocker2john

## 도구 개요

`bitlocker2john`은 BitLocker 볼륨, VHD 또는 디스크 이미지의 metadata에서 John the Ripper가 처리할 hash를 추출하는 변환 도구다. 볼륨을 직접 복호화하는 도구가 아니라 BitLocker 비밀번호 복구용 입력을 준비하는 데 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: BitLocker 이미지에 읽기 접근할 수 있는 Linux 호스트
- 필요한 입력: BitLocker 볼륨, VHD 또는 디스크 이미지와 추출 결과를 저장할 hash 파일
- 후속 환경: 출력 형식에 맞는 John 또는 Hashcat mode와 wordlist가 필요하다.


## 표준 사용법

```bash
bitlocker2john -i <bitlocker_image> > <hash_file>
```

## 대표 예시

### BitLocker 이미지에서 hash 추출

```bash
bitlocker2john -i Backup.vhd > backup.hashes
```

### 추출한 hash를 John으로 cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt backup.hashes
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-i` | 입력 BitLocker 이미지 지정 |
| `>` | 추출 결과를 hash 파일로 저장 |
| `--help` | 사용 가능한 옵션 확인 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `$bitlocker$...` hash line | BitLocker metadata에서 cracking용 hash를 추출 | John 또는 해당 hash mode로 복구 시도 |
| volume/image 식별자 포함 출력 | 어느 볼륨에서 추출했는지 구분 가능 | 원본 VHD/디스크 이미지와 결과 hash를 함께 보존 |
| `No signature found` 또는 읽기 오류 | 입력이 BitLocker 볼륨이 아니거나 image offset/권한이 맞지 않음 | 파티션 시작점, image 손상, 읽기 권한 확인 |
| John format 미인식 | 추출물과 설치된 John build의 BitLocker 지원이 맞지 않음 | `john --list=formats`와 추출 형식 재확인 |

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
