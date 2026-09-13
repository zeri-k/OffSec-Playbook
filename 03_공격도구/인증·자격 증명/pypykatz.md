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
test ! -e '<PYPYKATZ_OUTPUT>'
pypykatz lsa minidump '<LSASS_DUMP_PATH>' | tee '<PYPYKATZ_OUTPUT>'
test -s '<PYPYKATZ_OUTPUT>'
```

기존 출력 파일이 있으면 덮어쓰지 말고 다른 exact 경로를 정한다. `tee`의 종료 상태만으로 parser 성공을 확정하지 않고 화면과 파일에서 `FILE`, `LogonSession`, parser 오류를 함께 확인한다.


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

## 변경 영향과 복구

기본 분석은 입력 dump를 읽고 표준 출력에 표시할 뿐이다. `tee` 예시는 `<PYPYKATZ_OUTPUT>`을 추가로 만들며, 이 파일에도 hash·key·masterkey·평문 후보가 남을 수 있다. 필요한 후속 처리가 끝나면 이번 실행 전 없었던 exact 출력만 승인된 보존·폐기 정책에 따라 처리한다.

```bash
rm -- '<PYPYKATZ_OUTPUT>'
test ! -e '<PYPYKATZ_OUTPUT>'
```

입력 `<LSASS_DUMP_PATH>`는 이 도구가 만든 자원이 아니므로 여기서 삭제하지 않는다. terminal scrollback·shell logging과 이미 복사한 인증 자료는 출력 파일 삭제로 되돌릴 수 없다.

## 관련 공격기법

- [[LSASS 메모리 덤프]]

## 참고 링크

- [skelsec pypykatz: LSASS processing and minidump source](https://github.com/skelsec/pypykatz#lsass-processing)
