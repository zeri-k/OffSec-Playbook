---
tags:
  - 기능/자격증명수집
실행환경: ["Linux"]
필요권한: ["라이브 분석 시 root 또는 패킷 캡처 권한"]
필요조건: ["PCAP 파일 또는 캡처 가능한 네트워크 인터페이스"]
결과: ["자격증명", "해시", "쿠키", "community string"]
---

# PCredz

## 도구 개요

`PCredz`는 저장된 packet capture나 live interface의 트래픽을 분석해 평문 자격 증명, NetNTLM 인증, 쿠키와 SNMP community string 후보를 추출한다. 여러 프로토콜의 인증 단서를 패킷에서 한 번에 선별할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 오프라인 입력: 단일 pcap/pcapng 파일 또는 캡처 파일 디렉터리
- 라이브 입력: 캡처할 네트워크 인터페이스
- 권한 조건: 라이브 분석은 root 또는 해당 인터페이스의 패킷 캡처 권한 필요

## 표준 사용법

```bash
./Pcredz -f <capture_file> [options]
```

## 대표 예시

### 단일 packet capture에서 credential 추출

```bash
./Pcredz -f demo.pcapng -t -v
```

### 여러 pcap 파일이 있는 디렉터리 분석

```bash
./Pcredz -d ./pcaps -t
```

### live interface에서 credential 관찰

```bash
./Pcredz -i eth0 -v
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-f` | 단일 pcap/pcapng 파일 분석 |
| `-d` | pcap 파일이 있는 디렉터리 분석 |
| `-i` | live interface 캡처 분석 |
| `-t` | TCP stream 기반 분석 활성화 |
| `-v` | 상세 출력 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| credential/hash/cookie 출력 | 트래픽에서 인증 정보 단서 확보 | 원본 프로토콜과 재사용 가능한 서비스 확인 |
| SNMP community 또는 NTLM hash 확인 | 네트워크 장비/Windows 인증 단서 | `snmpwalk`, cracking, relay 가능성 검토 |
| 결과 없음 | 암호화 트래픽 또는 캡처 범위 부족 | pcap 필터, 프로토콜, 캡처 위치 확인 |
| 파싱 오류 | 파일 손상 또는 형식 문제 | pcap/pcapng 무결성과 도구 버전 확인 |

## 관련 공격기법

- [[네트워크 트래픽 자격증명 수집]]
