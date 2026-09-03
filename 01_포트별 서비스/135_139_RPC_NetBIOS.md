---
tags:
  - 환경/windows
  - 서비스/rpc
대표포트:
  - "T:135"
  - "U:137"
  - "U:138"
  - "T:139"
서비스:
  - MSRPC
  - NetBIOS
---

# 135_139_RPC_NetBIOS

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Microsoft Remote Procedure Call(MSRPC) Endpoint Mapper 또는 NetBIOS 이름·세션 서비스에 도달할 수 있고, 아직 Windows·Samba 계정이나 원격 관리 권한은 확인하지 않은 상태에서 시작한다. 이 포트들은 호스트 이름·워크그룹, RPC/Server Message Block(SMB) 익명 조회와 Windows Management Instrumentation(WMI) 전제 조건을 가르는 보조 진입점이다.

성공하면 익명·Guest·인증 계정별로 조회 가능한 사용자·그룹·공유·정책을 얻는다. Null session 실패 시 Guest와 알려진 계정을 분리해 확인하고, WMI 연결 실패 시 TCP 135 이후 협상된 동적 RPC 포트 차단과 대상 계정의 로컬 관리자·DCOM 권한을 각각 확인한다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | NetBIOS 이름, OS, 워크그룹 | `nmap -sS -sU --script nbstat,smb-os-discovery -p T:135,139,445,U:137,138 <TARGET>` | 호스트 역할과 SMB/Samba 후보를 확인한다. |
| 2 | Null RPC | `rpcclient -U "" -N <TARGET>` 후 `srvinfo`, `enumdomusers`, `netshareenumall` | 인증 없이 사용자·그룹·공유·정책을 조회할 수 있는지 구분한다. |
| 3 | RPC/SMB 자동 열거 | `enum4linux-ng -A <TARGET>` | 익명 또는 Guest로 노출되는 사용자·그룹·공유·정책을 교차 확인한다. |
| 4 | 인증된 RPC | `rpcclient -U '<USER>%<PASSWORD>' <TARGET>` | 익명 결과보다 조회 범위가 확장되는지 확인한다. |
| 5 | WMI 전제 | 135와 동적 RPC 포트, 445 접근성 | 유효 계정과 관리자·DCOM/WMI 조건을 확인할 경로가 있는지 판단한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| NetBIOS 이름·워크그룹 또는 Null session | [[SMB 익명 열거와 공유 권한 확인]] | `nbtscan`, `rpcclient`, `enum4linux-ng` | 호스트명·사용자·그룹·공유·정책의 익명 조회 범위 |
| 사용자 또는 RID 응답 | [[SMB 익명 열거와 공유 권한 확인]] | `rpcclient`, `enum4linux-ng` | 인증 검증에 사용할 사용자명 후보 |
| 139와 445 동시 오픈 | [[SMB 익명 열거와 공유 권한 확인]] | `smbclient`, `smbmap` | 공유별 익명·인증 READ/WRITE 후보 |
| WMI/RPC 단서와 대상 호스트의 로컬 관리자 권한이 있는 비밀번호·NT hash·Kerberos ticket | [[WMI 원격 명령 실행]] | `impacket-wmiexec`, `netexec` | [[135_WMI]]에서 검증할 원격 명령 실행 후보 |

## 서비스 고유 주의 사항

- 135/139가 열려 있어도 445가 막혀 있으면 SMB 기반 도구 결과가 제한될 수 있다.
- Null session 실패는 정상 설정일 수 있으며 Guest와 인증 계정 결과가 다를 수 있다.
- 로컬 계정과 도메인 계정의 인증 형식을 구분한다.
- WMI는 일반적으로 관리자 권한과 동적 RPC 포트 접근이 필요하다.
