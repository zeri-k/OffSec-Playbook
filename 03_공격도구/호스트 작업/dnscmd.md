---
tags:
  - 환경/windows
  - 서비스/dns
  - 기능/설정변경
실행환경: ["Windows CMD 또는 PowerShell"]
필요권한: ["조회·변경할 DNS 서버와 zone의 관리 권한"]
필요조건: ["대상 Windows DNS 서버 이름 또는 주소", "수행할 server·zone·record operation과 복구 기준선"]
결과: ["DNS server·zone 설정 조회", "DNS 설정·record 변경", "operation별 성공·오류 상태"]
---

# dnscmd

## 도구 개요

`dnscmd.exe`는 Windows DNS Server를 로컬 또는 원격 RPC로 조회·구성하는 기본 관리 CLI다. server-level plug-in, zone과 record처럼 영향 범위가 다른 operation을 명시적으로 선택하고 변경 전후 값을 비교할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 위치: `dnscmd`가 있는 Windows 관리 호스트.
- 대상: `<DNS_SERVER>` FQDN·hostname·IP. 생략하면 local server이므로 원격·로컬 대상을 혼동하지 않는다.
- 권한: DnsAdmins, DNS server administrator 또는 선택 operation에 위임된 동등한 권한.
- 변경 입력: exact parameter, 기존 값과 복구 값. 예를 들어 `<DNS_SERVER>`은 `dns01.corp.example`, `<ZONE_NAME>`은 `corp.example`, `<NODE_NAME>`은 `www`처럼 server·zone·record 역할을 구분한다. DNS service 재시작 권한은 `dnscmd` 설정 권한과 별도다.

## 표준 사용법

`<DNS_SERVER>`은 local 또는 remote DNS server, `<ZONE>`·`<NODE>`는 해당 server의 exact zone/record name, `<DLL_PATH>`는 DNS service host가 읽는 absolute DLL path다. 이 도구 문서는 server 설정·record 조회/변경과 그 출력만 다루며, msfvenom 생성·전송, ACL, restart, DnsAdmins membership, 로그오프/새 로그인 순서는 연결 기법의 입력·복구 흐름을 그대로 따른다.

```cmd
dnscmd [<DNS_SERVER>] <COMMAND> [<PARAMETERS>]
```

## 대표 예시

### server-level 설정 조회

```cmd
dnscmd <DNS_SERVER> /info /serverlevelplugindll
```

확인할 출력:

- 현재 `ServerLevelPluginDll` 값 또는 미설정 상태.
- 조회 성공은 설정 변경·DLL load·service 실행 권한을 뜻하지 않는다.

### custom DNS plug-in 경로 설정과 해제

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll <ABSOLUTE_PLUGIN_DLL_PATH>
dnscmd <DNS_SERVER> /info /serverlevelplugindll
```

기존 custom plug-in이 없었던 상태로 되돌릴 때 DLL path를 생략한다.

```cmd
dnscmd <DNS_SERVER> /config /serverlevelplugindll
dnscmd <DNS_SERVER> /info /serverlevelplugindll
```

확인할 출력:

- `Registry property serverlevelplugindll successfully reset`와 query에서 확인한 변경 값.
- 설정은 다음 DNS service start 때 사용할 path를 지정한다. 실제 DLL load와 code execution은 service 상태와 별도 관찰값으로 확인한다.

### zone record 조회

```cmd
dnscmd <DNS_SERVER> /enumrecords <ZONE_NAME> <NODE_NAME> /detail
```

확인할 출력:

- 지정 zone·node의 record와 상세 값. record read 권한과 server-level config 권한을 같은 것으로 보지 않는다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `/info [<SETTING>]` | server-level 설정 조회 | 변경 전·후 기준선 확인 |
| `/config <PARAMETER>` | server-level 설정 변경 | exact 복구 값을 확보한 변경 |
| `/serverlevelplugindll [<DLL_PATH>]` | custom DNS plug-in의 절대 경로 지정, path 생략 시 기존 custom plug-in 사용 중지 | plug-in 구성 검증과 원복 |
| `/enumrecords` | zone node의 record 열거 | DNS 관리 권한으로 record 확인 |
| `/recordadd`·`/recorddelete` | DNS record 추가·삭제 | 기존 record와 exact 복구를 기록한 변경 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Command completed successfully` | 요청 operation 처리 성공 | `/info`, `/zoneinfo` 또는 record 재조회로 실제 값 확인 |
| `Registry property ... successfully reset` | server-level 설정 write 성공 | service가 새 값을 load했는지는 별도 확인 |
| `ERROR_ACCESS_DENIED`·`Status = 5` | 현재 Windows client process access token의 해당 operation 권한 부족 또는 대상 DNS server authorization 거부 | client access token, 대상 server와 DnsAdmins·위임 범위 확인 |
| RPC server unavailable | DNS 관리 RPC 경로 실패 | 대상명·방화벽·DNS service와 네트워크 도달성 확인 |

## 관련 공격기법

- [[DnsAdmins DNS 서버 플러그인 DLL 실행]]
- [[AD DNS 레코드 열거]]

## 참고 링크

- [Microsoft Learn: dnscmd](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/dnscmd)
- [Microsoft Open Specifications: ServerLevelPluginDll](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dnsp/c9d38538-8827-44e6-aa5e-022a016ed723)
