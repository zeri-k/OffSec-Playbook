---
tags:
  - 환경/windows
  - 서비스/winrm
대표포트:
  - "T:5985"
  - "T:5986"
서비스:
  - WinRM
  - PowerShell Remoting
---

# WinRM 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Windows Remote Management(WinRM) HTTP 5985 또는 HTTPS 5986 listener에 연결할 수 있고, 아직 유효한 Windows 계정·원격 관리 권한·PowerShell 세션은 확인하지 않은 상태에서 시작한다. Web Services for Management(WS-Management, WSMan) 응답, 비밀번호·NT hash·Kerberos ticket 인증, Remote Management Users·로컬 관리자 권한과 실제 명령 실행을 분리한다.

**첫 화면 상태:** 지금 가능한 일은 listener 도달성, 보유한 인증 자료의 유효성, 세션 생성을 차례로 구분하는 것이다. 성공하면 인증된 사용자 컨텍스트의 원격 PowerShell 세션을 얻는다. listener 실패는 포트·방화벽·HTTP/HTTPS를, 인증 실패는 로컬/도메인 사용자 형식·NTLM/Kerberos·TLS를, 인증 후 권한 오류는 Remote Management Users·관리자 멤버십과 endpoint ACL을 확인한다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | WSMan 응답 | Windows에서 HTTP는 `Test-WSMan -ComputerName <TARGET> -Port 5985`, HTTPS는 `Test-WSMan -ComputerName <TARGET_FQDN> -UseSSL -Port 5986` | WinRM listener와 WSMan identification 응답을 확인한다. 비표준 listener를 발견했다면 실제 포트로 바꾼다. |
| 2 | 비밀번호 세션 | `evil-winrm -i <TARGET> -u <USER>` 후 password prompt | 인증 성공과 PowerShell 세션 생성을 확인한다. |
| 3 | hash 세션 | `evil-winrm -i <TARGET> -u <USER> -H <NTLM_HASH>` | NTLM 허용과 hash 기반 세션 권한을 확인한다. |
| 4 | 여러 호스트의 WinRM 권한 | `netexec winrm <TARGETS> -u <USER> -p <PASSWORD>` | credential이 통하는 호스트와 원격 관리 가능 범위를 구분한다. |
| 5 | HTTPS listener | 5986 인증서와 TLS 협상 | 인증서·호스트명 문제와 계정 인증·원격 관리 권한 오류를 분리한다. |

**출력 해석 경계:** `Test-WSMan`은 Windows에서 WS-Management identification 요청을 보내 WinRM 서비스 응답을 확인하는 명령이다. 응답은 유효 credential이나 원격 셸을 확정하지 않는다. `netexec winrm`의 인증 표시는 해당 인증 방식의 성공 후보이고, `evil-winrm`에서 `whoami` 같은 실제 출력이 있어야 세션과 실행 계정을 확정한다. 5986 TLS 오류는 인증 실패나 계정 권한 부족을 뜻하지 않는다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| 사용자 이름·비밀번호 후보 | [[원격 비밀번호 공격]] | `netexec`, `evil-winrm` | WinRM 인증과 원격 관리 권한의 분리된 결과 |
| 인증 성공과 PowerShell 세션 | [[WinRM 원격 PowerShell 세션]] | `evil-winrm`, `powershell` | 해당 사용자 컨텍스트의 원격 명령 실행 |
| NTLM hash | [[Pass the Hash]] | `evil-winrm`, `netexec` | 비밀번호 없는 WinRM 세션 가능성 |
| Kerberos ticket 또는 ccache | [[Pass the Ticket]] | `evil-winrm`, `netexec` | ticket 주체의 WinRM 세션 가능성 |
| 관리자 권한과 hive 접근 | [[Windows SAM SECURITY SYSTEM 덤프]] | `reg save`, `impacket-secretsdump` | SAM·SECURITY·SYSTEM hive 읽기의 별도 검증 |
| DC에 도달하며 복제 권한이 있는 AD 계정의 비밀번호·NT hash·Kerberos ticket | [[DCSync]] | `impacket-secretsdump`, `mimikatz` | 검증된 디렉터리 복제 가능성 |

## 서비스 고유 주의 사항

- 사용자가 유효해도 Remote Management Users 또는 관리자 등 원격 로그온 권한이 없으면 세션이 실패한다.
- Remote Management Users의 명령 권한과 로컬 관리자 권한을 구분한다.
- 5986은 인증서·TLS·호스트명 문제를 계정 인증 실패와 구분한다.
- 방화벽, 인증 방식, DNS와 시간 동기화가 WinRM·Kerberos 결과에 영향을 줄 수 있다.
- WinRM 2.0의 기본 listener는 HTTP 5985·HTTPS 5986이지만 임의 포트로 구성할 수 있다. 포트 번호만으로 WinRM을 확정하지 않고 WSMan 응답을 확인한다.

## 참고 링크

- [Microsoft: Test-WSMan](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman)
- [Microsoft: Installation and configuration for Windows Remote Management](https://learn.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management)
- [Microsoft: Service overview and network port requirements](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements)
