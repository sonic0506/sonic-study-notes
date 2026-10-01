---
aliases: [Dockerfile, 도커파일]
tags: [docker, devops]
prerequisites:
  - "[[docker]]"
related: []
status: draft
created: 2026-10-01
---

# Dockerfile

## 1. Dockerfile이란?

**Dockerfile**은 [[docker|Docker]] Image를 만드는 방법을 적어 둔 텍스트 파일입니다.

어떤 OS 위에서, 어떤 파일을 복사하고, 어떤 명령을 실행하고, 마지막에 무엇을 실행할지를 순서대로 적습니다.

```text
Dockerfile ──(docker build)──> Image ──(docker run)──> Container
  설계도                         완성품                   실행 중인 프로그램
```

요리로 비유하면 Dockerfile은 레시피입니다. 레시피만 있으면 누가 어디서 만들어도 같은 요리가 나옵니다.

## 2. 기본 예시

Node.js 애플리케이션을 예로 들어 보겠습니다.

```dockerfile
FROM node:22

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

위에서 아래로 한 줄씩 실행되며, 각 줄을 **명령어(instruction)**라고 부릅니다.

```text
1. node:22 Image에서 시작한다
2. 작업 디렉터리를 /app으로 정한다
3. package.json을 복사하고 의존성을 설치한다
4. 나머지 소스 코드를 복사한다
5. 3000번 포트를 쓴다고 표시한다
6. Container가 시작되면 npm start를 실행한다
```

## 3. 이미지 빌드하기

```bash
docker build -t my-app:1.0 .
```

| 부분 | 의미 |
|---|---|
| `-t my-app:1.0` | 만들어질 Image의 이름과 태그 |
| `.` | 빌드 컨텍스트(build context) 경로 |

### 빌드 컨텍스트

마지막의 `.`은 "현재 디렉터리의 파일을 Docker에게 넘긴다"는 뜻입니다.

`COPY`는 이 빌드 컨텍스트 안의 파일만 복사할 수 있습니다. 컨텍스트 밖의 파일(`../config` 같은 경로)은 복사할 수 없습니다.

```text
my-project/          ← 빌드 컨텍스트
├── Dockerfile
├── package.json
└── src/
```

다른 이름의 Dockerfile을 쓰려면 `-f` 옵션을 씁니다.

```bash
docker build -f Dockerfile.dev -t my-app:dev .
```

## 4. 주요 명령어

### FROM

어떤 Image에서 시작할지 정합니다. 보통 Dockerfile의 첫 줄에 옵니다.

```dockerfile
FROM node:22-alpine
```

`alpine`, `slim`처럼 크기가 작은 버전을 고르면 Image 크기를 줄일 수 있습니다.

### WORKDIR

이후 명령어가 실행될 작업 디렉터리를 정합니다. 디렉터리가 없으면 만들어 줍니다.

```dockerfile
WORKDIR /app
```

`RUN cd /app`은 그 줄에서만 효과가 있으므로, 디렉터리 이동은 `WORKDIR`로 합니다.

### COPY

빌드 컨텍스트의 파일을 Image 안으로 복사합니다.

```dockerfile
COPY package.json ./
COPY src/ ./src/
```

비슷한 명령어로 `ADD`가 있습니다. `ADD`는 압축 파일 자동 해제, URL 다운로드 같은 기능이 더 있지만, 단순 복사라면 동작이 명확한 `COPY`를 씁니다.

### RUN

Image를 만드는 빌드 시점에 명령을 실행합니다. 패키지 설치에 주로 씁니다.

```dockerfile
RUN npm install
RUN apt-get update && apt-get install -y curl
```

### ENV

환경 변수를 설정합니다. 빌드 중에도, Container 실행 중에도 남아 있습니다.

```dockerfile
ENV NODE_ENV=production
```

### ARG

빌드할 때만 쓰는 변수입니다. Container 실행 시에는 남지 않습니다.

```dockerfile
ARG NODE_VERSION=22
FROM node:${NODE_VERSION}
```

```bash
docker build --build-arg NODE_VERSION=20 -t my-app .
```

| | ARG | ENV |
|---|---|---|
| 빌드 중 사용 | O | O |
| 실행 중 사용 | X | O |
| 값 바꾸는 방법 | `--build-arg` | `docker run -e` |

비밀번호나 토큰을 `ARG`, `ENV`로 넣으면 Image 기록에 남을 수 있으므로 넣지 않습니다.

### EXPOSE

Container가 어떤 포트를 쓰는지 문서처럼 표시합니다.

```dockerfile
EXPOSE 3000
```

`EXPOSE`만으로는 외부에서 접속할 수 없습니다. 실제 연결은 `docker run -p 3000:3000`으로 합니다.

### USER

이후 명령과 Container 실행을 어떤 사용자로 할지 정합니다. 기본값은 root입니다.

```dockerfile
USER node
```

root로 실행하면 Container가 공격받았을 때 피해가 커질 수 있으므로, 가능하면 일반 사용자로 실행합니다.

### CMD

Container가 시작될 때 실행할 기본 명령입니다.

```dockerfile
CMD ["npm", "start"]
```

`RUN`은 빌드할 때, `CMD`는 Container를 실행할 때 동작합니다.

```text
RUN  → docker build 시점 → 결과가 Image에 저장됨
CMD  → docker run 시점   → Container의 메인 프로세스가 됨
```

`CMD`는 한 Dockerfile에서 마지막 하나만 적용되며, `docker run` 뒤에 명령을 적으면 덮어쓸 수 있습니다.

```bash
docker run my-app node -v   # CMD 대신 node -v 실행
```

### ENTRYPOINT

Container 실행 시 항상 실행할 명령입니다. `ENTRYPOINT`는 "무엇을 실행할지"를 고정하고, `CMD`는 "어떤 인자로 실행할지"의 기본값을 정합니다.

#### 최종 명령은 둘을 이어 붙인 것

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
```

