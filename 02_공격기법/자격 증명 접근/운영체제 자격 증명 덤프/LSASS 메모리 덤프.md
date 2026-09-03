---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 관리자급 또는 SYSTEM 세션 확보", "명령 실행 세션에서 LSASS 프로세스 접근 가능"]
필요권한: ["상승된 로컬 관리자 토큰, SeDebugPrivilege 또는 SYSTEM 중 선택한 덤프 방식에 필요한 권한"]
필요조건: ["대상 호스트의 LSASS PID", "덤프 파일을 저장·회수할 경로 또는 직접 조회를 허용하는 실행 환경"]
결과: ["NTLM hash", "Kerberos key·ticket", "DPAPI key 또는 master key", "존재할 때만 평문 비밀번호"]
---

# LSASS 메모리 덤프

## 한 줄 판단

상승된 Windows 세션에서 대상 호스트의 LSASS 메모리를 덤프하거나 직접 조회해 현재·최근 로그온의 NTLM hash, Kerberos key·ticket, DPAPI key 또는 master key와 존재할 때만 평문 비밀번호를 추출한다.

## 사용할 때

- 현재 보유 정보: 대상 Windows 호스트의 shell과 현재 사용자·상승 토큰 정보. 대상 사용자 비밀번호·hash·ticket은 수집 전에는 보유하지 않아도 된다.
- 명령 실행 위치: LSASS가 실행 중인 같은 Windows 호스트에서 덤프를 만들거나, 이미 회수한 `lsass.dmp`를 오프라인 분석 호스트에서 처리한다.
- 현재 권한과 대상: 로컬 Administrators 그룹 소속, 실제 상승된 관리자 토큰, `SeDebugPrivilege`, SYSTEM은 서로 다르며 보호된 LSASS에서는 각 방식의 접근 결과를 확인한다.
- 공격 대상과 결과: 현재·최근 로그온 세션에 남은 NTLM hash, Kerberos key·ticket과 DPAPI key를 얻는다. hash·key·ticket 획득은 평문 비밀번호 복구나 도메인 관리자 권한을 의미하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|
| 명령 실행 위치와 대상 | 덤프는 LSASS가 실행 중인 대상 Windows 호스트에서 수행 | hostname, 현재 shell, `Get-Process lsass` | 이미 회수한 dump가 있으면 오프라인 분석으로 분기 |
| 현재 권한 | 상승된 관리자 토큰, `SeDebugPrivilege` 또는 SYSTEM 중 방식에 필요한 권한 | `whoami /priv`, `whoami /groups` | UAC 상승, 현재 토큰과 LSASS 보호 상태 확인 |
| 대상 프로세스 | LSASS PID와 접근 가능한 프로세스 상태 | `tasklist /svc`, `Get-Process lsass` | PID 재확인, PPL·Credential Guard 등 보호 상태 확인 |
| 저장·회수 경로 | 덤프 파일을 쓸 로컬 경로와 분석 호스트로 옮길 경로 | 파일 생성 뒤 크기·ACL 확인 | 디스크 공간, 쓰기 권한과 준비한 회수 경로 확인 |

## 실행

1. 권한과 LSASS PID를 확인한다.
2. 가능하면 덤프 파일을 만들어 오프라인 분석한다.
3. `pypykatz` 또는 Mimikatz로 로그온 세션별 NTLM hash·Kerberos AES/RC4 key·ticket·평문 비밀번호 후보를 추출한다.
4. NTLM은 PtH, Kerberos key/ticket은 PtT/OverPass, DPAPI는 저장 credential 복호화에 연결한다.

### Windows 대상 호스트에서 덤프 생성

#### PID 확인과 덤프 생성

```powershell
Get-Process lsass
rundll32 C:\Windows\System32\comsvcs.dll, MiniDump <PID> C:\Windows\Temp\lsass.dmp full
```

확인할 출력:

- `lsass.dmp` 파일 생성.
- 파일 생성은 메모리 수집 성공일 뿐 credential 추출 성공은 아니다. 생성한 경로와 파일 크기를 확인한 뒤 오프라인 분석으로 넘긴다.

### Linux 분석 호스트에서 덤프 분석

#### 오프라인 분석

```bash
pypykatz lsa minidump lsass.dmp
```

확인할 출력:

- MSV NT hash, Kerberos key, logon session, DPAPI key 또는 master key.
- 평문 password 필드가 비어 있어도 NTLM hash·AES key·ticket·DPAPI key가 있으면 각각 별도의 인증 또는 복호화 후보로 기록한다.

### Windows 대상 호스트에서 직접 조회

#### Mimikatz 직접 조회

```cmd
mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
sekurlsa::ekeys
```

확인할 출력:

- NTLM, AES key, ticket, credential manager 결과.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `lsass.dmp` 파일 생성 | LSASS 메모리 덤프 성공 | 오프라인 분석 대상 확보 | pypykatz로 NTLM hash·Kerberos key·ticket·평문 후보 추출 |
| MSV NTLM hash 또는 Kerberos AES/RC4 key | 계정명이 연결된 NTLM hash 또는 Kerberos key 추출 | Pass the Hash 또는 새 TGT 요청 후보 | [[Pass the Hash]] 또는 [[OverPass the Hash]] |
| ticket 또는 DPAPI key | Kerberos 인증용 ticket 또는 저장소 복호화용 DPAPI key 추출 | ticket 또는 DPAPI key 확보 | [[Pass the Ticket]] 또는 [[Windows 저장 자격증명 수집]] |
| access denied | 현재 컨텍스트의 디버그 권한 부족 | LSASS 미접근 | 관리자/SYSTEM과 `SeDebugPrivilege` 확인 |
| 덤프 생성 차단 | 방어 제품 또는 보호 설정 개입 | 메모리 덤프 미획득 | 오프라인 hive와 secretsdump 등 다른 수집 경로 검토 |
| 평문 비밀번호 없음 | Credential Guard 또는 WDigest 설정 영향 가능 | 평문 미획득 | NTLM hash, AES key, ticket, DPAPI key 존재 여부 확인 |

## 확인할 출력과 권한

- 덤프 파일 생성과 그 안에서 NTLM hash·Kerberos key·ticket·평문 비밀번호 후보를 추출하는 것은 별도 성공 단계다.
- 로컬 관리자 그룹 멤버십만 보지 말고 실제 상승된 토큰, SYSTEM 또는 활성화된 `SeDebugPrivilege`를 확인한다.
- 추출된 도메인 계정의 NTLM hash·Kerberos key·ticket이 허용하는 실제 도메인 권한은 그룹과 서비스 인증을 통해 별도로 검증한다.

## 후속 공격 연결

- NTLM hash: [[Pass the Hash]]
- Kerberos key/ticket: [[OverPass the Hash]], [[Pass the Ticket]]
- 저장 credential: [[Windows 저장 자격증명 수집]]

## 관련 상태 라우터

- password 또는 hash를 획득했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- Kerberos ticket 또는 도메인 계정이 확인됐으면: [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 관련 도구

- [[mimikatz]]
- [[pypykatz]]
- [[powershell]]
- [[hashcat]]
