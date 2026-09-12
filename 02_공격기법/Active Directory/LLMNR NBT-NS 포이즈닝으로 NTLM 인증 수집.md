---
tags:
  - 환경/windows
  - 환경/ad
  - 서비스/llmnr
시작조건: ["내부 네트워크에서 LLMNR 또는 NBT-NS 요청 관찰", "포이즈닝할 인터페이스와 시간대 확인"]
필요권한: ["Linux의 패킷 캡처와 서비스 바인딩 권한 또는 Windows 로컬 관리자 권한"]
필요조건: ["피해 호스트와 이름 해석 요청을 관찰할 수 있는 네트워크 위치", "수신 포트 충돌이 없는 공격 호스트"]
결과: ["NetNTLM challenge-response", "인증 주체와 출발지 단서", "오프라인 크래킹 또는 NTLM relay 후보"]
---

# LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집

## 한 줄 판단

내부 링크에서 DNS 실패 뒤 발생하는 LLMNR/NBT-NS 요청을 먼저 수동 관찰하고, 대상 인터페이스에서 공격자 주소로 응답하여 NetNTLM 인증을 수집한 뒤 오프라인 크래킹과 실시간 relay 조건을 분리해 판단한다.

## 사용할 때

- 내부 네트워크 세그먼트에서 이름 해석 실패 요청이 발생하는지 확인할 때.
- 아직 유효한 도메인 credential이 없거나 더 가치 있는 인증 주체의 NetNTLM 응답을 수집하려 할 때.
- 수집한 인증을 평문 비밀번호 복구와 NTLM relay 중 어느 경로로 사용할지 결정해야 할 때.
- 현재 보유 정보: 공격 호스트의 인터페이스·주소와 관찰할 내부 세그먼트뿐이며, 대상 사용자 비밀번호나 NT hash는 수집 전에는 보유하지 않은 상태다.
- 실행 위치와 대상: 피해 호스트의 LLMNR UDP/5355 또는 NBT-NS UDP/137 broadcast를 볼 수 있는 같은 링크에서 listener를 실행한다. 다른 VLAN에서 이름만 아는 상태로는 수신할 수 없다.

## 전제 조건

| 구분 | 조건 | 확인 방법 |
|---|---|---|
| 시작 상태 | LLMNR UDP 5355 또는 NBT-NS UDP 137 요청을 볼 수 있음 | Responder 분석 모드 또는 패킷 캡처에서 요청 확인 |
| 필요 권한 | 패킷 캡처와 rogue listener 바인딩 가능 | Linux는 root, Windows는 필요한 포트를 열 수 있는 관리자 컨텍스트 확인 |
| 입력·환경 | 정확한 인터페이스, 대상 VLAN·호스트·시간대 | route와 인터페이스 주소 확인 |
| 충돌 방지 | 공격 호스트의 SMB·HTTP 등 수신 포트가 비어 있음 | 기존 서비스와 Responder/Inveigh listener 상태 확인 |

## 실행

### Linux 공격 호스트에서 실행

#### 1. Responder 분석 모드로 가능성 확인

Linux 공격 호스트에서는 먼저 Responder 분석 모드로 요청만 관찰한다.

```bash
sudo responder -I <INTERFACE> -A
```

확인할 출력:

- LLMNR 또는 NBT-NS 요청의 이름, 출발지와 빈도.
- 분석 모드이므로 공격자 응답이나 NTLM challenge가 전송되지 않는 상태.
- 요청이 전혀 없다면 인터페이스, VLAN과 관찰 시간을 다시 확인한다.

#### 2. Responder 활성 포이즈닝

Linux에서는 확인한 `<INTERFACE>`에서 Responder를 활성 모드로 시작하고, 정한 종료 시점 또는 대상 밖 요청이 보이면 중지한다.

기본 LLMNR·NBT-NS 포이즈닝과 rogue listener만 사용할 때 실행한다.

```bash
sudo responder -I <INTERFACE>
```

WPAD 요청과 NetBIOS `wredir` suffix 요청까지 응답하고, 요청 호스트를 식별하면서 상세 출력을 확인해야 할 때 다음 확장 실행을 사용한다.

```bash
sudo responder -I <INTERFACE> -wrfv
```

`-w`는 WPAD rogue proxy를 시작하고, `-r`은 NetBIOS `wredir` suffix 요청에도 응답한다. `-f`는 요청 호스트의 fingerprinting을 시도하고 `-v`는 상세 출력을 활성화한다. `-r` 응답은 정상 이름 해석과 파일·프린터 연결을 방해할 수 있으므로 분석 모드에서 해당 요청을 확인하고 영향 범위와 중지 조건을 정한 뒤 제한적으로 사용한다.

확인할 출력:

- 시작 요약에서 WPAD proxy와 host fingerprinting이 활성화되었는지 확인한다.
- LLMNR·NBT-NS·WPAD 요청의 출발지와 요청 이름, poisoner의 응답 전송 여부를 확인한다.
- `NTLMv1` 또는 `NTLMv2` capture가 기록되면 NetNTLM challenge-response와 인증 주체를 새 결과로 확보한 상태다.
- 사용자 인증 프롬프트, 파일·프린터 연결 실패 또는 대상 밖 응답이 보이면 즉시 `Ctrl+C`로 중지한다.

### Windows 공격 호스트에서 실행

#### Inveigh 활성 포이즈닝

C# Inveigh 실행 파일을 사용할 수 있다. 시작 요약에서 실제 활성화된 spoofer와 listener를 확인하고 사용하지 않을 프로토콜은 실행 전에 비활성화한다.

```powershell
.\Inveigh.exe
```

확인할 출력:

