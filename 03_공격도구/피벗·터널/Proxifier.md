---
tags:
  - 환경/windows
  - 기능/피벗
실행환경: ["Windows"]
필요조건: ["로컬 SOCKS proxy 주소·포트·protocol", "프록시를 적용할 Windows 애플리케이션"]
결과: ["지정한 Windows 애플리케이션의 SOCKS 경유 TCP 연결"]
---

# Proxifier

## 도구 개요

Proxifier는 자체 proxy 설정을 지원하지 않는 Windows 애플리케이션의 TCP 연결을 SOCKS 또는 HTTPS proxy로 보내는 GUI 도구다. Plink나 SocksOverRDP가 만든 로컬 SOCKS listener를 `mstsc.exe` 같은 Windows 클라이언트에 적용하고, 애플리케이션별 연결 목적지와 성공·실패를 확인할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 로컬 SOCKS listener와 대상 Windows 애플리케이션을 사용하는 Windows 호스트
- proxy 입력: `127.0.0.1`의 SOCKS 포트와 실제 listener protocol
- rule 입력: proxy를 적용할 애플리케이션 실행 파일과 필요 시 대상 hostname·IP·port
- 확인 조건: SOCKS listener 생성, Proxifier rule 적용, 최종 서비스 응답과 인증을 서로 다른 단계로 확인한다.

## 표준 사용법

1. `Profile > Proxy Servers`에서 SOCKS server를 추가한다.
2. Address, Port와 Protocol을 현재 터널의 listener와 일치시킨다.
3. `Profile > Proxification Rules`에서 대상 애플리케이션을 추가하고 생성한 proxy를 선택한다.
4. 애플리케이션을 실행해 Proxifier connection log와 최종 서비스 응답을 함께 확인한다.

## 대표 예시

### Plink 동적 포워딩에 적용

| 설정 | 값 |
|---|---|
| Address | `127.0.0.1` |
| Port | `9050` |
| Protocol | `SOCKS4` |
| Applications | `mstsc.exe` 또는 내부 서비스 클라이언트 |

Plink의 `-D 9050` 세션이 유지되는 동안 적용한다. 로컬 listener만으로 피벗 호스트에서 내부 서비스까지의 연결은 확인되지 않는다.

### SocksOverRDP에 적용

| 설정 | 값 |
|---|---|
| Address | `127.0.0.1` |
| Port | `1080` |
| Protocol | `SOCKS5` |
| Applications | `mstsc.exe` 또는 내부 서비스 클라이언트 |

SocksOverRDP plugin, RDP 세션과 server가 모두 동작하는 상태에서 적용한다.

## 주요 설정

| 설정 | 의미 | 확인할 것 |
|---|---|---|
| Proxy Servers | 로컬 proxy endpoint 등록 | listener 주소·포트·SOCKS version 일치 |
| Proxification Rules | 애플리케이션별 proxy 적용 | 대상 실행 파일이 예상 rule에 매칭되는지 확인 |
| Applications | rule을 적용할 실행 파일 | `mstsc.exe` 등 실제 프로세스 이름 확인 |
| Target Hosts·Ports | rule 적용 범위 제한 | 직접 연결 rule보다 우선하는지 확인 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 행동 |
|---|---|---|
| connection log에 `<INTERNAL_IP>:<PORT>`와 proxy endpoint 표시 | 애플리케이션 연결이 지정 SOCKS proxy로 전달됨 | 최종 서비스 배너·로그인 화면 확인 |
| log는 있지만 timeout | rule은 적용됐으나 터널 종단 또는 내부 hop 실패 가능 | 터널 server·client, 피벗 route와 대상 listener 확인 |
| log가 없음 | 애플리케이션이 rule에 매칭되지 않음 | 실행 파일명, rule 순서와 direct rule 확인 |
| proxy connection refused | 로컬 SOCKS listener가 없음 | Plink·SocksOverRDP 세션과 로컬 포트 확인 |
| 로그인 화면 또는 배너 표시 | proxy와 최종 TCP 서비스까지 도달 | 대상 서비스 인증과 현재 권한을 별도 확인 |

## 관련 공격기법

- [[SSH 포트 포워딩 피벗팅]]
- [[SocksOverRDP RDP 터널링]]
