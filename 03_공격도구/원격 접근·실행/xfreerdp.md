---
tags:
  - 환경/windows
  - 서비스/rdp
  - 기능/프로토콜접근
실행환경: ["Linux"]
필요조건: ["RDP 서버 주소", "RDP 계정과 비밀번호 또는 NTLM hash", "NTLM hash 사용 시 Restricted Admin Mode와 대상 로컬 Administrators 권한"]
결과: ["RDP GUI 세션", "파일 공유"]
---

# xfreerdp

## 도구 개요

`xfreerdp`는 Linux에서 Windows Remote Desktop Protocol(RDP) GUI 세션을 열고 해상도 조정과 로컬 드라이브 공유를 제공하는 클라이언트다. 비밀번호나 NTLM hash 기반의 그래픽 원격 접근과 파일 이동에 유용하지만 RDP 인증 성공은 로컬 관리자 권한을 뜻하지 않는다.

## 필요한 입력과 실행 환경

- 실행 환경: FreeRDP client가 설치된 Linux 호스트
- 입력: RDP 서버, 사용자명과 비밀번호 또는 NTLM hash
- 선택 입력: 도메인, 인증서 처리와 공유할 로컬 디렉터리


## 표준 사용법

```bash
xfreerdp /v:<TARGET> /d:<DOMAIN> /u:<USER> /cert:tofu
```

`/p`를 생략하면 비밀번호 prompt가 나타나므로 평문 비밀번호를 shell history·process 인자에 남기지 않을 수 있다. 로컬 계정이면 `/d:<DOMAIN>`을 제거하고 계정 범위를 명시한다.

## 대표 예시

### 비밀번호로 RDP GUI 접속

```bash
xfreerdp /v:<TARGET> /d:<DOMAIN> /u:<USER> /cert:tofu /dynamic-resolution
```

### Pass the Hash로 RDP 접속

```bash
xfreerdp /v:<TARGET> /d:<DOMAIN> /u:<USER> /pth:<NTLM_HASH> /cert:tofu
```

`/pth` 성공 후보는 대상의 Restricted Admin Mode 활성과 hash 주체의 대상 로컬 Administrators 권한이 둘 다 확인된 경우다. 일반 `Remote Desktop Users` 멤버십·`CanRDP` edge만으로는 충족되지 않는다.

### 로컬 폴더를 RDP 세션에 마운트

```bash
xfreerdp /v:<TARGET> /d:<DOMAIN> /u:<USER> /dynamic-resolution /cert:tofu /drive:<SHARE>,<LOCAL_PATH>
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `/v` | 대상 호스트 |
| `/u`, `/p` | 사용자명과 비밀번호 |
| `/d` | 도메인 지정 |
| `/pth` | NTLM hash로 RDP 인증 |
| `/dynamic-resolution` | 창 크기에 맞춰 해상도 자동 조정 |
| `/drive:name,path` | 로컬 디렉터리를 RDP 세션에 마운트 |
| `/cert:tofu` | 첫 접속의 인증서를 승인하고 이후 인증서 변경을 거부. 최초 fingerprint 확인 필요 |
| `/cert:fingerprint:<HASH>` | 사전에 확인한 인증서 fingerprint만 승인 |
| `/cert:ignore` | 인증서 검사를 전체 생략. 승인된 대상을 다른 경로로 확인한 제한 상황 외에는 기본값으로 사용하지 않음 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| RDP 데스크톱 표시 | RDP 인증과 세션 생성 성공 | 사용자 권한, 파일 접근, GUI 기반 정보 확인 |
| NLA/auth 실패 | credential 또는 RDP 권한 문제 | 도메인/로컬 계정 형식, RDP 그룹, NLA 설정 확인 |
| certificate warning | 자체 서명 또는 이름 불일치 | 대상 호스트명과 인증서 정보를 확인하고 접속 여부 판단 |
| black screen/disconnect | 그래픽, 세션 제한, 네트워크 문제 | 해상도, `/dynamic-resolution`, 네트워크·동시 세션 상태 확인 |
| 권한 거부 | RDP 로그온 권한 없음 | Remote Desktop Users, 로컬 정책과 계정 상태 확인 |
| 연결 실패 | RDP 포트 차단 또는 NLA/TLS 문제 | 포트 상태, 계정 범위, `/cert:tofu`의 기존 fingerprint과 보안 옵션 확인 |

## 관련 공격기법

- [[RDP 로그인과 GUI 세션]]
- [[Pass the Hash]]
- [[상황별 파일 전송]]

## 참고 링크

- [FreeRDP: command-line options](https://github.com/FreeRDP/FreeRDP/blob/master/client/common/cmdline.h)
- [Microsoft: Remote Credential Guard and Restricted Admin mode](https://learn.microsoft.com/en-us/windows/security/identity-protection/remote-credential-guard)
