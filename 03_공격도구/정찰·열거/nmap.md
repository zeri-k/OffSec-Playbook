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

`<TARGET_CIDR>`은 Linux 실행 호스트에서 도달 가능한 대역(예: `192.0.2.0/24`)이고, `<TARGET>`은 그 결과 또는 다른 관찰에서 선택한 단일 IP/FQDN이다. `<OPEN_TCP_PORTS>`는 앞 `-p-` 출력의 open TCP 포트를 쉼표로 연결한 값(예: `80,443`)이며 `$NMAP_OUT_DIR`은 이번 실행의 새 출력 디렉터리다.

```shell
nmap [스캔 방식] [옵션] <target>
```

권장 흐름은 먼저 호스트 발견을 하고, 살아 있는 대상에 대해 전체 TCP 포트를 확인한 뒤, 열린 포트만 골라 서비스/스크립트 스캔을 수행하는 방식이다. 결과 파일이 기존 자료와 섞이지 않도록 현재 작업 디렉터리 아래에 고유 디렉터리를 먼저 만든다.

```shell
NMAP_OUT_DIR="$(mktemp -d "${PWD}/nmap-scan.XXXXXX")"
sudo nmap -sn <TARGET_CIDR> -oA "$NMAP_OUT_DIR/ping-sweep"
sudo nmap -p- -n -Pn <TARGET> -oA "$NMAP_OUT_DIR/all-tcp"
sudo nmap -sC -sV -p <OPEN_TCP_PORTS> <TARGET> -oA "$NMAP_OUT_DIR/service"
```

`--min-rate`와 높은 timing template은 손실·오탐·대상 부하를 늘릴 수 있으므로 대표 흐름에 고정하지 않는다. 손실과 대상 안정성을 확인한 뒤에만 별도 값을 적용한다.

## 대표 예시

### 호스트 발견

```shell
sudo nmap -sn <TARGET_CIDR> -oA "$NMAP_OUT_DIR/ping-sweep"
```

포트 스캔 없이 살아 있는 호스트를 찾는다. 로컬 네트워크에서는 ARP 응답이 우선 사용될 수 있다.

### 전체 TCP 포트 확인

```shell
sudo nmap -p- -n -Pn <TARGET> -oA "$NMAP_OUT_DIR/all-tcp"
```

초기 포트 누락을 줄이기 위한 전체 TCP 스캔이다. 옵션 없는 기본 스캔은 일반적으로 자주 쓰는 TCP 1,000개를 대상으로 하므로 전체 포트 확인과 같지 않다. `-Pn`은 host discovery 결과와 무관하게 지정 대상을 스캔하지만 host가 실제로 살아 있음을 증명하지 않는다.

### 서비스/버전/기본 스크립트 스캔

```shell
sudo nmap -sC -sV -p <OPEN_TCP_PORTS> <TARGET> -oA "$NMAP_OUT_DIR/service"
```

열린 포트만 대상으로 서비스 배너, 버전, 기본 NSE 스크립트 결과를 수집한다.

기본 port scan의 `SERVICE` 열은 주로 포트 번호의 이름 매핑이며 실제 listener 식별과 다를 수 있다. `-sV` probe·수동 banner·전용 client 결과가 일치하는지 확인하고, package banner만으로 OS release나 취약 build를 확정하지 않는다.

`-sC`, `-A`, `--script`는 대상에 추가 요청을 보낸다. 실행 전에 `nmap --script-help <SCRIPT_OR_CATEGORY>`와 NSE 문서에서 선택된 script, 인자, `auth`·`brute`·`dos`·`exploit`·`external`·`fuzzer`·`intrusive` 범주 및 대상 상태 변경 가능성을 확인한다. `default`나 `safe` 범주도 무영향을 보장하지 않는다.


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
| `closed` | 대상이 응답했지만 해당 transport port에서 listener를 확인하지 못함 | 같은 host의 다른 port·protocol과 스캔 시점을 확인 |
| `filtered` / `open|filtered` | 필터 또는 응답 없는 protocol 때문에 상태를 단정할 수 없음 | TCP/UDP 구분, `--reason`, 재시도 시점·소스 위치 확인 |
| service/version/NSE 결과 | 제품, 버전, 설정 단서 확인 | 취약 버전 여부와 서비스별 우선 열거 명령 확인 |
| 결과가 배너와 불일치 | 탐지 오탐 또는 프록시/중간 장비 가능 | 수동 배너 확인, `curl`, `nc`, 서비스 클라이언트로 재확인 |

## 변경 영향과 복구

능동 scan은 대상 로그·탐지 시스템에 남을 수 있으며 클라이언트 정리로 되돌릴 수 없다. 로컬에는 각 `-oA` prefix마다 `.nmap`, `.xml`, `.gnmap` 파일이 생긴다. 보존하지 않을 때는 이번 고유 디렉터리에서 예상한 파일만 확인하고 제거한다.

```shell
find "$NMAP_OUT_DIR" -maxdepth 1 -type f -printf '%f %s bytes\n'
rm -- "$NMAP_OUT_DIR/ping-sweep.nmap" "$NMAP_OUT_DIR/ping-sweep.xml" "$NMAP_OUT_DIR/ping-sweep.gnmap"
rm -- "$NMAP_OUT_DIR/all-tcp.nmap" "$NMAP_OUT_DIR/all-tcp.xml" "$NMAP_OUT_DIR/all-tcp.gnmap"
rm -- "$NMAP_OUT_DIR/service.nmap" "$NMAP_OUT_DIR/service.xml" "$NMAP_OUT_DIR/service.gnmap"
rmdir -- "$NMAP_OUT_DIR"
test ! -e "$NMAP_OUT_DIR"
```

scan이 중단되어 일부 파일만 생겼다면 존재하는 정확한 경로만 제거한다. 디렉터리가 비지 않아 `rmdir`이 실패하면 재귀 삭제하지 말고 예상하지 않은 파일을 확인한다.

## 관련 공격기법

- [[AD 서비스 스캔으로 DC와 도메인 식별]]

- [[SMB 익명 열거와 공유 권한 확인]]
- [[웹 정찰과 경로 열거]]
- [[TFTP 설정 파일 수집]]
- [[R-Services trust 기반 원격 접근]]
- [[SMTP 서비스#Open Relay 검증|SMTP Open Relay 검증]]

## 참고 링크

- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [Nmap NSE usage and script categories](https://nmap.org/book/nse-usage.html)
- [Nmap output options](https://nmap.org/book/man-output.html)
