---
tags:
  - 환경/windows
시작조건: ["Windows 사용자 세션 또는 사용자 프로필 접근 확보"]
필요권한: ["현재 사용자 프로필과 DPAPI 보호 저장소 읽기", "LSASS 또는 다른 사용자의 DPAPI key 접근 시 로컬 관리자·SeDebugPrivilege"]
필요조건: ["현재 사용자 컨텍스트 또는 DPAPI master key·backup key", "저장 항목의 대상·사용자 정보"]
결과: ["저장 credential 대상·사용자 단서", "복호화된 평문 비밀번호 또는 token", "runas 실행에 사용할 대화형 저장 계정"]
---

# Windows 저장 자격증명 수집

## 한 줄 판단

현재 Windows 사용자 세션에서 DPAPI 복호화를 호출할 수 있거나 해당 프로필 파일과 DPAPI master key를 복호화할 로그온 비밀번호·NT hash·도메인 backup key가 있다면, Credential Manager·Windows Vault·브라우저·앱 저장소의 평문 비밀번호 또는 token을 복호화한다.

## 사용할 때

- 현재 보유 정보: Windows foothold의 현재 사용자와 프로필 경로를 알고 있거나 LSASS·SECURITY·사용자 프로필에서 해당 사용자의 DPAPI master key·DPAPI_SYSTEM secret을 확보한 상태다.
- 명령 실행 위치와 도달성: 대상 Windows 사용자의 세션에서 Credential Manager GUI, `cmdkey`, Mimikatz 또는 LaZagne를 실행할 수 있고, 검증할 원격 리소스가 있으면 그 서비스에도 접근할 수 있다.
- 현재 가능한 행동과 결과: 현재 사용자에게 보이는 저장 항목을 열거하거나 복호화하고, 저장된 대화형 계정은 별도 `runas` 실행 기법의 입력으로 넘긴다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 대상 Windows 사용자의 세션·프로필에서 명령 실행 가능, 원격 대상 검증 시 해당 서비스 접근 가능 | `hostname`, `whoami`, `%USERPROFILE%`와 대상 포트 확인 | 프로필 소유자와 현재 세션, 원격 서비스 도달성 확인 |
| 현재 계정 또는 인증 수단 | 현재 세션 사용자와 저장 항목에 기록된 사용자·대상이 식별됨 | `whoami`, `cmdkey /list`의 `Target`·`User` 확인 | 현재 사용자와 저장 credential 사용자를 같은 계정으로 가정하지 말고 각각 확인 |
| 현재 권한 | 현재 사용자 저장소는 해당 사용자 컨텍스트로 접근, LSASS·다른 사용자의 DPAPI key는 로컬 관리자·SeDebugPrivilege 필요 | 프로필 ACL, `privilege::debug` 결과와 도구 오류 확인 | 현재 사용자 범위로 축소하거나 상승 토큰·DPAPI master key 확보 여부 확인 |
| 공격 대상의 조건 | CredMan·Vault·브라우저·앱에 실제 저장 항목이 존재 | `cmdkey /list`, Vault 경로와 LaZagne 결과 확인 | 저장 항목이 없으면 [[Windows 파일 자격증명 검색]]으로 전환 |
| 필요한 파일·목록·주소 | DPAPI masterkey 또는 현재 사용자 세션, 저장 대상과 검증할 서비스 식별 | 사용자 SID·프로필·Target 정보를 함께 기록 | 서로 다른 사용자의 masterkey·프로필 혼용 여부 확인 |

## 실행

1. Credential Manager GUI와 `cmdkey /list`로 저장 credential 대상과 사용자를 확인한다.
2. 현재 권한이 허용하는 범위에서 Mimikatz/LaZagne로 CredMan·DPAPI·브라우저 credential을 추출한다.
3. 나온 평문 비밀번호와 token의 대상 서비스를 식별하고 계정 잠금 정책을 고려해 서비스별로 검증한다.

### 자격 증명 관리자 GUI 확인

```cmd
rundll32 keymgr.dll,KRShowKeyMgr
```

확인할 출력:

- 현재 사용자에게 표시되는 Windows·일반 자격 증명 항목과 대상·사용자.
- GUI에 보이는 항목과 `cmdkey /list` 결과의 차이. 내보내기 기능은 별도 암호화 백업 파일을 생성하므로 이 열거 단계에서는 실행하지 않는다.

### 저장 credential 확인

```cmd
whoami
cmdkey /list
```

확인할 출력:

- 현재 사용자 저장소에 등록된 `Target`, `Type`, `User`, persistence. 이 출력만으로 평문 비밀번호나 저장된 사용자 권한을 얻은 것은 아니다.

### 도구 기반 추출

```cmd
mimikatz.exe
privilege::debug
sekurlsa::credman
```

```cmd
LaZagne.exe all
```

확인할 출력:

- 항목별 사용자, 대상 서비스와 함께 표시된 CredMan password, 브라우저 저장 비밀번호 또는 token.
- `ERROR kuhl_m_privilege_simple ; RtlAdjustPrivilege` 또는 접근 거부가 나오면 현재 사용자 저장소 조회와 LSASS 접근에 필요한 권한을 구분한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `cmdkey /list`에 다른 계정의 `Domain:interactive` 항목 표시 | 대화형 로그온에 사용할 저장 계정 후보 | runas 실행 입력 | [[저장된 자격 증명으로 runas 프로세스 실행]] |
| CredMan 또는 브라우저의 평문 비밀번호/토큰 | 저장소 복호화 성공 | credential 또는 token 확보 | 대상 서비스에서 유효성과 권한 검증 |
| 항목은 있으나 평문 없음 | DPAPI master key 또는 사용자 컨텍스트 부족 | 저장 항목만 확인 | LSASS, DPAPI master key, 사용자 세션 확인 |
| 결과 없음 | 저장 기능 미사용 또는 현재 프로필 범위 제한 | 저장 credential 미확인 | 파일, 브라우저, 앱 config 검색 |

## 확인할 출력과 권한

- `cmdkey` 목록은 저장 항목 존재만 뜻하며 평문 비밀번호 확보나 사용 성공이 아니다.
- 현재 사용자, 저장 credential의 사용자, 로컬 계정과 도메인 계정을 구분한다.
- 평문 비밀번호, bearer token, DPAPI masterkey와 대화형 저장 계정은 서로 다른 결과이며, hash 또는 Kerberos ticket으로 바꾸어 기록하지 않는다.

## 후속 공격 연결

- 새 credential: [[원격 비밀번호 공격]]
- hash/key 획득: [[Pass the Hash]], [[Pass the Ticket]]
- 대화형 저장 계정: [[저장된 자격 증명으로 runas 프로세스 실행]]
- 로컬 권한 상승 단서: [[Windows 권한 상승 열거]]

## 관련 상태 라우터

- password, token 또는 ticket을 확인했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[mimikatz]]
- [[lazagne]]
