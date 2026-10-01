---
aliases: [Docker Compose, 도커 컴포즈]
tags: [docker, devops]
prerequisites:
  - "[[docker]]"
related: []
status: draft
created: 2026-10-01
---

# Docker Compose

## 1. Docker Compose란?

Docker Compose는 여러 [[docker|Docker]] Container의 구성과 실행 방법을 하나의 YAML 파일에 정의하고, 명령어 하나로 함께 관리하게 해주는 도구입니다.

Docker를 직접 사용하면 일반적으로 다음과 같이 Container를 하나씩 실행합니다.

```bash
docker build -t my-app .

docker run -d \
  --name my-app \
  -p 3000:3000 \
  -e NODE_ENV=development \
  my-app
```

Container가 하나라면 크게 복잡하지 않습니다.

애플리케이션이 다음처럼 여러 Container로 이루어져 있으면 챙길 것이 많아집니다.

```text
Application
├── Node.js
├── PostgreSQL
└── Redis
```

각 Container에 대해 다음과 같은 설정이 필요할 수 있습니다.

```text
Container 실행
Port 설정
환경 변수 설정
Volume 연결
Network 연결
Container 간 연결
```

이를 모두 `docker run` 명령어로 관리하면 명령어가 길고 복잡해집니다.

Docker Compose를 사용하면 이러한 설정을 하나의 파일에 작성할 수 있습니다.

```text
compose.yaml

├── App 설정
├── PostgreSQL 설정
├── Redis 설정
├── Network 설정
└── Volume 설정
```

그리고 다음 명령어 하나로 전체 환경을 실행할 수 있습니다.

```bash
docker compose up
```

---

# 2. Docker Compose가 필요한 이유

예를 들어 다음과 같은 애플리케이션을 개발한다고 가정합니다.

```text
Node.js Application
        ↓
   PostgreSQL
        +
      Redis
```

Docker Compose를 사용하지 않는다면 각각의 Container를 직접 실행해야 합니다.

먼저 [[docker-network|Network]]를 생성합니다.

```bash
docker network create my-network
```

PostgreSQL을 실행합니다.

```bash
docker run -d \
  --name postgres \
  --network my-network \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  postgres:16
```

Redis도 실행합니다.

```bash
docker run -d \
  --name redis \
  --network my-network \
  -p 6379:6379 \
  redis:7
```

애플리케이션 Image를 생성합니다.

```bash
docker build -t my-app .
```

그리고 애플리케이션 Container를 실행합니다.

```bash
docker run -d \
  --name app \
  --network my-network \
  -p 3000:3000 \
  -e DB_HOST=postgres \
  -e REDIS_HOST=redis \
  my-app
```

단계가 꽤 많습니다.

```text
Network 생성
     ↓
PostgreSQL 실행
     ↓
Redis 실행
     ↓
Application Build
     ↓
Application 실행
```

또한 다른 개발자가 프로젝트를 실행하려면 이러한 명령어와 설정을 모두 알아야 합니다.

Docker Compose를 사용하면 이 설정을 파일로 관리할 수 있습니다.

---

# 3. compose.yaml

Docker Compose에서는 일반적으로 `compose.yaml` 파일에 Container 구성을 정의합니다.

예를 들어:

```yaml
services:

  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DB_HOST: postgres
      REDIS_HOST: redis
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7

volumes:
  postgres-data:
```

그리고 다음 명령어를 실행합니다.

```bash
docker compose up
```

Docker Compose가 설정을 읽어 필요한 Container들을 구성합니다.

```text
compose.yaml
      ↓
docker compose up
      ↓

Docker Compose
│
├── app
│   └── App Container
│
├── postgres
│   └── PostgreSQL Container
│
└── redis
    └── Redis Container
```

---

# 4. Docker와 Docker Compose의 관계

Docker Compose는 Docker를 대체하는 기술이 아닙니다.

Docker의 기존 동작:

```text
Dockerfile
    ↓
docker build
    ↓
Image
    ↓
docker run
    ↓
Container
```

은 그대로 존재합니다.

Docker Compose는 이 과정을 설정 파일을 기반으로 대신 관리합니다.

```text
compose.yaml
      ↓
docker compose up
      ↓
필요한 Image Build
      ↓
Container 생성
      ↓
Network 연결
      ↓
Volume 연결
      ↓
Container 실행
```

> Docker Compose는 여러 Container의 Build, Run과 관련 설정을 선언적으로 관리하는 도구입니다. Build와 Run 과정 자체는 그대로 남아 있습니다.

