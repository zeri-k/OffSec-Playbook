---
tags:
  - 환경/windows
  - 서비스/smb
시작조건: ["대상 SMB 서비스와 주소 식별", "명령 실행 호스트에서 SMB 접근 가능"]
필요권한: ["명령 실행 호스트의 SMB 클라이언트 실행 권한"]
필요조건: ["Null Session·Guest 또는 유효 계정 중 시험할 인증 컨텍스트", "대상 139/TCP 또는 445/TCP 접근"]
결과: ["인증 컨텍스트별 SMB 로그인 여부", "공유 목록", "공유별 파일 읽기 후보"]
---

# SMB 익명 열거와 공유 권한 확인

## 한 줄 판단

SMB 클라이언트를 실행할 수 있는 호스트에서 대상 139/TCP 또는 445/TCP에 도달해 Null Session, Guest와 보유 계정을 각각 시험하고, 인증 컨텍스트별 공유 목록과 share 직접 접근·읽기 범위를 확인한다.

## 사용할 때

- SMB/Samba를 식별했고 인증 없이 보이는 공유가 있는지 확인할 때.
- Null Session, Guest와 보유 계정의 공유 노출 차이를 비교할 때.
- 공유 목록 표시와 실제 파일 읽기 권한을 구분해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 네트워크 경로 | 대상 139/TCP 또는 445/TCP 도달 | SMB 협상 응답 확인 | 이름 해석·대상 주소·SMB dialect 확인 |
| Null Session | 사용자명과 비밀번호를 보내지 않는 시험 | `-N`, 빈 사용자·비밀번호 확인 | Guest 성공과 구분 |
| Guest·보유 계정 | 실제 사용할 계정 형식 | 도메인·로컬 계정 형식과 인증 결과 | 인증 성공과 공유 열거 거부를 구분 |

## 실행

### Null Session 공유 목록

```bash
smbclient -L //<TARGET>/ -N
netexec smb <TARGET> -u '' -p '' --shares
```

### Guest 공유 목록

```bash
netexec smb <TARGET> -u guest -p '' --shares
```

### 보유한 계정의 공유 목록

```bash
netexec smb <TARGET> -d <DOMAIN> -u <USER> -p '<PASSWORD>' --shares
netexec smb <TARGET> --local-auth -u <USER> -p '<PASSWORD>' --shares
```

### share 직접 읽기 확인

```bash
smbclient //<TARGET>/<SHARE> -N
smb: \> ls
smb: \> get <FILE>
```

확인할 출력:

- 어느 인증 컨텍스트에서 로그인과 공유 목록이 반환됐는지.
- share 직접 접근, 목록 조회와 `get` 성공 여부.
- `STATUS_ACCESS_DENIED`, `STATUS_LOGON_FAILURE`, timeout을 서로 다른 단계로 기록한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| Null 또는 Guest로 공유 목록 반환 | 해당 컨텍스트에 공유 열거 허용 | 익명·Guest 공유 목록 | share 직접 접근과 READ 확인 |
| share에서 `ls`·`get` 성공 | 현재 컨텍스트에 파일 읽기 권한 | SMB 파일 접근 | [[SMB 공유 자격증명 수집]] |
| 도구가 WRITE 후보를 표시 | 직접 생성·조회·삭제 전에는 미확정 | SMB 쓰기 후보 | [[SMB 쓰기 가능한 공유 검증]] |
| 사용자·그룹 단서가 필요 | 공유 열거와 별도 RPC·LDAP 행동 | 계정 열거 필요 | [[인증 전 AD 사용자 목록 수집]] |
| 비밀번호 정책이 필요 | 공유 열거와 별도 정책 조회 행동 | 잠금 정책 필요 | [[AD 비밀번호 정책 열거 및 조회]] |
| 목록은 보이지만 share 접근 거부 | 목록 열거와 파일 권한이 분리됨 | 공유 이름만 확인 | 인증 컨텍스트와 share ACL 재확인 |

## 확인할 출력과 권한

- Null Session, Guest와 유효 계정의 결과를 서로 다른 접근 상태로 기록한다.
- 공유 목록은 share 존재만, `get` 성공은 해당 파일의 읽기만 확정한다.
- 사용자·정책 열거와 쓰기 검증은 연결된 세부 기법에서 수행한다.

## 후속 공격 연결

- 사용자 목록: [[인증 전 AD 사용자 목록 수집]]
- 잠금 정책: [[AD 비밀번호 정책 열거 및 조회]]
- READ share: [[SMB 공유 자격증명 수집]]
- WRITE 후보: [[SMB 쓰기 가능한 공유 검증]]
- signing 확인: [[NTLM Relay 조건 검토]]

## 관련 서비스

- [[445_SMB]]
- [[135_139_RPC_NetBIOS]]

## 관련 상태 라우터

- [[파일 서비스 접근 후 자격 증명과 초기 접근 연결]]

## 관련 도구

- [[smbclient]]
- [[smbmap]]
- [[netexec]]
