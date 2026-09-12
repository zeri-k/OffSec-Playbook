---
tags:
  - 환경/windows
  - 서비스/winrm
  - 기능/원격실행
실행환경: ["Linux"]
필요조건: ["WinRM 인증 정보"]
결과: ["세션", "파일", "명령 실행"]
---

# evil-winrm

## 도구 개요

`evil-winrm`은 Linux에서 비밀번호·NTLM hash·Kerberos ticket으로 Windows Remote Management(WinRM)에 인증해 원격 PowerShell 세션과 파일 전송 기능을 제공한다. 셸 중심의 Windows 원격 작업에 적합하지만 세션 획득 자체가 로컬 관리자나 SYSTEM 권한을 뜻하지는 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 WinRM TCP/5985 또는 TCP/5986에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 주소, 사용자와 비밀번호/NTLM hash 또는 Kerberos ticket
- HTTPS/Kerberos 조건: TLS 사용 여부, 도메인 형식, SPN/FQDN, 시간 동기화를 맞춘다.


## 표준 사용법

```bash
evil-winrm -i <TARGET> -u <USER>
```

비밀번호 방식은 `-p`를 생략하고 prompt에 입력해 shell history·process 인자에 평문을 남기지 않는다.

## 대표 예시

### 비밀번호로 WinRM PowerShell 세션 획득

```bash
evil-winrm -i <TARGET> -u <USER>
```

### Pass the Hash로 접속

```bash
evil-winrm -i <TARGET> -u Administrator -H <NTLM_HASH>
```

### Kerberos ticket으로 FQDN endpoint 접속

```bash
evil-winrm --version
evil-winrm -i <TARGET_FQDN> -r <DOMAIN_FQDN> -K <CCACHE_OR_KIRBI_FILE>
```

`-K`는 현재 공식 README에 문서화된 ticket 파일 옵션이다. 설치된 버전의 `evil-winrm -h`에 `-K`가 없으면 해당 버전의 `KRB5CCNAME`·`-r` 절차를 확인하고 신구 문법을 섞지 않는다. Kerberos는 `<TARGET_FQDN>`·realm·DNS·시간과 HTTP SPN이 맞아야 한다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-i` | 대상 IP 또는 호스트명 |
| `-u`, `-p` | 사용자명과 비밀번호 |
| `-H` | NTLM hash로 인증 |
| `-r` | Kerberos realm 지정 |
| `-S` | SSL/HTTPS 사용 |
| `-P` | WinRM 포트 지정 |
| `-K` | 현재 버전에서 ccache 또는 kirbi ticket 파일 지정. 설치 버전 도움말 확인 |
| `-s`, `-e` | local scripts/executables 경로 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| PowerShell 프롬프트 획득 | WinRM 인증과 원격 세션 성공 | `whoami /all`, `hostname`, `ipconfig`로 컨텍스트 확인 |
| 파일 업로드/다운로드 성공 | 후속 도구 전달과 결과 회수 가능 | 실행 가능한 경로와 AV/EDR 반응 확인 |
| `Access is denied` | 권한 부족 또는 WinRM 권한 제한 | 계정 그룹, 로컬 관리자 여부, UAC remote restriction 확인 |
| auth/transport 오류 | credential, SSL, 포트, SPN 문제 | `-S`, 포트, 도메인/로컬 계정 형식 확인 |
| 파일 전송 실패 | 경로 권한 또는 차단 | 쓰기 가능 경로, 파일명, Defender 차단 여부 확인 |
| 로컬 명령은 성공하나 PowerView LDAP에서 `FindAll` 오류 | Kerberos Double Hop 또는 DC 접근 문제 가능 | `klist`, DNS·LDAP 도달성 확인 후 [[WinRM Kerberos Double Hop 진단과 재인증]] |

## 관련 공격기법

- [[WinRM 원격 PowerShell 세션]]
- [[Pass the Hash]]
- [[Pass the Ticket]]
- [[WinRM Kerberos Double Hop 진단과 재인증]]

## 참고 링크

- [Hackplayers: Evil-WinRM](https://github.com/Hackplayers/evil-winrm)
- [Microsoft: Installation and configuration for Windows Remote Management](https://learn.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management)