---

# 5. Dockerfile과 compose.yaml의 차이

두 파일은 역할이 다릅니다.

## Dockerfile

[[docker-dockerfile|Dockerfile]]은 다음 질문에 대한 답입니다.

> "이 애플리케이션의 Docker Image를 어떻게 만들 것인가?"

예:

```dockerfile
FROM node:22

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

CMD ["npm", "start"]
```

Dockerfile을 통해:

```text
Source Code
    +
Dockerfile
    ↓
docker build
    ↓
Docker Image
```

가 만들어집니다.

---

## compose.yaml

Compose 파일은 다음 질문에 대한 답입니다.

> "여러 Container를 어떤 설정과 관계로 함께 실행할 것인가?"

예:

```yaml
services:

  app:
    build: .
    ports:
      - "3000:3000"

  database:
    image: postgres:16

  redis:
    image: redis:7
```

구조적으로 보면:

```text
Dockerfile
│
│ Image를 어떻게 만들까?
│
↓
App Image


compose.yaml
│
│ Container들을 어떻게 실행할까?
│
├── App Container
├── PostgreSQL Container
└── Redis Container
```

Dockerfile은 Image를 만드는 방법을, Compose 파일은 Container들을 함께 실행하는 방법을 맡습니다.

---

# 6. services

Compose 파일에서 가장 중요한 항목은 `services`입니다.

```yaml
services:
  app:
    ...
    
  postgres:
    ...
    
  redis:
    ...
```

각 Service는 일반적으로 하나의 Container 구성을 나타냅니다.

```text
services

├── app
│    ↓
│   App Container
│
├── postgres
│    ↓
│   PostgreSQL Container
│
└── redis
     ↓
    Redis Container
```

`app`, `postgres`, `redis`와 같은 이름은 개발자가 직접 정할 수 있습니다.

---

# 7. build

직접 만든 애플리케이션을 Container로 실행하려면 `build`를 사용할 수 있습니다.

```yaml
services:

  app:
    build: .
```

`.`은 현재 디렉터리를 Build Context로 사용한다는 의미입니다.

Docker Compose는 현재 디렉터리의 Dockerfile을 이용하여 Image를 Build합니다.

```text
compose.yaml

build: .
    ↓
Dockerfile
    ↓
docker build
    ↓
App Image
    ↓
App Container
```

Dockerfile이 다른 위치에 있다면 조금 더 상세하게 지정할 수도 있습니다.

```yaml
services:

  app:
    build:
      context: .
      dockerfile: Dockerfile
```

---

# 8. image

이미 만들어져 있는 Docker Image를 사용하는 경우 `image`를 사용합니다.

예를 들어:

```yaml
services:

  postgres:
    image: postgres:16

  redis:
    image: redis:7
```

이 경우 직접 Dockerfile을 작성해서 PostgreSQL이나 Redis Image를 만드는 것이 아닙니다.

```text
postgres:16 Image
       ↓
PostgreSQL Container


redis:7 Image
       ↓
Redis Container
```

필요한 Image가 로컬에 없다면 Docker가 Registry에서 가져올 수 있습니다.

---

# 9. build와 image의 차이

다음 두 설정은 의미가 다릅니다.

### build

```yaml
app:
  build: .
```

현재 프로젝트의 Dockerfile을 이용하여 Image를 직접 생성합니다.

```text
Dockerfile
    ↓
Build
    ↓
Image
    ↓
Container
```

### image

```yaml
redis:
  image: redis:7
```

이미 존재하는 Image를 사용합니다.

```text
redis:7 Image
      ↓
Container
```

일반적인 개발환경에서는 둘을 함께 사용하는 경우가 많습니다.

```yaml
services:

  app:
    build: .

  postgres:
    image: postgres:16

  redis:
    image: redis:7
```

```text
             Docker Compose
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓

   Dockerfile   postgres:16    redis:7
       ↓            ↓            ↓
     Build
       ↓
   App Image
       ↓            ↓            ↓
      App        PostgreSQL     Redis
   Container     Container    Container
```

---

# 10. ports

Container의 Port를 Host와 연결할 때 사용합니다.

```yaml
services:

  app:
    ports:
      - "3000:3000"
```

형식은 다음과 같습니다.

```text
Host Port : Container Port
```

따라서:

```yaml
ports:
  - "8080:3000"
```

이라면:

```text
내 컴퓨터
localhost:8080

      ↓

Container
localhost:3000
```

으로 연결됩니다.

