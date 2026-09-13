---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/열거
실행환경: ["Windows", "Linux"]
필요권한: ["로컬 세션"]
필요조건: ["현재 세션의 Kerberos ticket cache 또는 keytab"]
결과: ["정보", "티켓"]
---

# klist

## 도구 개요

`klist`는 Windows 로그온 세션이나 Linux ccache·keytab에 저장된 Kerberos 계정·서비스 이름, TGT·service ticket, 시간 필드와 암호화 유형을 표시한다. 현재 cache 자료를 점검하는 도구이며, 목록 표시만으로 KDC 발급 요청 성공·현재 유효성 또는 대상 서비스 수락을 검증하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: ticket를 사용하는 현재 Windows 또는 Linux 세션
- 필요한 입력: 현재 ticket cache, 지정 ccache 또는 keytab; Linux에서는 필요하면 `KRB5CCNAME`
- 확인 기준: `Default principal`에 표시된 계정, `Service principal`에 표시된 서비스, 암호화 형식, 시작/만료/갱신 시간을 함께 본다. 시간 범위와 현재 시각을 비교하고 실제 사용은 Kerberos 지원 서비스의 응답으로 별도 확인한다.
- `<CCACHE_FILE>`은 앞 단계가 만든 Linux cache의 절대 경로이며 `KRB5CCNAME`은 그 file을 가리킨다. cache 존재·principal 표시는 KDC ticket 발급이나 service access를 대신하지 않는다.


## 표준 사용법

Windows default cache와 Linux `<CCACHE_FILE>`은 current session 또는 `KRB5CCNAME`이 가리키는 서로 다른 cache source다. `Default principal`·`Service principal`·expiry는 cache state literal이며 ticket file 존재·listing은 KDC validation이나 target service acceptance를 뜻하지 않는다.

```bash
klist [options]
```

## 대표 예시

### 현재 Kerberos ticket cache 확인

```bash
klist
```

### keytab 파일에 들어 있는 계정·서비스 항목 확인

```bash
klist -k -t /opt/specialfiles/carlos.keytab
```

### 특정 ccache 파일을 지정해 ticket 확인

```bash
KRB5CCNAME=./<CCACHE_FILE> klist
```

### Windows에서 ticket cache 확인/정리

```cmd
klist
klist purge
```

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| ticket cache 목록 | 현재 세션에 표시된 ticket 자료 존재 | 시간 필드, SPN, realm, flag를 확인하고 실제 요청·사용 결과와 대조 |
| TGT/TGS 항목 존재 | cache에 해당 KDC·서비스 principal용 자료가 있음 | 현재 시각과 유효 시간, 접근하려는 SPN을 비교한 뒤 서비스 인증 시도 |
| cache 없음 | 현재 세션에 ticket 없음 | `runas`, `Rubeus`, `KRB5CCNAME` 등 ticket 확보 흐름 확인 |
| End Time 경과 또는 realm 불일치 | 표시 자료를 현재 요청에 사용할 수 없는 조건 | 시간 동기화, 도메인과 ticket 재발급 확인 |

KDC 발급은 AS·TGS 요청의 성공 응답으로, 서비스 인증은 SMB·LDAP·WinRM 등 대상 서비스가 제시된 자료를 수락한 응답으로 확인한다. `klist purge`는 현재 로그온 세션의 cache를 삭제하는 변경 명령이므로 단순 확인 과정에서 실행하지 않는다.

## 관련 공격기법

- [[Pass the Ticket]]
- [[Linux Kerberos keytab ccache 악용]]
- [[OverPass the Hash]]

## 주요 옵션과 명령

| 항목 | 설명 |
| --- | --- |
| `klist` | 현재 ticket cache 확인 |
| `-k` | keytab 파일 표시 |
| `-t` | keytab timestamp 표시 |
| `KRB5CCNAME` | 사용할 ccache 파일 지정 |
| `purge` | Windows에서 ticket cache 삭제 |
