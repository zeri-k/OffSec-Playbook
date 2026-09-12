---
tags:
  - 서비스/http
대표포트:
  - "T:80"
  - "T:443"
  - "T:8000"
  - "T:8008"
  - "T:8080"
  - "T:8081"
  - "T:8443"
  - "T:8888"
  - "T:9000"
서비스:
  - HTTP
  - HTTPS
---

# HTTP와 HTTPS 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>:80`의 Hypertext Transfer Protocol(HTTP) 또는 `<TARGET>:443`의 HTTP over TLS(HTTPS)에 요청을 보낼 수 있고, 아직 유효한 웹 계정이나 애플리케이션 권한은 확인하지 않은 상태에서 시작한다. IP와 Host header·Server Name Indication(SNI)별 응답, 기술 스택, Web Application Firewall(WAF)·Content Delivery Network(CDN), 숨은 경로와 인증·인가 경계를 먼저 분리한다.

**첫 화면 상태:** 지금 가능한 일은 실제 응답, virtual host(vhost), 인증 경계와 기능을 확인하는 것이다. 성공하면 수동 검증할 경로·입력값·계정 후보를 얻는다. 연결·TLS 실패는 포트·SNI·인증서를, 401은 인증 방식과 계정을, 403은 현재 주체·Host·경로의 인가를, 열거 결과가 모두 같으면 wildcard·custom 404를 다시 확인한다. 상태 코드나 도구 지문만으로 취약점·파일 쓰기·명령 실행은 확정되지 않는다.

## 서비스 고유 확인

| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | 헤더, 리다이렉션, 쿠키 | `curl -I http://<TARGET>/`, `curl -k -I https://<TARGET>/` | Server, Location, Set-Cookie와 IP/Host 응답 차이를 확인한다. |
| 2 | TLS 호스트 이름과 SNI | `openssl s_client -connect <TARGET>:443 -servername <HOST>` | CN/SAN, 조직명과 접속해야 할 vhost 후보를 수집한다. |
| 3 | 기술 스택과 WAF/CDN | `whatweb http://<TARGET>/`, `wafw00f http://<TARGET>/` | CMS·프레임워크·언어·버전과 응답 차단·변조 가능성을 구분한다. |
| 4 | 숨은 경로와 상태 코드 | `gobuster dir -u http://<TARGET>/ -w <WORDLIST>` | 200/301/302/401/403, wildcard와 custom 404를 비교한다. |
| 5 | 링크·JS·폼·단어 | `./finalrecon.py --headers --sslinfo --crawl --url https://<HOST>/`, `python3 ReconSpider.py http://<TARGET>/`, `cewl -d 2 -m 5 -w words.txt http://<TARGET>/` | API, 토큰, 내부 URL, 이메일, 폼 필드와 vhost·wordlist 후보를 수집한다. |
| 6 | 알려진 파일·구성 후보 | `nikto -h http://<TARGET>/` | 기본 파일과 위험 설정 후보를 `curl` 또는 브라우저로 수동 재현한다. |
| 7 | AD CS Web Enrollment | `curl -k -I https://<TARGET>/certsrv/` | `/certsrv`, NTLM 인증, redirect와 relay 보호 조건 확인 필요성을 판단한다. |

**출력 해석 경계:** 헤더·TLS 인증서·Host별 응답은 해당 요청의 웹 endpoint와 vhost 후보를 확정하지만 backend 주소나 권한 있는 접근을 확정하지 않는다. 200/301/302/401/403은 응답·인증 경계의 관찰이며 기능 사용 권한이나 파일 존재를 단독으로 증명하지 않는다. `nikto`, `gobuster`, WAF 지문과 `/certsrv` 노출은 후보이므로 수동 재현과 별도 권한 검증이 필요하다.


## 대체 포트에서 추가로 확인할 것

대체 포트의 HTTP·HTTPS도 같은 서비스 문서에서 다룬다. 80·443과 응답이 다를 때에만 관리 콘솔, API 문서, debug 기능, 프록시 동작처럼 포트별 차이를 별도로 확인한다.

## 대체 포트별 차이 확인