Container가 시작되면 Docker는 둘을 이어 붙여 실행합니다.

```text
ENTRYPOINT  +  CMD           =  실제 실행
["node"]    +  ["server.js"] =  node server.js
```

#### docker run 뒤의 인자는 CMD만 바꾼다

```bash
docker run my-app              # node server.js   ← CMD 기본값 사용
docker run my-app worker.js    # node worker.js   ← CMD가 worker.js로 바뀜
docker run my-app -v           # node -v          ← CMD가 -v로 바뀜
```

`docker run 이미지` 뒤에 적은 값은 `CMD` 자리만 덮어씁니다. `ENTRYPOINT`의 `node`는 그대로 남으므로, 무엇을 넘기든 항상 `node`로 실행됩니다.

`CMD`만 있으면 명령 전체가 바뀝니다.

```dockerfile
CMD ["node", "server.js"]
```

```bash
docker run my-app worker.js        # worker.js를 실행하려다 실패
docker run my-app node worker.js   # 명령 전체를 적어야 함
```

#### ENTRYPOINT 자체를 바꾸려면 --entrypoint

```bash
docker run --entrypoint sh my-app       # sh 실행 (CMD는 무시)
docker run -it --entrypoint sh my-app   # 디버깅용으로 셸 접속
```

바꿀 수는 있지만 옵션을 따로 줘야 하므로, 실수로 바뀌지 않게 고정하는 용도로 씁니다.

#### 조합별 동작

| Dockerfile | `docker run my-app` | `docker run my-app a.js` |
|---|---|---|
| `CMD ["node","server.js"]` | `node server.js` | `a.js` (명령 전체 교체) |
| `ENTRYPOINT ["node"]` | `node` | `node a.js` |
| `ENTRYPOINT ["node"]` + `CMD ["server.js"]` | `node server.js` | `node a.js` |

#### 언제 쓰나

이미지를 하나의 명령어처럼 쓸 때입니다.

```dockerfile
FROM alpine
RUN apk add --no-cache curl
ENTRYPOINT ["curl"]
CMD ["--help"]
```

```bash
docker run my-curl                       # curl --help
docker run my-curl https://example.com   # curl https://example.com
```

시작 전에 준비 작업이 필요할 때도 씁니다. 이때는 entrypoint 스크립트를 둡니다.

