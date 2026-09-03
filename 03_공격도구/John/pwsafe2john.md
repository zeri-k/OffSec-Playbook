---
tags:
  - 기능/형식변환
실행환경: ["Linux"]
필요조건: ["Password Safe 파일 접근"]
결과: ["해시"]
---

# pwsafe2john

## 도구 개요

`pwsafe2john`은 Password Safe `.psafe3` 데이터베이스와 백업 파일에서 master password용 John hash를 추출하는 도구다. 여러 백업을 각각 변환해 어느 데이터베이스의 비밀번호가 복구되는지 구분하며 처리할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- `.psafe3`, backup 파일 등에서 master password cracking을 시도할 때 쓴다.

- 대상: Password Safe 데이터베이스 파일
- 후속 도구: John the Ripper


## 표준 사용법

```bash
pwsafe2john <pwsafe_file> > <hash_file>
```

## 대표 예시

### Password Safe DB에서 hash 추출

```bash
pwsafe2john Employee-Passwords_OLD.psafe3 > pwsafe.hashes
```

### 여러 backup 파일을 한 번에 변환

```bash
for f in Employee-Passwords_OLD.psafe3 Employee-Passwords_OLD_*.ibak; do pwsafe2john "$f"; done > pwsafe.hashes
```

### master password cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt pwsafe.hashes
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| `<pwsafe_file>` | Password Safe 파일 |
| `>` | 추출 결과를 hash 파일로 저장 |
| 여러 파일 반복 | `for` 루프로 여러 backup 파일을 하나의 hash 파일에 합칠 수 있음 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `$pwsafe$*...`로 시작하는 한 줄 | Password Safe v3 master password hash가 추출됨 | `john <hash_file>`로 wordlist/rule 공격 시작 |
| 파일마다 하나의 hash line | 여러 backup이 각각 cracking 대상임 | 각 line의 파일명 label을 보존해 결과를 원본과 대응 |
| `pwsafe2john: ...` 오류 | 파일 읽기 또는 Password Safe 형식 파싱 실패 | 원본 파일 종류, 손상 여부, 읽기 권한 확인 |
| 출력이 비어 있음 | 대상이 Password Safe v3 DB가 아니거나 변환 실패 | 확장자 대신 파일 형식과 원본 확보 상태 재확인 |

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
- [[오프라인 해시 크래킹]]

## 실전 진입

Password Safe 파일을 확보했다면 이 문서에서 hash를 추출한 뒤 [[보호된 파일 및 아카이브 크래킹]]의 `john` 또는 `hashcat` 분기로 이어간다.
