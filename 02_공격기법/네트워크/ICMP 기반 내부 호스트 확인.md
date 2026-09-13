---
시작조건: ["내부 네트워크에 연결된 실행 호스트 또는 ICMP 전달이 가능한 피벗 경로 확보", "대상 CIDR 또는 /24 주소 접두사 확보"]
필요권한: ["현재 OS 계정 또는 Meterpreter 세션에서 ICMP Echo Request 전송 가능"]
필요조건: ["실행 환경에 맞는 fping, ping, Test-Connection 또는 Meterpreter 세션", "대상 CIDR 또는 /24 주소 접두사", "실행 호스트에서 대상 대역으로 ICMP를 보낼 route"]
결과: ["ICMP 응답 호스트 후보", "대상 범위의 응답 통계"]
---

# ICMP 기반 내부 호스트 확인

## 한 줄 판단

내부망에 연결된 Linux·Windows 호스트 또는 Meterpreter 세션에서 대상 대역으로 ICMP를 보낼 수 있으면, 실행 환경에 맞는 ping sweep으로 Echo Reply를 보낸 IP를 후속 서비스 열거 후보로 얻되 무응답 IP는 제외하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 경로 | 실행 호스트 또는 피벗이 대상 CIDR로 ICMP를 전달 | 인터페이스, route와 알려진 내부 주소에 대한 단일 ICMP 요청 확인 | TCP 전용 프록시인지 확인하고 [[내부망 경로 확보 후 피벗 구성]]에서 경로 수정 |
| 현재 계정 또는 인증 수단 | sweep 명령을 실행할 로컬 OS 계정 또는 Meterpreter 세션 | 사용할 명령이나 `ping_sweep` 모듈 실행 가능 여부 확인 | 명령·모듈 존재, 실행 권한과 세션 상태 확인 |
| 현재 권한 | raw socket 또는 시스템이 부여한 ICMP 송신 권한 | 단일 대상 실행에서 권한 오류 없이 요청 전송 | setuid·capability·현재 계정 권한을 확인하고 ICMP를 보낼 수 있는 실행 계정 사용 |
| 공격 대상의 조건 | 정확한 내부 CIDR 또는 /24 주소 접두사 | route·인터페이스 주소와 범위 대조 | 범위가 불명확하면 능동 sweep을 중단하고 수동 단서 재확인 |
| 필요한 파일·목록·주소 | `<TARGET_CIDR>` 또는 `<TARGET_PREFIX>`와 실행할 sweep 방식 | CIDR 표기, /24 전제와 자기 호스트 포함 여부 확인 | 네트워크 주소·prefix를 수정하고 예상 대상 수 확인 |

## 실행

`<TARGET_CIDR>`은 피벗 또는 내부 인터페이스에서 route가 확인된 대역(예: `198.51.100.0/24`)이고, `<TARGET_IP>`는 그 결과에서 개별 확인할 주소다. 아래 Linux·Windows 명령은 해당 ICMP를 실제 전송할 수 있는 피벗 또는 내부 명령 실행 호스트에서 실행하며, TCP 전용 SOCKS 경로에서는 같은 결과를 기대하지 않는다.

### Linux 피벗 호스트에서 fping 실행

대상 CIDR 전체에 ICMP Echo Request를 보내고 응답 IP와 집계를 함께 출력한다.

```bash
fping -asgq <TARGET_CIDR>
```

확인할 출력:

- 응답한 IP 목록.
- `targets`, `alive`, `unreachable`, `timeouts`, `ICMP Echo Replies` 집계.
- 공격 호스트 자체가 `alive` 수에 포함됐는지 여부.
- `Operation not permitted` 계열 오류는 현재 계정의 ICMP 송신 권한을, `Network is unreachable`은 route·피벗 경로를 먼저 확인한다. timeout만 있는 경우 권한·경로·ICMP 차단을 구분한다.

### Linux 피벗 호스트에서 ping 반복 실행

`fping`이 없고 대상이 `/24`이면 주소의 마지막 옥텟을 반복한다.

```bash
for i in {1..254}; do (ping -c 1 <TARGET_PREFIX>.$i | grep "bytes from" &); done
```

확인할 출력:

