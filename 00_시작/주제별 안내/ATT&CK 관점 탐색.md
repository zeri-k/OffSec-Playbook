---
tags:
  - 기능/탐색
---

# ATT&CK 복습과 범위 점검

이 문서는 Playbook의 기법을 복습하고 빠진 작업 영역을 점검하는 색인입니다. 아래 제목은 공식 ATT&CK 전술 매핑이 아니라 Vault 안에서 찾기 쉽게 묶은 로컬 탐색 항목이며, 실행 순서나 문서의 물리적 분류를 바꾸지 않습니다.

## 사용 방법

- 서비스·현재 상태에서 시작할 때는 `01_서비스` 또는 상태 라우터를 사용합니다.
- 복습·범위 점검이 필요할 때만 아래 링크를 사용합니다.
- 공식 전술·기법 ID를 확인하거나 실제 ATT&CK 분류를 수정할 때는 문서 끝의 MITRE ATT&CK Enterprise 원문을 기준으로 합니다.
- 실행 전에는 홈의 서비스·상태 라우터로 돌아가 연결된 기법의 시작조건과 결과 상태를 확인합니다.

## 정찰·탐색

- [[AD ACL 권한 열거와 공격 경로 식별]]
- [[AD DNS 레코드 열거]]
- [[AD GPO 쓰기 권한과 영향 범위 열거]]
- [[AD 계정 위험 속성과 Description 열거]]
- [[AD 계정의 디렉터리 복제 권한 확인]]
- [[AD 고권한 그룹과 중첩 구성원 열거]]
- [[AD 관계 그래프 수집과 공격 경로 식별]]
- [[AD 도메인 컨텍스트 기본 확인]]
- [[AD 도메인 트러스트 열거와 공격 경로 식별]]
- [[AD 비밀번호 정책 열거 및 조회]]
- [[AD 사용자 객체 열거]]
- [[AD 서비스 스캔으로 DC와 도메인 식별]]
- [[AD 외부 도메인 그룹 구성원 열거]]
- [[AD 원격 접근 권한 열거]]
- [[AD 컴퓨터 객체 열거]]
- [[SPN 계정 열거]]
- [[인증 후 AD 사용자와 컴퓨터 객체 열거]]
- [[Linux 권한 상승 열거]]
- [[Microsoft 365 사용자 열거]]
- [[웹 단서 기반 기능 열거]]
- [[웹 숨은 경로와 민감 파일 열거]]
- [[웹 정찰과 경로 열거]]
- [[웹 지문 확인과 공격면 분류]]
- [[Print Spooler 원격 인터페이스 노출 확인]]
- [[Windows 권한 상승 열거]]
- [[Windows 방어 제어 상태 확인]]
- [[원격 Windows 로그온 사용자와 로컬 관리자 단서 열거]]
- [[제한된 Windows 셸에서 AD와 호스트 열거]]
- [[DB 인증과 데이터 열거]]
- [[DNS 열거와 Zone Transfer]]
- [[ICMP 기반 내부 호스트 확인]]
- [[Oracle TNS SID 열거]]
- [[POP3 USER 사용자 열거]]
- [[SMB 익명 열거와 공유 권한 확인]]
- [[SMTP 사용자 열거]]
- [[SNMP OID 정보 열거]]
- [[내부망 수동 호스트 식별]]
- [[피벗팅 경로 식별과 내부망 열거]]

## 초기 접근·웹

- [[CoreFTP HTTP PUT Path Traversal]]
- [[Metasploit Web Delivery로 Meterpreter 세션 획득]]
- [[Public Exploit 검토와 검증]]
- [[WebDAV PUT 업로드와 실행]]
- [[웹 인가 경계 검증]]
- [[웹 파일 업로드와 Web Shell]]
- [[Meterpreter upload로 Windows 파일 반입]]

## 자격 증명 접근

- [[AD 사용자 비밀번호 강제 재설정]]
- [[AS-REP Roasting]]
- [[DCSync]]
- [[Kerberoasting]]
- [[LAPS 비밀번호 읽기 권한과 자격 증명 수집]]
- [[NTDS.dit 덤프]]
- [[OverPass the Hash]]
- [[SYSVOL GPP 자격 증명 수집]]
- [[Shadow Credentials]]
- [[표적 Kerberoasting]]
- [[Linux Shell History 자격증명 검색]]
- [[Linux 파일 자격증명 검색]]
- [[Linux 프로세스 메모리 자격증명 수집]]
- [[LSASS 메모리 덤프]]
- [[Pass the Hash]]
- [[Windows Cached Domain Credentials 추출]]
- [[Windows LSA Secrets 추출]]
- [[Windows SAM SECURITY SYSTEM 덤프]]
- [[Windows SAM 로컬 계정 해시 추출]]
- [[Windows 저장 자격증명 수집]]
- [[Windows 파일 자격증명 검색]]
- [[로컬 사용자 비밀번호 재설정]]
- [[확보한 AD 비밀번호로 runas netonly 네트워크 인증 컨텍스트 생성]]
- [[확보한 평문 비밀번호로 runas 사용자 프로세스 실행]]
- [[원격 비밀번호 공격]]
- [[IPMI hash 수집과 크래킹]]
- [[MSSQL 서비스 Hash 캡처]]
- [[SMB 공유 자격증명 수집]]
- [[SSH credential 및 키 인증 검증]]
- [[네트워크 트래픽 자격증명 수집]]

