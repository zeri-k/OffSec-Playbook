---
tags:
  - 환경/unix
  - 서비스/r-services
대표포트:
  - "T:512"
  - "T:513"
  - "T:514"
서비스:
  - R-Services
  - rexec
  - rlogin
  - rsh
---

# R-Services 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Berkeley R-Services인 TCP 512/rexec, 513/rlogin 또는 514/rsh에 연결할 수 있고, 아직 신뢰받는 source 호스트·사용자나 유효한 비밀번호는 확인하지 않은 상태에서 시작한다. rlogin·rsh의 `/etc/hosts.equiv`·`.rhosts` trust와 rexec의 사용자 이름·비밀번호 인증을 분리하며, 사용자·호스트 정보 노출과 실제 원격 로그인·명령 실행도 별도 상태로 본다.

**첫 화면 상태:** 지금 가능한 일은 rlogin·rsh trust 또는 rexec 계정 인증을 검증하는 것이다. 성공하면 지정한 원격 사용자 권한의 셸 또는 명령 출력을 얻는다. rlogin·rsh 거부 시 source IP·정방향/역방향 이름, 로컬 사용자 이름, privileged source port와 trust 항목을 확인하고, rexec 거부 시 사용자 이름·비밀번호를 확인한다. 전통적인 r-command의 privileged source port 사용에 필요한 로컬 root·setuid 권한은 대상 호스트 권한이 아니다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | rlogin 접근 | `rlogin -l <USER> <TARGET>` | 사용자명과 trust 조건으로 로그인되는지 확인한다. |
| 2 | 원격 명령 | `rsh -l <USER> <TARGET> id` | 원격 사용자 context와 실제 명령 출력을 확인한다. |
| 3 | rexec 계정 인증 | `rexec <TARGET> -l <USER> id` | trust 파일이 아니라 해당 사용자 비밀번호로 명령이 실행되는지 확인한다. |
| 4 | 별도 rusersd 정보 | `rusers -al <TARGET>` | 로그인 사용자명, 호스트명과 TTY 노출 여부를 확인한다. |
| 5 | 다른 경로에서 읽은 trust 파일 | `/etc/hosts.equiv`, `.rhosts` | 신뢰 호스트·사용자와 `+ +` 같은 과도한 구성을 확인한다. |

**출력 해석 경계:** `rusers` 출력은 로그인 사용자·TTY 노출만 확정하며 계정 비밀번호나 원격 접근 권한은 확정하지 않는다. `.rhosts`와 `hosts.equiv`는 대상의 신뢰 규칙 후보일 뿐 원본 IP·역방향 이름 해석·사용자 조합이 충족됐는지는 말해 주지 않는다. `rsh ... id`의 실제 출력만 해당 사용자로 명령이 실행됐음을 확정한다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| `.rhosts`, `hosts.equiv` 또는 trust 관계 | [[R-Services trust 기반 원격 접근]] | `rlogin`, `rsh` | 신뢰 사용자·호스트 조합의 인증 우회 가능성 |
| 비밀번호 없이 rlogin·rsh 접근 | [[R-Services trust 기반 원격 접근]] | `rlogin`, `rsh` | trust가 허용한 사용자 권한의 원격 셸 또는 명령 실행 |
| rexec 사용자 이름·비밀번호 확보 | [[R-Services trust 기반 원격 접근]] | `rexec` | 해당 사용자 권한의 원격 명령 실행 |
| `rusersd` 사용자 정보 | [[R-Services trust 기반 원격 접근]] | `rusers` | 사용자명·호스트 관계와 trust 검증 후보 |

## 서비스 고유 주의 사항

- tcpwrapped와 source IP 제한이 있을 수 있고 rlogin·rsh trust는 사용자명·호스트명·IP 조건에 민감하다.
- 통신이 암호화되지 않아 네트워크 구간에서 credential이 노출될 수 있다.
- 사용자 정보 노출, trust 설정, rexec 비밀번호 인증, 인증 우회와 실제 원격 명령 실행은 서로 다른 상태다.
