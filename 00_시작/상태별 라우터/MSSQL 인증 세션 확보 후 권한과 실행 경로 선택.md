---
tags:
  - 환경/windows
  - 서비스/mssql
시작상태: ["MSSQL 인증 성공", "MSSQL query 실행 가능", "MSSQL xp_cmdshell 명령 실행"]
목표: ["데이터베이스 권한 확인", "운영체제 명령 실행", "SQL Server 호스트 권한 상승", "내부 SQL Server 이동"]
현재계정: ["MSSQL 로그인", "Windows 통합 인증 계정", "SQL Server 서비스 계정"]
현재 가능한 행위: ["MSSQL query 실행", "xp_cmdshell로 Windows 명령 실행"]
필요권한: ["MSSQL 로그인 권한 또는 xp_cmdshell 실행 권한"]
필요정보: ["MSSQL 대상과 인스턴스", "현재 SQL 로그인", "서버 역할과 실행 결과"]
네트워크위치: ["대상 MSSQL 포트에 접근 가능한 위치"]
---

# MSSQL 인증 세션 확보 후 권한과 실행 경로 선택

## 상태 라우터 개요

MSSQL 인증에 성공하면 현재 SQL 로그인의 DB 권한, `IMPERSONATE`, linked server와 `xp_cmdshell` 실행 결과를 분리해 확인하고 데이터 수집·OS 명령 실행·로컬 권한 상승·내부 이동 중 실제 조건이 충족된 기법을 선택한다.

## 적용 조건

| 상태 축 | 조건 |
|---|---|
| 대상 플랫폼 | Microsoft SQL Server와 SQL Server가 실행 중인 Windows 호스트 |
| 현재 계정 | MSSQL 로그인 또는 Windows 통합 인증 계정 |
| 현재 가능한 행위 | TDS 세션에서 query 실행, 조건 충족 시 `xp_cmdshell`로 Windows 명령 실행 |
| 현재 권한 | 로그인 성공 이후 DB·서버 역할과 OS 실행 계정을 아직 구분해야 함 |
| 보유 정보 | 대상·인스턴스, 현재 로그인과 query 또는 명령 출력 |
| 네트워크 위치 | 대상 MSSQL 포트에 직접 또는 피벗 경유 접근 가능 |
| 목표 | 데이터·파일·자격 증명·OS 실행·로컬 고권한·linked server 경로 중 다음 상태 확보 |

## 판단 경로

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| MSSQL 인증에 성공했지만 DB 목록·서버 역할·현재 로그인 권한이 미확인임 | [[DB 인증과 데이터 열거]] | 접근 가능한 DB·테이블과 서버 권한 | 이 상태 라우터에서 다시 선택 | SQL 인증과 Windows 인증, 현재 DB·login과 `sysadmin`을 분리해 확인 |
| 현재 login이 `sysadmin`이거나 `xp_cmdshell` 실행 권한이 확인됨 | [[MSSQL xp_cmdshell 명령 실행]] | SQL Server 서비스 계정의 Windows 명령 실행 | 일반 계정이면 [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]], SYSTEM이면 [[고권한 세션 확보 후 후속 판단]] | DB 권한과 OS 계정·호스트를 `IS_SRVROLEMEMBER`, `hostname`, `whoami /all`로 구분 |
| `xp_cmdshell`의 `whoami /priv`에서 `SeImpersonatePrivilege`가 `Enabled`이고 실행 파일을 쓸 수 있음 | [[PrintSpoofer로 SeImpersonatePrivilege 권한 상승]] | SQL Server 호스트의 SYSTEM 명령 실행 | [[고권한 세션 확보 후 후속 판단]] | Windows build·arch, 실행 파일 경로, 현재 token과 `CreateProcessAsUser()` 결과 확인 |
| `xp_cmdshell`로 명령을 실행할 수 있고 Windows 호스트에서 공격 호스트 HTTP 포트에 연결 가능함 | [[Certutil로 Windows HTTP 파일 반입]] | SQL Server 호스트에 저장된 도구 파일 | 이 상태 라우터에서 파일을 사용할 기법 재선택 | 대상 쓰기 경로, URL·포트 도달성과 다운로드 결과를 확인 |
| 현재 login에 다른 login을 가장할 `IMPERSONATE` 권한이 있음 | [[MSSQL Impersonation 권한 상승]] | 가장한 SQL login의 DB·서버 권한 | 이 상태 라우터에서 다시 선택 | 가장 대상, 실행 전후 `SYSTEM_USER`와 `IS_SRVROLEMEMBER('sysadmin')` 확인 |
| `sys.servers`에서 linked server가 확인되고 query 또는 RPC 경로가 있음 | [[MSSQL Linked Server 내부 이동]] | 연결된 SQL Server의 query·login mapping | 이 상태 라우터에서 원격 서버 권한 재평가 | linked server가 가리키는 실제 호스트, login mapping, RPC Out과 원격 query 오류 확인 |
| SQL Server가 UNC 경로에 접근할 수 있고 수신 SMB 서버를 준비할 수 있음 | [[MSSQL 서비스 Hash 캡처]] | SQL Server 서비스 계정의 NetNTLM 인증 시도 | [[확보한 자격 증명으로 원격 접근 경로 선택]] | outbound SMB, 서비스 계정 종류와 수신 측 challenge·response 확인 |
| `BULK INSERT`·`OPENROWSET` 권한과 서버 파일 경로가 있음 | [[DB 서버 파일 수집]] | SQL Server 서비스 계정이 읽을 수 있는 파일 내용 | [[파일 서비스 접근 후 자격 증명과 초기 접근 연결]] | 클라이언트 경로와 DB 서버 로컬 경로를 구분하고 실제 파일 읽기 권한 확인 |

## 상태 재평가

- SQL `sysadmin`, Windows 로컬 관리자와 SYSTEM은 서로 다른 권한이다.
- `SeImpersonatePrivilege` 표시는 권한 상승 후보이며 `nt authority\system` 명령 출력이 있어야 SYSTEM 실행으로 전환한다.
- linked server query가 성공하면 현재 서버가 아니라 query가 실행된 원격 서버의 login·호스트·OS 권한을 다시 확인한다.
- 새 파일·Windows 셸·SYSTEM 실행·자격 증명·내부 SQL Server 경로를 얻어도 MSSQL 세션 상태를 버리지 않는다. SQL 고유 기능, 플랫폼 실행 환경, 파일·자격 증명 또는 피벗 중 현재 목표에 직접 답하는 라우터를 고른다.

## 관련 노트

- [[MSSQL 서비스]]
- [[impacket-mssqlclient]]
- [[MSSQL xp_cmdshell 명령 실행]]