- Responder의 poisoner 응답과 `NTLMv1` 또는 `NTLMv2` capture.
- Inveigh의 `response sent`, SMB/HTTP NTLM challenge와 캡처 수 증가.
- Inveigh 대화형 콘솔의 `GET NTLMV2UNIQUE`, `GET NTLMV2USERNAMES` 결과.
- `address already in use` 또는 listener 시작 실패가 보이면 포트 충돌을 해결하기 전 결과를 신뢰하지 않는다.
- `NTLMv2` capture는 challenge-response 수집 성공이다. Hashcat mode `5600`으로 평문이 복구되기 전에는 비밀번호를 얻지 못한 상태다.

### 3. 수집 결과를 목적별로 분기

- 저장한 NetNTLM challenge-response를 비밀번호 후보와 대조하려면 [[오프라인 해시 크래킹]]으로 넘긴다. NetNTLMv2는 Hashcat mode `5600` 후보이며 Pass the Hash에 직접 쓰지 않는다.
- relay는 저장된 응답을 나중에 재사용하는 절차가 아니라 들어오는 인증을 실시간으로 다른 서비스에 전달하는 흐름이다. SMB signing, EPA/CBT, 대상 서비스와 계정 권한은 [[NTLM Relay 조건 검토]]에서 별도로 확인한다.
- relay를 선택하면 Responder의 충돌하는 SMB/HTTP listener를 끄고 `ntlmrelayx` 수신 구성을 맞춘다. 같은 인증을 크래킹 분기와 relay 분기로 혼동하지 않는다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 분석 모드에서 LLMNR/NBT-NS 요청이 반복됨 | 활성 포이즈닝 가능성이 있으나 아직 인증은 수집하지 않음 | 이름 해석 요청 단서 | 대상 인터페이스와 중지 조건을 확인한 뒤 제한적으로 활성화 |
| `response sent` 뒤 NetNTLMv1/v2 응답이 기록됨 | 피해 호스트의 NTLM 인증이 공격 호스트에 도달함 | NetNTLM challenge-response와 인증 주체 단서 | 비밀번호 복구는 [[오프라인 해시 크래킹]], 실시간 전달은 [[NTLM Relay 조건 검토]] |
| Hashcat 또는 John이 평문을 복구함 | 재사용 가능한 credential 후보 확보 | 평문 비밀번호 후보 | [[확보한 자격 증명으로 원격 접근 경로 선택]]에서 계정 상태와 서비스 권한 검증 |
| relay가 인증에는 성공했지만 후속 작업이 거부됨 | relay된 주체에 대상 작업 권한이 없음 | 인증 relay만 성립 | 대상 서비스 ACL과 다른 relay 후보 재평가 |
| 요청은 보이지만 인증 응답이 없음 | 요청 유형, 캐시, 인증 방식 또는 listener 조건이 맞지 않음 | 이름 해석 요청만 확인 | 활성 프로토콜과 listener 오류를 확인하고 불필요한 응답은 중지 |
| 사용자 프롬프트, 연결 실패 또는 업무 서비스 이상이 관찰됨 | 활성 응답이 정상 이름 해석이나 인증 흐름에 영향을 줌 | 운영 영향 발생 | 즉시 포이즈닝 중지 후 아래 복구 절차 수행 |

## 확인할 출력과 권한

- 이름 해석 요청 관찰, 공격자 응답 전송, NTLM challenge-response 수집, 평문 복구와 relay 후 작업 성공을 각각 다른 상태로 기록한다.
- 캡처된 사용자 이름이나 머신 계정만으로 계정 권한을 추정하지 않는다.
- NetNTLM challenge-response는 NT hash가 아니며, 오프라인 크래킹 성공 전에는 평문 credential이 아니다.

## 변경 영향과 복구

| 변경 대상 | 예상 영향 | 검증 방법 | 복구 절차 |
|---|---|---|---|
| 링크 로컬 이름 해석 응답 | 잘못된 이름 요청이 공격 호스트로 향해 정상 연결 실패, 지연 또는 인증 프롬프트가 발생할 수 있음 | `response sent` 대상·시간과 사용자 또는 서비스 오류를 대조 | Responder는 `Ctrl+C`, Inveigh는 대화형 콘솔의 `STOP` 또는 PowerShell의 `Stop-Inveigh`로 즉시 중지 |
| 공격 호스트의 rogue SMB·HTTP·WPAD listener | 기존 서비스와 포트 충돌하거나 의도하지 않은 프로토콜에서 인증을 받을 수 있음 | 시작 요약과 listener 오류, 로컬 포트 점유 확인 | 프로세스를 중지하고 listener가 닫혔는지 확인한 뒤 변경한 Responder 설정을 작업 전 상태로 복원 |
| 대상 밖 요청 처리 | 다른 VLAN·호스트의 정상 통신에 영향을 줄 수 있음 | 캡처 로그의 출발지와 대상 목록 비교 | 대상 밖 응답이 보이면 즉시 중지하고 네트워크 정상화 및 관련 서비스 연결을 재검증 |

- 이 절차는 대상 호스트 설정을 영구 변경하지 않는다. 중지 후 공격 호스트가 더 이상 LLMNR/NBT-NS 응답이나 rogue service를 제공하지 않는지 확인한다.

## 관련 도구

- [[Responder]]
- [[Inveigh]]
- [[hashcat]]
- [[impacket-ntlmrelayx]]

## 관련 시나리오

- [[무인증 내부망에서 Responder로 AD 자격 증명 확보]]

## 관련 상태 라우터

- 평문 비밀번호를 복구했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]
- relay로 AD 객체 권한이나 AD 계정 접근을 얻었으면: [[AD Identity 확인 후 도메인 컨텍스트 열거]]
