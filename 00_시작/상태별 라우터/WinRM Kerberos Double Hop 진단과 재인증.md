---
tags:
  - 환경/windows
  - 서비스/winrm
시작상태: ["<JUMP_HOST>의 WinRM PowerShell 세션 확보", "점프 호스트 로컬 명령 성공", "점프 호스트에서 두 번째 Kerberos 리소스 접근 실패"]
목표: ["두 번째 홉 실패 조건 구분", "명시적 credential 재인증 기법 선택"]
현재계정: ["WinRM 세션의 원래 로그인 계정"]
현재 가능한 행위: ["점프 호스트에서 PowerShell 명령 실행", "점프 호스트에서 DNS와 대상 서비스 포트 확인"]
필요권한: ["원래 로그인 계정의 대상 리소스 읽기 권한"]
필요정보: ["현재 ticket 목록", "두 번째 대상의 FQDN·SPN·포트", "원래 로그인 계정의 plaintext password 보유 여부"]
네트워크위치: ["공격 호스트 -> <JUMP_HOST>:5985/5986 -> 두 번째 대상 서비스"]
---

# WinRM Kerberos Double Hop 진단과 재인증

## 상태 라우터 개요

`<JUMP_HOST>`의 WinRM 세션에서 로컬 명령은 성공하지만 DC 또는 두 번째 Kerberos 서비스 접근만 실패할 때, 현재 ticket·네트워크 경로·재인증 자료를 기준으로 명시적 자격 증명 재인증 기법을 선택한다.

## 적용 조건

| 상태 축 | 조건 |
|---|---|
| 대상 플랫폼 | AD에 연결된 Windows 점프 호스트와 두 번째 Kerberos 서비스 |
| 현재 계정 | WinRM 세션의 원래 도메인 계정 |
| 현재 가능한 행위 | `<JUMP_HOST>`의 PowerShell 명령 실행과 두 번째 대상의 DNS·포트 확인 |
| 현재 권한 | WinRM 로그온 권한은 확인됐지만 두 번째 리소스 권한은 별도 확인 필요 |
| 보유 정보 | 현재 ticket 목록, 두 번째 대상 FQDN·SPN·포트, 원래 계정 plaintext password 보유 여부 |
| 네트워크 위치 | 공격 호스트에서 `<JUMP_HOST>:5985/5986`, 점프 호스트에서 DC 또는 `<SECOND_HOST>`로 연결 |
| 목표 | 두 번째 서비스 재인증에 필요한 입력을 확인하고 적절한 공격기법 선택 |

## 판단 경로

| 현재 보유 상태·입력 | 선택할 공격기법 또는 수동 확인 | 성공하면 얻는 상태 | 다음 상태 라우터 | 선택 기준·미충족 시 확인 |
|---|---|---|---|---|
| WinRM 로컬 명령은 성공하지만 두 번째 Kerberos 리소스만 실패하고 현재 세션에 사용자 TGT가 없으며 원래 AD 계정의 plaintext password가 있음 | [[WinRM 두 번째 홉 명시적 자격 증명 재인증]] | 원래 AD 계정으로 두 번째 서비스 조회 또는 TGT를 가진 임시 세션 | [[AD Identity 확인 후 도메인 컨텍스트 열거]] | `<JUMP_HOST>`에서 대상 FQDN·Kerberos·LDAP 포트가 도달하는지, 현재 ticket과 원래 계정의 대상 리소스 권한을 확인 |
| WinRM 로컬 명령은 성공하지만 두 번째 Kerberos 리소스만 실패하고 plaintext password·재사용 가능한 TGT가 없음 | 수동 확인: DNS·FQDN·SPN·시간·대상 포트·모듈 상태와 현재 계정 ACL을 분리하고 재인증 가능한 자료를 확보 | Double Hop 외 원인 또는 부족한 인증 자료 식별 | 이 상태 라우터 유지 | `HTTP/<JUMP_HOST>` service ticket만으로 두 번째 서비스에 인증할 수 없으며 RunAs endpoint도 자격 증명 없이 만들지 않음 |
| 명시적 credential 조회가 성공해 원래 AD 계정으로 DC 객체를 읽을 수 있음 | [[AD 도메인 컨텍스트 기본 확인]] | 현재 AD 계정·도메인·DC와 사용 가능한 서비스 | [[AD Identity 확인 후 도메인 컨텍스트 열거]] | 조회 결과의 계정 Identity, 대상 도메인, LDAP 읽기 범위와 DC 서비스 도달성을 확인 |
| WinRM 서비스 재시작 또는 endpoint 변경 뒤 기존 세션이 끊김 | 수동 확인: 기본 endpoint와 임시 endpoint의 등록 상태를 확인하고 선택한 endpoint로 다시 연결 | 복구된 WinRM 세션 또는 제거할 임시 endpoint 식별 | [[Windows 셸 또는 세션 확보 후 컨텍스트 열거]] | 서비스 실행 상태, 5985·5986/TCP, endpoint 이름과 로그인 계정의 WinRM 권한을 확인 |

## 상태 재평가

- WinRM 인증 성공과 두 번째 Kerberos 서비스 인증 성공을 구분한다.
- `HTTP/<JUMP_HOST>` service ticket과 사용자 TGT는 같은 인증 자료가 아니다. AS·TGS 발급, cache 표시와 두 번째 서비스 사용의 차이는 [[Kerberos 인증 자료와 서비스 접근]]을 따른다.
- 명시적 credential 조회 성공 뒤에도 원래 계정의 AD 객체 권한과 두 번째 대상의 서비스 권한을 별도로 확인한다.
- 새로운 AD 계정·ticket·원격 세션 또는 고권한을 얻으면 해당 상태 라우터로 전환한다.

## 관련 노트

- [[WinRM 원격 PowerShell 세션]]
- [[WinRM 두 번째 홉 명시적 자격 증명 재인증]]
- [[PowerView]]
- [[evil-winrm]]
- [[klist]]
