---
aliases: [Docker, 도커]
tags: [docker, devops]
prerequisites: []
related: []
status: draft
created: 2026-10-01
---

# Docker

## 1. Docker란?

Docker는 애플리케이션과 그 실행 환경을 하나의 컨테이너로 묶어 실행하는 플랫폼입니다.

개발하다 보면 이런 문제를 자주 겪습니다.

- 개발자마다 운영체제가 다름
- Node.js, Java 등의 버전이 다름
- 설치된 라이브러리 버전이 다름
- 환경 설정이 다름
- 내 컴퓨터에서는 실행되지만 다른 컴퓨터에서는 실행되지 않음

예를 들어 어떤 애플리케이션이 다음 환경을 필요로 한다고 가정합니다.

```text
Node.js 22
특정 npm 패키지
특정 환경 설정
애플리케이션 코드
```

다른 컴퓨터에서 이 애플리케이션을 실행하려면 동일한 환경을 다시 구성해야 합니다.

Docker를 사용하면 필요한 실행 환경을 하나의 이미지로 정의할 수 있습니다.

```text
┌──────── Docker Container ────────┐
│                                  │
│  Node.js 22                      │
│  필요한 라이브러리                │
│  환경 설정                        │
│  애플리케이션                     │
│                                  │
└──────────────────────────────────┘
```

Docker의 목적은 실행 환경을 표준화해서, 어디서 실행하든 최대한 같은 환경을 만드는 것입니다.

---

# 2. Docker가 등장한 이유

## Docker가 없는 경우

개발자 A가 자신의 컴퓨터에서 애플리케이션을 개발했다고 가정합니다.

```text
개발자 A

macOS
Node.js 22
npm 11
특정 라이브러리
환경 설정
```

그런데 개발자 B의 환경은 다를 수 있습니다.

```text
개발자 B

Windows
Node.js 20
npm 10
다른 환경 설정
```

이 경우 같은 코드를 실행해도 문제가 발생할 수 있습니다.

```text
개발자 A

npm start
↓
정상 실행


개발자 B

npm start
↓
ERROR
```

이때 흔히 나오는 말이 있습니다.

> "제 컴퓨터에서는 잘 되는데요?"

Docker는 이런 환경 차이를 줄이는 데 씁니다.

Docker를 사용하면 개발자는 실행 환경 자체를 코드로 정의합니다.

```text
Docker 환경

Node.js 22
npm
필요한 라이브러리
애플리케이션

        ↓

개발자 A → 동일한 환경

개발자 B → 동일한 환경

다른 컴퓨터 → 동일한 환경
```

---

# 3. Docker의 핵심 개념

Docker에서 먼저 알아야 할 개념은 세 가지입니다.

```text
Dockerfile
    ↓
  build
    ↓
Docker Image
    ↓
   run
    ↓
Container
```

각각의 역할은 다음과 같습니다.

| 개념 | 설명 |
|---|---|
| Dockerfile | Docker Image를 어떻게 만들지 정의한 파일 |
| Image | 애플리케이션 실행 환경을 패키징한 결과물 |
| Container | Image를 실제로 실행한 상태 |

---

# 4. Dockerfile

[[dockerfile|Dockerfile]]은 Docker Image를 만드는 설계도입니다.

예를 들어 Node.js 애플리케이션을 Docker로 실행한다고 가정합니다.

```dockerfile
FROM node:22

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

CMD ["npm", "start"]
```

---

## FROM

```dockerfile
FROM node:22
```

기본으로 사용할 Image를 지정합니다.

Node.js 22가 설치된 Image를 기반으로 새 Image를 만든다는 뜻입니다.

```text
Node.js 22 환경
      ↓
우리 애플리케이션 환경 추가
      ↓
새로운 Image
```

이렇게 기반으로 쓰는 Image를 Base Image라고 합니다.

---

## WORKDIR

```dockerfile
WORKDIR /app
```

Container 내부에서 작업할 디렉터리를 지정합니다.

이후 명령어는 기본적으로 `/app`을 기준으로 실행됩니다.

```text
Container

/
└── app
    ├── package.json
    ├── node_modules
    └── src
```

---

## COPY

