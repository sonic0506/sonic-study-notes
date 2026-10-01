---
aliases: [gRPC, gRPC]
tags: [api, grpc]
prerequisites:
  - "[[rpc]]"
  - "[[http]]"
related: []
status: draft
created: 2026-10-01
---

# gRPC

## 1. gRPC란?

gRPC는 [[rpc|RPC]] 방식의 API를 만들기 위한 오픈소스 프레임워크입니다. 보통 **Protocol Buffers(Protobuf)**로 서비스와 메시지 타입을 정의하고, HTTP/2를 기반으로 통신합니다.

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  int64 id = 1;
}
```

이 `.proto` 파일은 API 계약이며, 여러 언어의 클라이언트와 서버 코드를 생성하는 데 사용됩니다.

## 2. 기본 동작 흐름

```text
.proto 계약 작성
→ 코드 생성
→ 클라이언트 Stub 호출
→ Protobuf 바이너리 직렬화
→ HTTP/2 전송
→ 서버 메서드 실행
→ 타입이 정해진 응답 반환
```

클라이언트는 `UserService.GetUser()` 같은 메서드를 호출하며, 생성된 Stub이 네트워크 처리를 담당합니다.

## 3. Protobuf

Protobuf는 데이터를 구조화하고 바이너리 형식으로 직렬화하는 방법입니다.

```protobuf
message User {
  int64 id = 1;
  string name = 2;
}
```

`= 1`, `= 2`는 단순한 순서 표시가 아니라 전송 데이터에서 필드를 식별하는 번호입니다. 배포 후 필드 이름을 바꾸는 것보다 번호를 잘 보존하는 것이 중요하며, 삭제한 번호는 재사용하지 않는 편이 안전합니다.

JSON보다 사람이 바로 읽기는 어렵지만 데이터 크기가 작고 파싱이 효율적이며 타입 계약이 명확합니다.

## 4. 네 가지 호출 방식

| 방식 | 요청 | 응답 | 예시 |
|---|---|---|---|
| Unary | 1개 | 1개 | 사용자 한 명 조회 |
| Server Streaming | 1개 | 여러 개 | 로그 연속 수신 |
| Client Streaming | 여러 개 | 1개 | 데이터 조각 업로드 |
| Bidirectional Streaming | 여러 개 | 여러 개 | 양방향 실시간 처리 |

gRPC에서는 단순 요청·응답과 스트리밍을 하나의 서비스 계약 안에서 함께 정의할 수 있습니다.

## 5. Deadline과 오류

원격 호출에는 언제까지 응답을 기다릴지 Deadline을 지정하는 것이 중요합니다. Deadline이 없으면 장애 난 서버를 계속 기다리면서 요청이 쌓일 수 있습니다.

gRPC는 `INVALID_ARGUMENT`, `NOT_FOUND`, `UNAUTHENTICATED`, `PERMISSION_DENIED`, `UNAVAILABLE` 같은 상태 코드를 제공합니다. 업무 오류와 일시적 네트워크 오류를 구분해야 안전한 재시도 정책을 만들 수 있습니다.

## 6. 장점과 한계

### 장점

- 계약에서 여러 언어의 코드를 생성할 수 있습니다.
- 바이너리 직렬화로 메시지가 작고 처리 효율이 좋습니다.
- HTTP/2 기반 다중화와 스트리밍을 활용합니다.
- 타입이 명확해 내부 서비스 간 계약 관리에 유리합니다.

### 한계

- 브라우저에서 일반 gRPC를 직접 쓰기 어렵고 gRPC-Web이나 중간 프록시가 필요할 수 있습니다.
- JSON처럼 요청·응답을 눈으로 바로 읽기 어려워 전용 도구가 필요합니다.
- `.proto` 호환성 규칙과 코드 생성 과정을 관리해야 합니다.
- 작은 서비스에는 운영 복잡성이 이점보다 클 수 있습니다.

## 7. REST보다 무조건 빠를까?

gRPC가 효율적인 경우가 많지만 “항상 더 빠르다”라고 단정할 수는 없습니다. 실제 성능은 데이터 크기, 네트워크, 서버 로직, DB 조회, 호출 횟수, 압축, 인프라 설정에 좌우됩니다. DB 쿼리가 병목이라면 전송 방식을 바꿔도 효과가 작습니다. 그래서 기술은 측정된 문제를 기준으로 골라야 합니다.

## 8. 언제 사용하면 좋은가?

- 마이크로서비스 사이의 고빈도 내부 통신
- 여러 언어로 작성된 서비스가 같은 계약을 공유함
- 스트리밍과 타입 안정성이 중요함
- 팀이 Protobuf와 관련 인프라를 운영할 수 있음

브라우저·외부 개발자용 공개 API는 REST나 GraphQL을 제공하고, 내부 통신만 gRPC로 구성하는 조합도 흔합니다.

## 9. 핵심 정리

> gRPC는 Protobuf 계약과 코드 생성을 중심으로 타입이 명확하고 효율적인 원격 호출을 제공하는 RPC 프레임워크다.
