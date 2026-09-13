---
tags:
  - 환경/ad
시작조건: ["내부망 호스트 후보 또는 IP 목록 확보", "명령 실행 호스트에서 대상 대역 접근 가능"]
필요권한: ["명령 실행 호스트에서 Nmap 실행 권한"]
필요조건: ["대상 호스트 목록", "대상 TCP 서비스에 도달 가능한 네트워크 경로", "승인된 능동 스캔 범위와 출력 basename"]
결과: ["Domain Controller 후보", "도메인 DNS 이름과 호스트 FQDN 후보", "AD 서비스별 후속 열거 대상"]
---

# AD 서비스 스캔으로 DC와 도메인 식별

## 한 줄 판단

내부망 호스트 후보와 대상 대역 접근이 있으면 Linux 공격 호스트에서 서비스·스크립트 스캔을 수행하여 DNS·Kerberos·LDAP·SMB·Global Catalog가 함께 나타나는 호스트와 도메인 이름을 찾고, Domain Controller 역할은 LDAP·DNS 결과로 다시 확인한다.

## 사용할 때

- 현재 보유 정보: 내부망에서 발견한 IP 또는 `hosts.txt`가 있지만 DC와 도메인명은 아직 모른다.
- 명령 실행 위치와 도달성: Nmap을 실행할 Linux 공격 호스트에서 후보 호스트의 TCP 서비스에 연결할 수 있다.
- 현재 계정과 권한: 대상 AD 계정은 필요하지 않다. 스캔 결과는 호스트·서비스 후보이며 인증이나 권한을 뜻하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상 목록 | 한 줄에 하나의 IP 또는 hostname | `hosts.txt`의 범위와 중복 확인 | [[내부망 수동 호스트 식별]]과 [[ICMP 기반 내부 호스트 확인]]으로 후보 보강 |
| 네트워크 경로 | 실행 호스트에서 대상 TCP 포트에 연결 가능 | route와 일부 알려진 서비스 응답 확인 | ICMP 무응답을 호스트 부재로 단정하지 말고 피벗·방화벽 확인 |
| 스캔 범위와 영향 | 승인된 host 목록과 AD 식별용 TCP 포트만 능동 확인 | 대상 수, 포트와 NSE 범위를 작업 전에 대조 | 산업·레거시 장비 등 불안정 가능 대상은 능동 스캔 승인 전 제외 |
| 결과 저장 위치 | Vault 밖의 승인된 작업 디렉터리와 새 `<SCAN_PREFIX>` | `.nmap`·`.xml`·`.gnmap` 부재 확인 | 기존 결과를 덮지 않는 basename 사용 |

## 실행

### 1. 대상과 출력 기준선 확인

Linux 공격 호스트에서 목록의 행 수와 범위를 확인하고, `-oA`가 만들 세 파일이 아직 없는지 확인한다.

```bash
wc -l '<HOST_LIST>'
test ! -e '<SCAN_PREFIX>.nmap' && test ! -e '<SCAN_PREFIX>.xml' && test ! -e '<SCAN_PREFIX>.gnmap'
```

확인할 출력:

- 대상 수와 승인 범위의 예상 호스트 수가 일치한다.
- `test`가 성공해 기존 출력 파일이 없음을 확인한다. 실패하면 기존 파일을 지우거나 덮지 말고 새 basename을 선택한다.

### 2. AD 식별 포트만 서비스·기본 스크립트로 확인

```bash
sudo nmap -n -Pn -sS -sC -sV -p53,88,135,139,389,445,464,636,3268,3269,3389 -iL '<HOST_LIST>' -oA '<SCAN_PREFIX>'
```

확인할 출력:

