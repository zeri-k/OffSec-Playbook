---
tags:
  - 서비스/nfs
  - 기능/열거
실행환경: ["Linux"]
필요조건: ["NFS mountd 또는 RPC 접근"]
결과: ["NFS export 정보", "접근 허용 범위"]
---

# showmount

## 도구 개요

`showmount`는 NFS 서버의 RPC mountd에 질의해 공개된 export 경로와 허용 클라이언트 범위를 보여 주는 열거 도구다. 마운트 후보를 빠르게 찾을 때 적합하며, 목록 노출은 실제 마운트나 파일 권한을 보장하지 않는다.

## 필요한 입력과 실행 환경

- 실행 환경: `showmount`가 설치된 Linux 호스트
- 입력: NFS 서버 주소
- 네트워크 조건: 대상의 RPC/mountd에 접근 가능


## 표준 사용법

```bash
showmount -e <target>
```

## 대표 예시

### NFS export 목록 확인

```bash
showmount -e <TARGET>
```

### 발견한 export를 로컬에 마운트

```bash
sudo mount -t nfs <TARGET>:/shared /mnt/nfs -o nolock
```

## 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `-e` | export 목록 표시 |
| `-a` | client와 directory mapping 표시 |
| `-d` | export된 directory만 표시 |


## 도구 고유 출력

| 출력/상태 | 의미 | 다음 행동 |
|---|---|---|
| export 경로와 접근 범위 출력 | NFS export 확인 | `mount -t nfs`로 읽기/쓰기와 UID 매핑 확인 |
| `everyone` 또는 넓은 네트워크 허용 | 접근 제어가 약할 가능성 | 민감 파일, 백업, 홈 디렉터리 노출 여부 확인 |
| `clnt_create: RPC: Timed out` | RPC/NFS 접근 불가 또는 필터링 | TCP/UDP 111, 2049와 mountd 포트 확인 |
| `Program not registered` / empty | mountd/NFS export 없음 | NFS 버전과 RPC 서비스 상태 재확인 |

## 관련 공격기법

- [[NFS export 마운트와 권한 매핑 검증]]
