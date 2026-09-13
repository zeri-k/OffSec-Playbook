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

`<PCAP_FILE>`과 `<PCAP_DIRECTORY>`는 Linux 분석 호스트의 읽기 입력이고, `<INTERFACE>`는 그 호스트의 캡처 인터페이스(예: `eth0`)다. `<PCREDZ_OUTPUT_DIR>`은 `mktemp`으로 만든 새 로컬 출력 디렉터리다.

## 표준 사용법

`<PCREDZ_OUTPUT_DIR>`은 명령을 실행하는 Linux 분석 호스트의 새 디렉터리다. live mode의 `<INTERFACE>`는 그 호스트에서 `ip link`로 확인한 이름이고 `$PCREDZ_PID`는 background 실행 직후 `$!`에서 얻어 종료 단계에 재사용한다.

```bash
./Pcredz -f '<PCAP_FILE>' -o '<PCREDZ_OUTPUT_DIR>' [options]
```

현재 upstream의 `-o`는 credential 유형별 `logs/` 파일과 `CredentialDump-Session.log`를 지정 경로에 저장한다. 구버전은 출력 option·파일 배치가 다를 수 있으므로 `./Pcredz -h`로 확인한다.

## 대표 예시

### 단일 packet capture에서 credential 추출

```bash
PCREDZ_OUTPUT_DIR="$(mktemp -d "${PWD}/pcredz.XXXXXX")"
./Pcredz -f '<PCAP_FILE>' -o "$PCREDZ_OUTPUT_DIR" -t -v
find "$PCREDZ_OUTPUT_DIR" -xdev -type f -printf '%p %s bytes\n'
```

### 여러 pcap 파일이 있는 디렉터리 분석

```bash
./Pcredz -d '<PCAP_DIRECTORY>' -o "$PCREDZ_OUTPUT_DIR" -t
```

### live interface에서 credential 관찰

```bash
sudo -s
PCREDZ_OUTPUT_DIR='<PCREDZ_OUTPUT_DIR>'
./Pcredz -i '<INTERFACE>' -o "$PCREDZ_OUTPUT_DIR" -v &
PCREDZ_PID=$!
ps -p "$PCREDZ_PID" -o pid=,user=,args=
```

elevated shell 안에서 실행해 `$!`가 sudo wrapper가 아닌 PCredz 작업 PID를 가리킨다. 출력 경로는 승격 전에 만든 이번 작업의 고유 디렉터리 절대 경로를 쓴다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-f` | 단일 pcap/pcapng 파일 분석 |
| `-d` | pcap 파일이 있는 디렉터리 분석 |
| `-i` | live interface 캡처 분석 |
| `-o` | 형식별 민감 로그를 저장할 출력 디렉터리. 설치 버전의 help에서 지원 확인 |
| `-t` | TCP stream 기반 분석 활성화 |
| `-v` | 상세 출력 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| credential/hash/cookie 출력 | 트래픽에서 인증 정보 단서 확보 | 원본 프로토콜과 재사용 가능한 서비스 확인 |
| SNMP community 또는 NTLM hash 확인 | 네트워크 장비/Windows 인증 단서 | `snmpwalk`, cracking, relay 가능성 검토 |
| 결과 없음 | 암호화 트래픽 또는 캡처 범위 부족 | pcap 필터, 프로토콜, 캡처 위치 확인 |
| 파싱 오류 | 파일 손상 또는 형식 문제 | pcap/pcapng 무결성과 도구 버전 확인 |

## 변경 영향과 정리

오프라인 분석은 입력 pcap을 수정하지 않지만 `<PCREDZ_OUTPUT_DIR>`에 credential·hash·session log를 생성한다. 라이브 분석은 정확한 PID 기록이 필요하다. 라이브 작업이면 먼저 기록한 프로세스만 종료한다.

```bash
kill "$PCREDZ_PID"
wait "$PCREDZ_PID"
ps -p "$PCREDZ_PID"
```

필요한 분석 뒤에는 생성 직후 기록한 파일 목록과 대조한 뒤 이번 실행의 고유 출력 경로만 정리한다.

```bash
find "$PCREDZ_OUTPUT_DIR" -xdev -depth -type f -delete
find "$PCREDZ_OUTPUT_DIR" -xdev -depth -type d -empty -delete
test ! -e "$PCREDZ_OUTPUT_DIR"
```

경로가 남으면 예상하지 않은 파일·소유자·열린 프로세스를 확인하고 정리 완료로 판정하지 않는다. 라이브 분기에서 연 elevated shell은 정리 확인 후 `exit`로 닫는다. 입력 pcap·터미널 scrollback·원격 탐지 로그는 출력 디렉터리 삭제로 되돌려지지 않는다.

## 관련 공격기법

- [[네트워크 트래픽 자격증명 수집]]

## 참고 링크

- [PCredz 공식 저장소](https://github.com/lgandx/PCredz)
