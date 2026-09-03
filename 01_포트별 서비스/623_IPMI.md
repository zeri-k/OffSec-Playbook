---
tags:
  - 환경/network-appliance
  - 서비스/ipmi
대표포트:
  - "U:623"
서비스:
  - IPMI
  - BMC
---

# 623_IPMI

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:623/UDP`의 Intelligent Platform Management Interface(IPMI)에 질의할 수 있고, 아직 Baseboard Management Controller(BMC) 계정이나 관리 role은 확인하지 않은 상태에서 시작한다. IPMI 버전·인증 방식, IPMI 2.0 Remote Authenticated Key-Exchange Protocol(RAKP) hash, 기본 계정 로그인과 실제 BMC 기능 권한을 단계별로 구분한다.

**첫 화면 상태:** 지금 가능한 일은 BMC 프로토콜과 계정 검증 범위를 확인하는 것이다. 성공하면 offline cracking 대상 RAKP hash 또는 유효한 BMC 계정의 콘솔·센서·전원·원격 미디어 범위를 얻는다. timeout이면 UDP 손실·source 제한·관리망 위치를, hash 수집 실패는 IPMI 버전·cipher·사용자 이름을, 로그인 후 기능 거부는 BMC role을 확인한다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | IPMI 버전과 인증 방식 | `nmap -sU --script ipmi-version -p623 <TARGET>` | IPMI 2.0과 RAKP hash 수집 가능성을 확인한다. |
| 2 | Metasploit 버전 정보 | `auxiliary/scanner/ipmi/ipmi_version` | IPMI 버전과 Auth 정보를 교차 확인한다. |
| 3 | RAKP hash | `auxiliary/scanner/ipmi/ipmi_dumphashes` | 사용자 hash와 기본 비밀번호 매칭 결과를 확인한다. |
| 4 | offline cracking | `hashcat -m 7300 ipmi.txt <WORDLIST> --backend-ignore-opencl -d 1 -O -w 3` | 재사용 가능한 평문 BMC 비밀번호 후보를 확인한다. |

**출력 해석 경계:** `ipmi-version`의 응답은 IPMI 구현·지원 인증 정보를 확정하지만 RAKP hash 추출이나 로그인 성공은 확정하지 않는다. `ipmi_dumphashes` 출력은 offline cracking 입력을 제공할 뿐 평문 비밀번호를 증명하지 않는다. cracking 결과와 웹·콘솔 로그인 성공은 각각 별도로 확인하며, BMC 로그인도 호스트 OS의 계정·권한을 확정하지 않는다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| IPMI 2.0 | [[IPMI hash 수집과 크래킹]] | `metasploit` | RAKP 사용자 hash 수집 가능성 |
| RAKP hash | [[IPMI hash 수집과 크래킹]] | `hashcat` | offline cracking으로 복구한 BMC 사용자 이름·비밀번호 후보 |
| `ADMIN:ADMIN` 등 기본 계정 단서 | [[IPMI hash 수집과 크래킹]] | `ipmitool`, `metasploit` | 낮은 빈도로 검증한 BMC 로그인 |
| 유효 BMC 계정 | [[IPMI hash 수집과 크래킹]] | 웹·SSH·Telnet·iKVM 클라이언트 | 콘솔·센서·전원·원격 미디어의 실제 관리 범위 |

## 서비스 고유 주의 사항

- UDP 응답 없음과 필터링을 구분하기 어렵고 BMC는 별도 관리망에 있을 수 있다.
- 기본 계정은 제품·버전과 잠금 정책을 확인한 뒤 낮은 빈도로 검증한다.
- hash 확보, 평문 복구, BMC 로그인과 관리 기능 권한은 각각 다른 상태다.
- BMC 권한과 호스트 OS 권한은 별도로 확인한다.