기존 Docker 명령어의:

```bash
docker run -p 8080:3000 my-app
```

과 같은 역할입니다.

---

# 11. environment

Container에 환경 변수를 전달할 수 있습니다.

```yaml
services:

  app:
    environment:
      NODE_ENV: development
      DB_HOST: postgres
      DB_PORT: 5432
```

Container 내부에서는:

```text
NODE_ENV=development
DB_HOST=postgres
DB_PORT=5432
```

와 같은 환경 변수를 사용할 수 있습니다.

기존 명령어의:

```bash
docker run \
  -e NODE_ENV=development \
  -e DB_HOST=postgres \
  my-app
```

과 비슷한 역할입니다.

---

# 12. env_file

환경변수가 많다면 별도의 `.env` 파일로 관리할 수도 있습니다.

```text
.env
```

```env
NODE_ENV=development
DB_HOST=postgres
DB_PORT=5432
```

Compose에서는:

```yaml
services:

  app:
    env_file:
      - .env
```

처럼 사용할 수 있습니다.

다만 비밀번호, API Key 등 민감한 값이 들어 있는 `.env` 파일을 Git 저장소에 그대로 올리지 않도록 주의해야 합니다.

---

# 13. volumes

[[docker-volume|Volume]]을 연결할 수도 있습니다.

예를 들어 PostgreSQL의 데이터를 보존한다고 가정합니다.

```yaml
services:

  postgres:
    image: postgres:16

    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

구조는 다음과 같습니다.

```text
PostgreSQL Container
        │
        │
        ↓
/var/lib/postgresql/data
        │
        ↓
postgres-data Volume
```

Container를 삭제하더라도 Volume을 유지한다면 데이터를 보존할 수 있습니다.

```text
기존 PostgreSQL Container
        ↓
       삭제

postgres-data
        ↑
        │
새 PostgreSQL Container
```

---

# 14. Bind Mount

로컬 디렉터리를 Container 내부에 직접 연결할 수도 있습니다.

예를 들어:

```yaml
services:

  app:
    volumes:
      - ./src:/app/src
```

구조는 다음과 같습니다.

```text
내 컴퓨터

./src
  │
  │ 실시간 연결
  ↓

Container

/app/src
```

개발 중 로컬 소스 코드 변경을 Container에서 바로 반영해야 하는 경우 등에 활용할 수 있습니다.

Volume과 Bind Mount는 비슷해 보이지만 차이가 있습니다.

```text
Named Volume

Docker가 관리
↓
postgres-data


Bind Mount

내 컴퓨터의 실제 경로 사용
↓
./src
```

---

# 15. Docker Compose Network

Docker Compose의 편리한 기능 중 하나가 Network입니다.

Compose로 여러 Service를 실행하면 기본적으로 해당 Compose 프로젝트를 위한 Network가 생성되고 Service들이 연결됩니다.

예를 들어:

```yaml
services:

  app:
    build: .

  postgres:
    image: postgres:16

  redis:
    image: redis:7
```

실행하면 개념적으로:

```text
Docker Compose Network

┌─────────────────────────────┐
│                             │
│ app                         │
│  │                          │
│  ├────────→ postgres        │
│  │                          │
│  └────────→ redis           │
│                             │
└─────────────────────────────┘
```

와 같은 환경이 만들어집니다.

---

# 16. Service 이름을 주소처럼 사용할 수 있다

Compose Network에서는 Service 이름을 이용해 다른 Container에 접근할 수 있습니다.

예를 들어:

```yaml
services:

  app:
    build: .

  postgres:
    image: postgres:16
```

Application에서 PostgreSQL에 접근할 때:

```text
localhost:5432
```

가 아니라:

```text
postgres:5432
```

처럼 접근할 수 있습니다. `postgres`가 Service 이름이기 때문입니다.

```text
App Container
     │
     │ postgres:5432
     ↓
PostgreSQL Container
```

---

# 17. Container 내부의 localhost

Docker를 처음 사용할 때 자주 실수하는 부분입니다.

App Container에서:

```text
localhost
```

는 내 컴퓨터를 의미하지 않습니다.

또한 PostgreSQL Container를 의미하는 것도 아닙니다.

현재 App Container 자기 자신을 의미합니다.

```text
Host Computer

├── App Container
│      │
│      └── localhost
│           ↓
│        App 자신
│
└── PostgreSQL Container
```

따라서 App에서 PostgreSQL Container에 접근하려면:

```text
localhost:5432
```

가 아니라:

```text
postgres:5432
```

처럼 Service 이름을 사용합니다.

---

# 18. depends_on

Service 간 실행 관계를 표현할 수 있습니다.

```yaml
services:

  app:
    build: .
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16

  redis:
    image: redis:7
