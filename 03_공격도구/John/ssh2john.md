---
tags:
  - 기능/형식변환
실행환경: ["Linux"]
필요조건: ["암호화된 SSH 개인키 접근"]
결과: ["해시"]
---

# ssh2john

## 도구 개요

`ssh2john`은 암호화된 PEM 또는 OpenSSH 개인키에서 passphrase용 John hash를 추출하는 변환 도구다. 개인키의 passphrase 복구 입력을 준비하며, 그 결과만으로 SSH 계정 인증 성공까지 확인하는 도구는 아니다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- `.ssh/id_rsa` 같은 키 파일을 얻었지만 passphrase가 걸려 있을 때 사용한다.

- 대상: passphrase로 보호된 SSH private key
- 후속 도구: John the Ripper


## 표준 사용법

```bash
ssh2john.py <private_key> > <hash_file>
```

## 대표 예시

### SSH private key에서 hash 추출

```bash
ssh2john.py SSH.private > ssh.hash
```

### key passphrase cracking

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
```

### crack 결과 확인

```bash
john ssh.hash --show
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| `<private_key>` | 암호화된 SSH private key |
| `>` | John 입력용 hash 파일 생성 |
| `john --wordlist` | passphrase 후보 목록 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `$ssh$...` 또는 `$sshng$...` hash line | PEM 또는 newer OpenSSH private key passphrase hash 추출 | 출력 접두사에 맞는 John format으로 cracking |
| key 파일명 label | 어느 private key에서 나온 hash인지 식별 | 복구 후 해당 key의 사용 권한과 SSH 연결을 별도 검증 |
| `not encrypted` | passphrase가 없는 private key | cracking 없이 파일 권한을 보호하고 직접 사용 가능 여부 확인 |
| key parse 오류 | private key 구조 손상 또는 지원하지 않는 encoding | 원본 key 형식과 도구 지원 여부 확인 |

## 관련 공격기법

- [[보호된 파일 및 아카이브 크래킹]]