```dockerfile
COPY package*.json ./
```

현재 컴퓨터에 있는 파일을 Image 내부로 복사합니다.

예를 들어:

```text
내 컴퓨터

package.json

        ↓ COPY

Docker Image

/app/package.json
```

이후 전체 프로젝트를 복사할 수도 있습니다.

```dockerfile
COPY . .
```

---

## RUN

```dockerfile
RUN npm install
```

Image를 만드는 과정에서 명령어를 실행합니다.

여기서는 필요한 npm 패키지를 설치합니다.

`RUN`은 보통 Image를 만드는 build 시점에 실행됩니다.

```text
docker build

    ↓

RUN npm install

    ↓

node_modules 생성

    ↓

Image에 포함
```

---

## CMD

```dockerfile
CMD ["npm", "start"]
```

Container가 실행될 때 기본적으로 실행할 명령어입니다.

```text
docker run
    ↓
Container 생성
    ↓
npm start
    ↓
애플리케이션 실행
```

`RUN`과 `CMD`는 실행 시점이 다릅니다.

```text
RUN
→ Image를 만들 때 실행

CMD
→ Container를 실행할 때 실행
```

---

# 5. Docker Image

Docker Image는 애플리케이션 실행에 필요한 환경과 파일을 패키징한 실행 템플릿입니다.

Dockerfile을 작성했으면 Image를 빌드합니다.

```bash
docker build -t my-app .
```

Docker는 Dockerfile을 읽어 Image를 생성합니다.

```text
Dockerfile

FROM node:22
WORKDIR /app
COPY ...
RUN ...
CMD ...

       ↓

docker build

       ↓

┌──────────────────┐
│ my-app Image     │
│                  │
│ Node.js          │
│ dependencies     │
│ application      │
│ configuration    │
└──────────────────┘
```

Image 자체는 실행 중인 프로그램이 아닙니다.

쉽게 비유하면:

```text
Image
=
프로그램을 실행할 수 있는 완성된 패키지
```

입니다.

---

# 6. Docker Container

Container는 Docker Image를 실제로 실행한 인스턴스입니다.

Image를 다음과 같이 실행할 수 있습니다.

```bash
docker run my-app
```

관계를 표현하면 다음과 같습니다.

```text
Docker Image
     ↓
 docker run
     ↓
Container
```

하나의 Image에서 여러 Container를 만들 수도 있습니다.

```text
              ┌→ Container 1
              │
my-app Image ─┼→ Container 2
              │
              └→ Container 3
```

각 Container는 서로 독립적으로 실행됩니다.

예를 들어:

```text
Container 1
Node.js App

Container 2
Node.js App

Container 3
Node.js App
```

처럼 동일한 애플리케이션을 여러 개 실행할 수 있습니다.

---

# 7. Image와 Container의 차이

Docker를 처음 배울 때 가장 헷갈리는 부분입니다.

간단하게 비유하면:

```text
Image = 붕어빵 틀

Container = 실제로 만들어진 붕어빵
```

또는 프로그램에 비유하면:

```text
Class
↓
Object

Image
↓
Container
```

처럼 생각할 수도 있습니다.

하나의 Image로 여러 Container를 만들 수 있습니다.

```text
Image

   ↓ run
Container A

   ↓ run
Container B

   ↓ run
Container C
```

---

# 8. Container는 독립된 실행 환경을 가진다

Docker Container는 다른 Container와 격리된 환경에서 실행됩니다.

예를 들어:

```text
컴퓨터

Docker
│
├─ Container A
│   └─ Node.js 22
│
├─ Container B
│   └─ Node.js 20
│
└─ Container C
    └─ Python
```

처럼 서로 다른 실행 환경을 동시에 사용할 수 있습니다.

Container A에 설치된 프로그램이 Container B에 자동으로 설치되는 것도 아닙니다.

각 Container는 논리적으로 분리된 실행 환경을 갖습니다.

---

# 9. Docker의 Port

Container는 기본적으로 격리되어 있기 때문에 Container 내부에서 서버를 실행했다고 해서 바로 외부에서 접근할 수 있는 것은 아닙니다.

예를 들어 Container 안에서 서버가 `3000` 포트를 사용한다고 가정합니다.