```

의미는:

```text
postgres
redis

   ↓

app
```

처럼 `app`이 `postgres`, `redis`에 의존한다는 것을 Compose에 알려주는 것입니다.

다만 `depends_on`만 사용한다고 해서:

> "PostgreSQL이 실제로 요청을 받을 준비가 완전히 끝날 때까지 기다린다."

라는 의미는 아닙니다.

Container의 시작 순서를 관리하는 것과 애플리케이션이 실제로 준비된 상태인 것은 다른 문제입니다.

이 때문에 필요한 경우 `healthcheck`를 함께 사용할 수 있습니다.

---

# 19. healthcheck

Container 내부 서비스가 정상적으로 동작할 준비가 되었는지 검사할 수 있습니다.

예를 들어:

```yaml
services:

  postgres:
    image: postgres:16

    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Docker가 주기적으로 상태를 확인합니다.

```text
Container 실행
      ↓
PostgreSQL 시작 중
      ↓
Health Check
      ↓
준비 안 됨
      ↓
Health Check
      ↓
정상
      ↓
healthy
```

이를 이용해 다른 Service가 정상적으로 준비된 이후 실행되도록 좀 더 명확한 의존 관계를 구성할 수 있습니다.

---

# 20. restart

Container 종료 시 재시작 정책도 지정할 수 있습니다.

```yaml
services:

  app:
    build: .
    restart: unless-stopped
```

대표적인 정책으로 다음과 같은 것들이 있습니다.

```text
no
always
on-failure
unless-stopped
```

예를 들어:

```yaml
restart: on-failure
```

는 애플리케이션이 오류로 종료되었을 때 다시 실행하도록 설정할 수 있습니다.

---

# 21. container_name

Container의 이름을 직접 지정할 수도 있습니다.

```yaml
services:

  app:
    build: .
    container_name: my-app
```

그러면:

```bash
docker ps
```

등에서 `my-app`이라는 이름으로 확인할 수 있습니다.

다만 Compose가 자체적으로 Container 이름을 관리할 수 있기 때문에 반드시 지정해야 하는 설정은 아닙니다.

---

# 22. Docker Compose 실행

Compose 파일이 준비되었다면 다음 명령어로 실행합니다.

```bash
docker compose up
```

전체 흐름은 다음과 같습니다.

```text
compose.yaml
      ↓
docker compose up
      ↓
설정 분석
      ↓
필요한 Image 확인
      ↓
필요하면 Build / Pull
      ↓
Network 생성
      ↓
Volume 생성
      ↓
Container 생성
      ↓
Container 실행
```

---

# 23. 백그라운드 실행

기본적으로:

```bash
docker compose up
```

을 실행하면 현재 터미널에 로그가 출력됩니다.

백그라운드에서 실행하려면:

```bash
docker compose up -d
```

를 사용합니다.

`-d`는 Detached Mode를 의미합니다.

```text
docker compose up

Terminal
└── 로그 계속 출력


docker compose up -d

Terminal
└── 명령 종료

Background
├── App
├── PostgreSQL
└── Redis
```

---

# 24. 다시 Build하면서 실행

애플리케이션 코드를 수정하여 Image를 다시 Build해야 한다면:

```bash
docker compose up --build
```

을 사용할 수 있습니다.

흐름은:

```text
Dockerfile
    ↓
Image 다시 Build
    ↓
Container 생성/교체
    ↓
실행
```

입니다.

다음 조합도 자주 씁니다.

```bash
docker compose up -d --build
```

이렇게 사용하면:

```text
Image Build
+
Container 실행
+
백그라운드 실행
```

을 한 번에 처리할 수 있습니다.

---

# 25. 실행 중인 Service 확인

```bash
docker compose ps
```

실행 중인 Compose Service와 Container 상태를 확인할 수 있습니다.

예:

```text
NAME        SERVICE     STATUS
app         app         running
postgres    postgres    running
redis       redis       running
```

---

# 26. 로그 확인

전체 Service 로그:

```bash
docker compose logs
```

특정 Service 로그:

```bash
docker compose logs app
```

실시간으로 확인:

```bash
docker compose logs -f app
```

개발 중 문제가 발생했을 때 자주 사용하는 명령어입니다.

---

# 27. Container 내부 명령 실행