- `bytes from <IP>`가 표시된 주소는 Echo Reply를 반환한 호스트 후보다.
- `<TARGET_PREFIX>`에는 `198.51.100`처럼 마지막 옥텟을 제외한 `/24` 접두사를 넣는다. 다른 prefix 길이에 이 반복 범위를 그대로 사용하지 않는다.

### Windows 피벗 호스트의 CMD에서 실행

```cmd
for /L %i in (1 1 254) do ping <TARGET_PREFIX>.%i -n 1 -w 100 | find "Reply"
```

확인할 출력:

- `Reply from <IP>`가 표시된 주소를 후속 서비스 확인 후보로 남긴다.
- 배치 파일 안에서는 반복 변수 `%i`를 `%%i`로 바꾼다.

### Windows 피벗 호스트의 PowerShell에서 실행

```powershell
1..254 | % {"<TARGET_PREFIX>.$($_): $(Test-Connection -Count 1 -ComputerName <TARGET_PREFIX>.$($_) -Quiet)"}
```

확인할 출력:

- 주소 뒤에 `True`가 표시되면 해당 시점에 ICMP 응답을 반환했다.
- `False`는 호스트 부재가 아니라 ICMP 차단·timeout·route 문제일 수도 있다.

### Meterpreter 세션에서 실행

공격 호스트의 Meterpreter 프롬프트에서 모듈을 실행하지만 ICMP 트래픽은 세션이 열린 피벗 호스트에서 대상 CIDR로 생성된다.

```text
meterpreter > run post/multi/gather/ping_sweep RHOSTS=<TARGET_CIDR>
```

확인할 출력:

- `Performing ping sweep for IP range <TARGET_CIDR>`는 모듈 실행 시작을 뜻한다.
- 응답 주소가 없으면 피벗 호스트에서 단일 IP에 ICMP를 보낼 수 있는지 확인한다. 시작 메시지만으로 대상 호스트가 없다고 판단하지 않는다.
- 첫 sweep에서는 ARP cache가 아직 구성되지 않아 응답을 놓칠 수 있으므로 경로가 정상인데 결과가 비어 있으면 같은 범위를 한 차례 더 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| IP가 출력되고 `alive` 집계에 포함됨 | 해당 시점에 ICMP 응답을 반환함 | 활성 호스트 후보 | 서비스 열거 대상으로 추가하고 [[무인증 내부 네트워크에서 AD 단서 확인]]에서 AD 서비스 여부 판단 |
| 수동 관찰 후보가 ICMP에 응답하지 않음 | ICMP 차단·라우팅·호스트 방화벽 가능성 | 호스트 부재 미확정 | 수동 단서를 유지하고 TCP 서비스 확인으로 교차 검증 |
| `unreachable` 또는 timeout이 대부분임 | 네트워크 정책 또는 경로 영향 가능성 | sweep 결과 신뢰도 낮음 | CIDR·route·피벗 경로를 다시 확인 |
| 첫 sweep은 비어 있고 같은 경로의 두 번째 sweep에서 응답함 | 초기 ARP cache 구성 지연 가능성 | ICMP 응답 호스트 후보 | 응답 주소를 서비스 열거 대상으로 추가 |
| DNS·SMB·Kerberos·LDAP 응답 호스트가 확인됨 | AD 역할 후보 존재 | DC 또는 AD 서비스 후보 | [[AD 도메인 컨텍스트 기본 확인]] |
| 예상 범위 밖 주소를 입력함 | 범위 오인 또는 라우팅 혼선 | 실행 중단 필요 | 능동 확인을 중단하고 정확한 CIDR을 재확인 |

## 확인할 출력과 권한

- `alive`는 ICMP 응답 사실만 뜻하며 운영체제, 역할, 서비스 또는 취약성을 증명하지 않는다.
- 출력된 IP는 실행 시점에 해당 경로로 Echo Reply를 보냈다는 사실만 확정한다. 무응답 IP의 부재, TCP·UDP 도달성 또는 양방향 세션 가능성은 확정하지 않는다.
- `unreachable`과 timeout을 호스트 부재로 단정하지 않는다.
- 서비스 스캔 대상에는 수동 관찰, DNS와 다른 프로토콜에서 확인된 호스트도 함께 유지한다.

## 관련 도구

- [[fping]]
- [[meterpreter]]

## 관련 상태 라우터

- [[무인증 내부 네트워크에서 AD 단서 확인]]
- [[내부망 경로 확보 후 피벗 구성]]
