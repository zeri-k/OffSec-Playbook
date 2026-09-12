---
tags:
  - 서비스/snmp
대표포트:
  - "U:161"
  - "U:162"
서비스:
  - SNMP
---

# SNMP 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Simple Network Management Protocol(SNMP) UDP 161 또는 trap 수신 포트 UDP 162에 도달할 가능성이 있고, 아직 유효한 community string이나 관리 권한은 확인하지 않은 상태에서 시작한다. SNMPv1/v2c community 인증, Object Identifier(OID) 읽기와 writable OID를 단계별로 구분하며 UDP 162 응답은 UDP 161 조회 권한을 뜻하지 않는다.

성공하면 community가 허용하는 호스트·인터페이스·라우팅·프로세스 정보를 얻는다. timeout이면 UDP 응답 손실·방화벽·source 제한·잘못된 community를, `authorizationError`나 빈 OID tree면 community view·SNMP 버전·OID 범위를 다시 확인하고, 읽기 성공만으로 장비 설정 권한을 단정하지 않는다.

MIB는 object의 이름·자료형·접근 속성과 OID 트리 위치를 정의하는 schema이고, 실제 값은 SNMP agent가 반환한다. 따라서 특정 OID subtree의 값이 보인다는 결과는 그 community와 source에 허용된 view의 읽기만 확정하며, 전체 MIB나 쓰기 권한이 공개됐다는 뜻은 아니다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | 161/162 UDP 응답 | `nmap -sU -sV -p161,162 <TARGET>` | SNMP 응답과 trap 포트 노출을 구분한다. |
| 2 | 기본 community READ | `snmpwalk -v2c -c public <TARGET>` | OID 출력이 있으면 읽기 community로 확인한다. |
| 3 | community 후보 | [[원격 비밀번호 공격]]에서 `onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET>` 실행 | 유효 community와 응답 장비를 확인한다. |
| 4 | OID 범위 | `braa <community>@<TARGET>:.1.3.6.*` | 호스트명, 인터페이스, 라우팅, 프로세스와 명령줄 등 OID별 값을 확인한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| community 후보만 있음 | [[원격 비밀번호 공격]] | `onesixtyone` | 실제 응답을 반환한 SNMPv1/v2c community 후보 |
| `public` 또는 다른 community 유효 | [[SNMP OID 정보 열거]] | `snmpwalk`, `braa` | 허용된 OID view의 호스트·OS·인터페이스·라우팅·프로세스 정보 |
| 사용자·프로세스·명령줄·네트워크 정보 | [[SNMP OID 정보 열거]] | `snmpwalk` | 계정·서비스·내부 네트워크 후보 |
| rwcommunity 또는 writable OID 단서 | 수동 확인: 장비 상태를 바꾸지 않는 OID에서 별도 쓰기 검증 설계 | `snmpwalk` | 읽기 결과와 분리된 쓰기 가능성 후보 |

## 서비스 고유 주의 사항

- SNMP는 UDP라 응답 없음과 필터링을 구분하기 어렵고 source IP 제한이 있을 수 있다.
- SNMPv3는 community가 아니라 사용자·인증·암호화 설정이 필요할 수 있다.
- rwcommunity 발견은 설정 변경 성공과 같지 않으며 서비스 장애를 만들 수 있는 실제 `SET`은 피한다.
- UDP 162 trap 노출은 UDP 161의 OID 조회 권한을 뜻하지 않는다.

## 참고 링크

- [Net-SNMP snmpwalk](https://www.net-snmp.org/docs/man/snmpwalk.html)
- [Net-SNMP snmpd.conf access control](https://www.net-snmp.org/docs/man/snmpd.conf.html)
