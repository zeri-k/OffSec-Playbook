---
tags:
  - 환경/unix
  - 서비스/r-services
시작조건: ["R-Services 식별", "사용자 또는 trust 후보 확보"]
필요조건: ["R-Services 접근 가능", "신뢰 관계 또는 유효 사용자 인증 정보", "사용자 후보"]
결과: ["정보", "세션", "명령 실행"]
---

# R-Services trust 기반 원격 접근

## 한 줄 판단

현재 명령 실행 위치에서 대상의 `rlogin`·`rsh`·`rexec` 서비스에 연결할 수 있고 사용자·source 호스트 trust 후보 또는 R-Services 비밀번호가 있다면, trust 기반 셸·trust 기반 단일 명령·비밀번호 기반 명령 실행을 각각 확인한다.

## 사용할 때

- 512/513/514가 열려 있고 오래된 Unix 원격 서비스가 보일 때.
- `rusers` 또는 이미 수신 중인 `rwhod` 정보에서 사용자/호스트 관계가 노출될 때.
- `.rhosts` 또는 `/etc/hosts.equiv`에 과도한 trust 설정이 의심될 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 서비스 접근 | `nmap -sV -p512,513,514 <TARGET>` | rexec/rlogin/rsh 식별 |
| 사용자 후보 | `rusers`, 로컬 `rwho`, 파일 단서 | 로그인할 사용자명 확보 |
| trust 기반 입력 | `.rhosts`, `hosts.equiv`, source 호스트 정보 | 특정 호스트/사용자 또는 `+ +` |
| 비밀번호 기반 입력 | 수집한 R-Services 사용자명과 비밀번호 | `rexec` 인증에 사용할 한 계정 |

## 실행

### 방식 선택

| 서비스 | 인증에 사용하는 입력 | 대상에서 얻는 결과 | 성공 판단 |
|---|---|---|---|
| `rlogin` | 대상 사용자명과 `.rhosts`·`hosts.equiv`가 신뢰하는 source 호스트 | 대화형 원격 셸 | 비밀번호 없이 셸이 열리고 `id`·`hostname`이 반환됨 |
| `rsh` | 대상 사용자명과 신뢰받는 source 호스트 | 지정한 단일 원격 명령의 출력 | 요청한 `id`의 출력이 반환됨 |
| `rexec` | 대상 사용자명과 평문 비밀번호 | 비밀번호 인증 후 단일 원격 명령의 출력 | 인증 오류 없이 요청한 `id`의 출력이 반환됨 |

`rlogin`과 `rsh`는 source 호스트·사용자 trust를 검증하지만, `rexec`는 명령에 제공한 비밀번호를 검증한다. `rexec` 성공을 `.rhosts` trust 성공으로 기록하지 않는다.

1. rlogin/rsh/rexec 서비스가 실제 응답하는지 확인한다.
2. 대상에 `rusersd`가 노출된 경우 `rusers`로 사용자명과 접속 호스트 후보를 모은다.
3. trust 파일 단서가 있으면 등록된 source 위치와 사용자 조합을 맞춘다.
4. `rlogin`·`rsh`의 trust 검증과 `rexec` 비밀번호 인증을 분리한다.
5. 로그인/명령 실행 성공 후 권한과 내부 네트워크를 확인한다.

### 명령과 확인할 출력

#### 서비스와 사용자 정보

```bash
nmap -sV -p512,513,514 <TARGET>
rusers -al <TARGET>
```

확인할 출력:

- 로그인 사용자, 호스트명, TTY, 서비스명.
- `rwho`는 현재 호스트가 `rwhod`로 이미 수신한 정보만 표시하며 임의 대상 질의 명령으로 사용하지 않는다.

#### `rlogin` trust 기반 셸

```bash
rlogin -l <USER> <TARGET>
```

확인할 출력:

- 비밀번호 입력 없이 원격 셸이 열리고 `id`, `hostname`이 반환된다.
- `permission denied`이면 대상 사용자명, 현재 source IP·호스트명의 정방향·역방향 해석, trust 파일의 허용 조합을 확인한다.

#### `rsh` trust 기반 단일 명령

```bash
rsh -l <USER> <TARGET> id
```

확인할 출력:

- 원격 대상에서 실행된 `uid=...` 결과.
- 연결은 되지만 명령 출력이 없으면 trust 거부, 원격 명령 경로와 stderr를 확인하며 대화형 셸 확보로 기록하지 않는다.

#### `rexec` 비밀번호 기반 단일 명령

```bash
rexec -l <USER> -p '<PASSWORD>' <TARGET> id
```

확인할 출력:

- 비밀번호 인증 뒤 원격 대상의 `uid=...` 결과.
- 인증이 거부되면 trust 설정이 아니라 `<USER>`·`<PASSWORD>` 조합과 `rexec` 서비스 정책을 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 비밀번호 없이 rlogin shell이 열림 | host/user trust 적용 확인 | Unix 원격 셸 | `id`, `hostname` 확인 후 [[Linux 셸 확보 후 초기 열거와 권한 상승]] |
| rsh로 원격 `id`가 반환됨 | trust 기반 비대화형 명령 실행 확인 | Unix 원격 명령 실행 | 필요하면 [[Reverse Shell 획득]], 결과는 [[Linux 셸 확보 후 초기 열거와 권한 상승]] |
| rexec 비밀번호 인증으로 명령 출력이 반환됨 | 유효한 R-Services credential 확인 | 서비스 인증과 명령 실행 | [[확보한 자격 증명으로 원격 접근 경로 선택]] |
| 신뢰된 사용자명과 호스트명 조합이 확인됨 | 재사용 가능한 trust 관계 확인 | 사용자·호스트 관계 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 SSH 등 별도 인증 검증 |
| permission denied | trust 조합 불일치 | 시작 상태 유지 | source IP/호스트명, 사용자명, DNS reverse 확인 |
| tcpwrapped | hosts.allow/deny 제한 | 시작 상태 유지 | 허용 대역, 피벗 위치 확인 |
| 서비스만 열림 | rwho/rusers 제한 | 시작 상태 유지 | 다른 계정 후보, 파일 공유/홈 디렉터리 확인 |

## 확인할 출력과 권한

- 판정 기준: 사용자·source 호스트 조합과 shell 또는 `uid=` 출력을 확인해 trust 기반 접근 권한을 판정한다.
- 권한 구분: 인증 전·익명·유효 계정 상태를 구분하고, 서비스 응답만으로 실제 권한을 추정하지 않는다.

## 후속 공격 연결

- shell 확보: [[sudo 권한 오남용]], [[Cron 권한 상승]]
- 사용자명 후보: [[원격 비밀번호 공격]]
- SSH 전환: [[SSH credential 및 키 인증 검증]]

## 관련 상태 라우터

- Unix 셸 또는 명령 실행: [[Linux 셸 확보 후 초기 열거와 권한 상승]]
- 비밀번호 또는 사용자 후보: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- 새 내부 인터페이스·route 확인: [[내부망 경로 확보 후 피벗 구성]]

## 관련 서비스

- [[512_513_514_R-Services]]

## 관련 도구

- [[nmap]]
- [[R-Services 클라이언트]]
