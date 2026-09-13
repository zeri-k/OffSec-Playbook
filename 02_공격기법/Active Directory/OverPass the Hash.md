---
tags:
  - 환경/ad
  - 환경/windows
  - 서비스/kerberos
시작조건: ["특정 도메인 계정의 NTLM/RC4 또는 AES key 확보", "Kerberos 명령을 실행할 Windows 호스트 세션 확보"]
필요권한: ["Rubeus 실행과 전용 Type 9 process 생성 권한", "Mimikatz `sekurlsa::pth`는 대상 Windows 호스트 관리자 권한"]
필요조건: ["실행 Windows 호스트에서 <DC_FQDN>:88 Kerberos와 DNS 접근 가능", "도메인·realm·DC FQDN과 시간 정합", "생성한 ticket으로 접근할 <HOST_FQDN>:<SERVICE_PORT> 도달 가능"]
결과: ["key 주체 도메인 계정의 Kerberos TGT", "전용 Windows 로그온 세션의 ticket 사용 상태", "ticket 주체 권한 범위의 대상 서비스 접근"]
---

# OverPass the Hash

## 한 줄 판단

도메인 계정의 NTLM/RC4 또는 AES key를 보유하고 현재 Windows 호스트에서 `<DC_FQDN>:88`로 도달할 수 있으면, 해당 계정의 Kerberos TGT를 발급·주입한 뒤 `<HOST_FQDN>:<SERVICE_PORT>`에서 ticket 주체의 실제 서비스 권한을 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | Windows 실행 호스트에서 DNS, `<DC_FQDN>:88`, `<HOST_FQDN>:<SERVICE_PORT>` 도달 가능 | DC·대상 FQDN 해석, Kerberos와 대상 서비스 포트 연결 확인 | DNS·route·방화벽, DC 시간 정합 확인 |
| 현재 계정 또는 인증 수단 | `<DOMAIN>\<USER>`의 NTLM/RC4 또는 AES key | key 출처와 계정·도메인 대응 확인 | 로컬·도메인 계정 구분, username·domain·key type 재확인 |
| 현재 권한 | Rubeus 실행·ticket 적용 가능, Mimikatz 사용 시 관리자·debug 권한 | Rubeus 실행 가능 여부, Mimikatz `privilege::debug` 결과 | 권한이 부족하면 Rubeus 방식 가능 여부 또는 현재 호스트 권한 상승 검토 |
| 공격 대상의 조건 | 도메인 KDC가 해당 계정·key type을 허용하고 대상 서비스가 Kerberos SPN을 사용 | TGT 발급 결과와 `<HOST_FQDN>` 기반 서비스 접속 결과 | RC4 차단 시 AES key, SPN·FQDN·대상 서비스 ACL 확인 |
| 필요한 파일·목록·주소 | `<DOMAIN>`, `<USER>`, `<NTLM_HASH>` 또는 `<AES256_KEY>`, `<DC_FQDN>`, `<HOST_FQDN>` | 수집 기록과 도메인·realm·FQDN 대응 확인 | 계정·key·domain 쌍과 FQDN 수정 |

## 실행

`<DOMAIN>\<USER>`와 key는 TGT 요청 대상 계정이고 현재 Windows 실행 계정과 다를 수 있다. `<DC_FQDN>`은 KDC, `<HOST_FQDN>`과 `<SERVICE_PORT>`는 후속 서비스 대상이며 PID·LUID는 ticket 생성 단계 출력에서 얻는다.

### 방식 선택

| 방식 | 선택 조건 | 실행 위치·필요 권한 | 성공 결과 |
|---|---|---|---|
| Rubeus `asktgt`와 `/createnetonly` | `<USER>`의 AES key 또는 NTLM/RC4 key로 TGT를 직접 요청할 때 | DC와 통신할 Windows 세션 | KDC가 발급한 TGT와 별도 Type 9 로그온 세션의 ticket 사용 상태 |
| Mimikatz `sekurlsa::pth` | NT hash로 별도 로그온 세션을 만들고 그 세션에서 Kerberos ticket을 요청해야 할 때 | 대상 Windows 호스트의 상승된 로컬 관리자·debug 권한 | NT hash가 적용된 새 프로세스·로그온 세션이며 TGT는 이후 `klist`로 별도 확인 |

AES key를 보유했다면 Rubeus의 AES 방식을 우선 검토한다. Mimikatz 새 cmd 창이 열렸다는 사실만으로 TGT 발급이나 원격 서비스 접근이 성공한 것은 아니다.

1. key가 로컬 계정이 아니라 `<DOMAIN>\<USER>` 도메인 계정의 것인지, 현재 Windows 호스트에서 DC·DNS·시간 조건이 맞는지 확인한다.
2. Windows 실행 호스트에서 가능하면 AES key를 우선 사용해 `<USER>`의 TGT를 요청한다.
3. 가능하면 `/createnetonly`로 전용 Type 9 로그온 세션을 만들고, 출력된 PID·LUID를 기록한 뒤 새 process 안의 `klist`로 principal·ticket 종류·만료 시각을 확인한다.
4. [[Pass the Ticket]] 절차로 `<HOST_FQDN>:<SERVICE_PORT>`에 연결하여 ticket 사용, 서비스 인증, 원격 실행·관리자 권한을 별도로 검증한다.

### 명령과 확인할 출력

#### Rubeus로 전용 세션에 TGT 요청

현재 Rubeus upstream의 `asktgt`가 `/createnetonly`를 지원하는지 `Rubeus.exe asktgt /?`에서 확인한다. `<OPTH_PROCESS_PID>`와 `<OPTH_LOGON_LUID>`는 출력된 값을 그대로 기록한다.

