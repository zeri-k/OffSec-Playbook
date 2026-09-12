---
tags:
  - 기능/열거
실행환경: ["Linux"]
필요권한: ["SYN, UDP, OS 탐지 등 raw socket 스캔 시 root 권한"]
필요조건: ["대상 IP, 호스트명, CIDR 또는 대상 목록"]
결과: ["호스트 상태", "포트 상태", "서비스 정보", "취약점 후보"]
---

# nmap

## 도구 개요

`nmap`은 호스트 발견, TCP·UDP 포트 상태 확인, 서비스·버전·운영체제 탐지와 NSE script 실행을 제공하는 네트워크 스캐너다. 대상 범위의 공격면을 단계적으로 파악하고 서비스별 열거 대상을 정리할 때 적합하며, 탐지 결과만으로 취약점이나 인증 권한을 확정하지 않는다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 입력: IP, 호스트명, CIDR 또는 `-iL` 대상 목록
- 선택 입력: 스캔할 TCP/UDP 포트, NSE script와 script 인자
- 권한 조건: SYN·UDP·OS 탐지처럼 raw socket을 사용하는 스캔은 root 권한 필요

## 표준 사용법

```shell
nmap [스캔 방식] [옵션] <target>
```

권장 흐름은 먼저 호스트 발견을 하고, 살아 있는 대상에 대해 전체 TCP 포트를 확인한 뒤, 열린 포트만 골라 서비스/스크립트 스캔을 수행하는 방식이다.

```shell
sudo nmap -sn <TARGET_CIDR> -oA scans/ping-sweep
sudo nmap -p- --min-rate 5000 <TARGET> -oA scans/all-tcp
sudo nmap -sC -sV -p 22,80,445 <TARGET> -oA scans/service
```

## 대표 예시

### 호스트 발견

```shell
sudo nmap -sn <TARGET_CIDR> -oA scans/ping-sweep
```

포트 스캔 없이 살아 있는 호스트를 찾는다. 로컬 네트워크에서는 ARP 응답이 우선 사용될 수 있다.

### 전체 TCP 포트 확인

```shell
sudo nmap -p- --min-rate 5000 -n -Pn <TARGET> -oA scans/all-tcp
```

초기 포트 누락을 줄이기 위한 전체 TCP 스캔이다. `-Pn`은 ping에 응답하지 않는 대상을 강제로 스캔할 때 쓴다.

### 서비스/버전/기본 스크립트 스캔

```shell
sudo nmap -sC -sV -p 22,80,445 <TARGET> -oA scans/service
```

열린 포트만 대상으로 서비스 배너, 버전, 기본 NSE 스크립트 결과를 수집한다.


## 주요 옵션

| 옵션 | 의미 |
| --- | --- |
| `-sn` | 포트 스캔 없이 호스트 발견만 수행 |
| `-Pn` | host discovery를 생략하고 대상이 살아 있다고 가정 |
| `-n` | DNS 조회 비활성화 |
| `-p 80,443` | 특정 포트 지정 |
| `-p-` | TCP 전체 포트 스캔 |
| `--top-ports <N>` | 자주 쓰는 상위 N개 포트 스캔 |
| `-iL <file>` | 파일에 있는 대상 목록 스캔 |
| `-sS` | SYN 스캔. 일반적으로 root 권한 필요 |
| `-sT` | TCP connect 스캔 |
| `-sU` | UDP 스캔 |
| `-sV` | 서비스/버전 탐지 |
| `-sC` | 기본 NSE 스크립트 실행 |
| `--script <name/category>` | 특정 NSE 스크립트 또는 카테고리 실행 |
| `--script-args <args>` | NSE 스크립트 인자 전달 |
| `-O` | OS 탐지 |
| `-A` | OS 탐지, 버전 탐지, 기본 스크립트, traceroute를 함께 실행 |
| `-T0`-`-T5` | 타이밍 템플릿. 숫자가 높을수록 빠르고 시끄러움 |
| `--min-rate <N>` | 초당 최소 패킷 전송률 힌트 |
| `--packet-trace` | 송수신 패킷 표시 |
| `--reason` | 포트/호스트 상태 판단 이유 표시 |
| `-oA <prefix>` | normal, grepable, XML 결과 동시 저장 |
| `-oN`/`-oG`/`-oX` | normal/grepable/XML 형식으로 저장 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `open` | 해당 포트에서 서비스 응답 확인 | 서비스 노트로 이동해 열거/공격 후보 선택 |
| `filtered` / `open|filtered` | 방화벽 또는 응답 없는 프로토콜 가능 | `-Pn`, TCP/UDP 구분, 재시도/소스 위치 확인 |
| service/version/NSE 결과 | 제품, 버전, 설정 단서 확인 | 취약 버전 여부와 서비스별 우선 열거 명령 확인 |
| 결과가 배너와 불일치 | 탐지 오탐 또는 프록시/중간 장비 가능 | 수동 배너 확인, `curl`, `nc`, 서비스 클라이언트로 재확인 |

## 관련 공격기법

- [[AD 서비스 스캔으로 DC와 도메인 식별]]

- [[SMB 익명 열거와 공유 권한 확인]]
- [[웹 정찰과 경로 열거]]
- [[TFTP 설정 파일 수집]]
- [[R-Services trust 기반 원격 접근]]
- [[SMTP 서비스#Open Relay 검증|SMTP Open Relay 검증]]
