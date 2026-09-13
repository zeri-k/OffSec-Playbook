---
tags:
  - 환경/ad
  - 환경/windows
  - 서비스/smb
시작조건: ["피해자 사용자 또는 머신 계정의 NTLM 인증을 수신할 수 있음", "relay 대상 서비스 식별"]
필요권한: ["인증 수신 도구 실행 권한", "relay된 사용자 또는 머신 계정이 대상 서비스에 가진 권한"]
필요조건: ["수신 호스트에서 relay 대상 서비스까지 접근 가능", "대상 서비스의 SMB signing, EPA 또는 CBT 방어 미흡"]
결과: ["대상 서비스 인증 세션", "relay된 계정 권한으로 수행한 명령·LDAP 작업·정보 접근", "AD CS certificate"]
---

# NTLM Relay 조건 검토

## 한 줄 판단

피해자 NTLM 인증을 받을 수 있는 호스트에서 SMB·LDAP·HTTP 대상에 접근할 수 있으면 SMB signing, EPA/CBT와 relay된 사용자 또는 머신 계정의 권한을 확인해 서비스 세션, LDAP 작업, 명령 실행 또는 AD CS certificate를 얻을 수 있는지 판단한다.

## 사용할 때

- 현재 보유 정보: SMB signing이 required가 아닌 호스트나 HTTP·LDAP·AD CS relay 후보와 피해자 사용자 또는 머신 계정의 NTLM 인증을 수신할 경로가 식별된 상태다.
- 명령 실행 위치와 도달성: Responder, coercion 또는 MSSQL hash capture를 실행하는 수신 호스트에서 피해자 인증을 받고 relay 대상의 SMB·LDAP·HTTP 서비스에 접근할 수 있다.
- 현재 가능한 행동과 결과: [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]] 등에서 새로 들어오는 NTLM 메시지를 중계하고, relay된 계정의 기존 권한으로 서비스 접근·객체 변경·명령 실행·certificate 발급 중 가능한 영향을 확인한다. 저장한 response와 live relay의 차이는 [[NTLM 인증 자료, 실시간 Relay와 서비스 권한 경계]]를 따른다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 수신 호스트가 피해자 NTLM 인증을 받을 수 있고 relay 대상의 SMB·LDAP·HTTP 서비스에 접근 가능 | Responder/ntlmrelayx 수신 로그와 대상 포트 연결 확인 | 피해자와 수신 호스트 사이의 이름 해석·coercion 경로, 수신 호스트와 대상 사이의 방화벽·라우팅 확인 |
| 현재 인증 수단 | 피해자 사용자 또는 머신 계정의 새 NTLM 교환이 relay listener에 실시간 도달 | ntlmrelayx에서 수신한 인증 주체와 target challenge 전달 확인 | 저장된 response와 구분하고 인증 유도 경로·피해자 protocol의 NTLM 사용 확인 |
| 현재 권한 | 수신 도구를 실행할 수 있고 relay된 계정이 대상 서비스에서 수행할 작업 권한을 보유 | 수신 호스트 권한과 대상 서비스의 share·LDAP ACL·enrollment 권한 확인 | relay 성공과 대상 작업 권한을 분리해 다른 계정 또는 대상 검토 |
| 공격 대상의 조건 | SMB signing이 required가 아니거나 HTTP·LDAP·AD CS에서 EPA/CBT 등 relay 방어가 미흡 | `nmap`, `netexec`와 대상 서비스 설정 확인 | 방어가 적용된 서비스는 제외하고 다른 프로토콜·대상 확인 |
| 필요한 파일·목록·주소 | 대상 URL 또는 호스트 목록과 실행할 relay 작업이 확정 | `targets.txt`, LDAP/HTTP URL과 ntlmrelayx action 검토 | 인증을 받기 전에 대상·프로토콜·기대 결과를 먼저 확정 |

## 실행

