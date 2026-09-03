---
tags:
  - 서비스/ftp
대표포트:
  - "T:21"
서비스:
  - FTP
---

# 21_FTP

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:21`의 File Transfer Protocol(FTP) 제어 채널에 연결할 수 있고, 아직 익명 로그인·사용자 계정·파일 권한은 확인하지 않은 상태에서 시작한다. 익명과 인증 사용자별 디렉터리 목록·파일 읽기·쓰기 권한을 구분하며, 서버 제품과 버전은 공개 취약점 후보를 고르는 단서일 뿐이다.

성공하면 익명 또는 특정 FTP 계정이 읽거나 쓸 수 있는 원격 경로와 파일을 얻는다. 로그인 실패 시 익명 허용·사용자 형식·TLS 요구를, 목록·전송 실패 시 현재 경로 권한과 Active/Passive 데이터 채널을 다시 확인하며, 파일 쓰기만으로 웹·서비스 실행 권한을 단정하지 않는다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | 익명 로그인과 현재 경로 | `ftp anonymous@<TARGET>` 또는 `ftp <TARGET>` 후 `pwd`, `status` | `Anonymous login successful`과 실제 디렉터리 접근 범위를 확인한다. |
| 2 | 목록·읽기·쓰기 권한 | FTP 세션에서 `ls`, `get`, `mget`, `put` | 익명/인증 사용자별 파일 읽기와 쓰기를 따로 구분한다. |
| 3 | STARTTLS와 인증서 | `openssl s_client -connect <TARGET>:21 -starttls ftp` | 평문 FTP인지, TLS 전환이 필요한지, 인증서 이름이 무엇인지 구분한다. |
| 4 | 익명 대량 읽기 범위 | `wget -m --no-passive ftp://anonymous:anonymous@<TARGET>` | 다운로드 가능한 파일과 디렉터리 경계를 확인한다. |

### 선택적 파일 쓰기 proof

FTP 세션에서 파일 쓰기를 확인해야 할 때는 기존 파일과 겹치지 않는 `<UNIQUE_PROOF>`를 사용한다. 업로드 파일을 다시 내려받아 내용이나 hash를 확인하고, 같은 세션에서 정확한 파일 하나를 삭제할 수 있을 때만 수행한다.

```bash
printf 'ftp write proof\n' > <LOCAL_PROOF>
ftp <TARGET>
ftp> put <LOCAL_PROOF> <UNIQUE_PROOF>
ftp> get <UNIQUE_PROOF> <DOWNLOADED_PROOF>
ftp> delete <UNIQUE_PROOF>
ftp> ls <UNIQUE_PROOF>
```

- `put` 성공만으로 서버 저장을 확정하지 않는다. 재다운로드한 파일의 내용 또는 hash가 원본과 일치해야 한다.
- 삭제 뒤 `<UNIQUE_PROOF>`가 더 이상 조회되지 않는지 확인한다.
- 업로드는 가능하지만 삭제가 거부되고 서버 측 정리 경로도 없으면 쓰기 proof를 만들지 않는다.

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| Anonymous 로그인 또는 READ 가능한 디렉터리 | [[FTP 익명 접근과 파일 수집]] | `ftp`, `wget` | 익명 읽기 범위와 설정·백업·문서·소스 코드 확보 |
| WRITE 가능한 디렉터리 | 이 문서의 선택적 파일 쓰기 proof | `ftp` | 검증된 FTP 경로의 파일 쓰기, 실행 가능성은 별도 확인 |
| SSH private key 발견 | [[SSH credential 및 키 인증 검증]] | `ssh` | 해당 키 주체의 SSH 인증 가능성 |
| STARTTLS 또는 평문 인증, 확보한 사용자 이름·비밀번호 | [[FTP 익명 접근과 파일 수집]] | `openssl`, `ftp` | 유효 FTP 계정과 인증된 사용자의 디렉터리·파일 권한 |
| 취약 제품·버전 단서 | [[Public Exploit 검토와 검증]] | `searchsploit`, `metasploit` | 배포판 패치까지 반영한 재현 가능한 취약점 후보 |
| FTP 로그인과 `PORT` 명령을 이용한 제3자 연결 허용 | [[FTP Bounce]] | `nmap` | FTP 서버에서 도달 가능한 내부 포트 단서 |
| CoreFTP before build 727, 별도 HTTP PUT endpoint, 인증 계정 | [[CoreFTP HTTP PUT Path Traversal]] | `curl` | 기본 업로드 디렉터리 밖의 파일 쓰기 |

## 서비스 고유 주의 사항

- Active/Passive 모드 차이로 제어 채널은 열려도 데이터 채널이 막힐 수 있다.
- 시스템 계정이어도 `/etc/ftpusers` 정책으로 FTP 로그인이 차단될 수 있다.
- 익명 접근, 계정 인증, 파일 READ, 파일 WRITE, 업로드 파일 실행 가능성은 서로 다른 상태다.
