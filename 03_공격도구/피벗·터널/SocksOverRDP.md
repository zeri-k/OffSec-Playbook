---
tags:
  - 환경/windows
  - 서비스/rdp
  - 기능/피벗
실행환경: ["Windows"]
필요권한: ["RDP client의 plugin DLL 등록은 상승된 Windows 관리자 권한", "RDP server 실행은 현재 사용자 권한"]
필요조건: ["Windows mstsc.exe client", "유효한 RDP 계정", "RDP Dynamic Virtual Channel 사용 가능", "client와 server 아키텍처에 맞는 구성 요소"]
결과: ["SOCKS 프록시", "네트워크 접근"]
---

# SocksOverRDP

## 도구 개요

SocksOverRDP는 RDP Dynamic Virtual Channel을 통해 RDP 피벗 호스트가 도달하는 내부 TCP 연결을 로컬 SOCKS 프록시로 전달하는 Windows용 도구다. 사용 중인 RDP 세션을 피벗 경로로 확장해 Windows GUI 애플리케이션이나 프록시 클라이언트를 내부망에 연결할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 환경: plugin DLL을 등록하고 `mstsc.exe`를 실행하는 Windows client와, RDP 세션 안에서 server EXE를 실행할 Windows 피벗 호스트
- 입력: RDP 계정, plugin DLL과 server 실행 파일
- 필요 권한: 공식 기본 절차에서 plugin DLL 등록은 client의 상승된 관리자, server EXE는 피벗 호스트의 일반 사용자도 실행 가능
- 네트워크 조건: RDP Dynamic Virtual Channel 사용 가능

## 표준 사용법

plugin을 Windows RDP client에 등록하고 그 client의 `mstsc.exe`로 RDP 세션을 연 뒤, 피벗 호스트에서 server를 실행한다. 두 구성 요소가 같은 호스트에서 연속 실행되는 명령으로 해석하지 않는다.

## 대표 예시

### plugin 등록

```cmd
regsvr32.exe "<CLIENT_PLUGIN_PATH>"
```

확인할 출력:

- RegSvr32 성공 메시지.

### SOCKS listener 확인

```cmd
netstat -antb | findstr 1080
```

확인할 출력:

- `127.0.0.1:1080` listen 상태.

### RDP server 측 구성 요소

```powershell
$SocksOverRdpServer = Start-Process -FilePath '<SERVER_EXE_PATH>' -PassThru
$SocksOverRdpServer | Select-Object Id,Path,StartTime
```

확인할 출력:

- server PID·경로·시작 시각. 이 EXE는 피벗 호스트의 일반 사용자로 실행할 수 있지만, client의 plugin 등록 권한까지 낮아지는 것은 아니다.

### [[Proxifier]]로 Windows 애플리케이션 연결

1. Proxifier의 proxy server에 `127.0.0.1`, `1080`, `SOCKS5`를 등록한다.
2. Proxification Rule에서 `mstsc.exe`처럼 터널을 사용할 애플리케이션을 이 proxy로 보낸다.
3. 애플리케이션에서 `<INTERNAL_IP>:<PORT>`에 연결하고 Proxifier connection log와 대상 서비스 응답을 함께 확인한다.

확인할 출력:

- Proxifier에 `<INTERNAL_IP>:<PORT>` 연결이 표시되고 대상 서비스의 로그인 화면이나 배너가 나타나야 DVC와 내부 TCP 경로가 모두 동작한 것이다.
- `127.0.0.1:1080` listener만 보이는 상태는 client 측 plugin이 준비된 것이며 SocksOverRDP server와 최종 내부 hop의 성공을 뜻하지 않는다.

## 주요 구성요소

| 요소 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `SocksOverRDP-Plugin.dll` | `mstsc.exe`를 실행하는 Windows client 구성요소 | RDP DVC 활성화와 client 측 SOCKS listener |
| `SocksOverRDP-Server.exe` | RDP로 접속한 Windows 피벗 호스트 구성요소 | DVC를 통해 피벗 호스트의 내부 연결 중계 |
| `127.0.0.1:1080` | SOCKS listener | Proxifier/프록시 클라이언트 연결 |
| Proxifier | GUI 기반 SOCKS 라우팅 보조 도구 | Windows GUI 앱을 SOCKS로 보낼 때 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| RegSvr32 성공 | plugin 등록 | RDP 재접속 또는 server 실행 |
| `127.0.0.1:1080` listen | client 측 SOCKS listener 준비 | SocksOverRDP server와 Proxifier rule을 확인한 뒤 내부 TCP 응답 검증 |
| Proxifier log에 `<INTERNAL_IP>:<PORT>` 연결 표시 | 지정 애플리케이션 트래픽이 SOCKS5 proxy를 통과 | 대상 서비스 응답과 인증을 별도 확인 |
| 내부 RDP 연결 성공 | 터널 동작 | 다음 호스트 권한/파일 확인 |
| DLL 등록 실패 | client의 권한 또는 DLL·`regsvr32` 아키텍처 불일치 | x64/x86와 상승된 관리자 권한 확인 |
| listener 없음 | plugin이 RDP 세션에 로드되지 않았거나 server가 미실행 | RDP 재접속과 server 실행 여부 확인 |
| 프록시 경유 실패 | Proxifier rule 또는 SOCKS endpoint 불일치 | 대상 앱, SOCKS host/port와 rule 순서 확인 |

## 관련 공격기법

- [[SocksOverRDP RDP 터널링]]

## 관련 도구

- [[Proxifier]]

PID·registry key·전송 파일·Proxifier rule의 기준선과 종료 순서는 [[SocksOverRDP RDP 터널링]]의 `변경 영향과 복구`를 따른다.

## 참고 링크

- [SocksOverRDP 공식 README](https://github.com/nccgroup/SocksOverRDP)