```text
Container

localhost:3000
```

외부에서 접근할 수 있도록 연결하려면 포트를 매핑할 수 있습니다.

```bash
docker run -p 8080:3000 my-app
```

의미는 다음과 같습니다.

```text
내 컴퓨터       Container

8080     →      3000
```

따라서 브라우저에서:

```text
localhost:8080
```

으로 접근하면 Container의 `3000` 포트로 요청이 전달됩니다.

형식은 다음과 같습니다.

```text
-p [Host Port]:[Container Port]
```

예:

```bash
docker run -p 3000:3000 my-app
```

```text
Host 3000
    ↓
Container 3000
```

---

# 10. Docker Volume

Container를 삭제하면 그 안에서 만든 데이터도 함께 사라질 수 있습니다.

예를 들어:

```text
Container
│
└── /data
    └── important.txt
```

Container를 삭제하면 이 데이터도 사라질 수 있습니다.

그래서 Docker는 [[docker-volume|Volume]]을 제공합니다.

```text
Docker Container
       │
       │
       ↓
Docker Volume
       │
       └── 실제 데이터
```

Container가 삭제되어도 Volume은 별도로 유지할 수 있습니다.

```text
Container A
    ↓
  Volume
    ↑
Container B
```

새로운 Container가 동일한 Volume을 연결하여 기존 데이터를 사용할 수도 있습니다.

Volume은 Container의 생명주기와 데이터를 분리하는 기능입니다.

---

# 11. Docker Network

여러 Container가 서로 통신해야 하는 경우도 있습니다.

예를 들어 애플리케이션 Container와 데이터베이스 Container가 있다고 가정합니다.

```text
┌──────────────────┐
│ App Container    │
└────────┬─────────┘
         │
         │ Network
         ↓
┌──────────────────┐
│ DB Container     │
└──────────────────┘
```

[[docker-network|Docker Network]]를 사용하면 Container끼리 네트워크를 통해 통신할 수 있습니다.

예를 들어:

```text
Docker Network

├─ app
│   └─ Node.js
│
└─ database
    └─ PostgreSQL
```

같은 Network에 연결되어 있다면 서로의 Container 이름 등을 이용하여 통신하도록 구성할 수 있습니다.

```text
app
 ↓
database:5432
```

Container 사이의 네트워크 구성도 Docker가 맡습니다.

---

# 12. Docker Layer

Docker Image는 여러 Layer를 쌓아서 만듭니다.

예를 들어:

```dockerfile
FROM node:22

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

CMD ["npm", "start"]
```

개념적으로 다음과 같이 Layer가 쌓입니다.

```text
┌──────────────────────┐
│ Application Source   │
├──────────────────────┤
│ npm dependencies     │
├──────────────────────┤
│ Working Directory    │
├──────────────────────┤
│ Node.js 22           │
├──────────────────────┤
│ Base Linux Layer     │
└──────────────────────┘
```

Docker는 변경되지 않은 Layer를 재사용할 수 있습니다.

이를 Docker Layer Cache라고 합니다.

예를 들어 소스 코드만 변경되었다면 기존에 설치했던 npm 패키지 Layer를 재사용할 수 있습니다.

그래서 Dockerfile을 작성할 때 다음과 같이 작성하는 경우가 많습니다.

```dockerfile
COPY package*.json ./

RUN npm install

COPY . .
```

`package.json`을 전체 프로젝트보다 먼저 복사하는 이유가 이것입니다.

소스 코드만 변경되고 `package.json`이 변경되지 않았다면:

```text
package.json 동일
        ↓
npm install Layer 재사용
        ↓
소스 코드 Layer만 다시 생성
```

할 수 있기 때문에 Image Build 시간을 줄일 수 있습니다.

---

# 13. .dockerignore

Docker Image를 만들 때 불필요한 파일까지 포함하지 않도록 `.dockerignore`를 사용할 수 있습니다.

예:

```text
node_modules
.git
.env
README.md
npm-debug.log
```

개념적으로 `.gitignore`와 비슷합니다.

```text
프로젝트

├─ src                 O
├─ package.json        O
├─ node_modules        X
├─ .git                X
└─ README.md           X

        ↓

Docker Image
```

