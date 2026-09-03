---
tags:
  - 서비스/snmp
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["SNMP community string", "대상 OID 또는 OID 범위"]
결과: ["정보"]
---

# braa

## 도구 개요

`braa`는 SNMPv1·v2c OID 값을 여러 대상에서 빠르게 조회하는 명령줄 도구다. 넓은 SNMP tree 전체보다 특정 OID 범위를 집중적으로 훑거나 여러 agent의 응답을 비교하는 작업에 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: SNMP agent의 UDP/161에 접근 가능한 Linux 호스트
- 필요한 입력: SNMP community string, 대상 주소, 조회할 OID 또는 OID 패턴
- 대상: SNMP agent가 노출된 네트워크 장비, 서버, 프린터


## 표준 사용법

```bash
braa <community>@<target>:<oid>
```

## 대표 예시

### community string을 알고 있을 때 OID 범위 훑기

```bash
braa public@<TARGET>:.1.3.6.*
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| `<community>` | SNMP community string |
| `<target>` | 조회할 IP 또는 호스트 |
| `<oid>` | 조회할 OID. `*`를 사용해 하위 범위를 훑을 수 있음 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| OID 값 다수 출력 | community string으로 SNMP 조회 성공 | 시스템, 인터페이스, 라우팅, 프로세스 관련 OID를 우선 확인 |
| 특정 OID만 응답 | view 제한 또는 OID 범위 문제 | `snmpwalk`로 기준 OID를 좁히고 필요한 tree만 재조회 |
| timeout/no response | community 불일치, UDP 필터링, SNMP 비활성화 | community string, SNMP 버전, UDP/161 접근성 확인 |
| 출력이 너무 많음 | 범위가 넓거나 장비 응답이 방대함 | OID 범위를 좁히고 결과를 파일로 저장해 검색 |

## 관련 공격기법

- [[SNMP OID 정보 열거]]
