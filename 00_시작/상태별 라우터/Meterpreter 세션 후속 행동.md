---
tags:
  - 환경/windows
  - 환경/linux
  - 기능/세션관리
시작상태: ["Meterpreter 세션 확보"]
목표: ["세션 상태 확인", "세션 안정화", "내부망 경로 확인", "플랫폼별 후속 공격 선택"]
현재계정: ["Meterpreter 세션에서 확인할 대상 사용자"]
현재 가능한 행위: ["대상 호스트에서 Meterpreter 명령 실행"]
필요권한: ["현재 Meterpreter 세션의 사용자 권한"]
필요정보: ["Meterpreter 세션", "대상 운영체제와 아키텍처"]
네트워크위치: ["공격 호스트와 연결된 대상 호스트"]
---

# Meterpreter 세션 후속 행동

## 상태 라우터 개요

Meterpreter 세션을 확보하면 대상 운영체제, 현재 사용자와 권한, 프로세스 아키텍처와 추가 네트워크 경로를 먼저 확인하고 파일 반입·피벗·권한 상승·자격 증명 수집 중 현재 가능한 공격기법을 고른다.

## 적용 조건

| 상태 축 | 조건 |
|---|---|
| 대상 플랫폼 | Windows 또는 Linux |
| 현재 계정 | `getuid`로 확인할 대상 호스트의 현재 사용자 |
| 현재 가능한 행위 | 대상 호스트에서 Meterpreter 명령 실행 |
| 현재 권한 | `getprivs`와 운영체제별 권한 확인 결과 |
| 보유 정보 | Meterpreter 세션 ID, 대상 운영체제와 아키텍처 |
| 네트워크 위치 | 공격 호스트와 연결된 대상 호스트 |
| 목표 | 세션 상태를 확정하고 플랫폼·권한·네트워크 조건에 맞는 후속 공격 선택 |

먼저 Meterpreter 프롬프트에서 `getuid`, `getprivs`, `sysinfo`, `pwd`로 현재 사용자·privilege·운영체제·architecture·작업 디렉터리를 수동 확인한다. 명령이 반복 실행되지 않으면 후속 모듈보다 세션 안정성과 연결 상태를 먼저 확인한다.

## 판단 경로

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| 세션이 불안정하거나 현재 프로세스가 종료될 가능성이 큼 | [[Meterpreter 프로세스 이동과 세션 안정화]] | 다른 프로세스에서 유지되는 Meterpreter 세션과 이동 뒤 사용자·권한 | 이 상태 라우터 재평가 | 프로세스 아키텍처, 현재 권한, 보호 프로세스 여부와 기존 세션 생존 여부 확인 |
| Windows Meterpreter 세션에서 대상에 도구나 스크립트를 반입해야 하고 쓰기 가능한 경로가 있음 | [[Meterpreter upload로 Windows 파일 반입]] | Windows 대상의 파일 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] | 활성 세션, 로컬 원본 경로와 대상 쓰기 경로를 확인 |
| 대상 호스트의 내부 NIC와 CIDR은 확인했지만 응답 호스트를 모름 | [[ICMP 기반 내부 호스트 확인]]의 Meterpreter `ping_sweep` | ICMP 응답 내부 호스트 후보 | [[내부망 경로 확보 후 피벗 구성]] | 세션 호스트의 route, 단일 ICMP 응답, 대상 방화벽과 첫 sweep의 ARP cache 지연 확인 |
| 대상 호스트에 추가 NIC·route·내부 대역이 있음 | [[Meterpreter 라우팅과 포트 포워딩]] | 내부 route·SOCKS·port forward | [[내부망 경로 확보 후 피벗 구성]] | 추가 대역의 route, 대상 서비스 포트 도달성과 Meterpreter 세션 유지 여부 확인 |
| Windows 일반 사용자 Meterpreter 세션이고 로컬 권한 상승 후보를 아직 열거하지 않음 | [[Windows 권한 상승 열거]] | Windows 권한 상승 후보 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] | 현재 사용자, token·그룹, 운영체제와 실행 가능한 열거 명령을 확인 |
| Linux 일반 사용자 Meterpreter 세션이고 로컬 권한 상승 후보를 아직 열거하지 않음 | [[Linux 권한 상승 열거]] | Linux 권한 상승 후보 | [[Linux 셸 확보 후 초기 열거와 권한 상승]] | 현재 사용자와 그룹, 운영체제와 실행 가능한 열거 명령을 확인 |
| Windows `ps`에서 현재 사용자와 다른 계정의 접근 가능한 프로세스가 확인됨 | [[Meterpreter 프로세스 토큰 탈취]] | 다른 Windows 사용자의 token이 적용된 Meterpreter 세션 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] | 대상 PID·사용자·architecture, 현재 privilege와 보호 프로세스 여부 확인 |
| Windows 로컬 관리자 또는 SYSTEM이며 로컬 계정 hash가 필요함 | [[Windows SAM SECURITY SYSTEM 덤프]] | 로컬 계정 NTLM hash와 LSA secret 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]] | 관리자 그룹 표시와 실제 토큰, SAM·SECURITY·SYSTEM 접근 권한 확인 |
| Windows 로그온 프로세스 메모리에 남은 자격 증명이 필요하고 필요한 권한을 보유함 | [[LSASS 메모리 덤프]] | NTLM hash·Kerberos ticket·평문 비밀번호 후보가 포함될 수 있는 덤프 | [[확보한 자격 증명으로 원격 접근 경로 선택]] | SYSTEM·SeDebugPrivilege, PPL·EDR 보호와 대상 프로세스 접근 권한 확인 |

`upload`, `download`, `steal_token`, `hashdump`와 `kiwi`의 문법과 도구 고유 오류는 [[meterpreter]], [[metasploit]], [[mimikatz]]에서 확인한다. 프로세스 이동은 [[Meterpreter 프로세스 이동과 세션 안정화]]에서 실행하고, 성공 뒤 `getuid`와 `getprivs`를 다시 확인한다.

## 상태 재평가

- `getuid`·`sysinfo`가 반복 실행되고 세션이 유지되는지 확인한 뒤 후속 공격을 선택한다.
- `hashdump` 결과는 SAM의 로컬 계정 hash이며 LSASS에 남은 로그온 자료와 구분한다.
- 파일 전송 성공 메시지와 대상 경로의 사용 가능 여부를 확인한다. 전송 손상이 의심되고 비교할 기준값이 있을 때만 송신본과 수신본을 비교한다.
- 현재 계정·권한, 대상 플랫폼, 세션 안정성 또는 내부망 경로가 바뀌면 이 상태 라우터를 다시 평가한다.

## 관련 노트

- [[meterpreter]]
- [[metasploit]]

## 관련 실전 시나리오

- Windows Web Shell 세션을 안정화하고 AD·피벗·SMB 원격 실행으로 이어갈 때: [[Windows Web Shell에서 내부망 SMB 관리자 명령 실행까지]]
