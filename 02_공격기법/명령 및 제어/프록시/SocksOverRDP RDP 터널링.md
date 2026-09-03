---
tags:
  - 환경/windows
  - 서비스/rdp
시작조건: ["운영 호스트에서 <PIVOT_IP>:3389/TCP RDP 세션 확보", "피벗 호스트에서 <INTERNAL_IP>:<PORT> TCP 연결 가능"]
필요권한: ["플러그인 등록과 SocksOverRDP server 실행이 가능한 Windows 관리자 권한"]
필요조건: ["첫 RDP 피벗에 로그인할 계정", "최종 내부 서비스에 인증할 별도 계정 또는 인증 수단", "RDP 세션에서 파일 전송 또는 도구 실행 가능", "SocksOverRDP client/server 구성 요소 준비"]
결과: ["RDP client 측 127.0.0.1:1080 SOCKS 프록시", "운영 호스트에서 <INTERNAL_IP>:<PORT>까지의 TCP 경로"]
---

# SocksOverRDP RDP 터널링

## 한 줄 판단

운영 호스트에서 `<PIVOT_IP>:3389`로 RDP 로그인할 수 있고 피벗 호스트에서 `<INTERNAL_IP>:<PORT>`에 연결할 수 있으면, RDP Dynamic Virtual Channel을 통해 client 측 `127.0.0.1:1080` SOCKS 프록시와 내부 TCP 경로를 만든다.

## 사용할 때

