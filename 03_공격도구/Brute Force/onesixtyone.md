---
tags:
  - 서비스/snmp
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["SNMP 대상 IP 또는 호스트 목록", "community string 목록"]
결과: ["유효 community string", "SNMP 응답"]
---

# onesixtyone

## 도구 개요

`onesixtyone`은 wordlist의 SNMPv1·v2c community string을 하나 이상의 대상에 대입해 응답하는 값을 찾는 도구다. 여러 agent에서 유효한 community 후보를 빠르게 선별하는 초기 SNMP 열거 작업에 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 입력: SNMP 대상 IP 또는 `-i` 호스트 목록
- 입력: `-c`로 지정할 community string wordlist
- 네트워크 조건: 대상 UDP/161로 패킷 전송 가능

## 표준 사용법

```bash
onesixtyone -c <community_wordlist> <target>
```

## 대표 예시

### 단일 대상 community string 추측

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET>
```

### 여러 대상에 같은 wordlist 적용

```bash
onesixtyone -c dict.txt -i hosts.txt
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-c` | community string wordlist 지정 |
| `-i` | 대상 IP 목록 파일 지정 |
| `-o` | 결과 저장 파일 지정 |
| `-w` | 응답 대기 시간 조정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| community string과 응답 확인 | 유효한 SNMP community string 발견 | `snmpwalk` 또는 `braa`로 OID tree 열거 |
| 일부 community만 응답 | 권한 또는 view 제한 가능 | `public`, `private`, 조직명 기반 후보와 OID 범위 확장 |
| 응답 없음/timeout | UDP 필터링, SNMP 비활성화, community 불일치 | UDP/161 접근성, SNMP 버전, 재시도 간격 확인 |
| too many retries/rate 영향 | UDP 손실 또는 장비 응답 제한 | 스레드와 재시도 횟수 조정 |
| SNMPv3만 가능 | v1/v2c community 방식 미지원 | SNMPv3 사용자/인증 정보 확보 여부 확인 |

## 관련 공격기법

- [[원격 비밀번호 공격]]
