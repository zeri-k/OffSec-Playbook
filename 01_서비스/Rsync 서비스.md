---
tags:
  - 환경/unix
  - 서비스/rsync
대표포트:
  - "T:873"
서비스:
  - Rsync
---

# Rsync 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:873`의 Rsync daemon에 연결할 수 있고, 아직 모듈명·daemon 계정·파일 권한은 확인하지 않은 상태에서 시작한다. 공개 모듈 목록, 알려진 모듈별 인증 요구와 읽기·쓰기를 구분한다. Rsync over Secure Shell(SSH)은 TCP 873 daemon과 별개이며 SSH 사용자·비밀번호 또는 개인키와 SSH 포트로 판단한다.

**첫 화면 상태:** 지금 가능한 일은 daemon 모듈별 목록·읽기·쓰기를 검증하는 것이다. 성공하면 파일 내용 또는 특정 모듈의 저장 권한을 얻는다. `#list` 거부 시 알려진 모듈명을 직접 확인하고, `auth failed`이면 rsync daemon 계정·secrets file 정책을, `Permission denied`이면 모듈의 read only·path와 서버 파일 권한을 확인한다. 목록 노출·파일 다운로드·쓰기와 원격 명령 실행은 서로 다른 상태다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | daemon greeting과 모듈 | `nc -nv <TARGET> 873` 후 `#list` | `@RSYNCD`, 모듈명과 설명을 확인한다. |
| 2 | 모듈 파일 목록 | `rsync -av --list-only rsync://<TARGET>/<MODULE>` | 익명 또는 인증된 READ 가능성과 파일·권한 단서를 확인한다. |
| 3 | 파일 동기화 | `rsync -av rsync://<TARGET>/<MODULE> ./loot/` | 실제 다운로드된 설정·secret·SSH 키·백업을 확인한다. |
| 4 | SSH 기반 rsync | `rsync -av -e "ssh -p<PORT>" <USER>@<TARGET>:<PATH> ./loot/` | SSH credential과 비표준 포트로 가능한 파일 접근을 확인한다. |

**출력 해석 경계:** `@RSYNCD` greeting과 `#list`는 daemon 응답과 공개된 모듈 목록만 확정하며 모든 모듈 접근은 확정하지 않는다. `--list-only` 성공은 나열된 경로의 읽기 가능성 단서이고 실제 파일 동기화 결과가 있어야 내용 접근을 확정한다. 업로드 완료 메시지는 무해한 파일의 재조회로 확인해야 하며, 검증된 WRITE도 배포·실행 권한은 뜻하지 않는다.

### 선택적 모듈 쓰기 proof

서버 측 실제 module 경로를 알고 SSH·셸·마운트 등으로 정확한 proof 파일을 제거할 수 있을 때만 수행한다.

```bash
PROOF="rsync-proof-$(date -u +%Y%m%dT%H%M%SZ)-$$.txt"
printf 'Rsync write proof: %s\n' "$PROOF" > "$PROOF"
rsync -av "$PROOF" rsync://<TARGET>/<MODULE>/"$PROOF"
rsync --list-only rsync://<TARGET>/<MODULE>/"$PROOF"
rsync -av rsync://<TARGET>/<MODULE>/"$PROOF" "downloaded-$PROOF"
sha256sum "$PROOF" "downloaded-$PROOF"
```

서버 관리 경로에서 정확한 파일 하나만 제거한 뒤 원격 조회가 실패하는지 확인한다. 전체 module에 `--delete`를 적용하지 않는다.

```bash
rm -f -- <RSYNC_MODULE_ROOT>/"$PROOF"
rsync --list-only rsync://<TARGET>/<MODULE>/"$PROOF"
```

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| module 목록 노출 | [[Rsync 공개 모듈 수집]] | `rsync`, `netcat` | 익명으로 확인 가능한 모듈명과 설명 |
| 인증 없는 READ 또는 `--list-only` 성공 | [[Rsync 공개 모듈 수집]] | `rsync` | 모듈 내 파일 목록과 파일 읽기 |
| WRITE 가능한 module | 이 문서의 선택적 모듈 쓰기 proof | `rsync` | 무해한 파일로 검증한 모듈 쓰기 |
| `.ssh`, `secrets.yaml`, backup 파일 | [[Rsync 공개 모듈 수집]] | `rsync` | 다른 서비스의 사용자 이름·비밀번호·키·설정 후보 |
| rsync over SSH의 사용자 이름·비밀번호·개인키 또는 제한 모듈 계정 | [[SSH credential 및 키 인증 검증]] | `rsync`, `ssh` | SSH 사용자 권한 범위의 파일 접근 |

## 서비스 고유 주의 사항

- 모듈 목록이 막혀 있어도 정확한 모듈명을 알면 접근될 수 있다.
- Rsync daemon과 Rsync over SSH를 구분하고 비표준 SSH 포트는 `-e "ssh -p<PORT>"`로 지정한다.
- 파일 목록, READ, WRITE와 파일 실행 가능성은 각각 다른 상태이며 쓰기는 무해하게 검증한다.

## 참고 링크

- [rsync(1) official manual](https://rsync.samba.org/ftp/rsync/rsync.1.html)
- [rsyncd.conf(5) official manual](https://rsync.samba.org/ftp/rsync/rsyncd.conf.5.html)
