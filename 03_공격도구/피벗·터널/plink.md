---
tags:
  - 서비스/ssh
  - 기능/피벗
실행환경: ["Windows"]
필요조건: ["피벗 호스트 SSH 계정과 인증 수단", "검증할 SSH host key fingerprint", "SSH 서버의 TCP forwarding 허용"]
결과: ["SSH 세션", "SOCKS 프록시", "포트 포워딩"]
---

# plink.exe

## 도구 개요

Plink는 Windows에서 SSH 연결과 동적 SOCKS 포워딩·로컬 단일 포트 포워딩을 명령줄로 만드는 클라이언트다. Windows 피벗 호스트에서 SOCKS 프록시를 열거나 내부 단일 서비스를 로컬 포트로 가져올 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 환경: `plink.exe`를 실행할 수 있는 Windows 호스트
- SSH 입력: 피벗 호스트 주소와 포트, 사용자명, PuTTY private key 또는 Pageant의 인증 키, 검증된 SSH host key fingerprint
- 동적 포워딩 입력: 열어 둘 로컬 SOCKS 포트
- 로컬 포워딩 입력: 로컬 listen 포트와 내부 목적지 IP·포트

## 표준 사용법

```cmd
plink.exe -ssh -N -D 127.0.0.1:<LOCAL_SOCKS_PORT> -hostkey "<SSH_HOST_KEY_FINGERPRINT>" -i "<PPK_PATH>" <USER>@<PIVOT_IP>
plink.exe -ssh -N -L 127.0.0.1:<LOCAL_PORT>:<INTERNAL_IP>:<INTERNAL_PORT> -hostkey "<SSH_HOST_KEY_FINGERPRINT>" -i "<PPK_PATH>" <USER>@<PIVOT_IP>
```

`-D`는 SOCKS 프록시를 만들고, `-L`은 특정 내부 서비스 하나를 로컬 포트로 당긴다.

## 대표 예시

### SOCKS dynamic forwarding

```cmd
plink.exe -ssh -N -D 127.0.0.1:9050 -hostkey "<SSH_HOST_KEY_FINGERPRINT>" -i "<PPK_PATH>" <USER>@<PIVOT_IP>
```

확인할 출력:

- Windows 로컬 `127.0.0.1:9050`에서 SOCKS proxy가 열리고 Plink 프로세스가 유지된다. listener만으로 내부 서비스 접근은 확인되지 않는다.

### 내부 RDP 단일 포트 포워딩

```cmd
plink.exe -ssh -N -L 127.0.0.1:13389:<INTERNAL_IP>:3389 -hostkey "<SSH_HOST_KEY_FINGERPRINT>" -i "<PPK_PATH>" <USER>@<PIVOT_IP>
```

확인할 출력:

- `127.0.0.1:13389` 접속이 내부 RDP `3389`로 전달된다.

## 주요 옵션

| 옵션 | 의미 | 자주 쓰는 상황 |
|---|---|---|
| `-ssh` | SSH 모드 지정 | SSH 터널 생성 |
| `-N` | 원격 shell·command 없이 SSH-2 forwarding만 유지 | 전용 터널 process |
| `-D <PORT>` | SOCKS dynamic forwarding | 여러 내부 TCP 서비스 접근 |
| `-L <LPORT>:<RHOST>:<RPORT>` | 로컬 포트 포워딩 | RDP/DB/웹 단일 포트 접근 |
| `-l <USER>` | 사용자 지정 | 사용자명 분리 입력 |
| `-i <KEY>` | private key 지정 | 키 기반 인증 |
| `-hostkey <FINGERPRINT>` | 허용할 서버 host key 지정 | registry 신뢰 상태를 바꾸지 않고 승인된 fingerprint 고정 |
| `-batch` | 대화형 prompt 대신 오류로 종료 | host key와 비대화형 인증을 미리 준비한 자동 실행 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| SSH 세션 유지 | 터널 생성 가능 | 로컬 포트 listen 확인 |
| 로컬 SOCKS 포트 listen | `-D` 성공 | [[Proxifier]] 또는 ProxyChains로 내부 접근 |
| 로컬 포트로 서비스 응답 | `-L` 성공 | 해당 서비스 클라이언트 실행 |
| SSH 로그인 실패 | 계정·키·host key 또는 SSH 서버 포트 문제 | `-v` 출력과 `-l`, `-i`, `-P`, 승인된 fingerprint 확인 |
| `The host key is not cached` 또는 host key 불일치 | 서버 신뢰 기준이 없거나 예상 fingerprint와 다름 | 임의 수락하지 말고 승인된 fingerprint와 `-hostkey` 값 대조 |
| `-D` 후 RDP가 직접 안 됨 | SOCKS 프록시와 단일 포워딩 혼동 | RDP 하나는 `-L 13389:<TARGET>:3389`로 검증 |
| SOCKS client 실패 | proxy 설정 오류 | SOCKS host/port, SOCKS4/5 지원 확인 |
| 내부 서비스 실패 | 피벗 호스트에서 내부 대상 접근 불가 | 피벗 호스트에서 `nc -vz <INTERNAL_IP> <PORT>` |

## 관련 공격기법

- [[SSH 포트 포워딩 피벗팅]]

## 관련 도구

- [[Proxifier]]

실행 PID·listener·전송 파일의 기준선과 exact 종료는 [[SSH 포트 포워딩 피벗팅]]의 Windows Plink 절차를 따른다.

## 참고 링크

- [PuTTY 0.84 Plink manual](https://the.earth.li/~sgtatham/putty/0.84/htmldoc/Chapter7.html)
- [PuTTY 0.84 command-line options](https://the.earth.li/~sgtatham/putty/0.84/htmldoc/Chapter3.html#using-cmdline)
