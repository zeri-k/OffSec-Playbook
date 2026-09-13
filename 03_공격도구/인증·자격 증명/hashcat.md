---
tags:
  - 기능/크래킹
실행환경: ["Linux", "Windows"]
필요조건: ["해시 파일"]
결과: ["자격증명"]
---

# hashcat

## 도구 개요

`hashcat`은 다양한 hash mode와 wordlist, rule, mask를 조합해 비밀번호를 오프라인으로 복구하는 도구다. GPU backend와 공격 mode를 세밀하게 선택할 수 있어 큰 후보 공간을 빠르게 처리하거나 후보 생성 규칙을 조정하는 작업에 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: Hashcat을 실행할 수 있는 Linux 또는 Windows 호스트; GPU 사용 시 호환 driver/runtime이 필요하다.
- 필요한 입력: 원본을 보존한 hash 파일, 정확한 hash mode, 공격 mode
- 후보 입력: wordlist, rule 또는 mask; 파일/볼륨 hash는 전용 변환 도구로 먼저 추출한다.


## 최우선 실행 원칙

이 Vault에서 작성하거나 안내하는 모든 Hashcat 명령에는 `--backend-ignore-opencl -d 1 -O -w 3`을 반드시 붙인다. 크래킹, 결과 확인, wordlist 생성과 장치 확인 명령에도 생략하지 않는다. 파이프 또는 리다이렉션이 있으면 이 옵션 조합을 Hashcat 명령의 마지막에 둔다.

## 표준 사용법

```bash
hashcat -m <hash_mode> -a <attack_mode> <hash_file> <wordlist_or_mask> [options] --backend-ignore-opencl -d 1 -O -w 3
```

## 대표 예시

### NTLM hash cracking

```bash
hashcat -m 1000 -a 0 ntlm.hash rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
```

### NetNTLMv2 hash에 rule 적용

```bash
hashcat -m 5600 -a 0 netntlmv2.hash rockyou.txt -r rules/best64.rule --backend-ignore-opencl -d 1 -O -w 3
```

### Linux sha512crypt hash cracking

```bash
hashcat -m 1800 -a 0 linux-sha512crypt.hash rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
```

### crack 결과 확인

```bash
hashcat --show -m 1000 ntlm.hash --backend-ignore-opencl -d 1 -O -w 3
```

### rule 후보 미리 보기

```bash
hashcat <WORDLIST> -r <RULE_FILE> --stdout --backend-ignore-opencl -d 1 -O -w 3
```

출력은 rule을 적용한 후보 문자열이며 hash 복구 결과가 아니다. 작은 표본으로 대소문자·치환·접미 규칙이 의도대로 적용되는지 확인한 뒤 실제 cracking에 `-r`을 사용한다. `--force`는 오류·경고를 우회하는 개발용 option이므로 후보 생성의 기본 조건으로 넣지 않는다.

### mask 공격

```bash
hashcat -m <HASH_MODE> -a 3 <HASH_FILE> '?u?l?l?l?l?d?s' --backend-ignore-opencl -d 1 -O -w 3
```

이 mask는 대문자 1자, 소문자 4자, 숫자 1자, 기호 1자의 정확한 7자리 후보 공간이다. 정책·관찰 근거 없이 길이와 문자 위치를 고정하지 않고 `?l`·`?u`·`?d`·`?s` 또는 `-1`~`-4` custom charset으로 실제 후보 범위를 표현한다. `Exhausted`는 그 mask만 소진됐다는 뜻이다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-m` | hash mode 지정. 예: NTLM `1000`, NetNTLMv2 `5600`, sha512crypt `1800` |
| `-a` | attack mode 지정. `0` straight, `3` mask, `6/7` hybrid |
| `-r` | rule 파일 적용 |
| `-o` | crack 결과 저장 파일 지정 |
| `--show` | 이미 crack된 결과 출력 |
| `--username` | `user:hash` 형식에서 username 필드 무시 |
| `--session`, `--restore` | 세션 이름 지정/중단 작업 재개 |
| `--potfile-path` | 복구 결과 cache를 작업별 파일로 분리 |
| `--restore-file-path` | 중단 상태 파일 경로를 작업별로 분리 |
| `--status` | 진행 상태 주기적 출력 |
| `--backend-ignore-opencl` | OpenCL backend를 비활성화 |
| `-d <device>` | 사용할 backend device 선택 |
| `-O` | 최적화 kernel 사용. 지원 가능한 후보 길이가 줄어들 수 있음 |
| `-w <0-4>` | workload profile. 기본값은 `2`, `3`은 높은 부하 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Hashes: <N> digests` | hash file과 mode가 로드됨 | `Hash.Mode`와 대상 hash 종류를 다시 확인하고 공격 진행 |
| `Status...........: Cracked`와 `Recovered` | 하나 이상 plaintext를 찾음 | `--show`로 결과를 분리하고 원본 접근 경로에서 제한적으로 검증 |
| `Status...........: Exhausted` | 현재 후보 공간을 모두 시도했지만 미복구 | rule, mask, 대상 기반 wordlist를 조정 |
| `Token length exception` 또는 `No hashes loaded` | hash 형식, mode, 추출물 불일치 | 변환 도구 결과와 `-m`을 재확인 |
| `Device #...`와 `Speed.#...` | 선택된 backend와 실제 처리 속도 | 기대 device가 아니면 `hashcat -I --backend-ignore-opencl -d 1 -O -w 3` 후 `-d`를 조정 |

## 버전과 환경 차이

- device 번호와 사용 가능한 backend는 driver/runtime과 설치된 Hashcat 버전에 따라 달라진다. 장비별 옵션을 적용하기 전 `hashcat -I --backend-ignore-opencl -d 1 -O -w 3`로 device를 확인한다.
- `--backend-ignore-opencl`은 OpenCL interface를 끄므로 CUDA 또는 다른 backend만 의도적으로 쓸 때 사용한다. `-d 1`은 어느 장비에서나 같은 GPU를 뜻하지 않는다.
- `-O`는 optimized kernel을 선택해 후보 길이 제한을 바꿀 수 있고, `-w 3`은 기본 `2`보다 높은 부하다. 장시간 실행 전 냉각과 시스템 사용량을 확인한다.
- 기본 potfile은 다른 작업의 복구 결과를 재사용한다. 민감 결과를 분리해야 하면 `--potfile-path <TASK_POTFILE>`과 `--restore-file-path <TASK_RESTORE>`를 지정하고, `--show`에도 같은 potfile을 사용한다. 종료된 작업의 exact 파일만 제거하며 shared potfile을 일괄 삭제하지 않는다.

## 관련 공격기법

- [[오프라인 해시 크래킹]]
- [[AS-REP Roasting]]
- [[Kerberoasting]]
- [[보호된 파일 및 아카이브 크래킹]]
- [[IPMI hash 수집과 크래킹]]

## 참고 링크

- [Hashcat 공식 Wiki](https://hashcat.net/wiki/doku.php?id=hashcat)
- [Hashcat potfile과 restore 동작](https://hashcat.net/wiki/doku.php?id=frequently_asked_questions)
