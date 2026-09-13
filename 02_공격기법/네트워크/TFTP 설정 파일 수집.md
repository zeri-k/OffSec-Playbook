---
tags:
  - 서비스/tftp
시작조건: ["TFTP 서비스 접근 가능", "요청할 파일명 후보 확보"]
필요조건: ["명령 실행 호스트에서 대상 UDP 69 접근", "정확한 파일명 후보"]
결과: ["설정·백업 파일", "자격 증명·SNMP community·내부 네트워크 후보", "선택적으로 확인한 TFTP 쓰기 권한"]
---

# TFTP 설정 파일 수집

## 한 줄 판단

현재 명령 실행 위치에서 대상 TFTP UDP 69에 요청을 보낼 수 있고 PXE·네트워크 장비·백업 파일명 후보가 있으면 디렉터리 목록 대신 정확한 파일명을 요청하여 설정·키·계정·내부 주소 단서를 수집하고, 서버 측 정리 경로가 있으면 고유 검증 파일로 쓰기 권한을 확인한다.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 네트워크 경로 | 대상 UDP 69 요청·응답 가능 | 실제 TFTP RRQ 응답 또는 오류 | `open|filtered`, timeout, source 위치를 구분 |
| 파일명 후보 | PXE·장비·백업 이름 | 정확한 원격 경로 후보 | 호스트명·장비명·날짜·확장자 단서 보강 |
| 전송 결과 | 로컬 파일 생성과 내용 | 파일 내용 확인 | 빈 파일·중단 전송을 제외 |

## 실행

`<TARGET>`은 UDP/69 TFTP 서버 주소(가상 예시 `192.0.2.69`)이고, `pxelinux.cfg/default`·`startup-config`은 다른 단서로 확인한 정확한 원격 filename 예시다. `$PROOF`는 공격 호스트에서 새로 만든 basename이며 업로드·재다운로드·cleanup에서 같은 값을 사용한다. 아래 명령은 TFTP 서버에 도달하는 공격 호스트에서 실행하며, 원본과 수신본 hash 비교는 쓰기 proof의 왕복 동일성을 판단하는 경우에만 유지한다.

```bash
tftp <TARGET>
tftp> binary
tftp> get pxelinux.cfg/default
tftp> get startup-config
tftp> get running-config
tftp> quit
```

확인할 출력:

- 파일 저장 성공과 0보다 큰 로컬 파일 크기.
- 장비 설정, boot 경로, 계정·hash·SNMP community·내부 IP 후보.

### 쓰기 권한 확인

TFTP는 목록 조회와 삭제 명령을 제공하지 않으므로, 서버 측에서 검증 파일을 제거할 경로가 있을 때만 고유 파일을 업로드하고 다시 내려받아 비교한다.

```bash
PROOF="tftp-proof-$(date -u +%Y%m%dT%H%M%SZ)-$$.txt"
printf 'TFTP write verification: %s\n' "$PROOF" > "$PROOF"
tftp <TARGET> -m binary -c put "$PROOF" "$PROOF"
tftp <TARGET> -m binary -c get "$PROOF" "downloaded-$PROOF"
sha256sum "$PROOF" "downloaded-$PROOF"
```

확인할 출력:

- 업로드와 다운로드 완료 응답.
- 원본과 다시 받은 파일의 SHA-256 일치.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `get` 성공과 파일 내용 확인 | 파일명이 실제 TFTP 객체와 일치 | 설정·백업 파일 수집 | 설정 문맥에서 계정·네트워크 단서 분리 |
| SNMP community 발견 | SNMP 인증 후보 | community 후보 | [[원격 비밀번호 공격]]으로 실제 응답 확인 후 [[SNMP OID 정보 열거]] |
| 계정·hash·SSH key 발견 | 후속 인증 후보 | 자격 증명·키 후보 | 대상 서비스와 형식을 확인해 검증 |
| 내부 IP·대역 발견 | 추가 서비스 조사 후보 | 내부 네트워크 단서 | 현재 명령 실행 위치에서 도달성 확인 |
| 검증 파일 업로드·재다운로드와 hash 일치 | TFTP 루트에 새 파일 생성 가능 | TFTP 쓰기 권한 | 서버 측 저장 위치와 파일 처리 방식을 별도로 확인 |
| `file not found` | 서비스는 응답하지만 후보가 틀릴 수 있음 | TFTP 접근만 확인 | 파일명 후보 조정 |

## 확인할 출력과 권한

- `open|filtered`와 timeout은 파일 읽기 성공이 아니다.
- 파일에서 발견한 문자열과 실제 서비스 인증 성공을 구분한다.

## 변경 영향과 복구

- 로컬의 `$PROOF`와 `downloaded-$PROOF`를 제거한다.
- TFTP 자체로는 원격 파일을 삭제할 수 없다. 서버 측 파일 접근 경로에서 정확한 고유 검증 파일을 제거할 수 없으면 쓰기 확인을 수행하지 않는다.

## 후속 공격 연결

- SNMP community 후보: [[원격 비밀번호 공격]]
- 유효한 community의 OID 조회: [[SNMP OID 정보 열거]]
- 장비·서비스 계정: [[원격 비밀번호 공격]]
- SSH key: [[SSH credential 및 키 인증 검증]]

## 관련 서비스

- [[TFTP 서비스]]

## 관련 상태 라우터

- [[파일 서비스 접근 후 자격 증명과 초기 접근 연결]]
- [[확보한 자격 증명으로 원격 접근 경로 선택]]
- [[내부망 경로 확보 후 피벗 구성]]

## 관련 도구

- [[tftp]]

## 참고 링크

- [GNU Inetutils tftp](https://www.gnu.org/software/inetutils/manual/inetutils.html#tftp-invocation)
- [Windows tftp 명령](https://learn.microsoft.com/en-ie/windows-server/administration/windows-commands/tftp)
