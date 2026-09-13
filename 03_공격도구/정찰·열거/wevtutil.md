---
tags:
  - 환경/windows
  - 기능/열거
실행환경: ["Windows CMD 또는 PowerShell"]
필요권한: ["조회할 event log channel의 read 권한"]
필요조건: ["event log 이름 또는 EVTX 경로", "XPath query·출력 형식·조회 상한"]
결과: ["event log 구성", "조건에 맞는 event text·XML", "event export file"]
---

# wevtutil

## 도구 개요

`wevtutil.exe`는 Windows 기본 event log CLI로 log·publisher를 열거하고 XPath 조건으로 event를 조회하거나 export한다. 공격 흐름에서는 채널별 읽기 권한과 event ID·시간 범위를 먼저 좁혀 대량 출력과 민감 자료의 불필요한 저장을 피할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: 조사할 Windows 호스트의 CMD·PowerShell 또는 `/r:<REMOTE_HOST>`로 접근 가능한 관리 호스트.
- 입력: log/channel 이름, XPath query, 역순 여부, 출력 형식과 최대 event 수.
- 원격 조회: 대상 RPC·Remote Event Log Management 방화벽 경로와 `/u`·`/p`로 지정할 계정.
- 권한: 각 channel security descriptor가 현재 Windows client process의 access token 또는 `/u`로 지정한 원격 인증 계정에 허용한 read 범위. 원격 Event Log service가 실제로 적용하는 authorization과 client process token을 혼동하지 않으며, 한 channel 조회 성공을 Security channel 권한으로 확대하지 않는다.

## 표준 사용법

```cmd
wevtutil qe <LOG_NAME> /q:"<XPATH_QUERY>" /rd:true /f:text /c:<MAX_EVENTS>
```

`<LOG_NAME>`은 channel 이름(예: `Security`), `<XPATH_QUERY>`는 event 조건, `<MAX_EVENTS>`는 양의 정수 상한(예: `50`)이다. `<REMOTE_HOST>`는 Windows host, `<DOMAIN>\<USER>`는 원격 인증 계정이며 명령은 대상 또는 관리 Windows 호스트에서 실행한다.

## 대표 예시

### log 목록과 channel 구성 확인

```cmd
wevtutil el
wevtutil gl Security
```

확인할 출력:

- 사용 가능한 channel 이름과 `enabled`, `retention`, `channelAccess` 같은 구성.
- 구성 조회 성공과 event 본문 읽기 성공은 별도이므로 `qe`로 확인한다.

### 최근 4688 event 조회

```cmd
wevtutil qe Security /q:"*[System[(EventID=4688)]]" /rd:true /f:text /c:<MAX_EVENTS>
```

확인할 출력:

- `Event ID: 4688`, 생성 시각, subject·creator와 process 정보.
- `Process Command Line`은 관련 audit policy가 활성화된 경우에만 값이 있다.

### 원격 host 조회

```cmd
wevtutil qe Security /r:<REMOTE_HOST> /u:<DOMAIN>\<USER> /p:<PASSWORD> /q:"*[System[(EventID=4688)]]" /rd:true /f:text /c:<MAX_EVENTS>
```

확인할 출력:

- 원격 대상에서 반환된 event와 명시적 인증의 성공 여부.
- 명령줄에 전달한 비밀번호는 현재 process command line과 audit log에 노출될 수 있으므로 가능한 경우 현재 인증 context를 우선하고, 일회성 입력만 사용한다.

## 주요 옵션

| 옵션 | 의미 | 사용하는 상황 |
|---|---|---|
| `qe` | log 또는 EVTX에서 event query | ID·provider·time 조건으로 event 조회 |
| `/q:<QUERY>` | XPath query | provider 측에서 event 범위를 줄일 때 |
| `/rd:true` | 최신 event부터 역순 반환 | 최근 활동을 제한적으로 볼 때 |
| `/f:text`·`/f:xml` | text 또는 XML 출력 | 사람이 읽거나 field 구조를 정확히 파싱할 때 |
| `/c:<COUNT>` | 반환 event 수 제한 | 전체 log dump를 피할 때 |
| `/r:<HOST>` | 원격 host 지정 | 원격 event log 관리 경로가 있을 때 |
| `/u:<USER>`·`/p:<PASSWORD>` | 원격 인증 계정 | 현재 context와 다른 계정이 필요할 때 |
| `/lf:true` | 입력 path를 live log가 아닌 log file로 처리 | 회수한 EVTX를 오프라인 조회할 때 |

## 도구 고유 출력

| 출력·상태 | 의미 | 다음 확인 |
|---|---|---|
| event text·XML 반환 | query와 channel read 성공 | event field와 현재 조사 목표의 관계 확인 |
| `Access is denied` | 현재 Windows client process access token 또는 지정 원격 인증 계정에 channel read 권한 없음 | channel ACL, client token과 server authorization을 구분 |
| 결과 없음 | 조건과 현재 retention 범위에서 event 없음 | log name, XPath, 시간·retention과 audit 설정 확인 |
| RPC·server unavailable | 원격 관리 경로 실패 | 대상명·방화벽·Remote Event Log Management 확인 |

`wevtutil cl`은 log를 clear하고, `epl`은 export file을 만든다. 조회 절차에서는 `cl`을 사용하지 않으며 `epl`을 사용했다면 exact output path와 보존·삭제 책임을 별도로 기록한다.

## 관련 공격기법

- [[Windows 이벤트 로그에서 민감 명령줄 검색]]

## 참고 링크

- [Microsoft Learn: wevtutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil)