실행 중인 Service 내부에서 명령어를 실행할 수도 있습니다.

```bash
docker compose exec app sh
```

그러면 App Container 내부 Shell에 접근할 수 있습니다.

```text
내 Terminal

    ↓

App Container

/app #
```

예를 들어 내부 파일을 확인하거나 환경변수를 확인할 때 사용할 수 있습니다.

---

# 28. Docker Compose 종료

실행 중인 Container를 중지하고 Compose 환경을 제거하려면:

```bash
docker compose down
```

을 사용합니다.

개념적으로:

```text
App Container       ─┐
PostgreSQL Container ├─ 삭제
Redis Container     ─┘

Network
   ↓
삭제
```

됩니다.

하지만 기본적으로 Named Volume까지 자동으로 삭제하는 것은 아닙니다.

Volume까지 삭제하려면:

```bash
docker compose down -v
```

를 사용할 수 있습니다.

데이터베이스 데이터를 Volume에 저장하고 있다면 `-v` 사용 시 데이터가 삭제될 수 있으므로 주의해야 합니다.

---

# 29. stop과 down의 차이

두 명령어의 차이도 알아두면 좋습니다.

## stop

```bash
docker compose stop
```

Container를 중지합니다.

```text
Container

Running
   ↓
Stopped

Container 자체는 존재
```

다시:

```bash
docker compose start
```

로 실행할 수 있습니다.

---

## down

```bash
docker compose down
```

Container를 중지하고 제거합니다.

```text
Container

Running
   ↓
Stopped
   ↓
Removed
```

필요한 경우 다음 `up` 실행 시 Container가 다시 생성됩니다.

---

# 30. 자주 사용하는 Docker Compose 명령어

| 명령어 | 설명 |
|---|---|
| `docker compose up` | 전체 Service 실행 |
| `docker compose up -d` | 백그라운드 실행 |
| `docker compose up --build` | Image를 Build하면서 실행 |
| `docker compose down` | Container와 Network 종료 및 제거 |
| `docker compose down -v` | Volume까지 함께 제거 |
| `docker compose stop` | Container 중지 |
| `docker compose start` | 중지된 Container 다시 실행 |
| `docker compose restart` | Container 재시작 |
| `docker compose ps` | 실행 상태 확인 |
| `docker compose logs` | 로그 확인 |
| `docker compose logs -f` | 실시간 로그 확인 |
| `docker compose exec app sh` | 특정 Container 내부 접속 |
| `docker compose build` | Image Build |
| `docker compose pull` | 필요한 Image 가져오기 |

---

# 31. Docker 명령어와 Compose 비교

Docker Compose에서 하는 일은 대부분 기존 Docker 명령어와 연결해서 이해할 수 있습니다.

### Port

Docker:

```bash
docker run -p 3000:3000 my-app
```

Compose:

```yaml
ports:
  - "3000:3000"
```

---

### Environment Variable

Docker:

```bash
docker run -e NODE_ENV=development my-app
```

Compose:

```yaml
environment:
  NODE_ENV: development
```

---

### Volume

Docker:

```bash
docker run -v my-data:/app/data my-app
```

Compose:

```yaml
volumes:
  - my-data:/app/data
```

---

### Image

Docker:

```bash
docker run redis:7
```

Compose:

```yaml
redis:
  image: redis:7
```

---

### Build

Docker:

```bash
docker build -t my-app .
```

Compose:

```yaml
app:
  build: .
```

Compose는 기존 Docker 기능을 YAML 설정으로 선언해서 관리하는 방식이라고 보면 됩니다.

---

# 32. 실제 예제

간단한 Node.js + PostgreSQL 환경을 구성한다고 가정합니다.

프로젝트 구조:

```text
my-project
│
├── src
│
├── package.json
├── Dockerfile
├── compose.yaml
└── .env
```

Dockerfile:

```dockerfile
FROM node:22

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

CMD ["npm", "start"]
```

`.env`:

```env
DB_HOST=postgres
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
DB_NAME=myapp
```

`compose.yaml`:

```yaml
services:

  app:
    build: .
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      - postgres

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

실행:

```bash
docker compose up -d --build
```

그러면:

```text
                Docker Compose

