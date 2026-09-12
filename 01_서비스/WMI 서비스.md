---
tags:
  - 환경/windows
  - 서비스/wmi
대표포트:
  - "T:135"
서비스:
  - WMI
  - DCOM
  - MSRPC
---

# WMI 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 TCP 135와 협상된 동적 Remote Procedure Call(RPC) 포트에 도달하고, 대상 호스트에서 로컬 관리자 권한을 갖는 계정의 비밀번호·NT hash 또는 Kerberos ticket을 보유한 경우에 연다. Windows Management Instrumentation(WMI)은 Distributed Component Object Model(DCOM)을 통해 TCP 135에서 시작해 동적 RPC 포트로 이동하는 관리 인터페이스다.

135/MSRPC 식별과 익명·인증 RPC 열거는 [[RPC와 NetBIOS 서비스]]가 담당한다. 성공하면 보유한 계정 권한으로 원격 명령을 실행한 호스트와 실행 사용자를 확인하고, RPC timeout이면 동적 포트·방화벽을, `access denied`이면 대상 로컬 관리자 멤버십·UAC 원격 제한·DCOM/WMI 권한을 다시 확인한다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | WMI 실행 전제 | 135와 동적 RPC 접근, 대상 호스트의 로컬 관리자 권한이 있는 비밀번호·NT hash·ticket, 필요 시 445 관리 공유 조건 | 방화벽·네트워크 오류와 인증·권한 오류를 구분한다. |
| 2 | WMI 명령 실행 검증 | [[WMI 원격 명령 실행]]의 최소 명령 | 실제 출력과 실행 사용자를 확인한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| NTLM hash | [[Pass the Hash]] | `impacket-wmiexec`, `netexec` | 비밀번호 없이 WMI 인증·명령 실행 가능성 |
| Kerberos ticket | [[Pass the Ticket]] | `impacket-wmiexec`, `netexec` | ticket 주체 권한의 WMI 인증 가능성 |
| 대상 호스트의 로컬 관리자 인증 자료와 WMI/DCOM/RPC 전제 충족 | [[WMI 원격 명령 실행]] | `impacket-wmiexec` | 원격 명령 출력과 실제 실행 사용자 |
| 관리자 권한과 hive 접근 | [[Windows SAM SECURITY SYSTEM 덤프]] | `reg save`, `impacket-secretsdump` | hive 읽기 가능성은 해당 기법에서 별도 검증 |

## 서비스 고유 주의 사항

- TCP 135만 열려 있어도 동적 RPC 포트가 막히면 WMI가 실패할 수 있다.
- 일반 사용자 인증, SMB 관리자 표시, WMI 원격 실행은 서로 다른 상태다.
- 445 접근과 관리 공유 조건을 확인하고, 방화벽 차단 시 WinRM/SMB 경로를 별도로 판단한다.
