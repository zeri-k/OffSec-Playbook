---
tags:
  - 기능/형식변환
실행환경: ["Linux"]
필요조건: ["Office 파일 접근"]
결과: ["해시"]
---

# office2john

## 도구 개요

`office2john`은 비밀번호로 보호된 Microsoft Office 문서의 encryption metadata를 John 형식의 hash로 변환하는 도구다. Office 문서를 직접 여는 도구가 아니라 문서 비밀번호를 오프라인으로 복구할 입력을 준비하는 데 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 문서 원본을 직접 열 수 없을 때 오프라인 password cracking 단계로 넘긴다.

- 대상: `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx` 등 보호된 Office 파일
- 후속 도구: John the Ripper, Hashcat


## 표준 사용법

```bash
office2john.py <office_file> > <hash_file>
```

## 대표 예시

### Office 문서에서 hash 추출

```bash
office2john.py Protected.docx > protected-docx.hash
```

### 추출한 Office hash cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt protected-docx.hash
```

### crack 결과 확인

```bash
john protected-docx.hash --show
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| `<office_file>` | 보호된 Office 문서 |
| `>` | 추출된 hash를 파일로 저장 |
| `john --format` | 필요 시 Office hash format 명시 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `$office$*...` hash line | Office encryption metadata를 John 형식으로 변환 | `john <hash_file>` 또는 해당 Hashcat mode로 진행 |
| 파일명 label과 hash | 여러 문서의 결과를 원본별로 구분 | 결과 label을 유지해 복구한 password를 원본에 대응 |
| `not encrypted` 또는 빈 출력 | 문서가 password-protected Office 형식이 아님 | 파일 보호 방식과 원본 형식을 확인 |
| parsing traceback/error | 손상된 OOXML/OLE 문서 또는 지원하지 않는 encryption | 원본 재수집과 도구 지원 형식 확인 |

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
