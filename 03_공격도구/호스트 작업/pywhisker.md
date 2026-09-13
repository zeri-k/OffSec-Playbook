---
tags:
  - 환경/ad
  - 서비스/ldap
  - 기능/자격증명수집
실행환경: ["Linux"]
필요권한: ["대상 객체 msDS-KeyCredentialLink 쓰기 권한"]
필요조건: ["도메인 인증 정보", "도메인 컨트롤러 LDAP 또는 LDAPS 접근", "수정할 사용자 또는 컴퓨터 객체"]
결과: ["PFX 인증서", "KeyCredential"]
---

# pywhisker

## 도구 개요

pyWhisker는 AD 사용자·컴퓨터 객체의 `msDS-KeyCredentialLink`를 추가·조회·삭제하고 인증용 certificate·PFX를 생성한다. Shadow Credentials 경로에서 KeyCredential 변경과 후속 PKINIT 인증 자료 생성을 함께 처리할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux
- 입력: 도메인, DC 주소, 인증 계정과 비밀번호·hash, 대상 사용자 또는 컴퓨터 객체
- 권한 조건: 대상 객체의 `msDS-KeyCredentialLink` 쓰기 권한
- 후속 입력: 생성된 PFX와 password를 사용할 PKINIT 도구
- `<AUTH_USER>`는 requester credential의 owner, `<TARGET_OBJECT>`는 KeyCredential을 변경할 user/computer object로 서로 다를 수 있다. DC address·domain·PFX·DeviceID는 출력에서 기록하고 remove는 같은 DeviceID만 대상으로 한다.

## 표준 사용법

`<AUTH_USER>`·password/hash는 KeyCredential write requester, `<TARGET_OBJECT>`은 변경 대상 user/computer object, `<DC_IP>`·`<DOMAIN>`은 directory endpoint/namespace다. generated `<PFX_FILE>`·password·`<DEVICE_ID>`는 output에서 기록하고 remove/clear는 same target·DeviceID를 exact하게 재사용한다.

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --target <TARGET> --action add
```

## 대표 예시

### KeyCredential 추가

```bash
pywhisker --dc-ip <TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --target <USER> --action add
```

### 추가된 KeyCredential 확인

```bash
pywhisker --dc-ip <TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --target <USER> --action list
```

### 정리

```bash
pywhisker --dc-ip <TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --target <USER> --action remove --device-id <DEVICE_ID>
```

## 주요 옵션

| 옵션 | 설명 |
|---|---|
| `--target` | KeyCredential을 추가할 대상 객체 |
| `--action add/list/remove` | 추가, 조회, 삭제 |
| `--dc-ip` | 도메인 컨트롤러 IP 지정 |
| `-d`, `-u`, `-p` | 도메인, 사용자, 비밀번호 |
| `--device-id` | 삭제할 KeyCredential 식별자 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| PFX 파일과 password 출력 | Shadow Credentials 추가 성공 | gettgtpkinit/certipy auth로 TGT 요청 |
| `msDS-KeyCredentialLink` 업데이트 | 대상 객체 수정 성공 | `list`로 확인하고 후속 접근 검증 |
| LDAP modify 실패 또는 insufficient access | 쓰기 권한 부족, 대상 객체 오류 또는 LDAPS 요구 | ACL, 대상 객체, LDAP/LDAPS, 계정 형식 확인 |
| 후속 PKINIT 실패 | DC 인증서, EKU, realm 또는 시간 조건 불일치 | gettgtpkinit/Certipy 출력과 DC certificate 조건 확인 |
| 정리 실패 | device-id 불일치 | `list`로 현재 값 재확인 |

## 관련 공격기법

- [[Shadow Credentials]]
