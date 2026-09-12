---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/중계
실행환경: ["Linux"]
필요조건: ["relay 가능한 대상", "인증 요청을 수신할 수 있는 네트워크 위치"]
결과: ["세션", "명령 실행", "자격증명"]
---

# impacket-ntlmrelayx

## 도구 개요

`impacket-ntlmrelayx`는 listener로 들어온 NTLM 인증을 SMB·HTTP·AD CS 같은 대상 서비스에 즉시 중계하고, 인증 계정 권한으로 서비스별 작업을 시도한다. hash를 복구하는 도구가 아니라 relay 도구이므로 signing·channel binding과 listener 충돌 여부에 따라 기능 범위가 달라진다.

## 필요한 입력과 실행 환경

- 실행 위치: 유도된 NTLM 인증을 수신하고 relay 대상 서비스에 접근 가능한 Linux 호스트
- 필요한 입력: 단일 target 또는 target list와 목적별 relay 옵션
- 전제: relay 프로토콜의 signing, EPA/channel binding과 relay 계정 권한을 사전에 확인한다.
- 리스너 조건: 같은 포트를 점유하는 Responder SMB/HTTP server와 충돌하지 않게 구성한다.


## 표준 사용법

```bash
impacket-ntlmrelayx -t <target_url_or_host> [options]
```

## 대표 예시

### SMB relay로 기본 SAM dump 시도

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET>
```

### AD CS Web Enrollment로 인증서 relay

```bash
impacket-ntlmrelayx -t http://<TARGET>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

### relay 성공 후 명령 실행

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <INTERNAL_TARGET> -c 'whoami'
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-t` | 단일 relay 대상 지정 |
| `-tf` | 대상 목록 파일 지정 |
| `-smb2support` | SMB2 지원 |
| `--no-http-server` | HTTP listener 비활성화 |
| `--adcs` | AD CS 공격 모드 |
| `--template` | 요청할 인증서 template 지정 |
| `-c` | relay 성공 후 실행할 명령 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| incoming connection / authenticated | NTLM 인증 유도와 relay 시도 발생 | relay 대상 서비스와 권한 결과 확인 |
| dump/add user/certificate 성공 | relay 후 영향 발생 | 획득 credential/cert/hash를 별도 도구로 검증 |
| 머신 계정 `SUCCEED`와 `GOT CERTIFICATE` | AD CS relay와 certificate 발급 성공 | certificate 주체·template 확인 후 PKINIT |
| signing required / relay failed | 대상 서비스가 relay 조건을 막음 | SMB signing, EPA, LDAP signing/channel binding 확인 |
| 연결은 오지만 영향 없음 | 권한 부족 또는 대상 선택 문제 | relay target, 사용자 권한, 프로토콜별 조건 확인 |
| 인증이 들어오지 않음 | 유도 트래픽 부재 | Responder/WPAD/강제 인증 트리거 확인 |
| 인증은 되지만 결과 없음 | 모듈/옵션 부적합 | `--dump`, `--delegate-access`, AD CS 옵션 등 목적별 옵션 확인 |

## 관련 공격기법

- [[NTLM Relay 조건 검토]]
- [[AD CS ESC8 NTLM Relay]]
