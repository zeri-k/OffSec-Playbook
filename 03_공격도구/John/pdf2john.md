---
tags:
  - 기능/형식변환
실행환경: ["Linux"]
필요조건: ["PDF 파일 접근"]
결과: ["해시"]
---

# pdf2john

## 도구 개요

`pdf2john`은 암호화된 PDF의 encryption dictionary를 John 형식의 hash로 추출하는 변환 도구다. PDF를 직접 복호화하는 대신 user 또는 owner password를 오프라인으로 복구할 입력을 만들 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- PDF 열람/편집 비밀번호를 오프라인으로 추측할 때 쓴다.

- 대상: 암호화된 PDF 파일
- 후속 도구: John the Ripper, Hashcat


## 표준 사용법

```bash
pdf2john.py <pdf_file> > <hash_file>
```

## 대표 예시

### PDF에서 hash 추출

```bash
pdf2john.py PDF.pdf > pdf.hash
```

### PDF password cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt pdf.hash
```

### crack 결과 확인

```bash
john pdf.hash --show
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| `<pdf_file>` | 보호된 PDF 파일 |
| `>` | 추출된 hash를 파일로 저장 |
| `john --wordlist` | wordlist 기반 cracking |
| `john --show` | crack 결과 확인 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `$pdf$...` hash line | PDF encryption dictionary를 John 입력으로 추출 | `john <hash_file>`로 user/owner password 복구 시도 |
| `V=`, `R=` 같은 encryption 정보 | PDF 암호화 revision과 kernel 선택 단서 | 지원 format과 후보 길이 제약을 확인 |
| `not encrypted` | password protection이 없는 PDF | cracking 대신 문서 내용을 직접 검토 |
| PDF parse 오류 | 손상 파일 또는 지원하지 않는 PDF 구조 | 원본 재수집과 다른 PDF parser로 형식 확인 |

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
