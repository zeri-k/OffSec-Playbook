---
tags:
  - 환경/linux
  - 환경/ad
  - 서비스/kerberos
문서역할: 오케스트레이터
시작조건: ["AD/Kerberos와 통합된 Linux 호스트 셸 확보", "현재 Linux 계정으로 keytab 또는 ccache 읽기 가능"]
필요권한: ["대상 keytab 또는 ccache 파일 읽기 권한", "원격 서비스는 ticket 주체의 해당 서비스 권한"]
필요조건: ["Kerberos realm·DNS·FQDN·시간 정합", "새 ticket 요청 시 <DC_FQDN>:88 접근 가능", "최종 <HOST_FQDN>:<SERVICE_PORT> 접근 가능"]
결과: ["보유 파일의 종류와 ticket 사용 경로 식별", "자격 증명 수집과 ticket 사용 기법으로 분기"]
---

# Linux Kerberos keytab ccache 악용

## 한 줄 판단

AD와 통합된 Linux 호스트에서 읽을 수 있는 keytab 또는 ccache를 발견했으면, 파일에서 principal·ticket 종류·만료 범위를 먼저 확인하고 자격 파일 수집은 [[Linux 파일 자격증명 검색]], ticket을 적용한 서비스 접근은 [[Pass the Ticket]]으로 나누어 진행한다.

## 사용할 때

- 현재 보유 접근: Linux 셸에서 Kerberos 설정과 읽을 수 있는 `*.keytab` 또는 `krb5cc_*` 파일을 확인했다.
- 명령 실행 위치: 파일 소유자·권한과 Kerberos 자료를 확인할 Linux 호스트다.
- 현재 계정·권한: 현재 Linux 계정이 파일을 읽을 수 있어야 한다. 로컬 파일 소유자와 ticket principal은 같은 계정이라고 가정하지 않는다.
- 성공 범위: 이 문서는 파일 발견, keytab으로 ticket을 발급할 경로, 기존 ccache를 사용할 경로를 선택한다. 실제 파일 수집과 원격 서비스 인증은 연결된 세부 기법에서 판정한다.

## 실행

### AD 통합 상태와 Kerberos 파일 확인

```bash
cat /etc/krb5.conf
realm list
find /tmp /var/tmp /home -name 'krb5cc_*' -o -name '*.keytab' 2>/dev/null
klist -c <CCACHE_FILE>
```

확인할 출력:

- Kerberos realm, ccache의 계정, ticket 종류·SPN과 만료 시각.
- 파일을 찾았더라도 현재 Linux 계정으로 읽을 수 있는지 소유자·모드·ACL을 별도로 확인한다.

### keytab으로 새 TGT 발급

```bash
klist -k -e <KEYTAB_FILE>
kinit -kt <KEYTAB_FILE> <PRINCIPAL>
klist
```

확인할 출력:

- keytab의 계정과 암호화 유형.
- `kinit` 오류 없이 `klist`에 해당 계정의 유효한 TGT가 표시되는지 확인한다.

### ccache 적용과 서비스 접근 확인

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
smbclient -k //<HOST_FQDN>/<SHARE>
impacket-wmiexec -k -no-pass <DOMAIN>/<USER>@<HOST_FQDN>
```

확인할 출력:

- `smbclient`의 공유·파일 목록은 해당 ticket 계정의 SMB 인증과 공유 권한을 보여 준다.
- `impacket-wmiexec`의 원격 prompt와 명령 출력은 WMI 원격 실행 권한을 보여 준다.
- ticket 보유, 서비스 인증과 실제 파일·원격 명령 권한을 각각 따로 판정한다.

## 판단 경로

| 관찰한 상태 | 의미 | 다음 기법 | 확인할 결과 |
|---|---|---|---|
| 설정·백업·홈·임시 경로에서 keytab 또는 ccache 후보를 발견함 | 로컬 자격 자료 후보만 식별됨 | [[Linux 파일 자격증명 검색]] | 파일 소유자·모드·ACL과 실제 읽기 가능 여부 |
| ccache에 유효한 특정 서비스용 TGS가 있음 | 표시된 SPN에만 즉시 재사용 가능 | [[Pass the Ticket]] | principal·SPN·만료 시각과 해당 서비스 인증 결과 |
| ccache에 유효한 TGT가 있음 | DC에서 필요한 서비스용 TGS를 새로 요청할 수 있음 | [[Pass the Ticket]] | realm·DNS·시간·KDC 도달성과 최종 서비스 인증 결과 |
| keytab에 principal과 사용할 수 있는 key가 있음 | 해당 principal의 새 TGT 발급 후보 | [[Linux 파일 자격증명 검색]]에서 파일과 principal을 확정한 뒤 [[Pass the Ticket]] | TGT 발급 성공과 ticket 주체의 최종 서비스 권한을 분리 확인 |
| ccache가 만료됐고 keytab도 없음 | 재사용 가능한 Kerberos 자료가 없음 | [[Linux 파일 자격증명 검색]] | 다른 백업·서비스 설정·홈 경로의 자격 자료 |
| ticket은 유효하지만 서비스 인증이 실패함 | SPN·FQDN·realm·시간·서비스 권한 중 하나가 맞지 않음 | [[Pass the Ticket]] | IP 대신 FQDN, 올바른 SPN과 ticket 주체의 서비스 권한 |

## 상태 구분

- keytab·ccache 파일 읽기 성공은 현재 Linux 계정의 로컬 파일 권한만 입증한다.
- keytab으로 TGT를 발급한 결과와 ccache에서 기존 TGT·TGS를 읽은 결과를 구분한다.
- ticket 보유는 원격 서비스 인증 성공이 아니며, 서비스 인증 성공도 share 쓰기·원격 명령·관리자·도메인 고권한을 의미하지 않는다.

## 다음 행동

- 자격 파일 후보를 찾고 읽을 수 있는지 확인: [[Linux 파일 자격증명 검색]]
- 유효한 TGT·TGS를 적용하고 서비스별 권한 확인: [[Pass the Ticket]]
- ticket으로 WinRM 세션을 열 수 있으면: [[WinRM 원격 PowerShell 세션]]
- ticket으로 WMI 명령 실행이 가능하면: [[WMI 원격 명령 실행]]
- 새 Kerberos 계정 또는 서비스 접근을 확보했으면: [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- 새 ticket 또는 key를 확보했지만 사용할 서비스가 정해지지 않았으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[klist]]
- [[impacket-ticketConverter]]
