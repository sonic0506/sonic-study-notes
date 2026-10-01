---
aliases: [Docker, 도커]
tags: [docker, devops]
prerequisites: []
related: []
status: draft
created: 2026-10-01
---

# 도커 정리

도커의 핵심 개념인 이미지와 컨테이너, 그리고 자주 쓰는 기본 명령을 정리합니다.

## 이미지와 컨테이너

이미지는 애플리케이션 실행에 필요한 것을 모두 담은 읽기 전용 템플릿이고, 컨테이너는 이 이미지를 실행한 인스턴스입니다. 이미지가 설계도라면 컨테이너는 그 설계도로 지은 건물입니다.

이미지는 레이어 단위로 쌓입니다. [[dockerfile|Dockerfile]]의 명령 한 줄이 레이어 하나가 되고, 바뀌지 않은 레이어는 캐시를 재사용합니다.

## 기본 명령

```bash
docker pull nginx:1.27
docker run -d -p 8080:80 --name web nginx:1.27
docker ps
docker stop web
```

`-p 8080:80`은 호스트의 8080 포트를 컨테이너의 80 포트로 연결합니다.

## 정리

도커를 쓰면 개발 환경과 운영 환경의 차이가 줄어듭니다.
