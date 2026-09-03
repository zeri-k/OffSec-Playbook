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

`klist`는 Windows 로그온 세션이나 Linux ccache·keytab에 저장된 Kerberos 계정·서비스 이름, TGT·service ticket, 만료 시간과 암호화 유형을 표시한다. ticket을 재사용하기 전 계정·서비스 범위와 유효 시간을 확인하거나 cache 상태를 진단할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: ticket를 사용하는 현재 Windows 또는 Linux 세션
- 필요한 입력: 현재 ticket cache, 지정 ccache 또는 keytab; Linux에서는 필요하면 `KRB5CCNAME`
- 확인 기준: `Default principal`에 표시된 계정, `Service principal`에 표시된 서비스, 암호화 형식, 시작/만료/갱신 시간을 함께 본다.


## 표준 사용법

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
| ticket cache 목록 | 현재 세션의 Kerberos ticket 확인 | 만료 시간, SPN, realm, forwardable 여부 확인 |
| TGT/TGS 존재 | Kerberos 인증 재사용 가능성 | 접근하려는 서비스 SPN과 ticket 범위 비교 |
| cache 없음 | 현재 세션에 ticket 없음 | `runas`, `Rubeus`, `KRB5CCNAME` 등 ticket 확보 흐름 확인 |
| 만료 또는 realm 불일치 | 인증 실패 가능성 | 시간 동기화, 도메인, ticket 재발급 확인 |

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
