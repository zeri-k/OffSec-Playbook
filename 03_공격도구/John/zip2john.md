---
tags:
  - 기능/형식변환
실행환경: ["Linux"]
필요조건: ["ZIP 파일 접근"]
결과: ["해시"]
---

# zip2john

## 도구 개요

`zip2john`은 비밀번호로 보호된 ZIP archive의 암호화 정보를 John 형식의 hash로 변환하는 도구다. archive와 암호화된 member를 구분해 오프라인 비밀번호 복구용 입력을 준비할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- ZIP 내부 파일을 열 수 없을 때 오프라인 password cracking 단계로 넘긴다.

- 대상: password-protected ZIP 파일
- 후속 도구: John the Ripper


## 표준 사용법

```bash
zip2john <zip_file> > <hash_file>
```

## 대표 예시

### ZIP archive에서 hash 추출

```bash
zip2john ZIP.zip > zip.hash
```

### ZIP password cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

### crack 결과 확인

```bash
john zip.hash --show
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| `<zip_file>` | 보호된 ZIP 파일 |
| `>` | John용 hash 파일 저장 |
| `john --show` | crack된 비밀번호 확인 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `$pkzip$...` 또는 `$zip2$...` hash line | ZIP encryption 방식에 맞는 John hash 생성 | 출력 접두사와 John format을 확인해 cracking |
| archive 파일명과 member 정보 | archive 안에서 암호화된 member 식별 | 복구 결과가 필요한 member에 실제로 적용되는지 확인 |
| `is not encrypted` | ZIP archive에 password protection이 없음 | cracking 없이 내용을 직접 확인 |
| `zipfile is corrupt` 또는 parse 오류 | archive 손상 또는 형식 문제 | 원본 archive 재수집과 무결성 확인 |

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
