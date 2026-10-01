---
aliases: [REST API, REST API]
tags: [api, http]
prerequisites:
  - "[[api]]"
  - "[[http]]"
related:
  - "[[rpc]]"
status: draft
created: 2026-10-01
---

# REST API

## 1. REST API란?

REST는 [[http|HTTP]]를 이용해 서버의 데이터를 **자원(Resource)**으로 바라보고, URL과 HTTP Method로 자원을 다루는 설계 스타일입니다.

```http
GET /users/1
```

- `/users/1`: 1번 사용자라는 자원
- `GET`: 그 자원을 조회하는 행동

REST 원칙을 잘 반영한 [[api|API]]를 RESTful API라고 부릅니다. 실무의 “REST API”가 모든 원칙을 엄격히 지킨다는 뜻은 아니며, 자원 중심의 HTTP API를 넓게 가리키는 경우도 많습니다.

## 2. 자원과 표현

서버는 내부의 실제 객체나 DB 행을 클라이언트가 사용할 수 있는 형태로 바꿔 표현한 뒤 전달합니다. JSON이 가장 흔합니다.

```json
{
  "id": 1,
  "name": "홍길동",
  "role": "USER"
}
```

자원과 표현을 구분하면 서버 내부 구조를 바꾸더라도 API 응답 계약은 유지할 수 있습니다.

## 3. URL은 자원, Method는 행동

URL에는 가급적 동사보다 명사를 사용합니다.

```http
GET    /products
GET    /products/10
POST   /products
PATCH  /products/10
DELETE /products/10
```

의미는 다음과 같습니다.

| 요청 | 의미 |
|---|---|
| `GET /products` | 상품 목록 조회 |
| `GET /products/10` | 10번 상품 조회 |
| `POST /products` | 새 상품 생성 |
| `PATCH /products/10` | 10번 상품 일부 수정 |
| `DELETE /products/10` | 10번 상품 삭제 |

## 4. 계층 관계 표현

댓글이 특정 게시글에 속한다면 다음처럼 표현할 수 있습니다.

```http
GET /posts/10/comments
POST /posts/10/comments
```

하지만 URL 중첩이 지나치게 깊어지면 사용하기 어려워집니다.

```http
GET /users/1/posts/10/comments/3/replies
```

필요하다면 댓글 자체를 독립 자원으로 다루거나 Query Parameter를 활용합니다.

## 5. PUT과 PATCH

- `PUT`: 자원의 전체 표현을 교체한다는 의미로 사용
- `PATCH`: 자원의 일부만 변경할 때 사용

```http
PATCH /users/1
Content-Type: application/json

{
  "nickname": "길동이"
}
```

실무에서는 팀 규칙이나 백엔드 프레임워크에 따라 PUT을 부분 수정처럼 사용하는 경우도 있습니다. 어느 쪽이든 명세에 의미를 분명히 적고 일관되게 사용하면 됩니다.

## 6. Stateless

REST의 중요한 제약 중 하나는 각 요청이 처리에 필요한 정보를 자체적으로 포함해야 한다는 것입니다. 서버가 이전 요청의 대화 흐름을 암묵적으로 기억한다고 가정하지 않습니다.

이는 “서버에 어떤 상태도 저장하면 안 된다”는 뜻이 아닙니다. DB 데이터나 로그인 세션을 저장할 수 있지만, 개별 요청은 어떤 사용자이며 무엇을 요청하는지 판별할 정보를 제공해야 합니다.

## 7. 상태 코드와 오류 응답

성공과 실패를 HTTP 상태 코드에 맞게 표현합니다.

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "code": "PRODUCT_NOT_FOUND",
  "message": "상품을 찾을 수 없습니다."
}
```

상태 코드만으로 세부 원인을 모두 설명하기 어려우므로, 서비스 내부 오류 코드와 사용자·개발자용 메시지를 함께 두는 방식이 흔합니다.

## 8. 멱등성

같은 요청을 여러 번 보내도 서버의 최종 상태가 같다면 멱등하다고 합니다.

- GET: 여러 번 조회해도 상태를 바꾸지 않으므로 멱등
- PUT: 같은 값으로 여러 번 교체해도 최종 상태가 같으므로 멱등
- DELETE: 이미 삭제된 결과까지 고려하면 최종 상태는 같으므로 멱등
- POST: 요청할 때마다 새 주문이 생길 수 있어 일반적으로 멱등하지 않음

결제·주문 API에서는 네트워크 재시도 때문에 중복 생성이 생기지 않도록 `Idempotency-Key` 같은 별도 장치를 사용하기도 합니다.

## 9. 장점과 한계

### 장점

- HTTP 표준과 잘 맞아 이해하기 쉽습니다.
- 브라우저, 서버, 모바일 등 거의 모든 환경에서 사용할 수 있습니다.
- HTTP 캐시와 프록시 같은 웹 인프라를 활용하기 좋습니다.
- 자원별 URL이 명확해 디버깅과 문서화가 쉽습니다.

### 한계

- 화면에 필요한 데이터가 여러 자원에 흩어지면 요청이 많아질 수 있습니다.
- 서버가 정한 응답 때문에 불필요한 필드가 포함될 수 있습니다.
- 실시간 양방향 통신 자체를 해결하는 방식은 아닙니다.
- “승인”, “정산” 같은 명령형 기능은 자원 중심으로 표현하기 어색할 수 있습니다.

## 10. 언제 사용하면 좋은가?

- 회원·상품·게시글 등 일반적인 CRUD
- 외부에 공개하는 범용 API
- 팀이 단순하고 익숙한 구조를 원하는 경우
- HTTP 캐시와 표준 도구를 적극 활용하려는 경우

실시간 기능이 추가되면 REST는 그대로 두고 WebSocket 또는 SSE를 함께 사용할 수 있습니다.

## 11. 핵심 정리

> REST API는 URL로 자원을 표현하고 HTTP Method로 행동을 표현하는 HTTP 기반 설계 스타일이다.
