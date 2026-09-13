---
tags:
  - 서비스/ldap
시작조건: ["장비 관리 화면에서 LDAP 연결 설정과 Test Connection 기능 확인", "현재 LDAP 전송 보호 방식 확인"]
필요권한: ["대상 장비의 LDAP 설정을 조회·변경할 관리 권한", "검증 호스트의 listener·packet capture 권한"]
필요조건: ["기존 LDAP 설정 기준선과 즉시 복원 경로", "장비에서 검증 호스트로의 TCP 경로", "plaintext LDAP simple bind 구성"]
결과: ["Test Connection의 LDAP bind 전송 여부", "평문 simple bind 자격 증명 노출 또는 보호·미전송 판정", "기존 LDAP 설정 복원 상태"]
---

# LDAP Test Connection 자격 증명 노출 검증

## 한 줄 판단

프린터·애플리케이션 관리 화면의 LDAP `Test Connection`이 TLS 없는 simple bind를 사용하는지 확인할 수 있다면, 기존 설정을 보존한 채 서버 주소만 통제된 listener로 잠시 전환해 bind 전송을 관찰하고 즉시 원래 서버로 복원한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상 관리 접근 | 장비의 LDAP server·port·TLS·bind identity를 조회하고 되돌릴 수 있음 | 관리 화면과 out-of-band 복구 경로 확인 | UI 접근이 끊길 수 있으면 변경하지 않음 |
| 기존 상태 | server, port, LDAP/LDAPS·StartTLS, 인증 방식, Base DN과 인증서 검증 값 기록 | 작업 기록에 화면·값 보존, 비밀번호 원문은 기록하지 않음 | 기존 값을 모르면 변경하지 않음 |
| 적용 범위 | `Test Connection` 한 번과 서버 주소·port만 변경 | 저장이 즉시 운영 인증에 적용되는지 장비 문서에서 확인 | 운영 로그인·주소록 등에 영향이 있으면 변경하지 않음 |
| 네트워크 | 장비→`<LISTENER_IP>:<LISTENER_PORT>` TCP 도달 | 방화벽·route와 listener 연결 관찰 | 연결 실패와 자격 증명 미전송을 구분 |
| 프로토콜 | TLS 없는 LDAP simple bind | 현재 scheme·port·StartTLS 설정 확인 | LDAPS·StartTLS·SASL이면 평문 simple bind 노출 검증으로 해석하지 않음 |

LDAP simple bind는 이름과 password를 BindRequest에 넣는다. RFC 4513은 TLS나 다른 기밀성 보호 없이 name/password 방식을 쓰는 것을 부적합하다고 설명한다. 반대로 LDAPS·StartTLS·SASL 트래픽이 보이지 않는다고 자격 증명 미사용으로 단정하지 않는다.

## 실행

`<LISTENER_IP>:<LISTENER_PORT>`는 장비가 도달할 수 있는 검증 호스트의 고유 listener(가상 예시 `192.0.2.50:1389`)이고, `<DEVICE_IP>`는 Test Connection을 수행할 장비 주소다. `<INTERFACE>`는 해당 트래픽이 지나는 검증 호스트 인터페이스, `<LDAP_TEST_PCAP>`은 새 pcap 파일 경로이며 `<TCPDUMP_PID>`·`<NC_PID>`는 시작 출력에서 기록한 정확한 PID다. 아래 Bash 명령은 검증 호스트에서 실행하고 같은 값으로 캡처·복구를 수행한다.

### 1. listener와 packet capture 기준선

검증 호스트에서 기존 listener 충돌과 pcap 파일 부재를 확인한다. 실제 bind 값은 Vault가 아닌 접근이 제한된 위치에서만 취급한다.

```bash
ss -lntp | grep ':<LISTENER_PORT> '
test ! -e '<LDAP_TEST_PCAP>'
```

packet capture와 raw TCP listener를 각각 시작하고 표시된 PID를 기록한다.

```bash
sudo tcpdump -ni <INTERFACE> -s0 -w '<LDAP_TEST_PCAP>' host <DEVICE_IP> and tcp port <LISTENER_PORT>
sudo nc -lvnp <LISTENER_PORT>
```

확인할 출력:

- tcpdump가 선택한 인터페이스와 `<LDAP_TEST_PCAP>` 기록 시작을 표시한다.
- netcat이 exact port에서 대기하며 기존 서비스와 충돌하지 않는다.
- root가 필요한 낮은 port 대신 장비가 허용하는 고유 high port를 우선 사용한다. 장비가 port 변경을 지원하지 않으면 기존 389 listener 충돌을 먼저 해결한다.

