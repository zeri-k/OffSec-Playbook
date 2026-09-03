---
tags:
  - 환경/windows
  - 서비스/llmnr
  - 기능/자격증명수집
실행환경: ["Linux"]
필요권한: ["root 또는 패킷 캡처와 서비스 바인딩 권한"]
필요조건: ["요청을 관찰할 수 있는 네트워크 위치"]
결과: ["이름 해석 요청", "NetNTLM challenge-response"]
---

# Responder

## 도구 개요

Responder는 Linux에서 LLMNR·NBT-NS·mDNS·WPAD 요청을 관찰하거나 응답하고 rogue 서비스를 통해 사용자·출발지와 NetNTLM challenge-response를 수집한다. 내부 이름 해석 트래픽을 분석하거나 인증을 유도할 때 사용하며, 캡처 결과는 계정의 NTLM hash 자체가 아니다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 입력: 관찰할 네트워크 인터페이스
- 권한 조건: packet capture와 rogue service 포트 bind가 가능한 root 권한
- 네트워크 조건: LLMNR/NBT-NS/mDNS/WPAD 요청을 관찰할 수 있는 위치
- relay 연계 조건: Responder의 SMB/HTTP server와 ntlmrelayx listener가 같은 포트를 점유하지 않도록 설정

## 표준 사용법

```bash
sudo responder -I <interface>
```

## 대표 예시

### 내부망 인터페이스에서 NetNTLM challenge-response 수집

```bash
sudo responder -I <INTERFACE> -v
```

콘솔에 `NTLMv1` 또는 `NTLMv2` capture가 표시되는지 확인한다. Kali 패키지 설치에서는 보통 `/usr/share/responder/logs`에 프로토콜·형식·출발지별 파일이 생성되며, 소스 저장소에서 직접 실행했다면 해당 저장소의 `logs` 디렉터리를 확인한다.

### WPAD와 wredir 응답을 포함한 확장 실행

```bash
sudo responder -I <INTERFACE> -wrfv
```

`-wrfv`는 `-w -r -f -v`를 붙여 쓴 형태다. WPAD rogue proxy와 NetBIOS `wredir` suffix 응답을 활성화하고, 요청 호스트 fingerprinting과 상세 출력을 함께 사용한다. 시작 요약에서 활성화된 poisoner·server를 확인하고 `NTLMv1` 또는 `NTLMv2` capture가 표시되는지 확인한다. `-r`은 정상 파일·프린터 연결에 영향을 줄 수 있으므로 승인된 범위와 중지 조건을 정한 경우에만 사용한다.

### 응답하지 않고 이름 해석 트래픽만 관찰

```bash
sudo responder -I eth0 -A
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-I` | 수신 인터페이스 지정 |
| `-A` | analyze mode. 응답하지 않고 관찰 |
| `-v` | 상세 출력 |
| `-w`, `--wpad` | WPAD rogue proxy 시작 |
| `-r`, `--wredir` | NetBIOS `wredir` suffix 요청에 응답. 정상 연결을 방해할 수 있으므로 제한적으로 사용 |
| `-f`, `--fingerprint` | 요청 호스트의 운영체제·버전 fingerprinting 시도 |
| 설정 파일 | relay 시 SMB/HTTP 서버를 끄는 등 동작 조정 |

소문자 `-f`는 host fingerprinting이고 대문자 `-F`는 WPAD 파일 요청에 인증을 강제하는 별도 옵션이다. 설치된 버전의 정확한 옵션은 `responder -h`로 확인한다.


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| NetNTLMv1/v2 challenge-response 캡처 | 내부망 인증 유도 성공 | cracking 또는 relay 가능성 검토 |
| LLMNR/NBT-NS/WPAD 요청 확인 | 이름 해석 트래픽 존재 | relay 대상과 캡처 계정 권한 확인 |
| 요청 없음 | 같은 L2 세그먼트가 아니거나 트래픽 유도 없음 | 인터페이스, VLAN, 강제 인증 트리거 확인 |
| 요청은 보이나 인증 응답 없음 | poisoning 비활성, 캐시 또는 인증 유도 미발생 | analyze mode 여부, protocol 옵션, 요청 유형 확인 |
| address already in use | Responder와 relay listener의 포트 충돌 | Responder SMB/HTTP server 설정과 listener 포트 확인 |

## 관련 공격기법

- [[내부망 수동 호스트 식별]]
- [[LLMNR NBT-NS 포이즈닝으로 NTLM 인증 수집]]
- [[네트워크 트래픽 자격증명 수집]]
- [[NTLM Relay 조건 검토]]

## 관련 시나리오

- [[무인증 내부망에서 Responder로 AD 자격 증명 확보]]
