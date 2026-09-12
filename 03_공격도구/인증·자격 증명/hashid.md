---
tags:
  - 기능/해시식별
실행환경: ["Linux"]
필요조건: ["해시 문자열 또는 파일"]
결과: ["정보"]
---

# hashid

## 도구 개요

`hashid`는 hash 문자열의 길이와 문자 형식을 바탕으로 가능한 알고리즘과 John·Hashcat 형식 후보를 제시한다. 크래킹 mode를 정하기 전에 후보를 좁히는 용도이며, 결과는 hash 출처와 prefix를 함께 확인해야 한다.

## 필요한 입력과 실행 환경

- 실행 위치: `hashid`를 실행할 수 있는 Linux 호스트
- 필요한 입력: 식별할 hash 문자열 또는 hash 파일
- 판단 조건: 출력은 형식 후보이므로 길이, prefix, 출처와 함께 John format/Hashcat mode를 교차 확인한다.


## 표준 사용법

```bash
hashid <hash>
```

## 대표 예시

### 단일 hash 형식 추정

```bash
hashid '5f4dcc3b5aa765d61d8327deb882cf99'
```

### John/Hashcat 형식 후보 함께 확인

```bash
hashid -m -j hashes.txt
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-m` | Hashcat mode 후보 출력 |
| `-j` | John format 후보 출력 |
| `-e` | extended mode로 더 많은 후보 표시 |
| `-o` | 결과 파일 저장 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 후보 알고리즘 여러 개 출력 | 해시 형식 후보가 좁혀짐 | 길이, 접두어, 출처를 함께 보고 John/Hashcat mode 선택 |
| 단일 후보에 가까움 | mode 선택 가능성이 높음 | 작은 wordlist로 빠르게 검증 |
| unknown 또는 후보 없음 | 해시가 아니거나 인코딩/포맷이 다름 | 원문, salt, prefix, base64/hex 여부 확인 |
| cracking 도구가 거부 | mode 추정이 틀렸을 수 있음 | 다른 후보 mode와 변환 도구 출력 재확인 |

## 관련 공격기법

- [[오프라인 해시 크래킹]]
