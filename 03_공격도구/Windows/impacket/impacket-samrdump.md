---
tags:
  - 환경/windows
  - 서비스/smb
  - 기능/열거
실행환경: ["Linux 또는 Impacket 실행 환경"]
필요권한: ["대상 SAMR 인터페이스가 요청자에게 허용한 조회 권한"]
결과: ["계정 이름, RID와 계정 설명"]
---

# impacket-samrdump

## 도구 개요

`impacket-samrdump`는 SMB/RPC의 Security Account Manager Remote(SAMR) 인터페이스를 조회해 대상이 반환하는 계정 이름, RID와 계정 정보를 출력하는 Impacket 예제 도구다. NULL Session 또는 보유 계정으로 SAMR 열거 가능성을 빠르게 확인할 때 적합하다.

## 필요한 입력과 실행 환경

- 실행 위치: Impacket 예제 명령을 실행할 수 있고 대상 SMB 445/TCP에 도달하는 호스트
- 필요한 입력: 대상 주소와 필요한 경우 `[domain/]username[:password]` 인증 정보
- 반환 범위: 대상의 SAMR 정책과 요청자 권한에 따라 달라짐

## 표준 문법

```bash
impacket-samrdump <TARGET>
impacket-samrdump '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>'
```

설치 형태에 따라 같은 예제가 `samrdump.py` 이름으로 제공될 수 있다.

## 대표 예시

```bash
impacket-samrdump <DC>
```

## 도구 고유 출력

| 출력 | 의미 | 다음 확인 |
|---|---|---|
| 계정명과 RID 항목 | 요청자에게 SAMR 계정 정보가 반환됨 | 사용자·컴퓨터·그룹 객체를 구분 |
| `rpc_s_access_denied` | SAMR 조회 권한 부족 | NULL Session 여부와 보유 계정 범위 확인 |
| 연결 오류 | 사용자 열거 전에 SMB/RPC 경로 실패 | DNS, 445/TCP와 대상 주소 확인 |

## 관련 공격기법

- [[인증 전 AD 사용자 목록 수집]]
