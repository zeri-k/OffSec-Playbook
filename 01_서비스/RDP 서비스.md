---
tags:
  - 환경/windows
  - 서비스/rdp
대표포트:
  - "T:3389"
  - "U:3389"
서비스:
  - RDP
---

# RDP 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Remote Desktop Protocol(RDP) TCP 3389에 연결할 수 있고, 아직 유효한 Windows 계정·RDP 로그온 권한·그래픽 세션은 확인하지 않은 상태에서 시작한다. UDP 3389 보조 전송, Network Level Authentication(NLA), Transport Layer Security(TLS) 인증서를 확인한 뒤 비밀번호·NT hash의 유효성, RDP 로그온 권한, Graphical User Interface(GUI) 세션과 드라이브 공유를 구분한다.

성공하면 대상 계정 권한의 원격 GUI 세션을 얻는다. TLS·handshake 실패는 프로토콜·보안 계층을, 인증 실패는 사용자 형식·비밀번호·NLA를, 인증 후 로그온 거부는 `Allow log on through Remote Desktop Services`와 Remote Desktop Users·로컬 정책을 확인한다. 포트 오픈이나 세션 성공만으로 SYSTEM 권한을 단정하지 않는다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | TCP·UDP RDP, NLA, 암호화 수준 | `nmap -p 3389 --script rdp-enum-encryption <TARGET>` | 안전한 식별 결과로 NLA 필요 여부와 지원 보안 계층을 확인한다. UDP 3389은 별도 UDP 스캔으로 확인한다. |
| 2 | RDP handshake | `nmap -sV -sC -p3389 --packet-trace --disable-arp-ping -n <TARGET>` | 응답 경로와 handshake 실패 지점을 확인한다. |
| 3 | 보안 레벨 | `rdp-sec-check.pl <TARGET>` | NLA와 RDP security 설정을 확인한다. |
| 4 | credential과 RDP 권한 | `xfreerdp /u:<USER> /v:<TARGET> /cert:tofu /dynamic-resolution` | 비밀번호는 prompt에 입력하고, 인증 성공과 실제 GUI 세션 생성을 확인한다. |
| 5 | 드라이브 공유 | `xfreerdp /v:<TARGET> /u:<USER> /drive:<SHARE>,<LOCAL_PATH> /cert:tofu` | 로그인 세션에서 로컬 드라이브 매핑이 되는지 확인한다. |
| 6 | 구형 Windows의 CVE-2019-0708 적용 후보 | 승인된 자산·패치 inventory에서 정확한 Windows 제품·버전·적용 업데이트 확인 | 3389 오픈·RDP handshake만으로 BlueKeep 취약을 확정하지 않는다. 충돌 위험이 있는 exploit은 기본 식별에서 제외한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| NLA·인증 방식 단서 또는 사용자 이름·비밀번호 | [[RDP 로그인과 GUI 세션]] | `xfreerdp` | 해당 계정의 RDP 로그온 권한과 원격 GUI 세션 |
| 사용자 이름·비밀번호 후보 | [[원격 비밀번호 공격]] | `xfreerdp`, `hydra` | 잠금 정책을 반영해 검증할 Windows 계정 |
| NTLM hash, Restricted Admin Mode와 대상 로컬 Administrators 권한 | [[Pass the Hash]] | `xfreerdp` | 비밀번호 없이 생성된 해당 계정의 GUI 세션 |
| 로그인 성공과 드라이브 공유 허용 | [[RDP 로그인과 GUI 세션]] | `xfreerdp` | 사용자 권한 범위의 GUI와 로컬·원격 파일 교환 |
| 같은 호스트에 다른 사용자의 RDP 세션이 있고 SYSTEM 명령 실행 가능 | [[RDP 세션 하이재킹]] | `tscon` | 대상 사용자의 기존 GUI 세션과 그 세션의 현재 권한 |

## 서비스 고유 주의 사항

- NLA가 켜져 있으면 인증 전 정보가 제한되고 유효 credential에도 별도 RDP 로그온 권한이 필요하다.
- 인증서 호스트명은 DNS·hosts 후보이며 계정 권한을 뜻하지 않는다.
- 기본 확인에는 `safe`, `discovery` 범주의 `rdp-enum-encryption`만 사용한다. `rdp-vuln-ms12-020`처럼 intrusive인 스크립트는 기본 식별에서 분리해 해당 취약점 검증 단계에서만 사용한다.
- hash 기반 RDP는 Restricted Admin Mode와 대상 로컬 Administrators 권한이 필요하다. 일반 `CanRDP` 후보만으로 충족되지 않는다.
- CVE-2019-0708은 미인증 RCE 취약점이지만 적용 여부는 Microsoft가 지정한 제품·패치 상태로 판정한다. Windows 8·10은 해당 취약점의 영향을 받지 않는다.

### CVE-2019-0708 판정 경계

CVE-2019-0708(BlueKeep)은 사용자 인증 전 RDP 연결 처리에서 발생하는 RCE다. 취약한 구형 Remote Desktop Services가 조작된 요청을 처리할 때 메모리 손상이 커널·LocalSystem 콘텍스트의 코드 실행으로 이어질 수 있다. 따라서 위험도는 포트 노출이 아니라 `정확한 제품·버전 + 보안 업데이트 미적용 + RDP 도달성`으로 판정한다. exploit으로 발생하는 세션과 SYSTEM 권한은 패치 적용 여부를 안전하게 확인하는 방법이 아니며, 시스템 충돌 위험 때문에 별도 승인 없이 실행하지 않는다.

## 참고 링크

- [Nmap: rdp-enum-encryption](https://nmap.org/nsedoc/scripts/rdp-enum-encryption.html)
- [Microsoft: Customer guidance for CVE-2019-0708](https://support.microsoft.com/en-us/topic/customer-guidance-for-cve-2019-0708-remote-desktop-services-remote-code-execution-vulnerability-may-14-2019-0624e35b-5f5d-6da7-632c-27066a79262e)
- [Microsoft Security Response Center: Prevent a worm by updating Remote Desktop Services](https://www.microsoft.com/msrc/blog/2019/05/prevent-a-worm-by-updating-remote-desktop-services-cve-2019-0708)
- [Palo Alto Networks Unit 42: Exploitation of Windows CVE-2019-0708](https://unit42.paloaltonetworks.com/exploitation-of-windows-cve-2019-0708-bluekeep-three-ways-to-write-data-into-the-kernel-with-rdp-pdu/)
- [Microsoft: Remote Credential Guard and Restricted Admin mode](https://learn.microsoft.com/en-us/windows/security/identity-protection/remote-credential-guard)
