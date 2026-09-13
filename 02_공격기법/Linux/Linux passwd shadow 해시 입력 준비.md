---
tags:
  - 환경/linux
시작조건: ["Linux shell에서 /etc/passwd와 /etc/shadow 읽기 가능", "또는 같은 호스트·시점의 passwd·shadow 사본 확보"]
필요권한: ["/etc/shadow 또는 수집한 shadow 사본 읽기 권한"]
필요조건: ["passwd·shadow 쌍의 출처 호스트와 수집 시점 확인", "John unshadow 유틸리티", "민감 산출물을 보호할 작업 경로"]
결과: ["Linux 계정명과 password hash가 대응된 unshadow 입력", "잠금·passwordless·별도 인증 상태 후보"]
---

# Linux passwd shadow 해시 입력 준비

## 한 줄 판단

같은 Linux 호스트와 수집 시점의 `passwd`·`shadow` 쌍을 읽을 수 있으면, `unshadow`로 계정 정보와 password hash를 대응시켜 [[오프라인 해시 크래킹]]에 넘길 작업별 입력을 만든다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 대상 Linux shell 또는 사본을 보유한 분석 호스트 | `hostname`, 사본 수집 기록과 파일 경로 확인 | 다른 호스트의 파일을 합치지 않고 출처를 다시 대조 |
| 현재 계정과 권한 | `/etc/passwd`와 `/etc/shadow` 읽기 또는 두 사본 읽기 가능 | `id`, `test -r`, `stat` | shadow 읽기 거부를 hash 부재로 해석하지 않고 권한 상태를 유지 |
| 파일 대응 | 두 파일이 같은 호스트·수집 시점의 계정 상태 | 호스트 식별자·수집 시각·hash를 Vault 밖 작업 기록에 보존 | 시점이 다른 사본은 사용자 대응이 틀릴 수 있으므로 다시 수집 |
| 도구 | 분석 호스트의 `unshadow` 사용 가능 | `command -v unshadow` | John the Ripper 패키지·build를 확인하고 원본 파일은 수정하지 않음 |
| 산출물 경로 | 이번 작업 전용 디렉터리와 hash 파일명 확정 | 경로 부재·디스크 용량·ACL 확인 | 기존 파일을 덮어쓰지 않고 새 고유 경로 선택 |

`/etc/passwd`의 두 번째 필드가 `x`면 일반적으로 password hash는 `/etc/shadow`에 있다. shadow password 필드가 `!`로 시작하거나 유효한 `crypt(3)` 값이 아닌 `!`·`*`면 UNIX password 로그인은 잠긴 상태지만 key·Kerberos 등 다른 인증 수단까지 없다고 확정할 수 없다. 빈 password 필드는 passwordless 로그인 가능성이지만 PAM·애플리케이션이 거부할 수 있어 실제 로그인 성공은 별도 검증 결과다.

## 실행

### 1. 대상 Linux shell에서 읽기 범위와 파일 쌍 확인

```bash
hostname
id
stat -c '%n %U:%G %a %s %y' /etc/passwd /etc/shadow
test -r /etc/passwd && test -r /etc/shadow
```

확인할 출력:

- 두 파일의 절대 경로·소유자·mode·수정 시간과 실행 호스트.
- `test` 성공은 현재 프로세스가 두 파일을 읽을 수 있다는 뜻이며 root 세션 획득이나 비밀번호 복구 성공을 의미하지 않는다.
- 수집은 원본을 수정하지 않는 경로를 사용한다. 분석 호스트로 사본을 옮겨야 하면 [[상황별 파일 전송]]에서 암호화·접근 제어·무결성을 만족하는 방식을 고른다.

### 2. Linux 분석 호스트에서 `unshadow` 입력 생성

```bash
umask 077
test ! -e '<LINUX_HASH_WORKDIR>' || exit 1
install -d -m 700 '<LINUX_HASH_WORKDIR>'
test ! -e '<LINUX_HASH_WORKDIR>/unshadowed.hashes' || exit 1
unshadow '<PASSWD_COPY>' '<SHADOW_COPY>' > '<LINUX_HASH_WORKDIR>/unshadowed.hashes'
stat -c '%n %a %s' '<LINUX_HASH_WORKDIR>/unshadowed.hashes'
```

