---
tags:
  - 환경/network-appliance
  - 서비스/ipmi
시작조건: ["명령 실행 위치에서 대상 BMC UDP/623 접근 가능", "IPMI 2.0 RMCP+ 응답 확인"]
필요권한: ["RAKP 응답 수집에는 사전 BMC 인증 불필요"]
필요조건: ["대상 BMC 주소", "BMC 사용자명 후보 또는 수집 도구의 사용자명 목록", "오프라인 크래킹 환경"]
결과: ["대상 BMC 계정의 RAKP HMAC challenge-response", "크래킹 성공 시 대상 BMC 계정의 평문 비밀번호"]
---

# IPMI hash 수집과 크래킹

## 한 줄 판단

명령 실행 위치에서 대상 Baseboard Management Controller(BMC)의 UDP/623에 접근할 수 있으면 IPMI 2.0 RMCP+의 RAKP HMAC challenge-response를 수집하고, 대상 BMC 계정의 평문 비밀번호를 오프라인으로 복구할 수 있는지 확인한다.

## 사용할 때

- 현재 보유 정보: 대상 BMC 주소와 BMC 사용자명 후보. 운영체제·도메인 계정이나 비밀번호는 필요하지 않다.
- 명령 실행 위치: 서버 운영체제가 아니라 관리 인터페이스의 UDP/623에 패킷을 보내고 응답을 받을 수 있는 네트워크 위치.
- 현재 권한: RAKP 응답 수집에는 BMC 로그인 권한이 필요하지 않지만 라이브 서비스에 도달할 네트워크 권한은 필요하다.
- 공격 대상과 결과: BMC 사용자 계정의 RAKP HMAC 값을 얻고, 크래킹 성공 시 그 BMC 계정의 평문 비밀번호를 얻는다. 호스트 root나 도메인 권한은 별도다.
- 수집 결과 경계: RAKP HMAC 한 줄은 오프라인 후보 대조용 응답이며, BMC 로그인 성공이나 운영체제 계정의 hash·비밀번호 복구를 의미하지 않는다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 명령 실행 위치와 네트워크 | 대상 BMC UDP/623에 접근하고 응답 수신 가능 | `nmap -sU --script ipmi-version`의 IPMI 응답 | 관리 VLAN, 라우팅, UDP 필터링과 대상 주소 확인 |
| 현재 인증 수단 | 사전 BMC credential 불필요 | 인증 없이 IPMI 2.0 RAKP 교환이 시작되는지 확인 | IPMI 버전과 익명 RAKP 응답 제한 여부 확인 |
| 공격 대상 계정 | 유효한 BMC 사용자명 후보 | 수집 출력의 사용자명과 RAKP HMAC 값 확인 | 기본 목록 또는 확인된 사용자 후보 재검토 |
| 오프라인 복구 환경 | RAKP 형식에 맞는 Hashcat mode와 wordlist | mode `7300` 입력 인식 | 캡처 형식과 전체 라인 확인 |

## 실행

1. UDP 623과 IPMI 버전을 확인한다.
2. Metasploit scanner로 IPMI hash dump 가능성을 확인한다.
3. hashcat으로 hash를 크래킹한다.
4. 복구한 credential은 BMC 웹/CLI 접근과 서버 제어 영향으로 분리해 판단한다.

### IPMI 식별

```bash
nmap -sU --script ipmi-version -p623 <TARGET>
```

확인할 출력:

- IPMI version, vendor, auth support.

### Metasploit hash 수집

```bash
msfconsole
use auxiliary/scanner/ipmi/ipmi_dumphashes
set RHOSTS <TARGET>
run
```

확인할 출력:

- 사용자명과 IPMI hash.
- 수집 실패 메시지만으로 BMC 계정이 없다고 판단하지 않는다. 응답 가능한 사용자명 후보, IPMI 2.0 지원과 UDP 경로를 먼저 분리한다.

### hashcat 크래킹

```bash
hashcat -m 7300 ipmi.hashes rockyou.txt --backend-ignore-opencl -d 1 -O -w 3
```

확인할 출력:

- `hash:password` 형태의 복구 결과.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| IPMI 버전과 BMC 응답 | 관리 인터페이스 식별 성공 | IPMI 접근 경로 확인 | RAKP hash 수집 가능성 확인 |
| 사용자명과 IPMI hash 출력 | 원격 hash 수집 성공 | 오프라인 크래킹 가능한 BMC hash 확보 | [[오프라인 해시 크래킹]] |
| Hashcat의 `hash:password` | BMC 비밀번호 복구 | 관리 credential 후보 | BMC 웹/CLI에서 인증과 기능 권한을 분리 검증 |
| UDP 응답 불안정 | 필터링, 손실 또는 네트워크 위치 문제 | 서비스 식별 불확실 | 재시도, rate와 네트워크 경로 확인 |
| hash 수집 실패 | IPMI 2.0 미지원 또는 설정 제한 | IPMI 서비스만 확인 | 기본 credential 여부와 웹 관리 포트 별도 확인 |
| 크래킹 실패 | 강한 비밀번호 또는 후보 부족 | IPMI hash만 확보 | 맞춤 wordlist 여부를 검토하고 우선순위 재평가 |

## 확인할 출력과 권한

- IPMI 버전 응답, hash 수집, 비밀번호 복구, 관리 콘솔 로그인은 각각 별도 상태다.
- hash 수집에는 사전 인증이 필요하지 않을 수 있지만 관리 기능은 복구한 계정의 BMC 역할에 제한된다.
- BMC 접근은 호스트 운영체제의 root 또는 도메인 권한과 동일하지 않다.
- 수집값은 BMC의 RAKP HMAC challenge-response이며 운영체제 계정의 NTLM hash나 Linux password hash가 아니다.

## 후속 공격 연결

- 크래킹: [[오프라인 해시 크래킹]]
- BMC credential 재사용: [[원격 비밀번호 공격]]

## 관련 상태 라우터

- BMC 비밀번호를 복구했으면: [[확보한 자격 증명으로 원격 접근 경로 선택]]

## 관련 서비스

- [[623_IPMI]]

## 관련 도구

- [[metasploit]]
- [[hashcat]]
- [[nmap]]
