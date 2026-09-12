---
tags:
  - 환경/windows
시작조건: ["Windows 피벗 호스트의 상승된 로컬 관리자 명령 실행", "공격 호스트에서 <PIVOT_IP>:<LISTEN_PORT>/TCP 연결 가능", "피벗 호스트에서 <INTERNAL_IP>:<PORT>/TCP 연결 가능"]
필요권한: ["Windows 피벗 호스트의 상승된 로컬 관리자 token"]
필요조건: ["공격 호스트에서 <PIVOT_IP>:<LISTEN_PORT>/TCP 연결 가능", "피벗 호스트에서 <INTERNAL_IP>:<PORT>/TCP 연결 가능", "listen 포트가 사용 중이지 않음"]
결과: ["<PIVOT_IP>:<LISTEN_PORT>에서 <INTERNAL_IP>:<PORT>로의 TCP 포워딩", "공격 호스트에서 내부 서비스까지의 TCP 경로"]
---

# Windows Netsh Portproxy 포트 포워딩

## 한 줄 판단

공격 호스트에서 `<PIVOT_IP>:<LISTEN_PORT>`에 연결할 수 있고 Windows 피벗 호스트에서 `<INTERNAL_IP>:<PORT>`에 연결할 수 있으면, 피벗의 상승된 관리자 세션에서 `netsh interface portproxy`를 실행해 두 TCP 구간을 연결한다.

## 사용할 때

- 현재 네트워크 위치: 공격 호스트에서는 `<INTERNAL_IP>:<PORT>`에 직접 연결할 수 없지만 `<PIVOT_IP>:<LISTEN_PORT>`에는 연결할 수 있고, 피벗 호스트에서는 최종 내부 포트에 연결할 수 있다.
- 명령 실행 위치: `netsh interface portproxy`는 Windows 피벗 호스트의 상승된 명령 프롬프트에서, `xfreerdp`와 최종 서비스 클라이언트는 공격 호스트에서 실행한다.
- 보유 계정·세션: 피벗 호스트에는 단순히 관리자 credential만 보유한 상태가 아니라, 해당 credential로 연 상승된 로컬 관리자 명령 실행 또는 세션이 필요하다.
- 현재 권한: Administrators 그룹 멤버십과 실제 elevated token을 구분한다. UAC로 제한된 세션에서는 rule 추가가 거부될 수 있다.
- 보유 인증 자료: portproxy는 TCP만 전달한다. `<INTERNAL_IP>`의 RDP·웹·DB 서비스에는 최종 대상에서 유효한 별도 계정이나 인증 수단이 필요하다.
- 성공 범위: 공격 호스트에서 `<PIVOT_IP>:<LISTEN_PORT>`로 연결해 `<INTERNAL_IP>:<PORT>`의 배너나 로그인 단계까지 도달한다. 최종 인증, 원격 명령 실행과 관리자 권한은 별도로 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 공격 호스트→`<PIVOT_IP>:<LISTEN_PORT>`와 피벗 호스트→`<INTERNAL_IP>:<PORT>` TCP 연결 가능 | 각 출발 호스트에서 목적 포트를 따로 확인 | listenaddress·인바운드 방화벽과 connectaddress·내부 방화벽을 분리 확인 |
| 현재 계정 또는 인증 수단 | Windows 피벗 호스트의 관리자 계정으로 연 명령 실행 또는 세션 | `whoami`, `hostname` | credential 대상 호스트와 현재 실행 컨텍스트 확인 |
| 현재 권한 | 현재 프로세스에 상승된 로컬 관리자 token | `whoami /groups`와 elevated cmd에서 rule 추가 가능 여부 확인 | UAC elevation, 로컬 Administrators 멤버십과 token 필터링 확인 |
| 공격 대상의 조건 | 피벗 호스트에서 `<INTERNAL_IP>:<PORT>`가 응답 | `Test-NetConnection` 또는 `nc` | 내부 주소·포트·route·방화벽과 대상 listener 확인 |
| 필요한 파일·목록·주소 | `<PIVOT_IP>`, 비어 있는 `<LISTEN_PORT>`, `<INTERNAL_IP>`, `<PORT>` | `netstat -ano`와 기존 `show v4tov4` 결과 확인 | 포트 충돌과 기존 portproxy rule 여부 확인 |

## 실행

### 선택 기준
| 단서 | 의미 | 다음 행동 |
|---|---|---|
| Windows 피벗 호스트가 내부 RDP에 접근 | portproxy로 RDP 중계 가능 | `listenport`를 내부 3389로 연결 |
| 관리자 권한 없음 | portproxy 설정 불가 | 다른 터널 또는 권한 상승 필요 |
| show 결과에는 있는데 접속 실패 | 방화벽/listen 주소 문제 | Windows Firewall, listenaddress 확인 |

