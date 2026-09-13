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

## 전제 조건

| 조건 | 확인 방법 | 충족 기준 |
|---|---|---|
| sudo 목록 | `sudo -l` | 실행 가능 명령 표시 |
| 비밀번호 필요 여부 | NOPASSWD/프롬프트 | 실행 가능 |
| 허용 실행 주체 | `sudo -l`의 `(RUNAS)` | root 또는 목표 사용자 확인 |
| 명령·인자 범위 | 허용된 절대 경로와 뒤따르는 인자 패턴 | 대표 명령의 인자를 정책이 허용함 |
| 악용 가능성 | GTFOBins/명령 기능 | shell/read/write/exec 중 실제 허용 기능 확인 |

## 실행
### sudo 권한 확인

이 블록은 대상 Linux 셸의 현재 사용자로 실행한다. 출력의 runas 사용자·절대 명령 경로·인자 규칙·`NOPASSWD` 여부를 다음 블록의 입력으로 그대로 사용하며, `sudo -l` 성공은 root shell을 뜻하지 않는다.

```bash
sudo -l
```

확인할 출력:

- `(ALL) ALL`, `NOPASSWD`, 특정 binary, 실행 대상 사용자.
- 경로 뒤에 인자가 없으면 sudoers에서는 원칙적으로 사용자가 임의 인자를 지정할 수 있다. `""`가 있으면 인자 없이만 실행할 수 있고, 특정 인자·wildcard·정규식이 있으면 실제 호출 전체가 그 규칙과 일치해야 한다.
- `sudo -l`에 보인 basename과 임의의 다른 경로를 바꾸어 쓰지 않는다. 허용된 절대 경로, runas 사용자, tag와 인자 범위를 한 행으로 읽는다.

### 전체 sudo 권한

```bash
sudo su -
sudo -i
```

확인할 출력:

- `whoami`가 `root`.

### 특정 사용자로 실행

`sudo -l`에서 대상 사용자로 `id` 또는 셸을 실행할 수 있다고 표시되는 경우에만 해당 명령을 사용한다.

`<USER>`는 바로 앞 `sudo -l`의 `(RUNAS)`에 표시된 사용자명(가상 예: `backup`)이며, 임의의 계정명으로 바꾸지 않는다.

```bash
sudo -u <USER> id
sudo -u <USER> /bin/bash
```

확인할 출력:

- `id`의 effective UID 또는 새 셸의 `whoami`가 대상 사용자와 일치하는지.

### 특정 허용 바이너리: `find` 대표 분기

아래 분기는 `sudo -l`에 `<RUNAS_USER>`로 실행 가능한 정확한 `/usr/bin/find`가 표시되고, 규칙이 임의 인자를 허용하거나 아래 인자 전체와 일치할 때만 사용한다. 단순히 시스템에 `find`가 설치됐다는 사실은 전제 조건이 아니다.

```text
Matching Defaults entries for <CURRENT_USER> on <HOST>:
    ...
User <CURRENT_USER> may run the following commands on <HOST>:
    (<RUNAS_USER>) NOPASSWD: /usr/bin/find
```

대상 Linux 셸에서 허용된 runas 사용자와 경로를 그대로 지정한다. `<RUNAS_USER>`는 `sudo -l`의 `(RUNAS)`에 나온 사용자명(가상 예: `root`)이며, `find`가 허용된 권한을 유지한 채 하위 셸을 실행하는지 먼저 `id`로 확인한다.

```bash
sudo -u <RUNAS_USER> /usr/bin/find . -exec /bin/sh -c 'id; exec /bin/sh' \; -quit
```

확인할 출력:

- 첫 `id`의 effective UID·그룹이 `<RUNAS_USER>`와 일치해야 그 사용자 컨텍스트의 명령 실행을 확정한다. `<RUNAS_USER>`가 root일 때만 root shell이다.
- 비밀번호 프롬프트가 나오면 `NOPASSWD`가 아니므로 보유한 현재 사용자 비밀번호가 필요한지 다시 확인한다.
- `command not allowed`가 나오면 sudoers의 절대 경로·runas·인자 패턴이 호출과 다르다. 허용되지 않은 변형을 계속 시도하지 말고 `sudo -l` 원문을 다시 비교한다.
- 셸이 생성되지 않으면 `NOEXEC`, AppArmor·SELinux, 실행 파일의 실제 기능과 셸 제약을 확인한다. `find` 실행 성공만으로 상승된 셸을 얻었다고 판단하지 않는다.

다른 특정 바이너리는 목록을 순서대로 대입하지 않는다. 허용 규칙과 실제 버전을 먼저 확정한 뒤 다음 기능 중 필요한 하나가 공식 문서·GTFOBins에 있는 경우에만 해당 분기로 바꾼다.

| 바이너리 기능 | 선택 결과 | 추가로 확인할 조건 |
|---|---|---|
| 하위 명령·셸 실행 | 목표 사용자 명령 실행 또는 셸 | runas, 허용 인자, privilege drop·`NOEXEC` |
| pager·editor 탈출 | 대화형 셸 후보 | TTY, 사용되는 pager/editor와 환경 변수 보존 여부 |
| 파일 읽기 | 목표 사용자 권한의 제한된 파일 접근 | 정확한 대상 파일과 출력 범위; 셸 획득과 구분 |
| 파일 쓰기 | 목표 사용자 권한의 상태 변경 | 기존 내용·owner·mode 보존 및 해당 기법의 복구 절차가 준비된 경우만 실행 |

## 관찰과 상태 전환

| 관찰 | 판단 | 결과 상태 | 다음 행동 |
|---|---|---|---|
| `sudo -l`에 `(ALL) ALL` 또는 목적 사용자 대상 명령이 표시됨 | 실행 가능한 sudo 경로가 있음 | 고권한 명령 실행 후보 | 허용 사용자와 비밀번호 요구 여부를 확인 |
| 실행 뒤 `whoami`가 `root` 또는 지정 사용자로 바뀜 | sudo 정책을 통해 컨텍스트가 전환됨 | root shell 또는 다른 사용자 shell | 새 권한과 접근 가능한 파일을 다시 열거 |
| shell은 없지만 허용 명령으로 파일 읽기·쓰기 또는 하위 명령 실행이 됨 | 제한된 sudo 기능을 악용할 수 있음 | 고권한 파일 접근 또는 명령 실행 | 허용 기능 범위에서 후속 기법 선택 |
| 허용된 `/usr/bin/find`에서 `id`가 `<RUNAS_USER>`를 표시함 | 특정 바이너리의 하위 명령 기능이 runas 권한으로 실행됨 | `<RUNAS_USER>` 명령 실행 또는 shell | 목표 사용자의 실제 권한·파일 접근 범위를 다시 확인 |
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

## 참고 링크

- [sudoers 공식 매뉴얼](https://www.sudo.ws/docs/man/1.9.14/sudoers.man.pdf)
- [GTFOBins find의 sudo 동작](https://gtfobins.org/gtfobins/find/)
