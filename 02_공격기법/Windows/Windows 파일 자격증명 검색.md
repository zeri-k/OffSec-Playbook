---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 shell 또는 파일 접근 경로 확보"]
필요권한: ["현재 세션 계정의 대상 디렉터리·파일 읽기 권한"]
필요조건: ["현재 사용자와 프로필 경로", "검색할 역할 기반 경로와 키워드"]
결과: ["파일에 기록된 평문 비밀번호·접속 문자열 후보", "SSH key·KeePass DB·보호 문서", "내부 호스트·경로·서비스 정보"]
---

# Windows 파일 자격증명 검색

## 한 줄 판단

대상 Windows 호스트의 현재 세션 계정으로 읽을 수 있는 프로필·문서·스크립트·config·PowerShell history에서 평문 비밀번호, 접속 문자열, key와 보호 파일을 찾아 적용 계정·서비스를 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 대상 Windows 셸 또는 회수 파일이 있는 분석 호스트에서 검색 가능 | `hostname`, `whoami`, 현재 디렉터리와 파일 경로 확인 | 명령을 실행하는 호스트와 검색 대상 파일의 실제 위치 확인 |
| 현재 계정 또는 인증 수단 | 현재 Windows 세션 계정과 프로필 소유자가 식별됨 | `whoami`, `$env:USERPROFILE` 확인 | 다른 사용자의 프로필을 현재 사용자 것으로 해석하지 말고 SID·경로 확인 |
| 현재 권한 | 현재 세션 계정이 검색할 디렉터리와 개별 파일을 읽을 수 있음 | `dir`, `Get-ChildItem`, 파일 내용 읽기 결과 확인 | 디렉터리 목록 권한과 파일 ACL을 분리하고 읽을 수 있는 경로로 범위 조정 |
| 공격 대상의 조건 | 사용자 역할상 사용자명·평문 비밀번호·NT hash·API token·개인키·배포 설정이 남을 가능성이 있는 경로 식별 | history, config, scripts, IT·dev 폴더와 파일명 확인 | 무차별 전체 검색 대신 역할·확장자·최근 변경 파일로 축소 |
| 필요한 파일·목록·주소 | 검색 키워드와 발견 값의 적용 서비스·호스트를 기록할 수 있음 | password·pwd·creds·key와 파일 경로 확인 | 발견 값만 떼어내지 말고 주변 계정명·호스트·서비스 문맥 확인 |

## 실행

검색 키워드와 파일 경로는 현재 셸이 읽을 수 있는 범위의 입력이며, 발견한 password·key·token은 반드시 파일 경로와 계정·서비스 문맥을 함께 기록한다. 이름만 같은 값이나 파일명만으로 실제 인증 가능성을 판단하지 않는다.

1. 현재 명령 실행 호스트, 세션 사용자와 읽을 수 있는 프로필·공유 경로를 확인한다.
2. 파일명과 내용 키워드로 후보를 좁히고, 발견 값과 파일 경로·적용 계정·서비스 문맥을 함께 기록한다.
3. 평문 비밀번호, 접속 문자열, key와 보호 파일을 형식별로 분리한다.
4. 검색 거부·결과 없음·credential 검증 실패를 파일 ACL, 검색 범위, 계정 형식과 대상 서비스 문제로 각각 분기한다.

### findstr 검색

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
findstr /SIM /C:"pwd" *.txt *.ini *.cfg *.config *.xml *.ps1
```

확인할 출력:

- 현재 계정이 읽을 수 있는 파일 경로와 그 안의 비밀번호 후보. `Access is denied`는 해당 경로·파일 ACL 미충족으로 분리한다.

### PowerShell history와 흔한 파일

```powershell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
Get-ChildItem -Recurse $env:USERPROFILE -Include *pass*,*cred*,*.kdbx,unattend.xml -ErrorAction SilentlyContinue
```

확인할 출력:

- 과거 명령이 기록된 사용자, KeePass DB·unattend·script의 실제 경로와 파일 소유자.

### LaZagne

```cmd
if not exist "<LAZAGNE_PATH>" echo MISSING
"<LAZAGNE_PATH>" all
```

확인할 출력:

- 브라우저, WinSCP, mail, sysadmin tool credential.
- `all`은 현재 권한으로 지원 모듈 전체를 검사한다. 대상 역할이 브라우저·sysadmin tool처럼 분명하면 [[lazagne]]의 해당 module부터 실행해 범위와 출력량을 줄인다.
- 실행 파일을 이번 작업에서 반입했다면 실행 전 부재를 확인한 exact 경로만 기록하고, 수집·검증 뒤 [[lazagne#변경 영향과 복구]]에 따라 제거한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 평문 비밀번호 또는 DB 접속 문자열 | 파일 기반 credential 노출 | 서비스 credential 후보 확보 | 계정 형식과 대상 서비스를 확인 |
| SSH 개인키, KeePass DB 또는 보호 문서 | 계정·호스트 단서가 있는 개인키 또는 비밀번호 보호 파일 발견 | 개인키 인증 후보 또는 오프라인 비밀번호 복구 대상 확보 | 개인키·파일의 암호화 여부 확인 후 관련 기법으로 전환 |
| unattend 또는 배포 스크립트의 credential | 배포 계정 정보 노출 | 로컬 또는 도메인 계정 후보 | 계정 범위를 구분해 낮은 영향으로 검증 |
| 정상 클라이언트로 다른 서비스 또는 사용자 인증 성공 | 수집한 credential이 해당 계정 형식과 서비스에서 현재 유효함 | 해당 Identity의 서비스 인증 성공 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 세션·파일 권한·관리자 권한 분리 확인 |
| 검색 결과 없음 | 범위 또는 키워드 부족 가능 | 파일 기반 credential 미확인 | 사용자 역할 기반 경로와 dev/IT 폴더 확인 |
| 암호화 파일 | 평문 접근 불가 | 보호 파일만 확보 | [[보호된 파일 및 아카이브 크래킹]] |
| credential 검증 실패 | 오래된 정보, 계정 형식 또는 서비스 불일치 | 유효 credential 미확인 | 계정 형식, 대상 서비스, 잠금 정책 확인 |

## 확인할 출력과 권한

- 파일 경로와 민감 값 또는 보호 파일 형식을 함께 확인한다.
- 파일 읽기 권한과 파일에 기록된 계정의 권한은 동일하지 않다.
- 인증 성공 후 `whoami`, 그룹, 서비스별 권한으로 로컬 사용자와 도메인 Identity를 구분한다.
- 평문 비밀번호, SSH private key, KeePass DB와 보호 문서는 각각 검증 절차가 다르므로 모두 `credential`로 뭉쳐 기록하지 않는다.

## 후속 공격 연결

- [[원격 비밀번호 공격]]
- [[Windows 저장 자격증명 수집]]
- [[보호된 파일 및 아카이브 크래킹]]

## 관련 상태 라우터

- 파일에서 credential을 확인했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[lazagne]]
- [[powershell]]
- [[netexec]]
