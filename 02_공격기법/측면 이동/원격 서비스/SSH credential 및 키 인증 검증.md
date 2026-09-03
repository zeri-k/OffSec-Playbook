---
tags:
  - 서비스/ssh
시작조건: ["SSH 서비스 식별", "비밀번호 또는 private key 확보"]
필요조건: ["비밀번호 또는 개인키","SSH 접근 가능"]
결과: ["세션", "파일 전송", "터널링"]
---

# SSH credential 및 키 인증 검증

## 한 줄 판단

현재 명령 실행 위치에서 대상 SSH 포트에 연결할 수 있고 사용자명과 비밀번호 또는 개인키가 있다면, 한 계정씩 실제 SSH 인증을 확인한다. 성공하면 그 계정의 셸·파일 전송·포트 포워딩 범위를 얻으며 관리자 권한은 로그인 후 별도로 확인한다.

## 사용할 때

- 22/TCP가 열려 있고 password 또는 publickey 인증이 허용될 때.
- FTP/SMB/NFS/웹/메일에서 계정, 비밀번호, private key가 나왔을 때.
- 초기 접근 이후 안정적인 셸과 터널링 경로가 필요할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 인증 방식 | `ssh -v` | `password`, `publickey` 확인 |
| credential 후보 | 수집 파일/스프레이/크래킹 | 사용자명과 비밀번호/키 |
| 키 권한 | `chmod 600 <KEY>` | SSH가 private key를 거부하지 않음 |

## 실행

### 인증 방식 선택

| 서버가 허용하는 방식 | 현재 보유할 입력 | 실행 결과 | 실패 시 먼저 확인할 항목 |
|---|---|---|---|
| `password` | `<USER>`와 그 계정의 평문 비밀번호 | 해당 사용자의 SSH 셸 | 사용자명·비밀번호, 계정 잠금, `PasswordAuthentication` 정책 |
| `publickey` | `<USER>`, 대응하는 `<KEY>`, 암호화된 키라면 passphrase | 해당 공개키가 등록된 사용자의 SSH 셸 | 키 파일 형식·권한, passphrase, 대상 계정의 `authorized_keys`와 알고리즘 정책 |

비밀번호와 개인키는 같은 credential이 아니다. 서버가 광고한 방식과 현재 보유한 입력이 일치하는 행 하나를 선택하며, 두 방식을 한 번의 성공으로 묶어 기록하지 않는다.

1. `ssh -v`로 허용 인증 방식을 확인한다.
2. 비밀번호와 키 인증을 분리해 검증한다.
3. 로그인 성공 후 `id`, `sudo -l`, 홈 디렉터리, 내부 네트워크를 확인한다.
4. 필요하면 `scp`, `sftp`, 포트 포워딩으로 후속 작업을 안정화한다.

### 명령과 확인할 출력

#### 서버가 허용하는 인증 방식 확인

```bash
ssh -v <USER>@<TARGET>
ssh -o PreferredAuthentications=password -v <USER>@<TARGET>
```

확인할 출력:

- `Authentications that can continue`에 표시되는 `password`, `publickey`.
- `Permission denied (publickey)`이면 SSH 포트 연결은 성공했지만 비밀번호 인증은 제공되지 않았거나 유효한 키가 선택되지 않은 상태다.

#### 비밀번호 인증

실행 위치: 대상 SSH 포트에 연결할 수 있는 현재 명령 실행 호스트.

일반 접속 명령은 로컬 agent의 키를 먼저 사용할 수 있다.

```bash
ssh <USER>@<TARGET>
```

