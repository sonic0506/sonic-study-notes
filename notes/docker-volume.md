---
aliases: [Docker Volume, 도커 볼륨]
tags: [docker, devops]
prerequisites:
  - "[[docker]]"
related: []
status: draft
created: 2026-10-01
---

# Docker Volume

## 1. 왜 Volume이 필요한가?

Container 안에서 만든 파일은 Container의 쓰기 가능한 레이어에 저장됩니다. 이 레이어는 Container와 생명주기가 같습니다.

```text
docker rm my-db
      ↓
Container 삭제
      ↓
Container 안에서 만든 데이터도 함께 삭제
```

데이터베이스를 Container로 띄웠는데 Container를 지울 때마다 데이터가 사라진다면 곤란합니다. 이미지를 새 버전으로 바꾸려면 Container를 지우고 새로 만들어야 하기 때문입니다.

**Volume**은 데이터를 Container 바깥에 저장해서, Container를 지워도 데이터가 남도록 하는 기능입니다.

```text
Container (지웠다 다시 만들어도 됨)
    │
    │ /var/lib/postgresql/data
    ↓
Volume (데이터는 여기에 계속 남음)
```

## 2. 데이터를 저장하는 세 가지 방법

[[docker|Docker]]에서 Container 바깥에 데이터를 두는 방법은 세 가지입니다.

```text
┌──────────────── 호스트 ────────────────┐
│                                         │
│  Docker 관리 영역       내 파일 시스템   │   메모리
│  ┌────────────┐        ┌──────────┐     │  ┌───────┐
│  │   Volume   │        │  ./src   │     │  │ tmpfs │
│  └─────┬──────┘        └────┬─────┘     │  └───┬───┘
└────────┼────────────────────┼───────────┘      │
         └──────────┬─────────┴──────────────────┘
                    ↓
               Container
```

| 방식 | 저장 위치 | 주 용도 |
|---|---|---|
| Volume | Docker가 관리하는 영역 | DB 데이터 등 보존할 데이터 |
| Bind Mount | 호스트의 내가 지정한 경로 | 개발 중 소스 코드 연결, 설정 파일 |
| tmpfs | 호스트 메모리 | 디스크에 남기면 안 되는 임시 데이터 |

### Volume

Docker가 만들고 관리합니다. 실제 파일 위치는 Linux 기준 `/var/lib/docker/volumes/` 아래이며, Docker Desktop에서는 Docker가 쓰는 가상 머신 안에 있습니다. 사용자는 위치를 몰라도 이름으로 다룰 수 있습니다.

### Bind Mount

호스트의 특정 경로를 그대로 Container에 연결합니다. 호스트에서 파일을 고치면 Container 안에서도 바로 바뀝니다.

### tmpfs

메모리에만 저장되고, Container가 멈추면 사라집니다. Linux에서만 쓸 수 있습니다.

## 3. Volume 사용하기

### 만들고 연결하기

```bash
docker volume create pg-data

docker run -d \
  --name my-db \
  -e POSTGRES_PASSWORD=secret \
  -v pg-data:/var/lib/postgresql/data \
  postgres:16
```

`-v 볼륨이름:Container경로` 형식입니다. 볼륨이 없으면 `docker run`이 자동으로 만들어 줍니다.

### 데이터가 남는지 확인

```bash
docker rm -f my-db

docker run -d \
  --name my-db-2 \
  -e POSTGRES_PASSWORD=secret \
  -v pg-data:/var/lib/postgresql/data \
  postgres:16
```

새 Container가 같은 `pg-data`를 연결하므로 이전 데이터를 그대로 사용합니다.

### 관리 명령어

```bash
docker volume ls               # 목록
docker volume inspect pg-data  # 상세 정보 (실제 저장 위치 등)
docker volume rm pg-data       # 삭제 (사용 중인 Container가 없어야 함)
docker volume prune            # 사용하지 않는 익명 볼륨 정리
docker volume prune --all      # 사용하지 않는 이름 있는 볼륨까지 정리
```

`docker volume prune`은 Docker 23부터 기본적으로 익명 볼륨만 지웁니다. `rm`, `prune`은 데이터를 지우므로 실행 전에 대상을 확인합니다.

