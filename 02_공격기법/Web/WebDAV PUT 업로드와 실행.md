---
tags:
  - 서비스/http
시작조건: ["HTTP 또는 HTTPS 서비스 식별", "PUT 또는 WebDAV 메서드 확인"]
필요권한: ["PUT/WebDAV 업로드 권한"]
필요조건: ["업로드 경로 접근 가능"]
결과: ["파일 쓰기", "웹 파일 접근", "서버 측 실행 후보"]
---

# WebDAV PUT 업로드와 실행

## 한 줄 판단

현재 명령 실행 위치에서 대상 WebDAV 경로에 HTTP 요청을 보낼 수 있고 `PUT` 또는 `MOVE`가 허용된다면, 고유한 시험 파일로 업로드·이동·HTTP 조회를 확인한다. 실행 가능한 확장자와 handler가 확인되면 서버 측 실행은 [[웹 파일 업로드와 Web Shell]]에서 별도로 검증한다.

## 시작 조건 해석


- `OPTIONS`에서 `PUT`, `MOVE`, `COPY`, `DELETE`, WebDAV 메서드가 보일 때.
- Nginx/Apache/IIS에서 업로드 디렉터리가 잘못 노출된 것 같을 때.
- 파일 쓰기 영향과 Web Shell 실행 가능성을 구분해야 할 때.

Nginx의 `ngx_http_dav_module`은 기본 빌드에 포함되지 않으며 `PUT`·`DELETE`·`MKCOL`·`COPY`·`MOVE`만 처리한다. 따라서 일부 WebDAV 메서드가 보이거나 `PUT`이 성공해도 완전한 WebDAV 기능이나 서버 측 실행을 뜻하지 않는다.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| 메서드 허용 | `curl -X OPTIONS` | `PUT` 또는 WebDAV 메서드 표시 |
| 업로드 경로 | `PUT` 테스트 | 파일 생성 성공 |
| 실행 경로 | HTTP GET/handler | 파일 조회 또는 코드 실행 |

## 실행

1. `OPTIONS`로 허용 메서드를 확인한다.
2. 식별 가능한 테스트 파일을 `PUT`으로 업로드한다.
3. `GET`으로 파일 접근 가능성을 확인한다.
4. 실행 가능한 확장자와 handler가 있으면 [[웹 파일 업로드와 Web Shell]]로 넘어간다.

### 명령과 확인할 출력

#### 메서드 확인

```bash
curl -i -X OPTIONS http://<TARGET>/<PATH>/
nmap --script http-methods -p80,443 <TARGET>
```

확인할 출력:

- `Allow: GET, HEAD, POST, OPTIONS, PUT, DELETE, MOVE`.

#### PUT 업로드

```bash
test ! -e '<LOCAL_PROOF>'
printf 'webdav write proof\n' > '<LOCAL_PROOF>'
curl -i 'http://<TARGET>/<PATH>/<UNIQUE_PROOF>.txt'
curl -i -X PUT --data-binary '@<LOCAL_PROOF>' 'http://<TARGET>/<PATH>/<UNIQUE_PROOF>.txt'
curl -i 'http://<TARGET>/<PATH>/<UNIQUE_PROOF>.txt'
```

확인할 출력:

- PUT 전 조회에서 `404`·`410`처럼 해당 고유 이름이 없음을 확인한다. 공통 오류 페이지가 `200`을 반환하면 응답 본문·길이를 기준 응답과 비교한다.
- 업로드 성공 status와 파일 내용 조회.

#### MOVE로 확장자 변경

```bash
curl -i -X MOVE -H 'Destination: http://<TARGET>/<PATH>/<UNIQUE_PROOF>.php' 'http://<TARGET>/<PATH>/<UNIQUE_PROOF>.txt'
```

확인할 출력:

- 파일 이동/이름 변경 성공 여부. 이 결과는 확장자 handler가 실행된다는 뜻이 아니며, 새 URL의 GET 응답과 서버 측 명령 출력은 별도로 확인한다.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| 인증 없이 또는 낮은 권한으로 파일이 업로드된다. | WebDAV 쓰기 권한 확인 | 파일 쓰기 | 업로드 URL에 GET 요청을 보내 저장 위치 확인 |
| 업로드 파일이 웹에서 접근 가능하다. | 웹 접근 가능한 파일 쓰기 확인 | 웹 파일 접근 | 서버 스택과 확장자 handler를 확인 |
| 실행 가능한 확장자와 서버 측 handler가 확인된다. | Web Shell 검증 전제 확인 | 서버 측 실행 후보 | [[웹 파일 업로드와 Web Shell]] |
| PUT 403 | 메서드 표시만 되고 경로 제한 | 시작 상태 유지 | 다른 경로, 인증 필요 여부 |
| PUT 성공, GET 실패 | 저장 경로가 비공개 | 파일 쓰기 | 웹 접근과 코드 실행은 미확정으로 유지 |
| 코드 미실행 | 정적 파일 처리 | 시작 상태 유지 | handler/확장자/서버 스택 확인 |

## 확인할 출력과 권한

- 판정 기준: PUT 성공, MOVE 결과와 GET 조회를 분리한다. 서버 측 코드 실행과 웹 서버 계정은 [[웹 파일 업로드와 Web Shell]]에서 확인한다.

## 변경 영향과 복구

업로드 전 대상 URL에 같은 이름이 없는지 확인하고 `<UNIQUE_PROOF>`를 사용한다. `<TARGET>`은 WebDAV 서버의 IP 또는 FQDN(가상 예: `192.0.2.20`), `<PATH>`는 해당 서버의 URL 경로(가상 예: `dav`), `<LOCAL_PROOF>`는 실행 호스트의 새 파일 경로다. `MOVE`를 사용했다면 원래 이름이 아니라 조회로 실제 존재를 확인한 최종 destination을 제거한다. 아래 `<FINAL_REMOTE_NAME>`은 MOVE를 쓰지 않았거나 MOVE가 실패했으면 `<UNIQUE_PROOF>.txt`, MOVE와 새 URL 조회가 성공했으면 `<UNIQUE_PROOF>.php`다.

```bash
curl -i -X DELETE 'http://<TARGET>/<PATH>/<FINAL_REMOTE_NAME>'
curl -i 'http://<TARGET>/<PATH>/<FINAL_REMOTE_NAME>'
rm -- '<LOCAL_PROOF>'
test ! -e '<LOCAL_PROOF>'
```

확인할 출력:

- `DELETE` 성공 뒤 `GET`에서 `404`·`410`처럼 파일이 더 이상 반환되지 않는다. 인증 화면·공통 오류 페이지의 `200`을 삭제 성공으로 해석하지 않는다.
- `DELETE`가 허용되지 않으면 WebDAV 관리 경로 또는 확보한 서버 셸로 정확한 파일을 제거한다.

## 후속 공격 연결

- [[웹 파일 업로드와 Web Shell]]
- [[상황별 파일 전송]]

## 관련 서비스

- [[HTTP와 HTTPS 서비스]]

## 관련 도구

- [[curl]]
- [[nmap]]
- [[metasploit]]

## 참고 링크

- [RFC 4918: WebDAV](https://www.rfc-editor.org/rfc/rfc4918)
- [NGINX ngx_http_dav_module](https://nginx.org/en/docs/http/ngx_http_dav_module.html)