### 절차
1. Windows 피벗 호스트에서 `<INTERNAL_IP>:<PORT>` 연결을, 공격 호스트에서 `<PIVOT_IP>:<LISTEN_PORT>`로 향하는 경로를 각각 확인한다.
2. 피벗 호스트의 상승된 명령 프롬프트에서 `netsh interface portproxy add v4tov4`로 listen 포트와 내부 대상 포트를 연결한다.
3. `show v4tov4`로 등록된 listen·connect 쌍을 확인한다.
4. 공격 호스트에서 피벗 호스트 listen 포트로 접속해 최종 서비스 응답을 확인한다.
5. 최종 서비스 계정으로 인증하고, 세션이 열리면 원격 Identity와 권한을 별도로 확인한다.
6. 사용 후에는 생성한 listen 주소와 포트가 일치하는 portproxy rule을 삭제한다.

### Windows 피벗 호스트에서 portproxy 추가

```cmd
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=<PIVOT_IP> connectport=3389 connectaddress=<INTERNAL_IP>
```

확인할 출력:

- 명령 오류 없이 rule이 추가된다. rule 등록은 설정 변경 성공이며 공격 호스트에서 listener에 연결되거나 내부 서비스가 응답한다는 뜻은 아니다.

### Windows 피벗 호스트에서 portproxy 확인

```cmd
netsh.exe interface portproxy show v4tov4
```

확인할 출력:

- `Listen on ipv4`와 `Connect to ipv4`에 의도한 IP/포트가 표시된다.

### Linux 공격 호스트에서 내부 RDP 접속

```bash
xfreerdp /v:<PIVOT_IP>:8080 /u:<USER> /p:'<PASSWORD>'
```

확인할 출력:

- 공격 호스트의 `<PIVOT_IP>:8080` 접속이 `<INTERNAL_IP>:3389`로 전달되어 RDP 인증 단계에 도달한다.
- RDP 인증 단계는 서비스 접근, GUI 세션은 `<USER>` 인증과 Remote Desktop 로그온 권한 확인이며 관리자 권한은 세션 안에서 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 피벗 호스트에서 내부 대상 포트가 직접 열림 | connect 대상까지의 TCP baseline이 유효함 | 내부 TCP 접근 | portproxy rule 추가 |
| `show v4tov4`에 의도한 listen·connect 쌍이 표시됨 | rule이 등록됨 | 포트 포워딩 구성 | 실제 listener와 방화벽 확인 |
| 공격 호스트 `nc -vz <PIVOT_IP> <LISTEN_PORT>`가 성공하고 `<INTERNAL_IP>:<PORT>`의 서비스가 응답함 | listener와 내부 전달이 모두 동작함 | 공격 호스트에서 내부 서비스 접근 | 최종 서비스 계정으로 인증하고 세션·권한 확인 |
| add 명령이 거부됨 | 관리자 권한 부족 또는 문법 오류 | rule 미생성 | elevated cmd와 주소·포트 인자 확인 |
| rule은 보이지만 공격 호스트 `nc`가 실패함 | listenaddress 또는 Windows Firewall 문제 | rule만 등록됨 | 실제 listen 상태와 인바운드 방화벽 확인 |
| listener 연결은 되지만 내부 응답이 없음 | connectaddress·connectport 또는 내부 경로 문제 | 외부 listener만 접근 가능 | 피벗 호스트에서 내부 포트를 직접 재확인 |
| RDP 인증만 실패함 | 포워딩은 성공했지만 계정·NLA·RDP 권한 문제 | RDP 인증 경로 | [[RDP 로그인과 GUI 세션]]으로 분기 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| 현재 권한 | 관리자 그룹과 elevated token | portproxy rule을 만들 수 있는 로컬 관리자 세션인지 확인 |
| 내부 hop | 피벗 호스트의 `Test-NetConnection` 또는 `nc` 성공 | 내부 대상 TCP baseline 확정 |
| listener | `show v4tov4`, 실제 listen 상태, 공격 호스트 `nc` | rule 등록과 외부 접근 가능성을 분리 |
| 최종 서비스 | 배너·로그인 화면·인증 결과 | 네트워크 포워딩과 서비스 권한을 구분 |

## 변경 영향과 복구

이번에 추가한 listen 주소와 포트가 일치하는 rule만 제거한다. 별도로 인바운드 방화벽 규칙을 만들었다면 그 규칙 이름도 기록해 함께 제거한다.

```cmd
netsh.exe interface portproxy delete v4tov4 listenport=<LISTEN_PORT> listenaddress=<PIVOT_IP>
netsh.exe interface portproxy show v4tov4
```

확인할 출력:

- 제거한 listen·connect 쌍이 `show v4tov4`에서 사라진다.
- 같은 주소와 포트의 기존 rule이 있었는지 추가 전에 확인하며, 기존 rule을 덮어썼다면 원래 값을 다시 등록한다.

## 관련 상태 라우터

- 포트 포워딩 뒤 내부 서비스 응답을 확인했으면: [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[netsh]]
- [[xfreerdp]]
- [[netcat]]
