---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/권한상승
실행환경: ["Linux"]
필요권한: ["자식 도메인 관리자 또는 자식 도메인 krbtgt를 DCSync할 수 있는 권한"]
필요조건: ["같은 포리스트의 자식·부모 신뢰 관계", "자식 도메인 인증 자료", "자식·부모 KDC와 부모 DC RPC 도달성"]
결과: ["부모 도메인 대상 계정의 NT hash와 Kerberos key", "선택 시 부모 DC 원격 셸"]
---

# impacket-raiseChild

## 도구 개요

`impacket-raiseChild`는 같은 포리스트의 자식 도메인에서 부모 도메인으로 이어지는 ExtraSids 공격 체인을 자동화한다. 자식 `krbtgt` DCSync, Golden Ticket 생성, 부모 계정 DCSync와 선택적 원격 실행을 단계별로 연속 검증할 때 사용한다.

기본 실행도 자식 `krbtgt`와 부모 대상 계정의 장기 key material을 출력한다. 이는 단순 trust 열거가 아니며, 수동 단계별 확인이 필요한 작업에서는 [[자식 도메인 ExtraSids Golden Ticket]]을 우선한다. `-target-exec`는 명시적으로 선택할 때만 부모 대상에 PsExec 방식의 service·파일·프로세스를 추가할 수 있다.

## 필요한 입력과 실행 환경

- 실행 위치: 자식·부모 도메인을 DNS로 해석하고 양쪽 KDC TCP/UDP 88번과 부모 DC RPC에 연결할 수 있는 Linux 호스트.
- 인증 입력: 자식 도메인 관리자 또는 자식 `krbtgt`를 DCSync할 수 있는 계정의 비밀번호·NT hash·AES key.
- 기본 추출 대상: 부모 도메인의 RID 500 계정. `-targetRID`로 변경할 수 있다.
- 선택적 원격 실행: `-target-exec`는 먼저 추출한 부모 대상 계정 credential을 사용해 지정 호스트에 접속한다.

## 표준 사용법

```bash
impacket-raiseChild [options] '<CHILD_FQDN>/<CHILD_USER>:<PASSWORD>'
```

## 대표 예시

### 자식 도메인 비밀번호 인증으로 자동 체인 실행

```bash
impacket-raiseChild '<CHILD_FQDN>/<CHILD_ADMIN>:<PASSWORD>'
```

### 자식 도메인 NT hash로 실행

```bash
impacket-raiseChild -hashes :<CHILD_ADMIN_NT_HASH> '<CHILD_FQDN>/<CHILD_ADMIN>'
```

### 부모 대상 RID와 원격 실행 호스트 지정

```bash
impacket-raiseChild -targetRID <ROOT_TARGET_RID> -target-exec <ROOT_DC_FQDN> '<CHILD_FQDN>/<CHILD_ADMIN>:<PASSWORD>'
```

### 생성한 Golden Ticket을 파일로 저장

기존 파일을 덮어쓰지 않는 exact 경로를 `<RAISECHILD_CCACHE_PATH>`로 정하고, 실행 전 부재를 확인한다.

```bash
test ! -e '<RAISECHILD_CCACHE_PATH>'
impacket-raiseChild -w '<RAISECHILD_CCACHE_PATH>' '<CHILD_FQDN>/<CHILD_ADMIN>:<PASSWORD>'
```

확인할 출력:

- 자식·부모 도메인 발견, 부모 Enterprise Admins SID, 자식 `krbtgt` key, Golden Ticket 생성, 부모 대상 계정 credential 추출이 단계별로 표시되어야 한다.
- 원격 셸이 열렸다면 앞 단계에서 추출한 부모 대상 계정 자격 증명으로 별도 인증한 결과다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-hashes` | 자식 도메인 요청자 계정의 LM:NT hash로 인증 | 평문 비밀번호 없이 자식 `krbtgt` DCSync를 요청할 때 |
| `-aesKey` | 자식 도메인 요청자 계정의 Kerberos AES key로 인증 | AES 기반 인증 자료를 보유했을 때 |
| `-k`, `-no-pass` | ccache 등 Kerberos 인증 사용 | 요청자 ticket이 있고 DNS·realm·시간이 정합할 때 |
| `-targetRID` | 부모 도메인에서 DCSync할 대상 RID 지정 | 기본 RID 500 이외 계정을 추출할 때 |
| `-target-exec` | 추출한 부모 대상 계정 credential로 원격 실행 | credential 추출과 셸 획득을 연속 검증할 때 |
| `-w <CCACHE_FILE>` | 생성한 Golden Ticket을 지정한 ccache 파일에 저장 | 자동화 중간 산출물을 수동 검증할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| 자식·포리스트 도메인과 SID 출력 | 도메인 발견 단계 성공 | 도메인 관계와 Enterprise Admins SID 확인 |
| 자식 `krbtgt` hash·key 출력 | 자식 DCSync 성공 | Golden Ticket 생성 단계 확인 |
| ticket 생성 출력 | ExtraSids ccache 생성 단계 성공 | 양쪽 KDC와 부모 realm service ticket 처리 확인 |
| 부모 대상 계정 hash·key 출력 | 부모 DRSUAPI DCSync 성공 | 추출 대상과 credential 종류 확인 |
| 원격 shell 출력 | 추출한 credential로 별도 원격 실행 성공 | 실제 `whoami`, `hostname`과 관리자 권한 확인 |
| 도메인 발견·KDC·DRSUAPI 오류 | 자동 체인의 해당 단계에서 중단 | 마지막 성공 단계 다음의 DNS·포트·인증·권한을 확인 |

## 변경 영향과 복구

공격 목표 달성과 정리는 별도로 판정한다. 부모 hash·AES key, 자식 `krbtgt` key, Golden Ticket을 화면·파일로 노출한 영향과 인증·복제 감사 기록은 파일·process 종료로 되돌릴 수 없다.

| 생성·변경 항목 | 작업 전 확인과 식별 | 종료·정리 | 완료 확인 |
|---|---|---|---|
| `-w`로 저장한 ccache | `test ! -e '<RAISECHILD_CCACHE_PATH>'`; exact 경로와 실행 shell의 기존 `KRB5CCNAME`을 Vault 밖에 기록 | 해당 ticket을 사용하는 client 종료 뒤 `kdestroy -c 'FILE:<RAISECHILD_CCACHE_PATH>'`; 기존 환경값을 복원하거나 원래 없었다면 `unset KRB5CCNAME` | `test ! -e '<RAISECHILD_CCACHE_PATH>'`와 원래 shell 환경 확인 |
| `-target-exec` 원격 shell·PsExec 자원 | 실행 전 target service 목록과 대상 경로를 좁게 확인; 도구 출력의 생성 service 이름, 원격 파일 경로와 로컬 client PID를 기록 | shell 안의 원격 정리를 먼저 마친 뒤 정상 `exit`; 연결이 끊겼다면 기록한 exact service·file·PID만 대상 host 관리자에게 대조·정리 요청 | 기록한 service·file·PID가 없음을 각각 확인. 식별값을 기록하지 못했다면 원격 정리 완료로 표시하지 않음 |

이름 pattern으로 모든 service나 실행 파일을 삭제하지 않는다. 정상 종료 시 PsExec가 자체 정리를 보고하더라도, 실행이 중단됐으면 실제 target 상태를 확인하기 전에는 완료로 간주하지 않는다. 이미 출력·전송된 credential과 KDC·DRSUAPI·서비스 접근 기록은 잔여 영향으로 남긴다.

## 관련 공격기법

- [[자식 도메인 ExtraSids Golden Ticket]]

## 참고 링크

- [Impacket raiseChild](https://github.com/fortra/impacket/blob/master/examples/raiseChild.py)
- [Impacket releases](https://github.com/fortra/impacket/releases)