1. 수신 호스트에서 피해자 인증 경로와 relay 대상 서비스까지의 도달성을 확인한다.
2. 대상별 SMB signing, EPA/CBT와 relay된 계정에 기대하는 작업 권한을 확인한다.
3. ntlmrelayx에서 인증 relay 성공과 후속 dump·LDAP action·명령 실행을 별도 출력으로 판정한다.
4. 인증만 수신했거나 relay 후 작업이 실패하면 challenge-response 형식, 대상 방어와 relay된 주체의 ACL을 차례로 확인한다.

### SMB signing 확인

```bash
nmap --script smb2-security-mode -p445 <TARGET>
netexec smb <TARGET>
```

확인할 출력:

- `Message signing enabled but not required`는 해당 SMB 서비스가 relay 대상 후보임을 뜻하며, 피해자 인증 수신이나 relay된 계정의 작업 권한까지 증명하지는 않는다.

### ntlmrelayx 기본 흐름

```bash
impacket-ntlmrelayx -tf targets.txt -smb2support
impacket-ntlmrelayx -t ldap://<DC> -smb2support
```

확인할 출력:

- relay된 사용자 또는 머신 계정, 대상 서비스의 인증 성공, 이어진 dump·LDAP action·command execution 결과.
- `Signing is required`, EPA/CBT 관련 거부 또는 LDAP 작업의 `insufficientAccessRights`가 나오면 대상 방어와 relay된 계정의 권한을 분리해 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Message signing enabled but not required` | 해당 SMB 대상이 relay 후보 | relay 가능성 확인 | relay 수신 주체와 대상 권한 확인 |
| ntlmrelayx에 대상 서비스 인증 성공과 relay된 주체가 표시됨 | 피해자 사용자 또는 머신 계정의 NTLM 인증이 해당 서비스에 relay됨 | relay된 계정의 서비스 인증 세션 확보 | share, LDAP, HTTP 등 서비스별 허용 작업 확인 |
| dump, LDAP action 또는 command execution 출력 | relay된 계정 권한으로 작업 성공 | 정보 접근, 객체 변경 또는 명령 실행 상태 | 실제 Identity와 대상 권한 확인 |
| AD CS에서 `.pfx` 발급 | certificate enrollment 성공 | 인증 certificate 확보 | [[AD CS ESC8 NTLM Relay]]에서 TGT 전환 |
| relay 실패 | signing required, EPA 또는 CBT 등 방어 조건 | 인증 relay 미성립 | 다른 프로토콜과 대상의 보호 조건 확인 |
| 인증만 수신되고 relay 대상 없음 | 인증 흐름은 확보했으나 사용할 대상 부재 | NetNTLM challenge-response만 확보 | [[오프라인 해시 크래킹]] 가능성과 대상 inventory 검토 |
| relay 성공이나 영향 없음 | relay된 계정에 대상 권한 없음 | 인증 세션만 확보 | relay된 주체와 대상 ACL 확인 |

## 확인할 출력과 권한

- NTLM 인증 수신, relay 성공, relay 후 작업 성공은 각각 별도 상태다.
- relay는 피해자 계정의 기존 권한을 전달할 뿐 새 관리자 권한을 만들지 않는다.
- SMB 인증 성공, 원격 명령 실행, LDAP 쓰기, certificate enrollment 권한을 서비스별로 구분한다.
- 수신한 NetNTLM challenge-response는 Windows 계정의 NT hash가 아니며, 그대로 [[Pass the Hash]]에 사용하는 값도 아니다.

## 후속 공격 연결

- 인증 수집 조건 확인: [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]
- [[AD CS ESC8 NTLM Relay]]
- [[MSSQL 서비스 Hash 캡처]]
- [[오프라인 해시 크래킹]]

## 관련 상태 라우터

- Windows 명령 실행이나 세션을 확보했으면: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- AD certificate 또는 객체 변경 결과를 확보했으면: [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 관련 서비스

- [[SMB 서비스]]
- [[MSSQL 서비스]]

## 관련 도구

- [[impacket-ntlmrelayx]]
- [[Responder]]
- [[netexec]]
- [[nmap]]
