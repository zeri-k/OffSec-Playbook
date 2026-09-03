---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/인증검증
  - 기능/열거
실행환경: ["Linux", "Windows"]
필요조건: ["도메인명", "사용자 목록"]
결과: ["정보", "자격증명"]
---

# kerbrute

## 도구 개요

Kerbrute는 Kerberos KDC 응답 차이를 이용해 AD 사용자 이름을 열거하고 password spraying 또는 사용자·비밀번호 조합 검증을 수행한다. 인증 전 사용자 후보를 정리하거나 Kerberos 기반 인증을 점검할 때 유용하며, `valid user`는 비밀번호 인증 성공을 뜻하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: DC의 Kerberos에 접근 가능한 Linux 또는 Windows 호스트
- 필요한 입력: DC 주소, 도메인, 사용자 목록과 명령에 따라 비밀번호 또는 password list
- 환경 조건: DNS/realm과 시간을 맞추고 spraying 전 계정 잠금 정책과 시도 간격을 확인한다.


## 표준 사용법

```bash
kerbrute <command> --dc <dc_ip> --domain <domain> <wordlist>
```

## 대표 예시

### Kerberos로 유효 사용자명 열거

```bash
./kerbrute_linux_amd64 userenum --dc <TARGET> --domain <DOMAIN> names.txt
```

### 단일 비밀번호로 password spraying

```bash
./kerbrute_linux_amd64 passwordspray --dc <TARGET> --domain <DOMAIN> users.txt '<PASSWORD>'
```

### Windows에서 유효 사용자명 열거

```powershell
.\kerbrute_windows_amd64.exe userenum --dc <TARGET> --domain <DOMAIN> .\names.txt
```

### Windows에서 단일 비밀번호로 Password Spraying

```powershell
.\kerbrute_windows_amd64.exe passwordspray -d <DOMAIN> .\adusers.txt '<PASSWORD>'
```

DNS로 DC를 찾을 수 없거나 특정 DC에 요청해야 할 때는 다음처럼 `--dc`를 추가한다. 잠금 징후를 감지했을 때 실행을 중단하려면 `--safe`를 함께 사용한다.

```powershell
.\kerbrute_windows_amd64.exe passwordspray --safe -d <DOMAIN> --dc <DC> .\adusers.txt '<PASSWORD>'
```

Windows 바이너리도 Linux 바이너리와 같은 명령과 옵션을 사용한다. 현재 Windows 도메인 사용자 세션은 필요하지 않지만 실행 호스트에서 DC의 Kerberos 88/TCP·UDP에 도달하고 도메인 realm과 시간이 맞아야 한다. `--dc`를 생략하면 도메인 DNS를 통해 DC를 찾을 수 있어야 한다.

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| valid user | Kerberos 반응으로 유효 사용자 확인 | password spraying 대상 목록에 반영 |
| valid credential | 사용자/비밀번호 조합 확인 | WinRM/SMB/LDAP 등 실제 서비스 로그인 검증 |
| lockout/disabled 메시지 | 계정 상태 또는 잠금 위험 | spraying 즉시 중단 또는 빈도 조정 |
| KDC/realm 오류 | 도메인명, DC, DNS 문제 | realm, DC IP, 시간 동기화, `/etc/hosts` 확인 |
| 전부 invalid | 사용자 목록 형식 오류 | `user` vs `user@domain`, 대소문자, 도메인 접미사 확인 |

## 주요 옵션과 명령

| 항목 | 설명 |
| --- | --- |
| `userenum` | 사용자 이름 유효성 확인 |
| `passwordspray` | 사용자 목록에 단일 비밀번호 시도 |
| `bruteforce` | 사용자:비밀번호 조합 시도 |
| `--dc` | 도메인 컨트롤러 지정 |
| `--domain` | Kerberos realm/domain 지정 |
| `--threads` | 병렬 작업 수 |
| `--safe` | lockout 감지 시 실행 중단 |

## 관련 공격기법

- [[인증 전 AD 사용자 목록 수집]]
- [[내부 AD Password Spraying]]
- [[원격 비밀번호 공격]]
- [[AS-REP Roasting]]