```dockerfile
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

```sh
#!/bin/sh
set -e
# 준비 작업 (예: DB 마이그레이션, 설정 파일 생성)
npm run migrate
# 넘겨받은 CMD를 실행
exec "$@"
```

스크립트가 준비 작업을 한 뒤 `exec "$@"`로 `CMD`(`node server.js`)를 실행합니다. `exec`를 쓰면 node가 스크립트 대신 메인 프로세스(PID 1)가 되어 `docker stop`의 종료 신호를 직접 받습니다. postgres, mysql 같은 공식 이미지도 이 방식을 씁니다.

### exec 형식과 shell 형식

`CMD`, `ENTRYPOINT`, `RUN`은 두 가지 형식으로 쓸 수 있습니다.

```dockerfile
CMD ["npm", "start"]   # exec 형식
CMD npm start          # shell 형식 (/bin/sh -c "npm start")
```

`CMD`와 `ENTRYPOINT`는 exec 형식을 권장합니다. shell 형식은 셸이 메인 프로세스가 되어 `docker stop`의 종료 신호(SIGTERM)가 애플리케이션까지 전달되지 않을 수 있습니다.

`ENTRYPOINT`를 shell 형식으로 쓰면 `CMD`와 `docker run` 뒤의 인자가 모두 무시됩니다.

```dockerfile
ENTRYPOINT node        # /bin/sh -c "node" 로 실행
CMD ["server.js"]      # 무시됨
```

`CMD`와 함께 쓸 `ENTRYPOINT`는 반드시 exec 형식으로 씁니다.

## 5. 레이어와 캐시

Dockerfile의 명령어(`RUN`, `COPY`, `ADD`)는 각각 하나의 레이어를 만듭니다. Docker는 레이어를 캐시해 두고, 바뀌지 않은 부분은 다시 실행하지 않습니다.

어떤 레이어가 바뀌면 그 아래 레이어는 모두 다시 빌드됩니다.

### 나쁜 순서

```dockerfile
COPY . .
RUN npm install
```

소스 코드 한 줄만 고쳐도 `COPY . .`가 바뀌므로 `npm install`이 매번 다시 실행됩니다.

### 좋은 순서

```dockerfile
COPY package*.json ./
RUN npm install
COPY . .
```

```text
package.json 그대로  → npm install 캐시 사용 (빠름)
소스 코드만 변경      → 마지막 COPY만 다시 실행
```

원칙은 자주 바뀌지 않는 것을 위에, 자주 바뀌는 것을 아래에 두는 것입니다.

### RUN 합치기

```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

설치와 정리를 한 `RUN`에서 해야 정리한 파일이 Image에 남지 않습니다. 다른 `RUN`에서 지우면 앞 레이어에 파일이 그대로 남아 Image 크기가 줄지 않습니다.

## 6. .dockerignore

빌드 컨텍스트에서 제외할 파일을 적습니다. 문법은 `.gitignore`와 비슷합니다.

```text
node_modules
.git
.env
*.log
dist
```

- 빌드 컨텍스트가 작아져 빌드가 빨라집니다.
- `.env` 같은 비밀 파일이 `COPY . .`로 Image에 들어가는 것을 막습니다.
- 로컬의 `node_modules`가 Container 안에서 설치한 것을 덮어쓰지 않습니다.

## 7. 멀티 스테이지 빌드

빌드에 필요한 도구와 실행에 필요한 파일은 다릅니다. 멀티 스테이지 빌드는 빌드 단계와 실행 단계를 나눠 최종 Image에는 실행에 필요한 것만 남깁니다.

### 하나의 파일, 여러 개의 FROM

단계별로 파일을 나누지 않습니다. 하나의 Dockerfile 안에 `FROM`을 여러 번 쓰고, `FROM`이 나올 때마다 새 단계(stage)가 시작됩니다. 빌드도 평소처럼 `docker build` 한 번이면 됩니다.

```dockerfile
# 1단계: 빌드
FROM node:22 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 2단계: 실행
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

```text
builder 단계: 소스 코드, devDependencies, 빌드 도구 → 버려짐
최종 단계:    dist 결과물, 운영 의존성만             → 최종 Image
```

`COPY --from=builder`로 앞 단계의 결과물만 가져옵니다. Image 크기가 줄고, 소스 코드나 빌드 도구가 운영 Image에 남지 않습니다.

### 단계마다 서로 다른 작업 공간

같은 파일 안에 있어도 단계끼리는 완전히 분리됩니다. 각 `FROM`이 깨끗한 새 Image에서 다시 시작하기 때문입니다.

```text
┌─── builder 단계 (node:22) ─────────┐     ┌─── 최종 단계 (node:22-alpine) ───┐
│ /app/src/          소스 코드        │     │ /app/package.json                │
│ /app/node_modules/ devDeps 포함     │     │ /app/node_modules/ 운영 의존성만  │
│ /app/dist/         빌드 결과물 ─────┼─────┼→ /app/dist/                       │
│ TypeScript, 빌드 도구               │     │                                   │
└─────────────────────────────────────┘     └───────────────────────────────────┘
          버려짐                                      최종 Image가 됨