| 우선순위 | 확인할 것 | 명령·도구 | 다음 판단 |
|---|---|---|---|
| 1 | 80/443과 포트별 애플리케이션 차이 | `curl -skI http://<TARGET>:<PORT>/` | title, Server, Location, 쿠키와 Host 처리 차이를 기본 포트 응답과 비교한다. |
| 2 | 관리·API·debug 기능 | `gobuster dir -u http://<TARGET>:<PORT>/ -w <WORDLIST>` | admin, manager, docs, swagger, api, actuator, debug처럼 대체 포트 고유 경로를 확인한다. |
| 3 | 프록시 또는 별도 관리 호스트 | `curl -i -X OPTIONS http://<TARGET>:<PORT>/`, `openssl s_client -connect <TARGET>:<PORT> -servername <HOST>` | 허용 메서드, SNI와 proxy 처리 차이를 실제 쓰기·내부 접근과 구분한다. |

**출력 해석 경계:** 포트별 header, title, TLS 인증서의 차이는 별도 endpoint 또는 vhost 후보를 확정하지만 별도 서버·관리자 권한을 확정하지 않는다. Swagger, actuator, debug의 응답은 공개된 메타데이터·경로만 확정하며 민감 정보의 유효성은 수동 요청으로 다시 확인한다. OPTIONS의 `PUT` 또는 프록시 형태 응답은 허용 메서드·동작 후보일 뿐 원격 파일 쓰기·내부 네트워크 접근을 증명하지 않는다.

## 단서별 다음 경로

| 관찰한 단서 | 다음 공격기법 | 주요 도구 | 예상 결과 상태 |
|---|---|---|---|
| Server, title, 기술 스택 | [[웹 지문 확인과 공격면 분류]] | `curl`, `whatweb` | 제품·버전과 수동 검증할 기능 후보 |
| CN/SAN, 301/302, Host 응답 차이 | [[웹 정찰과 경로 열거]] | `openssl`, `curl`, `gobuster` | 유효 vhost와 별도 애플리케이션 후보 |
| 200/301/401/403 경로 또는 백업·설정 파일 | [[웹 숨은 경로와 민감 파일 열거]] | `gobuster`, `curl` | 접근 가능한 숨은 경로·민감 파일 또는 인증 경계 |
| 링크, JS, 폼, API 단서 | [[웹 단서 기반 기능 열거]] | `FinalRecon`, `ReconSpider`, `cewl` | 엔드포인트·파라미터·토큰·계정 후보 |
| `/certsrv` 또는 CA Web Enrollment | [[AD CS ESC8 NTLM Relay]] | `curl`, `impacket-ntlmrelayx` | 인증 방식·template·발급 주체 권한을 포함한 ESC8 후보 |
| NTLM 인증 요구 | [[NTLM Relay 조건 검토]] | `curl`, `impacket-ntlmrelayx` | 보호 설정과 relay 계정 권한이 반영된 대상 후보 |
| 로그인 화면·SSO·사용자별 권한 차이 | [[웹 단서 기반 기능 열거]] | 브라우저/프록시, `curl` | 인증 방식, 사용자 형식, 잠금 정책과 재현된 인가 경계 |
| 소스·응답·문서에서 얻은 credential과 대상 서비스 | [[웹 단서 기반 기능 열거]] | 브라우저/프록시, `curl` | credential의 대상 프로토콜을 확정해 해당 서비스 카드에서 별도 검증 |
| OPTIONS에서 PUT/WebDAV 메서드 | [[WebDAV PUT 업로드와 실행]] | `curl`, `nmap` | 검증된 경로의 파일 쓰기와 별도 실행 가능성 |
| 업로드·검색·다운로드 기능 | [[웹 파일 업로드와 Web Shell]] | 브라우저/프록시, `curl` | 파일 처리 또는 입력값 검증 후보와 실제 영향 |
| DB와 웹 루트 경로 연결 | 수동 확인: DB 파일 쓰기와 HTTP 정적 조회를 분리한 뒤 [[웹 파일 업로드와 Web Shell]]에서 서버 측 실행 확인 | DB client, `curl` | DB 권한으로 쓴 파일의 웹 접근과 별도 서버 측 실행 여부 |

## 서비스 고유 주의 사항

- IP 접속과 Host 헤더 접속 화면이 다를 수 있고, HTTPS는 SNI와 Host를 함께 맞춰야 한다.
- 403은 보호된 리소스 단서일 수 있으며 접근 성공을 뜻하지 않는다.
- WAF/CDN은 `nikto`, `gobuster`와 자동 정찰 결과를 차단하거나 왜곡할 수 있다.
- `/certsrv` 노출은 단독 취약점이 아니며 NTLM relay 조건, template와 발급 주체 권한을 함께 확인한다.

## 참고 링크

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9114: HTTP/3](https://www.rfc-editor.org/rfc/rfc9114)
