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

여러 컨테이너를 YAML 파일 하나로 정의하고 한 번에 띄우는 도구다.
웹 서버와 DB처럼 같이 떠야 하는 컨테이너를 `docker run` 명령 여러 줄로 관리하다 보면 옵션이 금방 엉킨다. Compose는 그 옵션을 `compose.yaml`에 적어 두게 해 준다.

## 기본 구조

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "8080:80"
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
```

- `services` 아래 항목 하나가 컨테이너 하나다.
- `image`는 어떤 이미지로 컨테이너를 만들지 정한다.
- 같은 파일에 있는 서비스끼리는 서비스 이름(`db`)으로 서로 접근할 수 있다.

## 자주 쓰는 명령

```bash
docker compose up -d     # 백그라운드로 전부 실행
docker compose ps        # 상태 확인
docker compose logs web  # 특정 서비스 로그
docker compose down      # 컨테이너와 네트워크 정리
```
