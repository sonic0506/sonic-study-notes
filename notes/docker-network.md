---
aliases: [Docker Network, 도커 네트워크]
tags: [docker, devops]
prerequisites:
  - "[[docker]]"
related: []
status: draft
created: 2026-10-01
---

# Docker Network

## 1. Docker Network란?

Container는 각자 격리된 네트워크 환경을 가집니다. 자기만의 IP 주소와 네트워크 인터페이스가 있고, 기본적으로는 서로를 모릅니다.

**Docker Network**는 Container들을 같은 네트워크에 묶어서 서로 통신할 수 있게 해 주는 기능입니다.

```text
┌────────── my-net ──────────┐
│                             │
│  app ───────→ db:5432       │
│   │                         │
│   └─────────→ redis:6379    │
│                             │
└─────────────────────────────┘
```

같은 네트워크에 있는 Container끼리는 Container 이름을 주소처럼 써서 접속할 수 있습니다.

## 2. 네트워크 드라이버

[[docker|Docker]] 네트워크는 드라이버에 따라 동작 방식이 다릅니다.

| 드라이버 | 설명 | 주 용도 |
|---|---|---|
| `bridge` | 호스트 안에 가상 네트워크를 만듦 (기본값) | 한 서버 안의 Container 통신 |
| `host` | 호스트의 네트워크를 그대로 사용 | 네트워크 성능이 중요할 때 |
| `none` | 네트워크 없음 | 완전히 격리해야 할 때 |
| `overlay` | 여러 호스트를 하나의 네트워크로 연결 | Docker Swarm 클러스터 |
| `macvlan` | Container에 실제 네트워크의 MAC 주소 부여 | 물리 네트워크 장비처럼 보여야 할 때 |

대부분은 `bridge`만 알아도 충분합니다.

### bridge

호스트 안에 가상의 스위치를 만들고 Container를 연결합니다. Container는 이 안에서 사설 IP(예: `172.18.0.2`)를 받습니다.

```text
호스트
└── bridge 네트워크 (172.18.0.0/16)
    ├── app   172.18.0.2
    └── db    172.18.0.3
```

### host

Container가 호스트와 네트워크를 공유합니다. Container가 3000번 포트를 열면 호스트의 3000번 포트가 바로 열립니다. 격리가 없으므로 `-p` 포트 연결이 필요 없지만, 포트가 호스트와 겹칠 수 있습니다. 주로 Linux에서 씁니다.

```bash
docker run --network host my-app
```

### none

네트워크 인터페이스가 없습니다(loopback만 있음). 외부와 통신하지 않는 일회성 작업에 씁니다.

## 3. 기본 bridge와 사용자 정의 bridge

Docker를 설치하면 `bridge`라는 이름의 기본 네트워크가 있고, `--network`를 지정하지 않은 Container는 여기에 연결됩니다.

그런데 기본 bridge에서는 Container 이름으로 접속할 수 없습니다. IP 주소로만 통신할 수 있고, IP는 Container를 다시 만들 때마다 바뀔 수 있습니다.

직접 만든 네트워크(사용자 정의 bridge)에서는 Docker가 내장 DNS를 제공해서 Container 이름으로 접속할 수 있습니다.

| | 기본 bridge | 사용자 정의 bridge |
|---|---|---|
| Container 이름으로 접속 | X | O |
| 격리 | 모든 기본 Container가 한 곳에 | 네트워크별로 분리 |
| 실행 중 연결/해제 | 불가 | 가능 |

그래서 Container끼리 통신시킬 때는 네트워크를 직접 만들어 씁니다.

## 4. 네트워크 만들고 연결하기

```bash
# 네트워크 생성
docker network create my-net

# 같은 네트워크로 Container 실행
docker run -d --name db --network my-net \
  -e POSTGRES_PASSWORD=secret postgres:16

docker run -d --name app --network my-net my-app
```

`app` Container 안에서는 `db`라는 이름으로 데이터베이스에 접속합니다.

```text
DATABASE_URL=postgres://postgres:secret@db:5432/postgres
```

### 관리 명령어

```bash
docker network ls                      # 목록
docker network inspect my-net          # 연결된 Container, IP 확인
docker network connect my-net app      # 실행 중인 Container 연결
docker network disconnect my-net app   # 연결 해제
docker network rm my-net               # 삭제
docker network prune                   # 사용하지 않는 네트워크 정리
```

하나의 Container를 여러 네트워크에 연결할 수도 있습니다. 예를 들어 API 서버는 프론트용 네트워크와 DB용 네트워크에 모두 연결하고, DB는 DB용 네트워크에만 두면 프론트 Container가 DB에 직접 접근할 수 없습니다.

```text
frontend-net:  web ── api
backend-net:          api ── db
```

## 5. 포트 공개(-p)와 Container 간 통신

`-p`는 호스트 외부에서 Container로 들어오는 길을 여는 것입니다. Container끼리 통신할 때는 필요 없습니다.

```bash
docker run -d --name db --network my-net -p 5433:5432 postgres:16
```

```text
내 컴퓨터(호스트)에서 접속   → localhost:5433  (호스트 포트)
같은 네트워크 Container에서 → db:5432         (Container 포트)
```

Container끼리는 `-p`의 왼쪽 숫자(호스트 포트)가 아니라 Container 안의 포트로 접속합니다. 자주 헷갈리는 부분입니다.

DB처럼 외부에서 접근할 필요가 없는 Container는 `-p`를 빼면 같은 네트워크 안에서만 접근할 수 있어 더 안전합니다.

## 6. Container 안의 localhost

Container 안에서 `localhost`는 그 Container 자신을 가리킵니다. 호스트 컴퓨터가 아닙니다.

```text
app Container 안에서 localhost:5432
  → app Container 자기 자신의 5432 포트
  → DB가 없으므로 연결 실패
```

- 다른 Container에 접속: Container 이름 사용 (`db:5432`)
- 호스트에서 실행 중인 프로그램에 접속: `host.docker.internal` 사용

`host.docker.internal`은 Docker Desktop(Mac, Windows)에서 기본으로 쓸 수 있습니다. Linux에서는 실행할 때 직접 추가해야 합니다.

```bash
docker run --add-host=host.docker.internal:host-gateway my-app
```

## 7. Docker Compose의 네트워크

Docker Compose는 프로젝트마다 `<프로젝트명>_default` 네트워크를 자동으로 만들고 모든 서비스를 연결합니다. 서비스 이름이 곧 접속 주소입니다.

```yaml
services:
  app:
    build: .
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/postgres
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
```

네트워크를 나누고 싶으면 직접 정의합니다.

```yaml
services:
  web:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
```

## 8. 정리

- Container는 기본적으로 격리되어 있고, Docker Network로 묶어야 서로 통신한다.
- 대부분은 `bridge` 드라이버를 쓰고, 네트워크를 직접 만들어야 Container 이름으로 접속할 수 있다.
- `-p`는 호스트 외부 접속용이다. Container끼리는 Container 포트로 접속한다.
- Container 안의 `localhost`는 자기 자신이다.
- Docker Compose는 네트워크를 자동으로 만들고 서비스 이름을 주소로 쓰게 해 준다.