불필요한 파일을 빼면 Build Context가 줄어 Image 빌드가 가벼워집니다.

---

# 14. Docker 환경 변수

Container 실행 시 환경 변수를 전달할 수도 있습니다.

예를 들어:

```bash
docker run \
  -e NODE_ENV=production \
  my-app
```

Container에서는:

```text
NODE_ENV=production
```

환경 변수를 사용할 수 있습니다.

여러 환경 변수를 파일로 관리하는 경우:

```bash
docker run --env-file .env my-app
```

처럼 사용할 수도 있습니다.

단, 비밀번호나 API Key 같은 민감한 정보를 Docker Image 내부에 직접 포함하는 것은 피해야 합니다.

---

# 15. Docker 주요 명령어

## Image 만들기

```bash
docker build -t my-app .
```

---

## Image 목록 확인

```bash
docker images
```

---

## Container 실행

```bash
docker run my-app
```

---

## Port 연결하여 실행

```bash
docker run -p 3000:3000 my-app
```

---

## 백그라운드 실행

```bash
docker run -d -p 3000:3000 my-app
```

`-d`는 detached mode를 의미합니다.

터미널을 점유하지 않고 백그라운드에서 실행합니다.

---

## 실행 중인 Container 확인

```bash
docker ps
```

모든 Container를 확인하려면:

```bash
docker ps -a
```

---

## Container 중지

```bash
docker stop [container-id]
```

---

## Container 삭제

```bash
docker rm [container-id]
```

---

## Image 삭제

```bash
docker rmi [image-id]
```

---

## 로그 확인

```bash
docker logs [container-id]
```

실시간으로 확인하려면:

```bash
docker logs -f [container-id]
```

---

## Container 내부 접속

```bash
docker exec -it [container-id] sh
```

Container 내부에 들어가 파일이나 환경을 확인할 때 자주 사용합니다.

---

# 16. Docker의 전체 실행 흐름

Docker를 사용하는 가장 기본적인 흐름은 다음과 같습니다.

```text
1. 애플리케이션 개발

        ↓

2. Dockerfile 작성

        ↓

3. docker build

        ↓

4. Docker Image 생성

        ↓

5. docker run

        ↓

6. Container 생성

        ↓

7. 애플리케이션 실행
```

실제 명령어로 보면:

```bash
docker build -t my-app .
```

```text
Dockerfile
    ↓
Image
```

그리고:

```bash
docker run -p 3000:3000 my-app
```

```text
Image
  ↓
Container
  ↓
Application
```

---

# 17. Docker와 가상 머신(VM)의 차이

Docker를 이해할 때 가상 머신과의 차이를 알아두면 좋습니다.

가상 머신은 각각 별도의 운영체제를 실행합니다.

```text
Hardware
   ↓
Host OS
   ↓
Hypervisor
   │
   ├─ VM
   │   ├─ Guest OS
   │   └─ App
   │
   └─ VM
       ├─ Guest OS
       └─ App
```

Container는 일반적으로 Host의 OS Kernel을 공유하면서 격리된 환경을 제공합니다.

```text
Hardware
   ↓
Host OS
   ↓
Docker
   │
   ├─ Container
   │   └─ App
   │
   ├─ Container
   │   └─ App
   │
   └─ Container
       └─ App
```

그래서 Container는 보통 VM보다 가볍고 빨리 만들어집니다.

| 구분 | VM | Docker Container |
|---|---|---|
| 운영체제 | VM마다 Guest OS | Host Kernel 공유 |
| 크기 | 상대적으로 큼 | 상대적으로 작음 |
| 시작 속도 | 상대적으로 느림 | 빠름 |
| 리소스 사용 | 상대적으로 큼 | 상대적으로 적음 |
| 격리 | 강함 | 프로세스 수준 격리 |
| 배포 단위 | VM | Container |

---

# 18. Docker의 장점

## 실행 환경 표준화

```text
개발자 A
개발자 B
테스트 환경
실행 환경

        ↓

동일한 Docker Image
```

환경 차이로 발생하는 문제를 줄일 수 있습니다.

---

## 빠른 실행