## 4. Bind Mount 사용하기

```bash
docker run -d \
  -v $(pwd)/src:/app/src \
  my-app
```

경로가 `/`나 `./`로 시작하면 Bind Mount, 이름만 있으면 Volume으로 해석됩니다.

```text
-v pg-data:/data    → Volume
-v ./data:/data     → Bind Mount
-v /home/me/data:/data → Bind Mount
```

개발할 때 소스 코드를 Bind Mount로 연결하면, 이미지를 다시 빌드하지 않아도 코드 변경이 Container에 바로 반영됩니다.

## 5. -v와 --mount

같은 일을 `--mount`로도 할 수 있습니다. 더 길지만 각 값의 의미가 드러납니다.

```bash
# -v
docker run -v pg-data:/var/lib/postgresql/data postgres:16

# --mount
docker run --mount type=volume,source=pg-data,target=/var/lib/postgresql/data postgres:16
```

차이도 하나 있습니다. Bind Mount에서 호스트 경로가 없을 때 `-v`는 디렉터리를 새로 만들고, `--mount`는 오류를 냅니다. 경로 오타를 바로 알아차리고 싶다면 `--mount`가 안전합니다.

## 6. 읽기 전용으로 연결하기

Container가 파일을 읽기만 해야 한다면 `:ro`를 붙입니다.

```bash
docker run -v ./nginx.conf:/etc/nginx/nginx.conf:ro nginx
```

Container 안에서 이 파일을 수정하려고 하면 오류가 납니다.

## 7. 연결할 때 기존 파일은 어떻게 되나?

Container 경로에 이미 파일이 있는 상태에서 연결하면 방식에 따라 결과가 다릅니다.

| 상황 | 결과 |
|---|---|
| 빈 Volume을 연결 | 이미지의 기존 파일이 Volume으로 복사됨 |
| 내용이 있는 Volume을 연결 | Volume 내용이 보이고 이미지 파일은 가려짐 |
| Bind Mount를 연결 | 호스트 경로 내용이 보이고 이미지 파일은 가려짐 |

예를 들어 `/app`에 소스 코드가 들어 있는 이미지에 빈 호스트 디렉터리를 `/app`으로 Bind Mount하면, Container 안의 `/app`이 빈 디렉터리처럼 보입니다. "분명 이미지에 넣었는데 파일이 없다"는 문제의 흔한 원인입니다.

## 8. 익명 볼륨

이름 없이 Container 경로만 적으면 Docker가 무작위 이름으로 볼륨을 만듭니다.

```bash
docker run -v /data my-app
```

이름이 없어서 다른 Container에서 다시 연결하기 어렵습니다. `docker rm -v`로 Container를 지우거나 `docker run --rm`으로 실행한 경우에는 Container와 함께 삭제됩니다. 보존할 데이터에는 이름 있는 볼륨을 씁니다.

## 9. 백업과 복원

볼륨 내용은 임시 Container를 띄워 압축 파일로 꺼낼 수 있습니다.

```bash
# 백업: pg-data를 현재 디렉터리의 backup.tar.gz로
docker run --rm \
  -v pg-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz -C /data .

# 복원: backup.tar.gz를 pg-data로
docker run --rm \
  -v pg-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup.tar.gz -C /data
```

볼륨과 호스트 디렉터리를 동시에 연결한 뒤 그 사이에서 파일을 옮기는 방식입니다. 데이터베이스는 실행 중에 파일을 복사하면 일관성이 깨질 수 있으므로, Container를 멈추거나 `pg_dump` 같은 DB 백업 도구를 씁니다.

## 10. 정리

- Container의 데이터는 Container를 지우면 함께 사라진다.
- Volume은 데이터를 Container 바깥에 두어 생명주기를 분리한다.
- 보존할 데이터는 이름 있는 Volume, 개발 중 소스 연결은 Bind Mount, 메모리 임시 데이터는 tmpfs.
- 빈 Volume은 이미지 파일을 복사해 오고, Bind Mount는 기존 파일을 가린다.
- `docker volume rm`, `prune`은 데이터를 지우므로 주의한다.
