---
tags:
  - 환경/ad
  - 서비스/kerberos
  - 기능/형식변환
실행환경: ["Linux"]
필요조건: ["변환할 Kerberos 티켓 파일"]
결과: ["티켓"]
---

# impacket-ticketConverter

## 도구 개요

`impacket-ticketConverter`는 Windows 도구가 사용하는 Kerberos `.kirbi`와 Linux 도구가 사용하는 `.ccache` 형식을 서로 변환한다. 운영체제별 도구 사이에서 기존 ticket을 옮길 때 사용하며 ticket에 기록된 계정·서비스 이름, 만료 시간과 권한은 바꾸지 않는다.

## 필요한 입력과 실행 환경

- 실행 위치: Impacket이 설치된 Linux 호스트
- 필요한 입력: 변환할 `.kirbi` 또는 `.ccache` ticket 파일과 출력 경로
- 후속 환경: Linux에서는 `KRB5CCNAME`, Windows 도구에서는 `.kirbi`를 사용하는 흐름에 맞춰 출력 형식을 정한다.
- `<INPUT_TICKET>`과 `<OUTPUT_TICKET>`은 Linux 실행 host의 서로 다른 경로이며 확장자는 변환 방향을 나타낸다. 변환 성공은 KDC 또는 service가 ticket를 수락했음을 뜻하지 않는다.


## 표준 사용법

`<INPUT_TICKET>`은 existing `.kirbi`/`.ccache` Linux path, `<OUTPUT_TICKET>`은 변환할 새 path이며 방향에 맞는 extension을 쓴다. conversion result·cache inspection·KDC/service ticket acceptance는 별도 상태고 cleanup은 output path만 대상으로 한다.

```bash
test ! -e '<OUTPUT_TICKET>'
impacket-ticketConverter '<INPUT_TICKET>' '<OUTPUT_TICKET>'
```

## 대표 예시

### Linux ccache를 Windows kirbi로 변환

```bash
test ! -e '<TICKET_FILE>'
impacket-ticketConverter '<CCACHE_FILE>' '<TICKET_FILE>'
```

### Windows kirbi를 Linux ccache로 변환

```bash
test ! -e '<CCACHE_FILE>'
impacket-ticketConverter '<TICKET_FILE>' '<CCACHE_FILE>'
```

## 주요 옵션

| 항목 | 설명 |
| --- | --- |
| `<input_ticket>` | 변환할 ccache 또는 kirbi 파일 |
| `<output_ticket>` | 확장자에 따라 변환될 출력 파일 |
| `KRB5CCNAME` | Linux에서 ccache 사용 시 지정하는 환경 변수 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| `.kirbi` 또는 `.ccache` 생성 | Kerberos ticket 형식 변환 성공 | Windows/Linux 도구에 맞게 ticket 주입 또는 환경 변수 설정 |
| 변환 파일을 도구가 인식 | 후속 Kerberos 인증 준비 완료 | `klist`, `KRB5CCNAME`, `kerberos::ptt`로 확인 |
| parsing 오류 | ticket 파일 손상 또는 형식 불일치 | 원본 ticket 재수집, base64/바이너리 변환 여부 확인 |
| 인증 실패 | ticket 자체 조건 문제 | SPN, realm, 만료 시간, 시간 동기화 확인 |

형식 변환 성공은 ticket의 principal·SPN·유효 시간이 맞거나 KDC·서비스가 받아들였다는 뜻이 아니다. 입력과 출력의 ticket 정보를 각각 확인한 뒤 실제 서비스에서 사용 여부를 검증한다.

## 변경 영향과 복구

이 도구가 새로 만드는 상태는 실행 전 없음을 확인한 `<OUTPUT_TICKET>` 하나다. 후속 Windows·Linux process에서 ticket 사용을 먼저 종료한 뒤 이 exact 출력 파일만 제거한다. 원본 `<INPUT_TICKET>`은 이 명령이 만든 파일이 아니므로 삭제하지 않는다.

```bash
rm -- '<OUTPUT_TICKET>'
test ! -e '<OUTPUT_TICKET>'
```

마지막 명령이 성공해야 변환 사본 정리가 끝난 것이다. 다른 위치로 복사한 ticket, 주입된 Windows 로그온 세션과 KDC·서비스 감사 기록은 이 파일 삭제로 제거되지 않는다.

## 관련 공격기법

- [[Pass the Ticket]]
- [[Linux Kerberos keytab ccache 악용]]

## 참고 링크

- [Fortra Impacket: ticketConverter implementation](https://github.com/fortra/impacket/blob/master/examples/ticketConverter.py)