### 2. 장비 설정을 최소 변경하고 한 번 확인

관리 화면에서 기존 LDAP server·port를 `<LISTENER_IP>`·`<LISTENER_PORT>`로 바꾸되 TLS·StartTLS·인증 방식·bind identity·Base DN은 바꾸지 않는다. 저장이 필요한 경우 운영 영향 시간을 최소화하고 `Test Connection`을 한 번만 실행한다.

확인할 출력:

- listener 또는 pcap에 장비의 TCP 연결이 도착했는지.
- Wireshark/tshark에서 LDAP BindRequest와 simple authentication 값이 평문으로 해석되는지.
- 연결만 있고 BindRequest가 없거나 protocol 오류만 있으면 자격 증명 전송을 확인하지 못한 것이다.

```bash
tshark -r '<LDAP_TEST_PCAP>' -Y 'ldap' -V
```

출력에 bind DN·password가 보이면 필요한 최소 판정만 하고 화면·터미널 내용을 Vault에 복사하지 않는다. raw listener가 정상 LDAP BindResponse를 보내지 않으므로 장비의 `Test Connection failed`는 예상할 수 있으며, 그 자체는 자격 증명 유효성이나 실제 디렉터리 인증 실패의 증거가 아니다.

## 변경 영향과 복구

Test 결과를 얻었는지와 관계없이 장비의 서버·port를 작업 전 값으로 즉시 복원한다. TLS·인증 방식 등 다른 값은 처음부터 바꾸지 않는다. 원래 LDAP 서버에 `Test Connection`을 실행해 기존 상태의 연결 결과가 돌아왔는지 확인한다.

복구 완료 조건:

- 현재 server·port·TLS/StartTLS·인증 방식·Base DN과 인증서 검증 값이 작업 전 기준선과 일치한다.
- 원래 서버를 대상으로 한 Test 결과가 작업 전과 같은 상태다. 성공 여부가 원래부터 불명확했다면 `설정값 복원 확인, 서비스 동작 미확인`으로 남긴다.
- 관리 연결이 끊겨 값을 확인하지 못하면 원상복구 완료로 표현하지 않는다.

그 뒤 기록한 exact listener·capture PID만 종료하고 보존이 필요하지 않은 pcap만 제거한다.

```bash
sudo kill <TCPDUMP_PID> <NC_PID>
ps -p <TCPDUMP_PID>,<NC_PID>
rm -- '<LDAP_TEST_PCAP>'
test ! -e '<LDAP_TEST_PCAP>'
```

`ps`에 대상 PID가 없고 pcap 부재가 확인돼야 로컬 정리 완료다. bind credential이 listener 화면, shell scrollback, pcap과 장비·네트워크 로그에 남을 수 있으며 원격 로그는 클라이언트에서 되돌릴 수 없다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| TCP 연결과 평문 simple BindRequest | Test 기능이 보호 없이 bind 자격 증명을 전송 | 자격 증명 노출 확인 | 즉시 설정 복원·listener 종료·민감 자료 처리 후 credential rotation 권고 |
| TCP 연결만 있고 LDAP bind 없음 | 연결 경로만 확인 | 자격 증명 전송 미확인 | 먼저 원복하고 장비 protocol·Test 동작 확인 |
| TLS ClientHello 또는 암호화 세션 | 전송 보호가 시도됨 | 평문 노출 미확인 | 인증서 검증·TLS 성공 여부를 별도 확인; 암호문 부재를 credential 미사용으로 해석하지 않음 |
| 연결 없음 | route·방화벽·저장 미적용 가능 | Test 동작 미확인 | 먼저 원복하고 listener IP·port·장비 egress 확인 |
| 기준선 일치와 원래 Test 상태 회복 | 장비 설정 복원 확인 | 원상복구 완료 또는 서비스 동작 제한적 확인 | 로컬 listener·pcap 정리 |

## 관련 서비스

- [[LDAP 서비스]]

## 관련 도구

- [[netcat]]
- `tcpdump`
- `tshark`

## 참고 링크

- [RFC 4511: LDAP Bind Operation](https://www.rfc-editor.org/rfc/rfc4511.html#section-4.2)
- [RFC 4513: LDAP Authentication Methods and Security Mechanisms](https://www.rfc-editor.org/rfc/rfc4513.html)
