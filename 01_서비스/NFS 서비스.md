---
tags:
  - 환경/unix
  - 서비스/nfs
대표포트:
  - "T:111"
  - "U:111"
  - "T:2049"
  - "U:2049"
서비스:
  - NFS
  - rpcbind
---

# NFS 서비스

## 발견 시 판단

현재 명령 실행 호스트에서 `<TARGET>`의 Network File System(NFS) 또는 Remote Procedure Call(RPC) 등록 서비스인 rpcbind에 도달할 수 있고, 아직 export 마운트나 서버 파일 권한은 확인하지 않은 상태에서 시작한다. NFSv3의 mountd와 NFSv4 pseudo-root, export별 허용 source 대역, 마운트, 읽기·쓰기, 숫자 사용자·그룹 식별자(UID/GID)와 squash 정책을 구분한다.

로컬 `sudo`는 NFS 마운트에 필요한 실행 호스트 권한일 뿐 서버 root 권한이 아니다. 성공하면 export 범위의 파일 읽기·쓰기와 서버 측 UID/GID 매핑을 얻고, `showmount` 실패 시 NFSv4-only 여부를, `access denied` 시 export·source 대역·버전을, 마운트 후 거부 시 서버 파일 권한과 squash를 다시 확인한다.

## 서비스 고유 확인

| 우선순위 | 현재 가진 정보로 확인할 것 | 도구 | 확인 출력과 다음 판단 |
|---|---|---|---|
| 1 | NFS 프로그램·버전·mountd | `nmap --script nfs-showmount,nfs-ls,nfs-statfs -sV -p111,2049 <TARGET>` | v3의 MNT/mountd 노출과 v4-only 가능성을 구분한다. |
| 2 | v3 export와 허용 대역 | `showmount -e <TARGET>` | MNT가 응답하면 마운트 가능한 경로와 source 제한을 확인한다. |
| 3 | v3/v4 마운트 | `sudo mount -t nfs -o vers=3 <TARGET>:/<EXPORT> ./target-NFS` 또는 `sudo mount -t nfs -o vers=4 <TARGET>:/<EXPORT> ./target-NFS` | v3 export 또는 v4 pseudo-root에서의 마운트 성공과 경계를 확인한다. |
| 4 | 이름·숫자 소유권과 접근 | `ls -la`, `ls -n` | 사용자·그룹명, UID/GID와 실제 READ/WRITE 차이를 비교한다. |
| 5 | 무해한 쓰기 확인 | 고유한 작은 테스트 파일 생성 후 `cat`과 `ls -n`으로 재확인 | 생성·재조회·소유자 매핑이 모두 확인될 때만 WRITE로 판단한다. |

### 선택적 파일 쓰기와 UID·GID 매핑 proof

읽기 전용 마운트를 먼저 해제하고, 쓰기·소유자 매핑을 확인해야 할 때만 같은 export를 `rw`로 다시 마운트한다.

`<3_OR_4>`는 확인한 NFS 버전(예: `3`), `<TARGET>`은 NFS 서버 IP 또는 FQDN(예: `192.0.2.10`), `<EXPORT>`는 `showmount` 또는 NFSv4 pseudo-root에서 확인한 export 절대 경로(예: `/srv/share`)다. 이 명령은 현재 실행 호스트의 `./target-NFS` 마운트 지점에서 고유 파일을 만들고 삭제한다.

```bash
sudo umount ./target-NFS
sudo mount -t nfs -o vers=<3_OR_4>,rw,nosuid,nodev <TARGET>:/<EXPORT> ./target-NFS
PROOF="nfs-proof-$(date -u +%Y%m%dT%H%M%SZ)-$$.txt"
printf 'NFS write proof: %s\n' "$PROOF" > "./target-NFS/$PROOF"
stat -c '%n %s bytes uid=%u gid=%g mode=%a' "./target-NFS/$PROOF"
rm -f -- "./target-NFS/$PROOF"
test ! -e "./target-NFS/$PROOF" && echo 'proof removed'
sudo umount ./target-NFS
```

- `findmnt`에서 실제 mount mode가 `rw`인지 먼저 확인한다.
- root로 만든 파일의 UID가 anonymous UID로 바뀌면 `root_squash`가 적용된 것이다.
- 삭제가 거부되면 정확한 proof 경로와 숫자 소유자를 확인하고 해당 UID 또는 서버 관리 경로로 제거한 뒤 마운트를 해제한다.

## 단서별 다음 경로

| 관찰 단서·현재 권한 | 지금 가능한 기법 | 도구 | 성공 결과 |
|---|---|---|---|
| export 목록 또는 마운트 성공 | [[NFS export 마운트와 권한 매핑 검증]] | `showmount`, `mount` | export별 허용 대역과 마운트 가능 상태 |
| READ 가능한 export | [[NFS export 마운트와 권한 매핑 검증]] | `mount`, `find`, `grep` | 문서·백업·설정 파일의 읽기 |
| WRITE 가능한 export | 이 문서의 선택적 파일 쓰기와 UID·GID 매핑 proof | `mount` | export 범위의 검증된 파일 쓰기 |
| `rw`, `no_root_squash`, UID/GID 또는 SUID 단서 | 이 문서의 선택적 파일 쓰기와 UID·GID 매핑 proof | `mount` | 서버 측 squash와 실행 경로까지 충족한 권한 영향 후보 |
| 홈 디렉터리 또는 SSH private key 노출 | [[SSH credential 및 키 인증 검증]] | `mount`, `ssh` | 해당 키 주체의 SSH 인증 가능성 |

## 서비스 고유 주의 사항

- NFSv3의 `showmount`는 MNT 서비스에 의존한다. v4-only 서버는 MNT/export 목록을 노출하지 않을 수 있으므로 `showmount` 실패만으로 NFS 접근 불가로 단정하지 않는다.
- rpcbind, mountd, nlockmgr 등 동적 포트는 주로 v3에서 함께 필요할 수 있다.
- export가 특정 source IP나 대역에만 허용될 수 있다.
- `root_squash`가 있으면 클라이언트 root가 서버의 root 권한으로 유지되지 않는다.
- 마운트 성공은 export 진입만 증명하며 파일별 `Permission denied`는 UID/GID 매핑과 서버 파일 모드·ACL을 따로 확인한다.
- 테스트 후에는 `sudo umount ./target-NFS`로 마운트를 해제한다.

## 참고 링크

- [nfs(5) — Linux NFS client options](https://man7.org/linux/man-pages/man5/nfs.5.html)
- [exports(5) — NFS export and identity mapping options](https://man7.org/linux/man-pages/man5/exports.5.html)
- [showmount(8) — MNT service and NFSv4-only limitation](https://man7.org/linux/man-pages/man8/showmount.8.html)