- DNS 53, Kerberos 88, LDAP 389·636, SMB 445, Global Catalog 3268·3269가 같은 호스트에서 함께 나타나는지 확인한다.
- LDAP script, `smb-os-discovery`, `rdp-ntlm-info`가 반환한 DNS domain, NetBIOS domain, hostname과 FQDN을 비교한다.
- `-sS` 권한 오류가 나면 root/raw socket 권한을 확인한다. 승인된 환경에서 raw socket을 쓸 수 없을 때만 `-sT` connect scan으로 바꾸고, 연결이 대상 로그에 남을 수 있음을 고려한다.
- `-sC`는 기본 NSE script를 실행하므로 대상 안정성·탐지 요구와 승인 범위를 확인한다. 단순 포트 상태만 필요하면 먼저 `-sC`를 빼고 확인한다.
- 원천의 `-A`는 OS 탐지·버전 탐지·기본 script·traceroute를 함께 켠다. 이 문서의 첫 선택에는 필요 범위를 넘으므로 사용하지 않고, 추가 프로파일링은 후보 하나와 승인된 옵션으로 제한한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| Kerberos·LDAP·SMB·GC와 같은 도메인명이 함께 확인됨 | DC 가능성이 높은 호스트 | DC·도메인 후보 | [[AD 도메인 컨텍스트 기본 확인]]에서 DNS SRV와 LDAP RootDSE 재확인 |
| SMB·RDP에서 Windows hostname만 확인됨 | Windows 멤버 호스트일 수 있음 | Windows 호스트 후보 | 식별된 [[SMB 서비스]] 또는 RDP 서비스 흐름 확인 |
| LDAP는 열렸지만 AD naming context가 확인되지 않음 | 일반 LDAP 또는 제한된 응답 가능 | LDAP 서비스 후보 | [[LDAP 서비스]]에서 RootDSE와 TLS 경로 확인 |
| 후보 전체가 timeout | 대상 부재보다 route·방화벽·피벗 문제 가능 | 서비스 상태 미확인 | 실행 호스트의 route와 [[내부망 경로 확보 후 피벗 구성]] 확인 |

## 확인할 출력과 권한

- 포트 열림은 서비스 접근 가능성, script 반환은 해당 응답에서 얻은 식별 정보다. AD 계정 유효성이나 객체 읽기 권한을 확정하지 않는다.
- DNS·LDAP·SMB에서 같은 도메인과 FQDN이 반복되는지 확인한 뒤 DC 후보를 정한다.

## 변경 영향과 복구

이 절차는 대상 설정을 바꾸지 않지만 능동 연결·NSE 요청이 대상 로그와 탐지 시스템에 남을 수 있으며 클라이언트에서 되돌릴 수 없다. 로컬에는 `-oA`가 `<SCAN_PREFIX>.nmap`, `<SCAN_PREFIX>.xml`, `<SCAN_PREFIX>.gnmap`을 만든다.

검토·인계 뒤 결과를 폐기하기로 했다면 기록한 세 exact 경로만 제거하고 부재를 확인한다. 같은 디렉터리의 다른 scan 결과를 pattern으로 삭제하지 않는다.

```bash
rm -- '<SCAN_PREFIX>.nmap' '<SCAN_PREFIX>.xml' '<SCAN_PREFIX>.gnmap'
test ! -e '<SCAN_PREFIX>.nmap' && test ! -e '<SCAN_PREFIX>.xml' && test ! -e '<SCAN_PREFIX>.gnmap'
```

스캔이 중단돼 일부 형식만 만들어졌다면 실제 존재하는 이번 basename의 파일만 개별 확인·삭제한다. 삭제 실패 시 경로·소유권과 결과를 열고 있는 process를 먼저 확인한다. 실제 대상·호스트 목록과 출력은 Vault에 저장하지 않는다.

## 관련 도구

- [[nmap]]

## 관련 상태 라우터

- [[무인증 내부 네트워크에서 AD 단서 확인]]

## 참고 링크

- [Nmap: Port Scanning Techniques](https://nmap.org/book/man-port-scanning-techniques.html)
- [Nmap: Output](https://nmap.org/book/man-output.html)
- [Nmap: Options Summary](https://nmap.org/book/man-briefoptions.html)
