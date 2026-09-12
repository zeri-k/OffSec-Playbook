---
tags:
  - 기능/크래킹
실행환경: ["Linux", "Windows"]
필요조건: ["해시 파일"]
결과: ["자격증명"]
---

# john

## 도구 개요

`john`은 hash에 wordlist, rule, single, incremental mode를 적용해 비밀번호와 passphrase를 오프라인으로 복구하는 도구다. 다양한 `*2john` 변환 결과를 한 흐름에서 처리하고 복구 상태를 확인하거나 중단한 세션을 이어갈 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: John the Ripper를 실행할 수 있는 Linux 또는 Windows 호스트
- 필요한 입력: John 형식의 hash 파일과 필요하면 `--format`
- 후보 입력: wordlist/rule, single 또는 incremental mode; 보호 파일은 대응하는 `*2john` 도구로 먼저 변환한다.


## 표준 사용법

```bash
john [options] <hash_file>
```

## 대표 예시

### wordlist 기반 cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

### 사용자 정보 기반 single mode

```bash
john --single passwd.hash
```

### brute force 성격의 incremental mode

```bash
john --incremental hashes.txt
```

### crack된 결과 확인

```bash
john hashes.txt --show
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `--wordlist` | wordlist 기반 cracking |
| `--single` | 사용자 정보 기반 single crack mode |
| `--incremental` | brute force 성격의 incremental mode |
| `--format` | hash format 명시 |
| `--rules` | word mangling rule 적용 |
| `--show` | 이미 crack된 결과 출력 |
| `--session`, `--restore` | 세션 저장/재개 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Loaded <N> password hash` | John이 hash와 format을 인식 | 필요하면 `--format`을 명시하고 wordlist/rule 실행 |
| `<plaintext> (<account>)` | 후보 plaintext와 hash label을 찾음 | `john --show <hash_file>`로 전체 결과 확인 |
| `Session completed` | 현재 mode의 후보 처리가 끝남 | 다른 rule, wordlist, incremental 설정 검토 |
| `No password hashes loaded` | 변환 결과 또는 format을 인식하지 못함 | `*2john` 출력과 `--format`을 재확인 |
| `<N> password hashes cracked, <N> left` | `--show` 기준 복구/미복구 수 | 미복구 hash에만 다음 후보군 적용 |

## 관련 공격기법

- [[오프라인 해시 크래킹]]
- [[AS-REP Roasting]]
- [[Kerberoasting]]
- [[보호된 파일 및 아카이브 크래킹]]

## 참고 링크

- [John the Ripper 문서](https://www.openwall.com/john/doc/)