```cmd
Rubeus.exe asktgt /domain:<DOMAIN> /user:<USER> /aes256:<AES256_KEY> /createnetonly:"C:\Windows\System32\cmd.exe" /show
Rubeus.exe asktgt /domain:<DOMAIN> /user:<USER> /rc4:<NTLM_HASH> /createnetonly:"C:\Windows\System32\cmd.exe" /show
REM 새로 열린 cmd.exe에서 실행
klist
```

확인할 출력:

- `TGT request successful`은 `<USER>` key로 KDC에서 TGT를 발급받은 상태다. `ProcessID`·`LUID`, `Ticket successfully imported`와 새 process의 `klist`에 표시된 `<USER>` TGT는 전용 로그온 세션에서 ticket을 사용할 수 있는 상태다. 둘 다 특정 원격 서비스 권한은 입증하지 않는다.

#### Mimikatz 방식

```cmd
mimikatz.exe
privilege::debug
sekurlsa::pth /domain:<DOMAIN> /user:<USER> /ntlm:<NTLM_HASH>
```

확인할 출력:

- 출력된 PID와 LUID를 각각 `<OPTH_PROCESS_PID>`·`<OPTH_LOGON_LUID>`로 기록한다. 새 cmd 세션은 key가 가리키는 `<DOMAIN>\<USER>`의 네트워크 인증 자료를 사용할 수 있는 별도 로그온 세션이다. 이 세션에서 Kerberos 요청이 가능한지 `klist`와 대상 서비스 접속으로 확인한다.

## 변경 영향과 복구

먼저 전용 cmd에서 시작한 SMB·WinRM·WMI client를 정상 종료한다. 상승된 셸에서 exact LUID를 대상으로 ticket을 지울 수 있으면 다음 순서로 정리한다.

```cmd
Rubeus.exe purge /luid:<OPTH_LOGON_LUID>
taskkill /PID <OPTH_PROCESS_PID> /T
tasklist /FI "PID eq <OPTH_PROCESS_PID>"
```

`/luid` purge는 상승된 권한이 필요하다. Rubeus를 일반 사용자로 실행했거나 purge가 거부되면 다른 로그온 세션을 건드리지 말고 기록한 process tree만 종료한다. 마지막 `tasklist`에 해당 PID가 없어야 전용 로그온 세션 정리가 끝난 것이다. 현재 공유 로그온 세션에 `/ptt`로 직접 주입하면 기존 TGT를 보존하면서 이번 ticket만 분리 복구하기 어렵기 때문에 격리된 일회성 세션이 아닌 곳에서는 fallback으로 사용하지 않는다. KDC·서비스 감사 기록과 이미 수행한 원격 접근은 되돌릴 수 없다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `TGT request successful`과 `klist`에 `<USER>` TGT가 표시됨 | NTLM/AES key가 도메인 KDC에 유효하고 현재 세션에서 ticket 사용 가능 | `<DOMAIN>\<USER>` Kerberos TGT | [[Pass the Ticket]]에서 대상 SPN의 service ticket 발급과 `<HOST_FQDN>:<SERVICE_PORT>` 접근 확인 |
| 새 ticket으로 SMB·WinRM·WMI 인증이 성공함 | ticket 주체가 해당 서비스에서 인증됨 | 인증된 서비스 접근 | WinRM은 세션 생성을 [[WinRM 원격 PowerShell 세션]]에서, SMB는 share 목록·읽기·쓰기 권한을 확인 |
| KDC preauth 실패 | hash/key 불일치 | 시작 상태 유지 | 계정명, 도메인, key type 확인 |
| RC4 탐지/차단 | AES만 허용 또는 정책 | 시작 상태 유지 | AES key 수집 시도 |
| ticket은 있으나 접근 실패 | key·TGT는 유효하지만 SPN·FQDN 불일치 또는 서비스 권한 부족 | Kerberos TGT 보유, 대상 서비스 경로 미확보 | SPN·FQDN·시간, 계정 그룹과 대상 ACL 확인 |

## 확인할 출력과 권한

- key 보유: `<DOMAIN>\<USER>`의 NTLM/RC4·AES key를 가진 상태이며 KDC 인증 성공 전이다.
- TGT 발급·주입: `TGT request successful`, `Ticket successfully imported`, `klist`의 principal·만료 시각으로 확인하며 원격 서비스 권한은 아직 미확인이다.
- 서비스 인증: `<HOST_FQDN>:<SERVICE_PORT>`의 Kerberos 성공 응답으로 확인하고, share 열거·원격 prompt·명령 출력으로 실제 허용 행동을 추가 확인한다.
- 권한 구분: 인증 성공, 서비스 접근, 원격 명령 실행, 로컬 관리자, 도메인 고권한을 서비스 응답만으로 추정하지 않는다.

## 후속 공격 연결

- [[Pass the Ticket]]
- [[WinRM 원격 PowerShell 세션]]
- [[SMB 공유 자격증명 수집]]

## 관련 상태 라우터

- Kerberos TGT를 만들었지만 원격 서비스를 선택하지 않았으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- ticket 주체로 AD 서비스 인증이 확인됐으면: [[AD Identity 확인 후 도메인 컨텍스트 열거]]

## 관련 도구

- [[rubeus]]
- [[mimikatz]]
- [[klist]]

## 참고 링크

- [GhostPack Rubeus: asktgt와 createnetonly](https://github.com/GhostPack/Rubeus)
- [gentilkiwi Mimikatz: sekurlsa module](https://github.com/gentilkiwi/mimikatz/wiki/module-~-sekurlsa)
- [Microsoft klist](https://learn.microsoft.com/windows-server/administration/windows-commands/klist)