## 원격 접근·측면 이동

- [[Pass the Ticket]]
- [[Snaffler로 도메인 SMB 공유 민감 파일 탐색]]
- [[SSH Authorized Keys 등록]]
- [[PrintNightmare 원격 코드 실행]]
- [[RDP 로그인과 GUI 세션]]
- [[RDP 세션 하이재킹]]
- [[SMB 공유로 Windows 파일 반입]]
- [[WMI 원격 명령 실행]]
- [[WinRM 두 번째 홉 명시적 자격 증명 재인증]]
- [[WinRM 원격 PowerShell 세션]]
- [[MS17-010 EternalBlue SMB RCE]]
- [[MSSQL Linked Server 내부 이동]]
- [[R-Services trust 기반 원격 접근]]
- [[SMB 쓰기 가능한 공유 검증]]
- [[SMB 인증 공유 파일 수집]]

## 권한 상승

- [[NoPac sAMAccountName 스푸핑 권한 상승]]
- [[Cron 권한 상승]]
- [[SUID GTFOBins 권한 상승]]
- [[sudo 권한 오남용]]
- [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]]
- [[SeBackupPrivilege로 보호된 파일과 hive 복사]]
- [[SeTakeOwnershipPrivilege로 보호 파일 ACL 변경]]
- [[MSSQL Impersonation 권한 상승]]

## 피벗·명령 및 제어

- [[Meterpreter 라우팅과 포트 포워딩]]
- [[SocksOverRDP RDP 터널링]]
- [[Windows Netsh Portproxy 포트 포워딩]]
- [[Chisel SOCKS 터널링]]
- [[Rpivot Reverse SOCKS 피벗팅]]
- [[SSH 포트 포워딩 피벗팅]]
- [[TUN 라우팅 피벗 구성]]
- [[dnscat2 DNS 터널링]]
- [[ptunnel-ng ICMP 터널링]]
- [[sshuttle SSH 피벗팅]]

## 파일·수집

- [[Linux HTTP 파일 회수]]
- [[Certutil로 Windows HTTP 파일 반입]]
- [[Windows HTTP 파일 회수]]
- [[보호된 파일 및 아카이브 크래킹]]
- [[상황별 파일 전송]]
- [[제한 환경 파일 반입]]
- [[DB 서버 파일 수집]]
- [[FTP 익명 접근과 파일 수집]]
- [[TFTP 설정 파일 수집]]

## 실행·공통

- [[AD CS ESC8 NTLM Relay]]
- [[AD 그룹 구성원 추가로 권한 확대]]
- [[AD 보안 구성과 GPO 감사]]
- [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]
- [[NTLM Relay 조건 검토]]
- [[내부 AD Password Spraying]]
- [[인증 전 AD 사용자 목록 수집]]
- [[임시 SPN 설정]]
- [[자식 도메인 ExtraSids Golden Ticket]]
- [[Kernel Exploit 후보 검증]]
- [[Linux Kerberos keytab ccache 악용]]
- [[Linux 개인키 검색]]
- [[Microsoft 365 Password Spraying]]
- [[Meterpreter 프로세스 이동과 세션 안정화]]
- [[Meterpreter 프로세스 토큰 탈취]]
- [[Windows Defender 실시간 보호 비활성화]]
- [[로컬 관리자 그룹 구성원 추가]]
- [[저장된 자격 증명으로 runas 프로세스 실행]]
- [[Bind Shell 획득]]
- [[MSFVenom Payload 생성과 Handler 수신]]
- [[Reverse Shell 획득]]
- [[Socat 셸 리디렉션]]
- [[TTY 업그레이드]]
- [[오프라인 해시 크래킹]]
- [[Ettercap L2 MITM DNS Spoofing]]
- [[FTP Bounce]]
- [[IMAP POP3 메일함 수집]]
- [[MSSQL xp_cmdshell 명령 실행]]
- [[NFS export 마운트와 권한 매핑 검증]]
- [[Rsync 공개 모듈 수집]]

## 참고 링크

- [MITRE ATT&CK Enterprise](https://attack.mitre.org/matrices/enterprise/)
