---
tags:
  - 서비스/snmp
시작조건: ["대상 SNMP UDP 161 식별", "유효한 SNMPv1/v2c community 확보"]
필요권한: ["유효한 community로 OID view를 읽을 권한"]
필요조건: ["명령 실행 호스트에서 대상 UDP 161 접근", "유효한 community와 SNMP 버전"]
결과: ["community가 반환한 OID view의 호스트·계정·프로세스·소프트웨어·내부 네트워크 단서"]
---

# SNMP OID 정보 열거

## 한 줄 판단

유효한 SNMPv1/v2c community와 대상 UDP 161 경로가 있으면 해당 community가 반환하는 Object Identifier(OID) view를 조회하여 호스트·프로세스·소프트웨어·인터페이스 단서를 수집하고, community 후보 대입은 원격 비밀번호 공격으로 분리한다.

## 사용할 때

- `onesixtyone` 등으로 실제 응답을 반환하는 community를 이미 확인했을 때.
- 네트워크 장비·프린터·서버가 공개하는 시스템·프로세스·인터페이스 정보를 조사할 때.
- community 유효성과 특정 OID view 접근 범위를 구분해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 네트워크 경로 | 명령 실행 호스트에서 대상 UDP 161로 요청 가능 | 실제 SNMP 요청과 응답 확인 | `open|filtered`, timeout, source 제한을 구분 |
| 인증 수단 | 응답이 확인된 community와 SNMP 버전 | 좁은 system OID가 값을 반환하는지 확인 | [[원격 비밀번호 공격]]에서 후보와 잠금·소음 조건 재검토 |
| 조회 범위 | community로 읽을 수 있는 OID view | OID별 값 또는 authorization 오류 | 특정 OID 거부와 community 전체 실패를 구분 |

## 실행

### 주요 OID 조회

```bash
snmpwalk -v2c -c <COMMUNITY> <TARGET> 1.3.6.1.2.1.1
snmpwalk -v2c -c <COMMUNITY> <TARGET> 1.3.6.1.2.1.25.4.2.1.2
snmpwalk -v2c -c <COMMUNITY> <TARGET> 1.3.6.1.2.1.25.6.3.1.2
```

확인할 출력:

- 시스템 설명·hostname, 프로세스 이름, 설치 소프트웨어.
- 특정 OID만 거부되면 community 전체가 실패한 것으로 해석하지 않는다.

### 넓은 OID tree 조회

```bash
braa <COMMUNITY>@<TARGET>:.1.3.6.*
```

출력량이 크면 필요한 OID로 범위를 좁혀 같은 값을 재확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| OID와 값이 반환됨 | 해당 community와 source에서 읽을 수 있는 view | SNMP 정보 접근 | 시스템·프로세스·인터페이스 단서를 서비스별로 재확인 |
| 일부 OID만 반환됨 | 제한된 view | 제한된 SNMP 정보 접근 | 반환 범위만 기록하고 전체 tree 접근으로 확대하지 않음 |
| 사용자명·프로세스 인자·내부 IP가 반환됨 | 후속 검증이 필요한 정보 단서 | 계정·네트워크·서비스 후보 | 계정은 [[원격 비밀번호 공격]], 내부 주소는 서비스 도달성 확인 |
| timeout | UDP 필터링, 버전 불일치 또는 source 제한 가능 | SNMP 접근 미확정 | UDP 경로·버전·source 위치 확인 |
| authorization 오류 | 서비스는 응답하지만 현재 OID view가 거부됨 | 조회 범위 미확정 | community와 요청 OID를 분리해 재확인 |

## 확인할 출력과 권한

- OID 반환은 해당 community, source와 view의 읽기만 확정한다.
- writable OID, 운영체제 로그인과 다른 서비스 자격 증명 유효성은 별도 상태다.

## 후속 공격 연결

- community 후보 대입: [[원격 비밀번호 공격]]
- OID에서 발견한 계정·비밀번호 후보: [[원격 비밀번호 공격]]
- 설정 파일이나 패킷이 필요할 때: [[TFTP 설정 파일 수집]], [[네트워크 트래픽 자격증명 수집]]

## 관련 서비스

- [[SNMP 서비스]]

## 관련 도구

- [[snmpwalk]]
- [[braa]]
