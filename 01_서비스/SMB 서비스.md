---
tags:
  - 환경/windows
  - 서비스/smb
대표포트:
  - "T:139"
  - "T:445"
서비스:
  - SMB
  - CIFS
---

# SMB 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:445` 또는 NetBIOS session 포트 139의 Server Message Block(SMB)에 연결할 수 있고, 아직 유효한 Windows·Samba 계정이나 공유·호스트 관리 권한은 확인하지 않은 상태에서 시작한다. 운영체제·시간·SMB signing을 확인한 뒤 Null session, Guest 매핑, 계정 인증, 공유별 읽기·쓰기와 대상 호스트의 로컬 관리자·원격 실행 권한을 분리한다.

**첫 화면 상태:** 지금 가능한 일은 익명·Guest·보유 계정의 세션 유형과 공유별 접근을 확인하는 것이다. 성공하면 각 세션이 볼 수 있는 공유와 실제 파일 읽기·쓰기 범위를 얻는다. 로그온 실패는 로컬/도메인 사용자 형식·비밀번호·NT hash를, `ACCESS_DENIED`는 공유 권한과 파일 시스템 ACL을, 관리자 표시 후 실행 실패는 UAC 원격 제한·관리 공유·서비스 생성/WMI 권한을 다시 확인한다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | OS, SMB signing, 시간 | `nmap --script smb-os-discovery,smb2-security-mode,smb2-time -p445 <TARGET>` | 호스트·시간과 signing required 여부를 확인한다. |
| 2 | 익명 공유 | `smbclient -L //<TARGET>/ -N` | Null session으로 보이는 공유 목록을 확인한다. |
| 3 | Null과 Guest 차이 | `netexec smb <TARGET> -u '' -p '' --shares`, `netexec smb <TARGET> -u guest -p '' --shares` | 두 세션의 인증 결과와 공유별 권한을 비교한다. |
| 4 | 인증된 공유 권한 | `netexec smb <TARGET> -u <USER> -p <PASSWORD> --shares` | 로그인 성공, 공유별 READ/WRITE와 관리자 표시를 구분한다. |
| 5 | 공유 내부 권한 | `smbclient`, `smbmap` | 공유 접근 권한과 실제 파일 시스템 READ/WRITE 차이를 확인한다. |

**출력 해석 경계:** `smb2-security-mode`의 signing 결과는 relay 보호 조건 하나를 확정할 뿐 relay 성공을 보장하지 않는다. `netexec`의 로그인·관리자 표시는 인증 또는 권한 후보이고, 실제 원격 명령 실행·hive 읽기는 별도 확인한다. 공유 목록과 `READ`/`WRITE`는 해당 공유의 관찰 시점 권한만 확정하며 파일 시스템 ACL, 업로드 파일의 실행 가능성은 확정하지 않는다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| Null 또는 Guest 접근 | [[SMB 익명 열거와 공유 권한 확인]] | `smbclient`, `rpcclient`, `enum4linux-ng` | 익명·Guest의 공유·사용자·정책 조회 범위 |
| 사용자명·비밀번호 후보 | [[원격 비밀번호 공격]] | `netexec` | 유효 SMB credential과 공유 권한 |
| READ 가능한 공유 | [[SMB 공유 자격증명 수집]] | `smbclient`, `smbmap`, `manspider` | 설정·백업·문서·스크립트에 저장된 사용자 이름·비밀번호·key 후보 |
| 인증 계정과 읽을 공유·파일 경로를 알고 있음 | [[SMB 인증 공유 파일 수집]] | `smbclient` | 공격 호스트로 회수한 지정 파일 |
| DC의 `SYSVOL`, `Groups.xml`, `Registry.xml` 또는 logon script | [[SYSVOL GPP 자격 증명 수집]] | `smbclient`, `crackmapexec`, `gpp-decrypt` | 스크립트·Group Policy Preferences(GPP)·autologon의 계정·비밀번호 후보 |
| SSH private key 발견 | [[SSH credential 및 키 인증 검증]] | `smbclient`, `ssh` | 해당 키 주체의 SSH 인증 가능성 |
| WRITE 가능한 공유 | [[SMB 쓰기 가능한 공유 검증]] | `smbclient`, `smbmap` | 검증된 공유 경로의 파일 쓰기, 실행 여부는 별도 |
| signing not required | [[NTLM Relay 조건 검토]] | `nmap`, `netexec`, `ntlmrelayx` | 네트워크·인증 흐름과 relay 계정 권한을 반영한 대상 후보 |
| Kerberos ticket | [[Pass the Ticket]] | `smbclient`, `netexec` | ticket 주체의 SMB 접근 범위 |
| NTLM hash | [[Pass the Hash]] | `netexec`, `impacket-psexec`, `evil-winrm` | hash 주체의 인증·원격 관리 가능성 |
| 관리자 표시 또는 ADMIN$ 접근 후보 | [[Windows SAM SECURITY SYSTEM 덤프]] | `netexec`, `impacket-psexec`, `impacket-wmiexec`, `impacket-secretsdump` | SAM·LSA hive와 서비스 생성 권한의 별도 검증 |
| 관리자 권한과 WMI 접근 | [[WMI 원격 명령 실행]] | `impacket-wmiexec` | 실제 명령 출력과 실행 사용자 |
| DC에 도달하며 복제 권한이 있는 AD 계정의 비밀번호·NT hash·Kerberos ticket | [[DCSync]] | `impacket-secretsdump`, `mimikatz` | 검증된 디렉터리 복제 가능성 |
| 구형 Windows, SMBv1, MS17-010 scanner 취약 표시 | [[MS17-010 EternalBlue SMB RCE]] | `metasploit` | exploit 안정성까지 확인한 SYSTEM 세션 후보 |
| Windows 10·Server 1903/1909 계열, SMB 3.1.1 compression과 CVE-2020-0796 업데이트 상태 미확인 | [[Public Exploit 검토와 검증]] | 자산 build·업데이트 정보, 공식 CVE | SMBGhost 영향 가능 버전·패치 후보. 버전 단서만으로 취약·RCE를 확정하지 않음 |

## 서비스 고유 주의 사항

- 로컬 계정과 도메인 계정 인증 형식을 구분하고 Null과 Guest를 모두 확인한다.
- SMB signing 미요구는 relay 가능성에 영향을 주지만 단독 취약점은 아니다.
- 공유 권한과 파일 시스템 권한이 다를 수 있으며 WRITE가 파일 실행을 뜻하지 않는다.
- 관리자 표시는 원격 명령·SAM/LSA·서비스 생성 권한의 후보이므로 실제 가능한 작업을 별도 확인한다.

## 참고 링크

- [Microsoft — What is SMB File Sharing](https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview)
- [Microsoft — SMB signing overview](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing-overview)
- [Samba smbclient manual](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)
- [Microsoft Security Response Center — CVE-2020-0796](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2020-0796)
