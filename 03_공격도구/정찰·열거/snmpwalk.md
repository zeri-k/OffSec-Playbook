---
tags:
  - 서비스/snmp
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["유효한 community string"]
결과: ["OID", "시스템 정보", "네트워크 정보"]
---

# snmpwalk

## 도구 개요

`snmpwalk`는 SNMP OID 트리를 순차 조회해 시스템·프로세스·인터페이스·네트워크 정보를 수집하는 도구다. 넓은 OID 범위를 한 번에 열거하거나 특정 하위 트리를 좁혀 확인할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: Net-SNMP client가 설치된 Linux 호스트
- 입력: SNMP 서버, 버전과 community string
- 선택 입력: 조회할 OID


## 표준 사용법

```bash
snmpwalk -v<version> -c <community> <target> [oid]
```

## 대표 예시

`<TARGET>`은 SNMP agent IP/FQDN(예: `snmp.example.test`)이며, `public`은 실제로 확인한 community string으로 바꾼다. 명령은 agent에 UDP/161으로 도달 가능한 Linux 호스트에서 실행한다.

### community string으로 기본 mib-2 subtree 순회

```bash
snmpwalk -v2c -c public <TARGET>
```

OID를 생략하면 Net-SNMP는 전체 OID namespace가 아니라 기본적으로 `SNMPv2-SMI::mib-2` 아래를 순회한다. vendor enterprise OID나 다른 subtree가 필요하면 해당 시작 OID를 명시한다.

### 시스템 정보 OID만 조회

```bash
snmpwalk -v2c -c public <TARGET> 1.3.6.1.2.1.1
```

### 숫자 OID로 결과 정리

```bash
snmpwalk -v2c -c public -On <TARGET>
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-v1`, `-v2c`, `-v3` | SNMP 버전 지정 |
| `-c` | community string 지정 |
| `-On` | 숫자 OID로 출력 |
| `-Os` | 짧은 symbolic OID로 출력 |
| OID 인자 | 특정 OID 하위만 조회 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `sysDescr`, interface, process OID 출력 | community string으로 SNMP 열거 성공 | OS, 장비명, 인터페이스, 라우팅, 실행 프로세스 정리 |
| 민감 설정값 또는 community 노출 | 후속 접근 단서 확보 | 네트워크 장비/서비스 credential 재사용 가능성 확인 |
| `No Such Object` | OID가 없거나 view 제한 | 다른 OID tree와 SNMP 버전 확인 |
| timeout/authorization error | community 불일치 또는 접근 제한 | community string, SNMP version, UDP/161 접근성 확인 |

## 관련 공격기법

- [[SNMP OID 정보 열거]]

## 참고 링크

- [Net-SNMP snmpwalk](https://www.net-snmp.org/docs/man/snmpwalk.html)
