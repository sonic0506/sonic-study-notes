---
aliases: [HTTP, HTTP 기초]
tags: [network, http]
prerequisites: []
related:
  - "[[api]]"
status: draft
created: 2026-10-01
---

# HTTP 기초

## 1. HTTP란?

**HTTP(Hypertext Transfer Protocol)**는 클라이언트와 서버가 데이터를 주고받기 위한 통신 규칙입니다. 브라우저가 웹페이지를 받을 때뿐 아니라 웹·앱의 [[api|API]] 통신에도 널리 사용됩니다.

```text
클라이언트 ── HTTP 요청 ──> 서버
클라이언트 <─ HTTP 응답 ─── 서버
```

HTTP는 API 자체가 아니라 API가 데이터를 전달할 때 사용할 수 있는 기반 프로토콜입니다. REST와 GraphQL 모두 일반적으로 HTTP 위에서 동작합니다.

## 2. 요청과 응답

요청은 대체로 다음 요소로 구성됩니다.

- Method: 무엇을 하려는지 표현
- URL: 요청 대상의 위치
- Header: 인증, 데이터 형식 등의 부가 정보
- Body: 서버에 보낼 실제 데이터

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer access-token

{
  "name": "홍길동",
  "email": "hong@example.com"
}
```

응답에는 상태 코드, 헤더, 본문이 포함됩니다.

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": 1,
  "name": "홍길동"
}
```

## 3. HTTP Method

| Method | 일반적인 의미 | 예시 |
|---|---|---|
| GET | 조회 | 상품 목록 조회 |
| POST | 생성 또는 처리 요청 | 주문 생성 |
| PUT | 전체 교체 | 회원 정보 전체 교체 |
| PATCH | 일부 수정 | 닉네임만 변경 |
| DELETE | 삭제 | 게시글 삭제 |

Method의 의미를 일관되게 사용하면 주소만 보고도 API의 의도를 이해하기 쉬워집니다.

## 4. 상태 코드

상태 코드는 요청 처리 결과를 세 자리 숫자로 표현합니다.

| 범위 | 의미 | 대표 코드 |
|---|---|---|
| 2xx | 성공 | 200, 201, 204 |
| 3xx | 이동·캐시 | 301, 302, 304 |
| 4xx | 클라이언트 요청 문제 | 400, 401, 403, 404, 409 |
| 5xx | 서버 처리 문제 | 500, 502, 503 |

자주 헷갈리는 코드는 다음과 같습니다.

- `401 Unauthorized`: 인증 정보가 없거나 유효하지 않음
- `403 Forbidden`: 인증은 되었지만 해당 작업 권한이 없음
- `404 Not Found`: 요청 대상을 찾지 못함
- `409 Conflict`: 현재 상태와 요청이 충돌함

## 5. Header와 Body

Header는 요청과 응답을 해석하는 데 필요한 부가 정보입니다.

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer token
Cache-Control: max-age=60
```

- `Content-Type`: 보내는 본문의 형식
- `Accept`: 받고 싶은 응답 형식
- `Authorization`: 인증 정보
- `Cache-Control`: 캐시 정책

Body에는 JSON, 텍스트, 파일 등 실제 데이터가 들어갑니다. GET 요청은 보통 Body를 사용하지 않고 URL의 경로나 쿼리 문자열로 조건을 전달합니다.

## 6. URL, Path Parameter, Query Parameter

```http
GET /products/10
```

`10`은 특정 상품을 가리키는 Path Parameter입니다.

```http
GET /products?category=computer&page=2
```

`category`와 `page`는 필터·정렬·페이지 같은 선택 조건을 전달하는 Query Parameter입니다.

## 7. HTTP는 기본적으로 상태를 기억하지 않는다

HTTP는 각 요청을 독립적으로 처리하는 **Stateless** 성격을 가집니다. 서버가 사용자의 로그인 상태를 유지하려면 쿠키, 세션, 토큰 같은 별도 장치가 필요합니다.

```text
로그인 성공
→ 서버가 세션 쿠키 또는 토큰 발급
→ 이후 요청에 인증 정보 포함
→ 서버가 사용자를 식별
```

## 8. HTTP 버전은 왜 여러 개인가?

- HTTP/1.1: 널리 사용된 기본 버전
- HTTP/2: 하나의 연결에서 여러 요청을 효율적으로 처리하고 헤더를 압축
- HTTP/3: TCP 대신 QUIC 기반으로 연결 지연과 손실 상황을 개선

API의 논리 구조는 비슷하지만, 전송 효율과 연결 방식이 발전해 왔다고 이해하면 됩니다. gRPC는 일반적으로 HTTP/2의 스트리밍과 다중화 기능을 적극 활용합니다.

## 9. HTTPS

HTTPS는 HTTP 통신을 TLS로 암호화한 방식입니다. 요청·응답을 암호화하고, 연결한 서버가 올바른 서버인지 확인하며, 통신 중 데이터 변조를 탐지합니다. 개인정보나 인증 정보가 없어도 현대 서비스에서는 HTTPS 사용이 기본입니다.

## 10. 핵심 정리

> HTTP는 클라이언트와 서버가 요청과 응답을 주고받는 규칙이며, Method·URL·Header·Body·Status Code가 핵심 구성 요소다.


