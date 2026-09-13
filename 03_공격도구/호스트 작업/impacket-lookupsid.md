---
tags:
  - 환경/ad
  - 서비스/smb
  - 기능/열거
실행환경: ["Linux"]
필요권한: ["대상 SAMR·LSAT 조회를 허용하는 인증 계정 또는 null session"]
필요조건: ["대상 SMB/RPC 접근", "필요 시 도메인 또는 로컬 계정 인증 자료"]
결과: ["도메인 SID", "사용자·그룹 이름과 RID 대응"]
---

# impacket-lookupsid

## 도구 개요

`impacket-lookupsid`는 Windows의 SMB/RPC 인터페이스를 통해 도메인 또는 로컬 SAM SID와 사용자·그룹 이름에 대응하는 Relative Identifier(RID)를 열거한다. LDAP 조회가 어렵거나 ticket 생성에 필요한 SID·RID 대응 관계를 확인할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 TCP/445와 관련 RPC에 접근 가능한 Impacket 설치 Linux 호스트.
- 인증 입력: 대상이 허용하는 도메인·로컬 계정의 비밀번호·NT hash·Kerberos ticket. null session이 허용되면 빈 인증도 시도할 수 있다.
- 결과 사용: 도메인 SID, 실제 사용자 이름과 RID를 ticket PAC·대상 계정 식별에 사용한다.

## 표준 사용법

```bash
impacket-lookupsid [options] '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>'
```

## 대표 예시

### 도메인 계정으로 DC SID·RID 열거

```bash
impacket-lookupsid '<DOMAIN>/<USER>:<PASSWORD>@<DC_FQDN>'
```

### NT hash로 인증

```bash
impacket-lookupsid -hashes :<NT_HASH> '<DOMAIN>/<USER>@<DC_FQDN>'
```

### null session 시도

```bash
impacket-lookupsid -no-pass '<TARGET>'
```

확인할 출력:

- `Domain SID is: <DOMAIN_SID>`와 `<RID>: <DOMAIN>\<NAME> (SidTypeUser)` 형식의 대응 관계.
- 사용자 이름과 RID가 같은 행에 표시되어야 위조 ticket의 `-user-id` 입력으로 사용할 수 있다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-hashes` | LM:NT hash로 인증 | 비밀번호 대신 NT hash를 보유했을 때 |
| `-k`, `-no-pass` | Kerberos ticket 사용 또는 비밀번호 입력 생략 | ccache를 사용하거나 null session을 시험할 때 |
| `-target-ip` | 명령의 대상 이름은 유지하고 실제 연결할 IP 지정 | NetBIOS·FQDN을 사용하지만 이름 해석이 되지 않을 때 |
| `-port 139` 또는 `-port 445` | SMB named pipe 연결 포트 선택 | 대상이 한쪽 SMB 포트만 허용할 때 |
| `-domain-sids` | 로컬 계정 도메인 SID 대신 대상의 주 도메인 SID 조회 | 도메인 SID 열거가 필요하고 요청이 DC로 전달될 수 있을 때 |
| `<MAX_RID>` | 검사할 최대 RID를 두 번째 위치 인자로 지정 | 기본 4000보다 넓거나 좁은 범위를 확인할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Domain SID is:` | 도메인 또는 로컬 SAM SID 확인 | 대상이 도메인 SID인지 로컬 SID인지 구분 |
| `SidTypeUser` | RID가 사용자 계정에 대응 | 사용자 이름·RID를 함께 기록 |
| `SidTypeGroup` 또는 `SidTypeAlias` | RID가 그룹·별칭에 대응 | 그룹 권한과 도메인·로컬 범위 확인 |
| `rpc_s_access_denied` | SID 조회 권한 부족 | 인증 계정·null session 허용 여부와 RPC 접근 확인 |
| `STATUS_LOGON_FAILURE` | 인증 실패 | 도메인·계정·비밀번호·hash 형식 확인 |

## 관련 공격기법

- [[자식 도메인 ExtraSids Golden Ticket]]
- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]

## 참고 링크

- [Impacket lookupsid](https://github.com/fortra/impacket/blob/master/examples/lookupsid.py)