Container는 전체 운영체제를 새로 부팅하는 방식이 아니기 때문에 비교적 빠르게 시작할 수 있습니다.

---

## 격리된 환경

각 Container는 서로 분리된 환경을 가질 수 있습니다.

```text
Container A
Node.js 22

Container B
Node.js 20

Container C
Python
```

서로 다른 실행 환경을 하나의 컴퓨터에서 운영할 수 있습니다.

---

## 재현 가능한 환경

Dockerfile이 존재하면 어떤 환경이 필요한지 코드로 확인할 수 있습니다.

```text
Dockerfile
=
실행 환경을 코드로 문서화
```

따라서 특정 개발자의 컴퓨터에 의존하는 환경을 줄일 수 있습니다.

---

## 쉬운 생성과 제거

필요한 Container를 생성했다가 사용하지 않으면 삭제할 수 있습니다.

```text
Image

 ↓ run

Container

 ↓ stop

중지

 ↓ rm

삭제
```

Image가 남아 있다면 동일한 환경의 Container를 다시 생성할 수 있습니다.

---

# 19. Docker의 단점과 주의점

Docker가 모든 문제를 해결해주는 것은 아닙니다.

## 학습해야 할 개념이 늘어남

단순히 애플리케이션만 실행하던 것과 달리 다음과 같은 개념을 알아야 합니다.

```text
Image
Container
Dockerfile
Volume
Network
Port
Layer
환경 변수
```

---

## Image 크기 관리

잘못 작성된 Dockerfile은 Image 크기가 매우 커질 수 있습니다.

예를 들어:

```text
좋지 않은 Image

2GB
```

불필요한 파일과 패키지를 제거하면:

```text
최적화된 Image

300MB
```

처럼 줄어들 수 있습니다.

Image 크기도 계속 관리해야 합니다.

---

## 데이터 관리

Container 내부에 중요한 데이터를 저장하면 Container 삭제 시 데이터가 사라질 수 있습니다.

따라서 영구적으로 보존해야 하는 데이터는 Volume 등을 이용하여 Container와 분리해야 합니다.

---

## 보안

Docker Image 안에 다음과 같은 정보를 직접 넣으면 안 됩니다.

```text
DB_PASSWORD
API_KEY
PRIVATE_KEY
ACCESS_TOKEN
```

특히 다음과 같이 Dockerfile에 직접 작성하는 것은 피해야 합니다.

```dockerfile
ENV API_KEY=my-secret-key
```

Image에 민감한 정보가 남을 수 있기 때문입니다.

---

# 20. Docker를 이해할 때 가장 중요한 개념

처음 공부할 때는 명령어를 외우기 전에 아래 흐름부터 이해하는 게 좋습니다.

```text
Dockerfile
    │
    │ docker build
    ↓
Docker Image
    │
    │ docker run
    ↓
Docker Container
    │
    ├─ Port
    ├─ Volume
    ├─ Network
    └─ Environment Variables
```

각각을 한 문장으로 정리하면 다음과 같습니다.

### Dockerfile

> Image를 어떻게 만들 것인지 정의한 설계도

### Image

> 애플리케이션과 실행 환경을 패키징한 실행 템플릿

### Container

> Image를 실제로 실행한 인스턴스

### Port

> Container 내부 서비스와 외부 네트워크를 연결

### Volume

> Container와 데이터를 분리하여 데이터를 지속적으로 보관

### Network

> Container끼리 통신할 수 있도록 연결

### Layer

> Docker Image를 효율적으로 저장하고 재사용하기 위한 계층 구조

---

# 21. 최종 정리

기존에는:

```text
Application

+

개발자가 직접 구성한 실행 환경
```

이었다면 Docker를 사용하면:

```text
Application
+
Runtime
+
Dependencies
+
Configuration

        ↓

Docker Image
```

형태로 실행 환경을 표준화할 수 있습니다.

그리고 이 Image를 실행하면:

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
    ↓
Application 실행
```

이라는 흐름이 만들어집니다.

따라서 Docker를 한 문장으로 정리하면 다음과 같습니다.

> Docker는 애플리케이션과 실행에 필요한 환경을 Image로 표준화하고, 이를 격리된 Container로 실행하는 플랫폼이다.