`<LINUX_HASH_WORKDIR>`은 분석 호스트의 작업 전 존재하지 않는 절대 디렉터리(가상 예: `/tmp/linux-hash-20260914a`)다. `<PASSWD_COPY>`와 `<SHADOW_COPY>`는 같은 Linux 호스트·수집 시점에서 얻은 읽기 전용 사본 경로를 사용한다.

확인할 출력:

- `unshadowed.hashes`의 각 행에 계정명·password hash·UID·GID·GECOS·home·shell 필드가 원본 계정과 대응한다.
- 파일이 비어 있거나 `unshadow` 오류가 나면 사본 경로·읽기 권한·형식과 같은 호스트 쌍인지를 먼저 확인한다.
- hash prefix는 알고리즘 후보이며, John format·Hashcat mode는 [[오프라인 해시 크래킹]]에서 전체 필드와 수집 출처를 함께 보고 확정한다.

### 3. 선택적 password history 단서 구분

```bash
test -r /etc/security/opasswd && stat -c '%n %U:%G %a %s %y' /etc/security/opasswd
```

`pam_pwhistory`가 기본 경로를 쓴 호스트의 `/etc/security/opasswd`는 이전 비밀번호 history를 담을 수 있다. 파일 존재·읽기 성공은 현재 password hash 확보가 아니며, 복구된 과거 값도 현재 인증 성공을 입증하지 않는다. Kerberos·LDAP 계정의 중앙 password history로 확대해석하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `unshadow` 입력에 유효한 hash·계정 label이 대응 | 현재 로컬 password hash 후보 확보 | 오프라인 cracking 입력 준비 | [[오프라인 해시 크래킹]] |
| password 필드가 `!`·`*` 또는 잠금 prefix | UNIX password 로그인은 잠금 후보 | 다른 인증 수단 미확인 | key·Kerberos·service-specific 인증을 별도 확인 |
| password 필드가 비어 있음 | passwordless 로그인 후보지만 PAM·service 정책 미확인 | 인증 결과 미확정 | 정상 client로 해당 서비스의 수락 여부를 최소 검증 |
| `/etc/security/opasswd`만 읽힘 | 과거 로컬 password history 후보 | 현재 credential 미확인 | 현재 계정·서비스와 분리해 후보 패턴만 평가 |
| shadow 읽기 거부 | 현재 권한으로 hash 수집 불가 | 일반 Linux shell 유지 | [[Linux 셸 확보 후 초기 열거와 권한 상승]]에서 다른 읽기·권한 경로 선택 |

## 변경 영향과 복구

`passwd`·`shadow`·`opasswd`는 읽기 입력이며 수정하지 않는다. 이 문서에서 새로 만드는 항목은 분석 호스트의 `<LINUX_HASH_WORKDIR>`과 `unshadowed.hashes`다. 검토가 끝나면 이번 작업이 생성한 정확한 파일과 빈 디렉터리만 정리한다.

```bash
rm -- '<LINUX_HASH_WORKDIR>/unshadowed.hashes'
rmdir -- '<LINUX_HASH_WORKDIR>'
test ! -e '<LINUX_HASH_WORKDIR>'
```

원본 사본·John pot·session·복구 평문이 별도 경로에 있다면 [[오프라인 해시 크래킹]]의 식별자별 정리를 따른다. shared cache나 안전하게 분리할 수 없는 사본이 남으면 전체 정리 완료로 기록하지 않는다. 삭제 실패 시 경로·소유권·파일을 열고 있는 프로세스를 먼저 확인한다.

## 관련 상태 라우터

- [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 공격기법

- [[오프라인 해시 크래킹]]
- [[상황별 파일 전송]]

## 관련 도구

- [[john]]
- [[hashcat]]

## 참고 링크

- [Linux `passwd(5)` file format](https://man7.org/linux/man-pages/man5/passwd.5.html)
- [shadow-utils `shadow(5)` file format](https://man7.org/linux/man-pages/man5/shadow.5.html)
- [Linux-PAM `pam_pwhistory(8)`](https://man7.org/linux/man-pages/man8/pam_pwhistory.8.html)
- [Openwall John the Ripper usage examples — `unshadow`](https://www.openwall.com/john/doc/EXAMPLES.shtml)