┌───────────────────────────────────┐
│                                   │
│  App Container                    │
│       │                           │
│       │ postgres:5432             │
│       ↓                           │
│  PostgreSQL Container             │
│       │                           │
│       ↓                           │
│  postgres-data Volume             │
│                                   │
└───────────────────────────────────┘
```

와 같은 개발 환경이 만들어집니다.

---

# 33. Docker Compose의 장점

## 여러 Container를 한 번에 관리

기존:

```text
docker run ...
docker run ...
docker run ...
```

Compose:

```bash
docker compose up
```

---

## 실행 환경을 코드로 관리

긴 Docker 명령어를 문서로 전달할 필요가 없습니다.

```text
"Postgres는 이 명령어로 실행하고
Redis는 이 옵션을 추가하고
Network는 이것을 사용하세요."
```

대신:

```text
compose.yaml
```

파일 자체가 실행 환경에 대한 설명이 됩니다.

---

## 팀 개발 환경 통일

프로젝트를 받은 개발자가:

```bash
docker compose up
```

만으로 필요한 환경을 실행할 수 있도록 구성할 수 있습니다.

```text
개발자 A ─┐
개발자 B ─┼→ compose.yaml
개발자 C ─┘
                ↓
          동일한 개발 환경
```

---

## 복잡한 명령어 감소

다음과 같은 설정을:

```text
Port
Environment
Volume
Network
Image
Build
Dependency
Health Check
```

하나의 YAML 파일에서 관리할 수 있습니다.

---

# 34. Docker Compose를 이해할 때 중요한 점

Docker Compose를 새로운 Container 기술이라고 생각할 필요는 없습니다.

Docker의 기본 구조는 여전히:

```text
Dockerfile
    ↓
Image
    ↓
Container
```

입니다.

Docker Compose는 그 위에서:

```text
                compose.yaml

                     ↓

            여러 Container 관리

         ┌───────────┼───────────┐
         ↓           ↓           ↓

       App        Database      Redis
         │           │           │
         └───────────┼───────────┘
                     ↓
                  Network

                     +

                  Volumes
```

처럼 여러 Container와 관련 설정을 하나의 애플리케이션 환경으로 묶어서 관리합니다.

---

# 35. Docker와 Docker Compose의 관계 정리

각각을 한 문장씩 정리하면 다음과 같습니다.

### Dockerfile

> 하나의 Docker Image를 어떻게 만들 것인지 정의한다.

### Docker Image

> Container를 만들기 위한 실행 환경과 애플리케이션의 패키지다.

### Docker Container

> Image를 실제로 실행한 인스턴스다.

### Docker Compose

> 여러 Container를 어떤 설정과 관계로 함께 실행할 것인지 정의하고 관리한다.

전체 관계를 표현하면 다음과 같습니다.

```text
                    Dockerfile
                        │
                        ↓
                      Image
                        │
                        ↓
                    Container
                        ↑
                        │
                Docker Compose
                        │
          ┌─────────────┼─────────────┐
          │             │             │
        Port          Volume       Network
          │             │             │
          └─────────────┼─────────────┘
                        │
                 compose.yaml
```

조금 더 실제적인 구조로 보면:

```text
compose.yaml
│
├── app
│    ├── build: .
│    ├── ports
│    └── environment
│
├── postgres
│    ├── image: postgres
│    └── volume
│
└── redis
     └── image: redis

          ↓

docker compose up

          ↓

┌──────────────────────────────────┐
│                                  │
│ Docker Network                   │
│                                  │
│  App ─── PostgreSQL ─── Redis    │
│           │                      │
│           ↓                      │
│         Volume                   │
│                                  │
└──────────────────────────────────┘
```

---

# 36. 최종 요약

Docker Compose는 여러 Docker Container로 구성된 애플리케이션 환경을 YAML 파일 하나로 정의하고 관리하기 위한 도구입니다.

기존 Docker 방식에서는:

```text
docker build
docker network create
docker volume create
docker run ...
docker run ...
docker run ...
```

처럼 각각의 작업을 직접 수행했다면 Docker Compose에서는:

```text
compose.yaml
      ↓
docker compose up
```

으로 필요한 구성을 한 번에 관리할 수 있습니다.

네 가지의 관계를 정리하면 다음과 같습니다.

```text
Dockerfile
=
Image를 어떻게 만들 것인가?


Docker Image
=
Container의 실행 기반


Docker Container
=
실제로 실행되는 애플리케이션


Docker Compose
=
여러 Container를 어떻게 구성하고
함께 실행할 것인가?
```

한 문장으로 정리하면:

> Docker Compose는 여러 Docker Container의 Image, Port, Environment, Volume, Network 등의 설정을 YAML 파일 하나에 선언하고, 전체 환경을 한 단위로 실행하고 관리하게 해주는 도구입니다.