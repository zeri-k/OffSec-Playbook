---
tags:
  - 서비스/tftp
대표포트:
  - "U:69"
서비스:
  - TFTP
---

# TFTP 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:69/UDP`의 Trivial File Transfer Protocol(TFTP)에 요청을 보낼 수 있고, 계정 인증 없이 확인할 원격 파일명 후보가 있는 상태에서 시작한다. TFTP에는 디렉터리 listing이 없으므로 파일별 읽기 요청(RRQ)과 쓰기 요청(WRQ)의 결과를 직접 구분한다.

**첫 화면 상태:** 지금 가능한 일은 `<REMOTE_FILE>` 후보의 인증 없는 읽기·쓰기를 검증하는 것이다. 성공하면 해당 파일의 내용 또는 검증된 저장 경로를 얻는다. TFTP ERROR code 1은 해당 서버 관점의 파일 부재를, code 2는 경로·파일 권한 거부를 뜻하지만 timeout은 파일 부재·UDP 필터링·응답 경로 문제를 구분하지 못한다. 쓰기 성공도 설정 반영·파일 실행을 뜻하지 않는다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | UDP 69 응답 | `nmap -sU -sV -p69 <TARGET>` | TFTP 응답, open\|filtered와 timeout을 서로 다른 관찰 상태로 구분한다. |
| 2 | 대상과 전송 모드 | `tftp <TARGET>` 후 `status`, `binary` | 올바른 대상과 binary 모드에서 파일 요청을 준비한다. |
| 3 | 파일명 후보의 READ | `get <REMOTE_FILE> <LOCAL_FILE>` | 로컬 파일 생성과 내용으로 읽기 가능성을 확인한다. |
| 4 | 파일명 후보의 WRITE | `put <LOCAL_FILE> <REMOTE_FILE>` 후 같은 파일 재수신 | 검증용 작은 파일의 저장과 재조회가 모두 되는지 확인한다. |

**출력 해석 경계:** `open|filtered`와 timeout은 UDP 응답 관찰일 뿐 TFTP 서비스 존재·차단·파일 부재를 확정하지 않는다. `get`으로 생성된 로컬 파일과 내용만 해당 원격 파일의 읽기를 확정한다. `put`의 전송 완료 메시지는 서버 저장을 보장하지 않으므로 같은 파일을 재수신해야 쓰기를 확정하며, 그 결과도 파일 적용·실행은 확정하지 않는다.

### 선택적 파일 쓰기 proof

TFTP에는 원격 삭제 명령이 없다. 서버의 셸·관리 인터페이스·공유 경로에서 정확한 proof 파일을 제거할 수 있을 때만 쓰기를 확인한다.

`<TARGET>`은 TFTP 서버 IP 또는 FQDN(예: `192.0.2.10`)이다. 이 명령은 현재 실행 호스트에서 `<TARGET>`에 전송하며, `PROOF`는 현재 디렉터리에 새로 만드는 고유 파일명이고 재수신 파일은 `downloaded-$PROOF`다. 양쪽 hash 비교는 업로드 원본과 재수신본의 동일성이 WRITE 판단을 바꾸므로 유지한다.

```bash
PROOF="tftp-proof-$(date -u +%Y%m%dT%H%M%SZ)-$$.txt"
printf 'TFTP write proof: %s\n' "$PROOF" > "$PROOF"
tftp <TARGET> -m binary -c put "$PROOF" "$PROOF"
tftp <TARGET> -m binary -c get "$PROOF" "downloaded-$PROOF"
sha256sum "$PROOF" "downloaded-$PROOF"
```

- 업로드와 재다운로드 hash가 모두 일치해야 원격 파일 쓰기로 판단한다.
- 서버 측 경로에서 정확한 `$PROOF`를 삭제한 뒤 같은 이름의 `get`이 `File not found`를 반환하는지 확인한다.
- 서버 측 cleanup 경로가 없으면 `put`을 수행하지 않는다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| 설정·백업·펌웨어 파일명 후보 | [[TFTP 설정 파일 수집]] | `tftp` | 설정, 계정, SNMP community, 내부 IP 또는 장비 정보 확보 |
| PXE 또는 네트워크 장비 단서 | [[TFTP 설정 파일 수집]] | `tftp` | 부팅·장비 설정 파일의 읽기 범위 |
| `get` 성공 | [[TFTP 설정 파일 수집]] | `tftp` | 특정 파일의 인증 없는 읽기 |
| `put` 후 재수신 성공 | 이 문서의 선택적 파일 쓰기 proof | `tftp` | 검증된 원격 파일 쓰기, 설정·실행 영향은 별도 확인 |

## 서비스 고유 주의 사항

- TFTP timeout은 파일 없음, UDP 필터링, 응답 경로 문제를 구분하기 어렵다.
- 디렉터리 listing이 없으므로 환경 단서에서 정확한 파일명을 만드는 과정이 중요하다.
- 인증 부재, 파일 READ, 파일 WRITE는 서로 다른 상태이며 쓰기는 무해한 파일로만 검증한다.
