---
tags:
  - 환경/ad
  - 환경/windows
  - 서비스/smb
시작조건: ["피해자 사용자 또는 머신 계정의 NTLM 인증을 수신할 수 있음", "relay 대상 서비스 식별"]
필요권한: ["인증 수신 도구 실행 권한", "relay된 사용자 또는 머신 계정이 대상 서비스에 가진 권한"]
필요조건: ["수신 호스트에서 relay 대상 서비스까지 접근 가능", "대상 서비스의 SMB signing, EPA 또는 CBT 방어 미흡"]
결과: ["relay 대상 서비스의 방어·권한 조건", "실시간 relay 가능성"]
---

# NTLM Relay 조건 검토

## 한 줄 판단

피해자 NTLM 인증을 받을 수 있는 호스트에서 SMB·LDAP·HTTP 대상에 접근할 수 있으면 SMB signing, EPA/CBT와 relay된 사용자 또는 머신 계정의 권한을 확인해 실시간 relay의 기술 조건을 판단한다.

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
3. 이 문서는 relay 후 dump·LDAP 변경·명령 실행을 수행하지 않고, 해당 작업은 복구 절차를 갖춘 별도 공격기법으로 분리한다.

### SMB signing 확인

```bash
nmap --script smb2-security-mode -p445 <TARGET>
netexec smb <TARGET>
```

확인할 출력:

- `Message signing enabled but not required`는 해당 SMB 서비스가 relay 대상 후보임을 뜻하며, 피해자 인증 수신이나 relay된 계정의 작업 권한까지 증명하지는 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Message signing enabled but not required` | 해당 SMB 대상이 relay 후보 | relay 가능성 확인 | relay 수신 주체와 대상 권한 확인 |
| SMB signing required 또는 HTTP·LDAP의 EPA/CBT 적용 확인 | 확인한 서비스에서 relay 방어 조건 충족 | 해당 대상 제외 | 다른 프로토콜·대상의 보호 조건을 같은 방식으로 확인 |
| signing·EPA·CBT 조건은 후보지만 action별 복구 절차가 없음 | 후속 작업의 영향과 복구 경계 미확정 | 실제 relay 실행 보류 | 복구를 갖춘 연결 기법이 있는 AD CS ESC8만 선택하고 일반 SMB·LDAP action은 `미확인`으로 유지 |

## 확인할 출력과 권한

- NTLM 인증 수신과 relay 가능성은 별도 상태다.
- relay는 피해자 계정의 기존 권한을 전달할 뿐 새 관리자 권한을 만들지 않는다. 실제 대상 작업은 해당 작업의 조건·복구 절차를 가진 별도 문서에서만 판정한다.
- 수신한 NetNTLM challenge-response는 Windows 계정의 NT hash가 아니며, 그대로 [[Pass the Hash]]에 사용하는 값도 아니다.

## 후속 공격 연결

- 인증 수집 조건 확인: [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]
- AD CS HTTP enrollment 조건과 복구가 충족될 때: [[AD CS ESC8 NTLM Relay]]
- [[MSSQL 서비스 Hash 캡처]]
- [[오프라인 해시 크래킹]]

일반 SMB dump·명령 실행이나 LDAP 객체 변경은 이 Vault에 각 action과 복구를 함께 다루는 별도 공격기법이 없으므로 실제 relay 실행 경로로 연결하지 않고 `미확인`으로 남긴다.

## 관련 상태 라우터

- Windows 명령 실행이나 세션을 확보했으면: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]

## 관련 서비스

- [[SMB 서비스]]
- [[MSSQL 서비스]]

## 관련 도구

- [[impacket-ntlmrelayx]]
- [[Responder]]
- [[netexec]]
- [[nmap]]
