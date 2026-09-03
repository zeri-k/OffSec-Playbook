---
tags:
  - 서비스/ssh
대표포트:
  - "T:22"
서비스:
  - SSH
---

# 22_SSH

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:22`의 Secure Shell(SSH)에 연결할 수 있고, 아직 유효한 대상 사용자·비밀번호·개인키나 셸 권한은 확인하지 않은 상태에서 시작한다. 서버가 허용한 password·publickey·keyboard-interactive 인증과 보유한 사용자 이름·비밀번호·개인키의 조합을 먼저 맞춘다.

성공하면 인증된 사용자 권한의 원격 셸 또는 파일 전송 채널을 얻는다. `Permission denied`이면 서버가 제시한 인증 방식, 사용자 이름과 key 주체를, timeout이면 네트워크·source 제한을, 로그인 후 제한 셸이면 해당 계정의 shell·command 제한을 확인한다. OpenSSH 배너는 공개 취약점 조사 단서일 뿐이며 로그인 성공도 관리자 권한을 뜻하지 않는다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | 허용 인증 방식 | `ssh -v <USER>@<TARGET>` | `Authentications that can continue`에서 password, publickey, keyboard-interactive를 구분한다. |
| 2 | 확보한 개인키와 사용자명 조합 | `ssh -i <KEY_FILE> <USER>@<TARGET>` | 키 인증 성공 여부와 키 암호화 여부를 구분한다. |
| 3 | 확보한 credential의 단일 검증 | `ssh -o PreferredAuthentications=password -v <USER>@<TARGET>` | 비밀번호 인증 경로와 알려진 credential의 결과를 확인한다. |
| 4 | Password Spraying 필요성 | [[원격 비밀번호 공격]]의 잠금 정책·사용자 형식·시도 간격 확인 | 잠금 가능성과 시도 간격을 확인한 뒤 마지막 단계로 제한한다. |

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| 비밀번호 인증 허용 또는 확보한 사용자 이름·비밀번호 | [[SSH credential 및 키 인증 검증]] | `ssh`, `sshpass`, `hydra` | 인증된 사용자 권한의 SSH 셸 또는 명시적 거부 |
| `publickey` 허용 또는 private key 확보 | [[SSH credential 및 키 인증 검증]] | `ssh` | 해당 키 주체의 SSH 셸 |
| 암호화된 private key | [[보호된 파일 및 아카이브 크래킹]] | `john`, `hashcat` | 검증 가능한 개인키 passphrase 후보 |
| 사용자 이름 후보만 확보 | [[원격 비밀번호 공격]] | `ssh`, `hydra` | 잠금 정책을 반영해 검증할 사용자 이름·비밀번호 조합 |
| 오래된 OpenSSH와 OS·패치 단서 | [[Public Exploit 검토와 검증]] | `searchsploit` | 실제 배포판에 적용 가능한 취약점 후보 |
| 로그인 성공 후 파일 전송 필요 | [[상황별 파일 전송]] | `scp`, `sftp` | 로그인 사용자 권한 범위의 파일 송수신 |
| 로그인 호스트에서만 내부 대상에 도달 | [[SSH 포트 포워딩 피벗팅]] | `ssh` | SSH 세션을 경유한 제한된 내부 TCP 접근 |

## 서비스 고유 주의 사항

- `Permission denied (publickey)`는 사용자 부재가 아니라 키 인증만 허용하는 구성일 수 있다.
- private key 파일 권한은 `chmod 600 <KEY_FILE>`로 맞춘다.
- 사용자명이 로컬 계정인지 디렉터리·서비스 계정인지 구분하고, 계정 잠금 정책이 있으면 반복 인증을 중단한다.
- SSH 포워딩은 로그인 성공과 서버의 포워딩 허용이 모두 필요한 별도 네트워크 접근 상태다.