```

- 뒤 단계는 앞 단계의 파일을 자동으로 물려받지 않습니다.
- 앞 단계의 파일이 필요하면 `COPY --from=단계이름`으로 직접 가져옵니다.
- `docker build`가 만드는 Image에는 마지막 단계만 남습니다. 앞 단계는 빌드 중에만 쓰이고 결과 Image에 포함되지 않습니다.

### 단일 단계와 비교

| | 단일 단계 | 멀티 스테이지 |
|---|---|---|
| 소스 코드 | 남음 | 없음 (dist만) |
| devDependencies, 빌드 도구 | 남음 | 없음 |
| 베이스 Image | `node:22` (큼) | `node:22-alpine` (작음) |
| Image 크기 | 크다 | 작다 |

Image가 작으면 배포가 빨라지고, 공격에 쓰일 수 있는 도구도 남지 않습니다.

Go처럼 실행 파일 하나로 컴파일되는 언어는 효과가 더 큽니다.

```dockerfile
FROM golang:1.23 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app

FROM scratch                 # 아무것도 없는 빈 Image
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

컴파일러가 든 수백 MB짜리 Image에서 빌드하고, 최종 Image에는 실행 파일 하나만 남깁니다.

### 알아 두면 좋은 것

단계에는 `AS builder`처럼 이름을 붙입니다. 이름이 없으면 순서 번호(`--from=0`)로 가리켜야 해서, 단계를 추가하면 번호가 어긋납니다.

`--target`으로 특정 단계까지만 빌드할 수 있습니다. 빌드 결과를 확인하거나 테스트 단계만 돌릴 때 씁니다.

```bash
docker build --target builder -t my-app:build .
```

단계는 여러 개 둘 수 있습니다.

```dockerfile
FROM node:22 AS deps      # 의존성 설치
FROM node:22 AS builder   # 빌드 (COPY --from=deps)
FROM node:22 AS test      # 테스트 (--target test로만 실행)
FROM node:22-alpine       # 최종 (COPY --from=builder)
```

최신 Docker(BuildKit)는 최종 단계에 필요 없는 단계를 건너뜁니다. 위 예시에서 `test` 단계는 `--target test`를 줄 때만 실행됩니다.

`--from`에는 단계 이름 말고 다른 Image 이름도 쓸 수 있습니다.

```dockerfile
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/
```

## 8. 작성 체크리스트

- 작은 베이스 Image(`alpine`, `slim`)를 쓰고 태그를 명시한다 (`latest` 대신 `node:22`).
- 자주 바뀌지 않는 명령을 위에 둬서 캐시를 활용한다.
- `.dockerignore`로 불필요한 파일과 비밀 파일을 제외한다.
- 비밀 값을 `ENV`, `ARG`로 넣지 않는다.
- 가능하면 `USER`로 root가 아닌 사용자로 실행한다.
- `CMD`, `ENTRYPOINT`는 exec 형식으로 쓴다.
- 빌드 결과물만 필요하면 멀티 스테이지 빌드를 쓴다.

## 9. 정리

| 명령어 | 시점 | 역할 |
|---|---|---|
| `FROM` | 빌드 | 시작 Image 선택 |
| `WORKDIR` | 빌드·실행 | 작업 디렉터리 설정 |
| `COPY` | 빌드 | 파일 복사 |
| `RUN` | 빌드 | 명령 실행 (패키지 설치 등) |
| `ENV` | 빌드·실행 | 환경 변수 |
| `ARG` | 빌드 | 빌드 변수 |
| `EXPOSE` | - | 사용 포트 표시 (문서용) |
| `USER` | 빌드·실행 | 실행 사용자 |
| `CMD` | 실행 | 기본 실행 명령 (덮어쓰기 가능) |
| `ENTRYPOINT` | 실행 | 항상 실행할 명령 |

Dockerfile은 "이 애플리케이션을 실행하려면 무엇이 필요한가"를 코드로 적은 것입니다. 한 번 작성해 두면 누구든 `docker build` 한 번으로 같은 Image를 만들 수 있습니다.
