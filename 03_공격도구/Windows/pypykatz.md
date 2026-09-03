---
tags:
  - 환경/windows
  - 기능/자격증명수집
실행환경: ["Linux", "Windows"]
필요조건: ["LSASS 덤프 파일"]
결과: ["자격증명", "NTLM 해시", "Kerberos 키", "DPAPI 단서"]
---

# pypykatz

## 도구 개요

Pypykatz는 Windows LSASS minidump를 오프라인으로 분석해 로그온 세션별 NTLM hash·Kerberos key·ticket과 평문 후보를 추출한다. LSASS dump를 원본 Windows 호스트 밖에서 분석하거나 Mimikatz를 직접 실행하지 않고 자격 증명 흔적을 확인할 때 사용한다.

## 필요한 입력과 실행 환경

- 실행 환경: Linux, Windows
- 입력: `lsass.dmp` 같은 Windows LSASS minidump 파일
- 파일 조건: 분석 호스트에서 읽을 수 있고 전송 과정에서 손상되지 않은 dump
- 선택 입력: 긴 분석 결과를 받을 shell 출력 파일

## 표준 사용법

```bash
pypykatz lsa minidump <lsass_dump>
```

## 대표 예시

### LSASS minidump에서 credential 추출

```bash
pypykatz lsa minidump <OPERATOR_HOME>/Documents/lsass.dmp
```

### 결과를 파일로 저장하며 분석

```bash
pypykatz lsa minidump lsass.dmp | tee pypykatz.out
```


## 주요 옵션

| 명령·인자 | 의미 | 사용하는 상황 |
|---|---|---|
| `lsa` | Windows LSA credential parser 선택 | LSASS 분석 |
| `minidump` | minidump 파일 입력 방식 지정 | 오프라인 `lsass.dmp` 분석 |
| `<lsass_dump>` | 분석할 dump 경로 | 실제 입력 파일 지정 |

## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| 로그온 세션별 NTLM hash·Kerberos key·ticket·평문 후보 출력 | LSASS dump에서 계정별 인증 자료 추출 | hash는 Pass the Hash·cracking, ticket은 Pass the Ticket, 각 계정 권한은 대상 서비스에서 별도 확인 |
| parsing 완료지만 credential 없음 | dump에 재사용 가능한 값이 없거나 보호됨 | dump 시점, 사용자 세션, Credential Guard 여부 확인 |
| parsing 오류 | dump 형식/architecture 불일치 또는 손상 | 원본 dump 재수집, minidump 형식, 도구 버전 확인 |
| DPAPI/Vault 단서 | 추가 복호화 대상 존재 | masterkey와 사용자 context 확보 여부 확인 |

## 관련 공격기법

- [[LSASS 메모리 덤프]]
