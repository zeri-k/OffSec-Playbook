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

Password Safe의 master password는 암호화된 데이터베이스를 여는 비밀번호이다. 데이터베이스 안에 저장된 각 서비스 계정의 비밀번호와는 다른 자료다. master password 복구 성공은 해당 파일 열기 후보이며, 내부 항목의 서비스·계정 대응과 실제 인증 성공은 별도로 확인한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- `.psafe3`, backup 파일 등에서 master password cracking을 시도할 때 쓴다.

- 대상: Password Safe 데이터베이스 파일
- 후속 도구: John the Ripper
- 구분할 식별자: 원본·backup 파일의 고유 경로와 각 hash line의 label. 백업마다 별도 파싱·복구 대상으로 취급한다.


## 표준 사용법

```bash
pwsafe2john <pwsafe_file> > <hash_file>
```

## 대표 예시

### Password Safe DB에서 hash 추출

```bash
test ! -e '<PWSAFE_HASH_FILE>'
pwsafe2john '<PWSAFE_FILE>' > '<PWSAFE_HASH_FILE>'
```

### 여러 backup 파일을 한 번에 변환

```bash
test ! -e '<PWSAFE_HASH_FILE>'
for f in '<PWSAFE_FILE_1>' '<PWSAFE_FILE_2>'; do pwsafe2john "$f"; done > '<PWSAFE_HASH_FILE>'
```

### master password cracking

```bash
john --wordlist='<WORDLIST>' '<PWSAFE_HASH_FILE>'
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
| 파일마다 하나의 hash line | 여러 backup이 각각 cracking 대상임 | 각 line의 파일명 label을 보존해 결과를 원본과 대응. 한 파일의 master password 복구를 다른 backup 열기 성공으로 확대하지 않음 |
| `pwsafe2john: ...` 오류 | 파일 읽기 또는 Password Safe 형식 파싱 실패 | 원본 파일 종류, 손상 여부, 읽기 권한 확인 |
| 출력이 비어 있음 | 대상이 Password Safe v3 DB가 아니거나 변환 실패 | 확장자 대신 파일 형식과 원본 확보 상태 재확인 |

## 변경 영향과 로컬 산출물 정리

- 원본 `.psafe3`·backup은 읽기 입력이므로 삭제하지 않는다. 이번 작업이 새로 만든 `<PWSAFE_HASH_FILE>`만 정확한 경로로 구분한다.
- 파생 hash와 John pot/session은 master password를 노출할 수 있는 민감 산출물이다. 가능하면 [[보호된 파일 및 아카이브 크래킹]]의 작업별 pot·session 경로를 사용한다. 기본 shared `john.pot`을 사용했다면 이번 작업의 값만 안전하게 분리해 삭제하는 범용 절차를 제공하지 않으므로 잔여 가능성을 남긴다.

```bash
rm -- '<PWSAFE_HASH_FILE>'
test ! -e '<PWSAFE_HASH_FILE>'
```

삭제 실패 시 경로·소유권과 John 프로세스가 파일을 열고 있는지 먼저 확인한다. 정리 완료는 파생 hash 부재와 작업별 pot·session 정리를 각각 확인했을 때만 표현한다.

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
- [[오프라인 해시 크래킹]]

## 실전 진입

Password Safe 파일을 확보했다면 이 문서에서 hash를 추출한 뒤 [[보호된 파일 및 아카이브 크래킹]]의 `john` 또는 `hashcat` 분기로 이어간다.

## 참고 링크

- [Password Safe 공식 저장소](https://github.com/pwsafe/pwsafe)
