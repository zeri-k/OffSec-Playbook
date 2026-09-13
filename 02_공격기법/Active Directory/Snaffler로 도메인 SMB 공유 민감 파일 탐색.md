---
tags:
  - 환경/windows
  - 환경/ad
  - 서비스/smb
시작조건: ["Windows 도메인 사용자 세션 확보", "도메인과 DC 식별", "도메인 호스트의 SMB 공유에 연결 가능한 위치"]
필요권한: ["현재 도메인 계정으로 호스트·공유 열거", "후보 공유와 파일을 읽을 권한"]
필요조건: ["Snaffler.exe", "검색할 AD 도메인명", "로그 저장 경로"]
결과: ["현재 계정으로 읽을 수 있는 도메인 SMB 공유", "자격 증명·키·설정·민감 파일 후보", "수동 검증할 UNC 파일 경로"]
---

# Snaffler로 도메인 SMB 공유 민감 파일 탐색

## 한 줄 판단

Windows 도메인 사용자 세션에서 도메인 호스트의 SMB에 연결할 수 있으면 Snaffler로 공유 metadata와 rule hit UNC 경로를 선별한다. 반환 경로는 실제 파일 내용 READ 또는 자격 증명 존재의 증거가 아니므로, 해당 공유·파일 ACL과 별도 읽기 결과를 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | Windows 호스트에서 도메인 DNS와 SMB 대상에 연결 가능 | `hostname`, `whoami`, DC 확인과 445/TCP 연결 확인 | DNS suffix·route·방화벽과 SMB 도달성 확인 |
| 현재 계정 또는 인증 수단 | 도메인 사용자 세션 또는 해당 AD 계정의 네트워크 인증 컨텍스트 | `whoami`, 필요 시 원격 공유 접근 결과 | 로컬 계정인지 도메인 계정인지, `/netonly` 사용 시 실제 원격 인증 결과 확인 |
| 현재 권한 | 호스트·공유 열거와 일부 후보 파일 READ 가능 | Snaffler의 share 순회와 파일 접근 출력 | 인증 성공, share READ와 개별 파일 READ를 각각 구분 |
| 공격 대상의 조건 | 도메인 호스트에 접근 가능한 SMB 공유가 존재 | Snaffler share 열거 출력 | 도메인명, 컴퓨터 목록, SMB 445/TCP와 현재 계정의 공유 접근 범위 확인 |
| 필요한 파일·목록·주소 | `Snaffler.exe`, `<DOMAIN>`, `<OUTPUT_FILE>` | 실행 파일과 출력 경로의 쓰기 가능 여부 확인 | 파일 반입·실행 차단과 로그 경로 ACL 확인 |

## 실행

`<DOMAIN>`은 현재 AD DNS 도메인, `<DC>`는 그 DC, `<OUTPUT_FILE>`은 Windows 실행 호스트의 새 결과 파일(예: `C:\\Temp\\snaffler.txt`)이다. 출력의 `\\<HOST>\\<SHARE>\\<PATH>`는 rule hit 경로이며 파일 내용을 읽을 권한을 보장하지 않는다.

### 1. Windows 공격 호스트의 계정과 도메인 경로 확인

```powershell
hostname
whoami
nltest /dsgetdc:<DOMAIN>
Test-NetConnection <DC> -Port 445
```

확인할 출력:

- `whoami`의 현재 계정 또는 별도 네트워크 인증 컨텍스트의 사용자명.
- `<DOMAIN>`의 DC와 해당 DC 445/TCP 연결 성공.
- DC 연결만으로 다른 도메인 호스트의 공유를 읽을 수 있다고 판단하지 않는다.

### 2. Snaffler로 도메인 공유와 민감 파일 후보 탐색

```powershell
if (Test-Path -LiteralPath '<OUTPUT_FILE>') { throw 'Output file already exists' }
.\Snaffler.exe -s -d <DOMAIN> -o <OUTPUT_FILE> -v data
```

경로 부재 guard가 통과해야 한다. 기존 파일이 있으면 덮어쓰지 않고 이번 실행의 고유 경로를 다시 정한다.

확인할 출력:

- 순회한 호스트·공유와 현재 계정으로 읽을 수 있는 디렉터리.
- rule에 일치한 `\\<HOST>\<SHARE>\<PATH>` 형식의 파일 경로와 분류.
- `<OUTPUT_FILE>`에 저장된 동일한 결과와 실행 중 발생한 접근 오류.

### 3. 후보 파일의 metadata와 내용을 수동 검증

Snaffler 결과에서 우선순위가 높은 파일 하나를 선택해 현재 계정으로 metadata를 조회하고 내용을 READ할 수 있는지 확인한다. 이 두 명령은 ACL 전체를 열거하거나 별도 권한을 증명하지 않는다.

```powershell
Get-Item '\\<HOST>\<SHARE>\<PATH>'
Get-Content -LiteralPath '\\<HOST>\<SHARE>\<PATH>'
```

확인할 출력:

- `Get-Item`으로 경로 metadata를 조회하고 `Get-Content`으로 내용 READ가 되는지.
- 사용자명과 비밀번호, token, private key, connection string 또는 내부 서비스 주소가 함께 있는지.
- rule hit만 있고 파일을 읽지 못하면 자격 증명 확보로 기록하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 공유와 파일 경로가 열거되지만 rule hit가 없음 | 현재 rule·권한 범위에서 우선순위 파일을 찾지 못함 | 자격 증명 미확보 | 특정 공유·확장자·최근 변경 파일을 [[SMB 공유 자격증명 수집]]에서 수동 확인 |
| rule hit 파일을 실제로 읽음 | 현재 AD 계정에 해당 파일 READ 권한이 있음 | 민감 파일 접근 | 내용과 계정·서비스 대응 관계 확인 |
| 계정명이 연결된 평문 비밀번호·token·key·접속 문자열 확인 | 재사용 가능성 있는 자격 증명 자료 발견 | 자격 증명 후보 확보 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 값의 종류와 대상 서비스별 인증 검증 |
| share 순회는 되지만 파일 접근이 거부됨 | share 목록 권한과 파일 ACL이 다름 | 경로만 확인 | 읽을 수 있는 하위 경로로 범위를 좁힘 |
| 도메인·컴퓨터 열거 실패 | 현재 계정·DNS·도메인 컨텍스트 또는 네트워크 문제 | 결과 미판정 | [[AD 도메인 컨텍스트 기본 확인]]에서 도메인·DC·DNS와 현재 계정을 다시 확인 |

## 확인할 출력과 권한

- Snaffler의 rule hit는 민감 파일 후보이며 자격 증명 확보 결과가 아니다.
- 공유 열거, 디렉터리 목록, 개별 파일 READ와 파일 내용의 자격 증명 식별을 각각 구분한다.
- 파일에서 얻은 계정이 로컬 계정인지 AD 계정인지, 어느 호스트·서비스에 적용되는지 확인한다.
- Snaffler 로그에는 UNC 경로와 민감 문자열 후보가 포함될 수 있으므로 실행 결과를 다른 재사용 문서에 복사하지 않는다.

## 변경 영향과 로컬 산출물 정리

Snaffler는 원격 공유를 읽고 `<OUTPUT_FILE>`에 민감 경로·match를 남긴다. `-m`으로 파일 사본을 저장하는 모드는 이 문서의 대표 절차에서 사용하지 않았으므로 복구 대상에 추가하지 않는다. 사용 후 생성 전 부재를 확인한 정확한 log만 삭제한다.

```powershell
Remove-Item -LiteralPath '<OUTPUT_FILE>' -Force
Test-Path -LiteralPath '<OUTPUT_FILE>'
```

`False`가 반환되어야 log 정리가 확인된다. `Snaffler.exe`를 이번 작업에서 반입했다면 업로드 전 부재를 확인한 exact path만 별도로 제거한다. 원격 SMB·AD 감사 로그와 이미 전달된 credential은 로컬 log 삭제로 되돌려지지 않는다.

## 관련 공격기법

- [[SMB 공유 자격증명 수집]]
- [[AD 도메인 컨텍스트 기본 확인]]

## 관련 도구

- [[Snaffler]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 참고 링크

- [Snaffler 공식 저장소 — output·scope option](https://github.com/SnaffCon/Snaffler)