비밀번호 자체를 검증할 때는 public key fallback을 끈다.

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -v <USER>@<TARGET>
```

확인할 출력:

- verbose 출력에서 비밀번호 인증 성공이 확인되고 shell prompt가 열린 뒤 `whoami`, `id`가 `<USER>`의 계정과 일치한다.
- `Permission denied`이면 SSH 서비스 접근 성공과 계정 인증 실패를 구분하고 사용자명·비밀번호·잠금 정책을 확인한다.

#### 개인키 인증

실행 위치: `<KEY>`를 보관하고 대상 SSH 포트에 연결할 수 있는 현재 명령 실행 호스트.

```bash
chmod 600 <KEY>
ssh -i <KEY> <USER>@<TARGET>
```

지정한 개인키 자체를 검증할 때는 다른 agent key와 비밀번호 fallback을 끈다.

```bash
ssh -o IdentitiesOnly=yes -o PreferredAuthentications=publickey -o PasswordAuthentication=no -i <KEY> -v <USER>@<TARGET>
```

확인할 출력:

- verbose 출력에서 지정한 key의 publickey 인증 성공을 확인하고, passphrase가 필요한 키라면 해제한 뒤 shell prompt와 `whoami`, `id`가 `<USER>`와 일치한다.
- `UNPROTECTED PRIVATE KEY FILE`이면 로컬 키 파일 권한을, `invalid format`이면 키 형식을, `Permission denied (publickey)`이면 대상 계정과 공개키 등록·알고리즘 정책을 확인한다.
- 개인키가 passphrase를 요구하고 값을 모르면 [[보호된 파일 및 아카이브 크래킹]]에서 키 형식을 추출하고 오프라인 복구한 뒤 같은 명령으로 다시 검증한다.

#### 검증 후 기본 확인

```bash
id
hostname
sudo -l
ip addr
```

확인할 출력:

- 현재 권한, sudo 가능 명령, 내부 인터페이스.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| SSH shell이 열리고 `id`·`hostname`이 확인됨 | SSH 인증과 셸 권한 확인 | SSH 세션 | [[Linux 셸 확보 후 초기 열거와 권한 상승]] |
| private key 또는 재사용 비밀번호로 다른 호스트 접근이 가능함 | 동일 인증 정보의 재사용 확인 | 다른 호스트 SSH 접근 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 계정 범위 재평가 |
| 피벗 호스트에서 추가 인터페이스·route가 확인됨 | SSH 세션이 새 네트워크 위치를 제공함 | 피벗 호스트 세션 | [[내부망 경로 확보 후 피벗 구성]] |
| `ssh -L`, `-D`, `-R` 또는 sshuttle로 내부 서비스가 응답함 | SSH 경유 TCP 경로 확인 | 내부 서비스 접근 | [[SSH 포트 포워딩 피벗팅]] 또는 [[sshuttle SSH 피벗팅]] 검증 후 [[내부망 경로 확보 후 피벗 구성]] |
| key 권한 오류 | private key 파일 권한 과다 | 시작 상태 유지 | `chmod 600 <KEY>` |
| 개인키 passphrase를 모름 | 키 파일은 확보했지만 아직 인증에 사용할 수 없음 | 암호화된 키 | [[보호된 파일 및 아카이브 크래킹]] |
| publickey만 허용 | 비밀번호 로그인 비활성 | 시작 상태 유지 | NFS/SMB/웹에서 private key 찾기 |
| 로그인은 되지만 제한 shell | rbash/scp-only | 시작 상태 유지 | TTY/환경 변수/허용 명령 확인 |

## 확인할 출력과 권한

- 판정 기준: `Authentications that can continue`, shell prompt, `id`, `sudo -l`을 확인해 인증과 실제 사용자 권한을 구분한다.
- 권한 구분: 인증 전·익명·유효 계정 상태를 구분하고, 서비스 응답만으로 실제 권한을 추정하지 않는다.

## 후속 공격 연결

- 셸 확보 후: [[sudo 권한 오남용]], [[Cron 권한 상승]], [[상황별 파일 전송]]
- 암호화된 개인키 복구: [[보호된 파일 및 아카이브 크래킹]]
- 대상 계정의 `authorized_keys`에 쓸 수 있고 재접속 경로가 필요함: [[SSH Authorized Keys 등록]]
- 단일 포트·SOCKS·reverse callback: [[SSH 포트 포워딩 피벗팅]]
- 내부 CIDR TCP 라우팅: [[sshuttle SSH 피벗팅]]

## 관련 상태 라우터

- SSH 셸 확보: [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- credential의 다른 서비스 재사용: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- 추가 내부망 또는 forwarding 경로: [[내부망 경로 확보 후 피벗 구성]]

## 관련 서비스

- [[22_SSH]]

## 관련 도구

- [[ssh]]
- [[sshpass]]
- [[hydra]]
