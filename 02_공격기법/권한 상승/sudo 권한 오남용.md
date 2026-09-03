---
tags:
  - 환경/linux
시작조건: ["Linux shell", "sudo 정책 조회 가능"]
필요권한: ["sudo 실행 권한"]
필요조건: ["Linux shell", "sudo 정책상 요구되는 경우 사용자 비밀번호"]
결과: ["root shell", "다른 사용자 shell", "명령 실행"]
---

# sudo 권한 오남용

## 한 줄 판단

현재 Linux 셸에서 `sudo -l`에 표시된 명령을 비밀번호 없이 또는 보유한 현재 사용자 비밀번호로 실행할 수 있다면, 해당 프로그램의 기능을 이용해 root·다른 사용자 셸이나 제한된 고권한 파일 접근을 얻는다.

## 사용할 때

- Linux shell을 얻었고 사용자 비밀번호가 있거나 NOPASSWD sudo가 의심될 때.
- 특정 binary를 root 또는 다른 사용자로 실행할 수 있을 때.
- 제한된 명령이 파일 읽기/쓰기/명령 실행 기능을 포함할 때.

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| sudo 목록 | `sudo -l` | 실행 가능 명령 표시 |
| 비밀번호 필요 여부 | NOPASSWD/프롬프트 | 실행 가능 |
| 악용 가능성 | GTFOBins/명령 기능 | shell/read/write/exec 가능 |

## 실행
### sudo 권한 확인

```bash
sudo -l
```

확인할 출력:

- `(ALL) ALL`, `NOPASSWD`, 특정 binary, 실행 대상 사용자.

### 전체 sudo 권한

```bash
sudo su -
sudo -i
```

확인할 출력:

- `whoami`가 `root`.

### 특정 사용자로 실행

`sudo -l`에서 대상 사용자로 `id` 또는 셸을 실행할 수 있다고 표시되는 경우에만 해당 명령을 사용한다.

```bash
sudo -u <USER> id
sudo -u <USER> /bin/bash
```

확인할 출력:

- `id`의 effective UID 또는 새 셸의 `whoami`가 대상 사용자와 일치하는지.

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `sudo -l`에 `(ALL) ALL` 또는 목적 사용자 대상 명령이 표시됨 | 실행 가능한 sudo 경로가 있음 | 고권한 명령 실행 후보 | 허용 사용자와 비밀번호 요구 여부를 확인 |
| 실행 뒤 `whoami`가 `root` 또는 지정 사용자로 바뀜 | sudo 정책을 통해 컨텍스트가 전환됨 | root shell 또는 다른 사용자 shell | 새 권한과 접근 가능한 파일을 다시 열거 |
| shell은 없지만 허용 명령으로 파일 읽기·쓰기 또는 하위 명령 실행이 됨 | 제한된 sudo 기능을 악용할 수 있음 | 고권한 파일 접근 또는 명령 실행 | 허용 기능 범위에서 후속 기법 선택 |
| `password required`이고 비밀번호가 없음 | 현재 sudo 경로를 실행할 수 없음 | 기존 일반 사용자 shell 유지 | `NOPASSWD` 항목 또는 자격 증명 수집 확인 |
| `command not allowed` 또는 환경 변수가 제거됨 | sudoers·`secure_path`·환경 정책이 제한함 | 허용 명령 범위만 유지 | 절대 경로, wildcard, 해당 binary의 GTFOBins 조건 확인 |

## 확인할 출력과 권한

| 확인 지점 | 확인할 출력 | 권한 판단 |
|---|---|---|
| sudo 정책 | `sudo -l`의 실행 대상 사용자, `NOPASSWD`, 허용 경로 | root와 다른 사용자 권한을 구분 |
| 실행 후 identity | `whoami`, `id`, 생성 파일 소유자 | 프롬프트 모양이 아니라 실제 사용자와 effective 권한으로 확정 |
| 제한 명령 | root 전용 파일 접근 또는 하위 명령 결과 | shell과 단일 고권한 기능을 별도 결과로 구분 |

## 후속 공격 연결

- [[Linux 파일 자격증명 검색]]
- [[Linux Shell History 자격증명 검색]]
- [[Linux 개인키 검색]]
- [[상황별 파일 전송]]

## 관련 상태 라우터

- root 실행 컨텍스트를 확인했으면: [[고권한 세션 확보 후 후속 판단]]
- 다른 사용자 셸 또는 제한된 명령 실행만 확보했으면: [[Linux 셸 확보 후 초기 열거와 권한 상승]]

## 관련 도구

- [[linpeas]]
- [[LinEnum]]
