---
aliases: [REST vs gRPC, REST와 gRPC 비교]
tags: [api, grpc]
prerequisites:
  - "[[rest-api]]"
  - "[[grpc]]"
related: []
status: draft
created: 2026-10-01
---

# REST vs gRPC

## 1. 먼저 비교 범위를 이해하기

[[rest-api|REST]]는 HTTP 기반 API 설계 스타일이고, [[grpc|gRPC]]는 RPC API를 구현하는 프레임워크입니다. 정확히 같은 종류의 개념은 아니지만, 서비스 통신 방식을 고를 때 자주 비교됩니다.

```text
REST: GET /users/1
gRPC: UserService.GetUser({ id: 1 })
```

## 2. 핵심 비교

| 항목 | REST | gRPC |
|---|---|---|
| 관점 | 자원 중심 | 서비스·메서드 중심 |
| 데이터 형식 | 주로 JSON | 주로 Protobuf 바이너리 |
| 전송 | 일반적인 HTTP | 주로 HTTP/2 |
| 계약 | OpenAPI 등을 선택 | `.proto`가 핵심 |
| 코드 생성 | 선택 사항 | 적극 활용 |
| 사람이 읽기 | 쉬움 | 전용 도구 필요 |
| 브라우저 사용 | 매우 쉬움 | gRPC-Web 등이 필요할 수 있음 |
| 스트리밍 | 별도 방식 필요 | 네 가지 호출 형태로 지원 |
| 주 사용처 | 외부 웹·앱 API | 내부 서비스 간 통신 |

## 3. 같은 기능 표현하기

REST:

```http
GET /users/1
```

```json
{
  "id": 1,
  "name": "홍길동"
}
```

gRPC:

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

REST는 URL과 Method가 계약의 중심이고, gRPC는 서비스·메서드·메시지 타입이 중심입니다.

## 4. REST가 더 어울리는 경우

- 브라우저와 모바일 앱이 직접 사용함
- 외부 개발자에게 공개할 API
- curl, 개발자 도구 등으로 쉽게 확인해야 함
- 일반 CRUD가 중심이고 성능 요구가 특별히 높지 않음
- 단순한 운영과 폭넓은 호환성이 중요함

## 5. gRPC가 더 어울리는 경우

- 내부 마이크로서비스 간 호출이 많음
- 여러 언어의 서버가 같은 타입 계약을 공유함
- 작은 메시지를 빈번하게 주고받음
- 양방향을 포함한 스트리밍이 필요함
- 코드 생성과 엄격한 계약 관리가 유리함

## 6. 함께 사용하는 구조

```text
브라우저·앱
    │ REST / GraphQL
    ▼
API Gateway 또는 BFF
    │ gRPC
    ├─ 사용자 서비스
    ├─ 주문 서비스
    └─ 결제 서비스
```

외부에는 호환성이 좋은 REST를 제공하고, 내부에는 gRPC를 적용할 수 있습니다. 다만 서비스가 작다면 모든 통신을 REST로 통일하는 편이 운영상 더 나을 수도 있습니다.

## 7. 성능만으로 선택하지 않기

gRPC의 바이너리 메시지가 작고 효율적인 것은 장점이지만, 시스템 병목이 DB나 외부 API에 있다면 체감 효과가 작을 수 있습니다. 다음 비용도 함께 봐야 합니다.

- Protobuf 계약 관리
- 코드 생성과 빌드 과정
- 프록시와 로드밸런서 설정
- 관측·디버깅 도구
- 팀 학습 비용

## 8. 핵심 정리

> REST는 범용성과 접근성이 강하고, gRPC는 엄격한 타입 계약과 효율적인 내부 서비스 통신에 강하다.