- 현재 네트워크 위치: 운영 호스트에서 `<PIVOT_IP>:3389/TCP`에는 연결할 수 있지만 `<INTERNAL_IP>:<PORT>`에는 직접 연결할 수 없고, 피벗 호스트에서는 해당 내부 포트에 연결할 수 있다.
- 명령 실행 위치: plugin은 RDP client 측 Windows 환경에 등록하고, server 구성 요소는 `<PIVOT_IP>`의 RDP 세션에서 실행한다. 최종 서비스 클라이언트는 `127.0.0.1:1080`을 사용할 운영 호스트에서 실행한다.
- 현재 계정·권한: 첫 RDP 계정은 피벗 세션을 여는 용도이며, plugin 등록과 server 실행에 필요한 Windows 관리자 권한을 별도로 확인한다.
- 보유 인증 자료: `<PIVOT_IP>` RDP 인증 성공은 `<INTERNAL_IP>`의 RDP 또는 다른 서비스 인증을 보장하지 않는다. 최종 대상에는 그 호스트·서비스에서 유효한 별도 계정이나 인증 수단이 필요하다.
- 성공 범위: 운영 호스트의 `127.0.0.1:1080`에서 `<INTERNAL_IP>:<PORT>`까지 TCP가 전달된다. 로그인 화면이나 서비스 배너는 서비스 접근이며, 인증 성공·원격 세션·관리자 권한은 각각 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 운영 호스트→`<PIVOT_IP>:3389`와 피벗→`<INTERNAL_IP>:<PORT>` TCP 연결 가능 | 첫 RDP 접속 후 피벗 세션에서 최종 포트를 확인 | 첫 RDP 방화벽·NLA와 피벗 호스트의 route·대상 포트를 분리 확인 |
| 현재 계정 또는 인증 수단 | 피벗 호스트용 RDP 계정과 최종 내부 서비스용 인증 자료를 역할별로 보유 | `xfreerdp` 또는 `mstsc`로 첫 세션을 열고 최종 계정은 별도 검증 | 계정의 대상 호스트, 도메인, 로그온 유형과 만료 상태 확인 |
| 현재 권한 | plugin 등록과 SocksOverRDP server 실행 가능 | 각 실행 위치에서 현재 Identity와 관리자 권한 확인 | 관리자 세션, x86·x64 아키텍처와 파일 경로 확인 |
| 공격 대상의 조건 | 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` 서비스가 응답 | 피벗 RDP 세션에서 TCP 연결 또는 서비스 클라이언트로 확인 | 내부 DNS·route·방화벽과 대상 listener 확인 |
| 필요한 파일·목록·주소 | client/server 구성 요소와 `<PIVOT_IP>`, `<INTERNAL_IP>`, `<PORT>` | RDP drive redirection 또는 기존 전송 경로로 파일 존재 확인 | 파일 전송 방식과 구성 요소 버전 재확인 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| RDP만 연결 가능한 Windows 구간 | SSH/Chisel 대안 필요 | SocksOverRDP 검토 |
| plugin 등록 성공 | DVC 플러그인 준비 | RDP 재접속 후 listener 확인 |
| `127.0.0.1:1080` listen | SOCKS proxy 생성 | Proxifier 또는 SOCKS 지원 도구 연결 |

### 절차
1. 운영 호스트에서 `<PIVOT_IP>:3389`로 RDP 로그인하고 피벗 세션의 현재 Identity와 권한을 확인한다.
2. RDP client 측 Windows 환경에서 SocksOverRDP plugin DLL을 등록한다.
3. 피벗 호스트의 RDP 세션에서 SocksOverRDP server를 실행한다.
4. RDP client 측 `127.0.0.1:1080` SOCKS listener를 확인한다.
5. Proxifier에 SOCKS5 `127.0.0.1:1080`을 등록하고 `mstsc.exe` 등 사용할 애플리케이션의 rule에 적용한다.
6. `<INTERNAL_IP>:<PORT>`까지 TCP 응답을 확인한 뒤, 최종 대상용 인증 자료로 로그인 또는 명령 실행을 별도 검증한다.

### Windows RDP 클라이언트에서 plugin DLL 등록

```cmd
regsvr32.exe SocksOverRDP-Plugin.dll
```

확인할 출력:

- DLL 등록 성공 대화상자 또는 성공 메시지. 이 출력은 plugin 등록만 입증하며 SOCKS listener나 내부 서비스 접근을 입증하지 않는다.

### Windows RDP 클라이언트에서 SOCKS listener 확인

```cmd
netstat -antb | findstr 1080
```

확인할 출력:

- RDP client 측 `127.0.0.1:1080` listen 상태. listener 생성 뒤에도 `<INTERNAL_IP>:<PORT>`의 실제 응답을 별도로 확인한다.

### Windows RDP 피벗 호스트에서 server 실행

```cmd
SocksOverRDP-Server.exe
```

확인할 출력:

- server가 RDP Dynamic Virtual Channel 연결을 처리하는 상태가 되어야 한다. 프로세스 시작만으로 client 측 listener나 내부 서비스 접근을 판단하지 않는다.

### Windows RDP 클라이언트에서 Proxifier 설정

- Proxy Server: `127.0.0.1:1080`
- Protocol: `SOCKS5`
- Proxification Rule: `mstsc.exe` 또는 실제로 내부 서비스에 연결할 애플리케이션

확인할 출력:

- Proxifier connection log에 `<INTERNAL_IP>:<PORT>`가 표시되고 대상 서비스가 응답해야 한다.
- listener는 있으나 connection log가 없으면 애플리케이션 rule을, log는 있으나 대상 응답이 없으면 SocksOverRDP server·피벗 route·최종 listener를 확인한다.

### Linux 공격 호스트에서 RDP 접속

```bash
xfreerdp /v:<PIVOT_IP> /u:<USER> /p:'<PASSWORD>' /drive:share,<LOCAL_PATH>
```

확인할 출력:

- `<PIVOT_IP>:3389` RDP 인증 성공, 파일 복사와 피벗 GUI 세션 진입 가능. 이 계정의 최종 내부 호스트 권한은 아직 확인되지 않은 상태다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 첫 Windows 피벗에 RDP 로그인 성공 | `<PIVOT_IP>`에서 DVC와 server를 운반할 인증된 RDP 세션이 확보됨 | 피벗 호스트의 Windows 사용자 세션 | 현재 Identity·관리자 권한과 plugin·server 구성 요소 확인 |
| DLL 등록 성공 후 `127.0.0.1:1080`이 listen | RDP DVC를 통한 SOCKS listener가 생성됨 | SOCKS 프록시 | Proxifier의 SOCKS protocol·port 확인 |
| SOCKS 규칙을 적용한 `<INTERNAL_IP>:<PORT>`의 RDP·서비스가 응답함 | DVC와 최종 TCP 경로가 모두 동작함 | 운영 호스트에서 내부 서비스 접근 | 최종 대상 계정의 인증, 원격 세션과 실제 권한을 차례로 확인 |
| DLL 등록이 실패함 | 관리자 권한, x86·x64 또는 경로가 맞지 않음 | plugin 미등록 | 권한과 아키텍처를 재확인 |
| 등록은 됐지만 `1080` listen이 없음 | RDP 재접속 또는 server 활성화가 필요함 | DVC 구성만 일부 완료 | RDP 세션 재연결과 server 실행 여부 확인 |
| SOCKS 연결은 되지만 RDP 인증이 실패함 | 터널보다 대상 계정·NLA·로그온 권한 문제일 수 있음 | 내부 RDP 인증 경로 | [[RDP 로그인과 GUI 세션]]으로 분기 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 첫 RDP hop | GUI 세션과 현재 `whoami` | plugin 설치가 가능한 관리자 세션인지 확인 |
| plugin | DLL 등록 성공과 RDP 재접속 | 파일 존재와 DVC 활성화를 구분 |
| listener·protocol | `127.0.0.1:1080` listen, SOCKS client 규칙 | 포트 생성과 실제 proxy 사용을 분리 |
| 최종 hop | 내부 RDP 로그인 화면·인증 결과 | TCP 접근과 대상 호스트 권한을 구분 |

## 변경 영향과 복구

작업 후 RDP client 측에 등록한 plugin과 피벗 호스트에서 실행한 server를 정리한다. 기존 등록 여부를 먼저 확인하고 이번 작업에서 추가한 구성 요소만 제거한다.

```cmd
regsvr32.exe /u SocksOverRDP-Plugin.dll
```

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| RDP client 측 plugin 등록 | RDP client가 SocksOverRDP DVC를 로드함 | DLL 등록 결과와 RDP 재접속 후 `1080` listener 확인 | server 종료 후 이번에 등록한 DLL만 `/u`로 해제 |
| 피벗 호스트의 SocksOverRDP server 프로세스 | RDP DVC를 통해 내부 TCP를 전달함 | 프로세스와 `127.0.0.1:1080` listener 확인 | server 프로세스를 종료하고 listener가 사라졌는지 확인 |

## 관련 상태 라우터

- RDP 경유 SOCKS로 내부 대상에 도달했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[SocksOverRDP]]
- [[Proxifier]]
- [[xfreerdp]]

## 관련 노트

- [[RDP 로그인과 GUI 세션]]
