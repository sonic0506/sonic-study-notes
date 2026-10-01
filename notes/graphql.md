---
aliases: [GraphQL, GraphQL]
tags: [api, graphql]
prerequisites:
  - "[[api]]"
related:
  - "[[websocket]]"
status: draft
created: 2026-10-01
---

# GraphQL

## 1. GraphQL이란?

GraphQL은 클라이언트가 **필요한 데이터의 모양을 직접 선언**할 수 있게 하는 [[api|API]] 질의 언어이자 실행 방식입니다. 서버는 제공 가능한 데이터 타입과 관계를 스키마로 정의합니다.

```graphql
query {
  user(id: 1) {
    name
    posts {
      title
    }
  }
}
```

클라이언트는 사용자 이름과 게시글 제목만 요청했고, 서버는 그 모양에 맞춰 응답합니다.

## 2. 왜 등장했을까?

REST에서는 화면 하나를 구성하기 위해 여러 Endpoint를 호출해야 할 수 있습니다.

```text
GET /users/1
GET /users/1/posts
GET /posts/최근글/comments
```

또는 응답에 필요하지 않은 데이터까지 포함되는 Over-fetching, 반대로 필요한 데이터가 부족해 추가 요청이 필요한 Under-fetching이 생길 수 있습니다. GraphQL은 클라이언트가 필요한 필드를 선택하고 연관 데이터까지 한 질의로 요청할 수 있게 합니다.

## 3. 스키마와 타입

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
}

type Query {
  user(id: ID!): User
}
```

- `type`: 데이터의 구조
- `ID!`: null이 될 수 없는 ID
- `[Post!]!`: 목록과 목록의 각 항목이 null이 아님
- `Query`: 조회할 수 있는 진입점

스키마는 문서이자 클라이언트와 서버 사이의 계약입니다. 타입을 기반으로 자동 완성, 검증, 코드 생성 도구를 활용할 수 있습니다.

## 4. Query, Mutation, Subscription

GraphQL에서는 서버의 기능을 크게 세 가지 형태로 구분합니다.

```text
Query
→ 데이터 조회

Mutation
→ 데이터 생성·수정·삭제

Subscription
→ 데이터 변경을 실시간으로 전달받기
```

REST API와 비교하면 다음과 비슷하게 이해할 수 있습니다.

| GraphQL      | 주요 역할        | REST API와 비교             |
| ------------ | ------------ | ------------------------ |
| Query        | 데이터 조회       | GET                      |
| Mutation     | 데이터 생성·수정·삭제 | POST, PUT, PATCH, DELETE |
| Subscription | 실시간 데이터 수신   | WebSocket, SSE 등의 실시간 통신 |

단, GraphQL과 REST는 구조가 달라서 완전히 같은 개념은 아니고, 위 표는 역할을 대략 비교한 것입니다.

### 4.1 Query

Query는 서버의 데이터를 **조회할 때** 사용합니다.

쉽게 말하면 서버에 다음과 같이 질문하는 것입니다.

> 1번 사용자의 이름과 이메일을 알려주세요.

```graphql
query {
  user(id: 1) {
    name
    email
  }
}
```

서버는 클라이언트가 요청한 필드만 반환합니다.

```json
{
  "data": {
    "user": {
      "name": "홍길동",
      "email": "hong@example.com"
    }
  }
}
```

이름만 필요하다면 `name`만 요청할 수도 있습니다.

```graphql
query {
  user(id: 1) {
    name
  }
}
```

응답에도 이름만 포함됩니다.

```json
{
  "data": {
    "user": {
      "name": "홍길동"
    }
  }
}
```

연관된 데이터도 함께 요청할 수 있습니다.

```graphql
query {
  user(id: 1) {
    name
    posts {
      id
      title
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "name": "홍길동",
      "posts": [
        {
          "id": 10,
          "title": "GraphQL 공부하기"
        },
        {
          "id": 11,
          "title": "REST API와 비교하기"
        }
      ]
    }
  }
}
```

REST API라면 사용자 정보와 게시글 목록을 서로 다른 Endpoint에서 조회해야 할 수 있습니다.

```http
GET /users/1
GET /users/1/posts
```

GraphQL에서는 필요한 관계를 하나의 Query로 표현할 수 있습니다.

정리하면 Query는 다음과 같은 기능에 사용합니다.

```text
사용자 정보 조회
상품 목록 조회
게시글 상세 조회
주문 내역 조회
검색 결과 조회
```

Query는 데이터를 읽는 용도이므로 일반적으로 서버의 데이터를 변경하지 않도록 설계합니다.

---

### 4.2 Mutation

Mutation은 서버의 데이터를 **생성하거나 변경하거나 삭제할 때** 사용합니다.

쉽게 말하면 서버에 다음과 같은 작업을 요청하는 것입니다.

```text
새로운 게시글을 만들어주세요.
사용자 이름을 변경해주세요.
10번 게시글을 삭제해주세요.
```

새로운 게시글을 생성하는 Mutation은 다음과 같이 작성할 수 있습니다.

```graphql
mutation {
  createPost(
    input: {
      title: "GraphQL 공부하기"
      content: "GraphQL의 기본 개념을 정리합니다."
    }
  ) {
    id
    title
    content
  }
}
```

`createPost`는 서버에 정의된 게시글 생성 기능입니다.

`input`에는 게시글 생성에 필요한 데이터를 전달합니다.

```text
createPost
→ 실행할 기능

input
→ 서버에 전달할 데이터

id, title, content
→ 작업이 끝난 후 응답받을 필드
```

서버는 게시글을 생성한 뒤 클라이언트가 요청한 결과를 반환합니다.

```json
{
  "data": {
    "createPost": {
      "id": 20,
      "title": "GraphQL 공부하기",
      "content": "GraphQL의 기본 개념을 정리합니다."
    }
  }
}
```

사용자 이름을 수정하는 Mutation은 다음과 같이 만들 수 있습니다.

```graphql
mutation {
  updateUser(
    id: 1
    input: {
      name: "김길동"
    }
  ) {
    id
    name
  }
}
```

게시글을 삭제할 때도 Mutation을 사용합니다.

```graphql
mutation {
  deletePost(id: 20) {
    id
    success
  }
}
```

GraphQL에서는 생성·수정·삭제를 모두 Mutation으로 처리합니다.

```text
Query
→ 서버의 데이터를 읽음

Mutation
→ 서버의 데이터를 변경함
```

REST API에서는 HTTP Method를 통해 작업을 구분합니다.

```http
POST   /posts
PATCH  /posts/20
DELETE /posts/20
```

GraphQL에서는 보통 Mutation 안에 정의된 기능 이름으로 작업을 구분합니다.

```graphql
createPost()
updatePost()
deletePost()
```

Mutation의 장점 중 하나는 작업 결과로 필요한 데이터를 바로 요청할 수 있다는 점입니다.

예를 들어 게시글을 생성한 뒤 생성된 게시글의 ID와 작성 시간을 바로 받을 수 있습니다.

```graphql
mutation {
  createPost(
    input: {
      title: "새 게시글"
    }
  ) {
    id
    title
    createdAt
  }
}
```

정리하면 Mutation은 다음과 같은 기능에 사용합니다.

```text
회원가입
게시글 생성
사용자 정보 수정
주문 생성
결제 요청
게시글 삭제
좋아요 등록과 취소
```

---

### 4.3 Subscription

Subscription은 서버에서 발생한 이벤트나 데이터 변경을 클라이언트가 **실시간으로 전달받을 때** 사용합니다.

Query와 Mutation은 기본적으로 요청을 보내고 응답을 한 번 받습니다.

```text
Query

클라이언트 → 데이터 조회 요청
클라이언트 ← 조회 결과
```

```text
Mutation

클라이언트 → 데이터 변경 요청
클라이언트 ← 변경 결과
```

Subscription은 연결을 유지하면서 서버가 새로운 데이터를 계속 전달할 수 있습니다.

```text
Subscription

클라이언트 → 실시간 이벤트 구독
클라이언트 ← 새로운 이벤트
클라이언트 ← 새로운 이벤트
클라이언트 ← 새로운 이벤트
```

채팅방의 새 메시지를 구독하는 예시는 다음과 같습니다.

```graphql
subscription {
  messageAdded(roomId: 10) {
    id
    senderName
    message
    createdAt
  }
}
```

이 Subscription을 실행하면 클라이언트는 10번 채팅방에 새 메시지가 등록되는 것을 기다립니다.

새로운 메시지가 등록될 때마다 서버가 다음과 같은 데이터를 전달합니다.

```json
{
  "data": {
    "messageAdded": {
      "id": 101,
      "senderName": "홍길동",
      "message": "안녕하세요.",
      "createdAt": "2026-10-01T15:00:00"
    }
  }
}
```

다른 메시지가 등록되면 같은 연결을 통해 새로운 응답이 다시 전달됩니다.

```json
{
  "data": {
    "messageAdded": {
      "id": 102,
      "senderName": "김철수",
      "message": "반갑습니다.",
      "createdAt": "2026-10-01T15:00:05"
    }
  }
}
```

Subscription은 다음과 같은 기능에 사용할 수 있습니다.

```text
채팅 메시지
실시간 알림
주문 상태 변경
배송 위치 변경
주식 가격 변경
작업 진행 상태
온라인 사용자 상태
```

Subscription은 실시간 통신이 필요하므로 일반적으로 [[websocket|WebSocket]] 같은 지속 연결 기술과 함께 구현합니다.

다만 GraphQL은 어떤 데이터를 구독하고 어떤 형태로 받을지를 정의하고, WebSocket은 해당 데이터를 실시간으로 전달하는 통신 수단입니다.

```text
GraphQL Subscription
→ 어떤 이벤트를 어떤 데이터 구조로 받을 것인가?

WebSocket
→ 이벤트를 실시간으로 전달하는 통신 연결
```

GraphQL Subscription이 반드시 WebSocket만 사용해야 하는 것은 아닙니다. 서버와 클라이언트 구현에 따라 SSE나 다른 스트리밍 방식을 사용할 수도 있습니다.

---

### 4.4 세 가지 기능을 함께 사용하는 예

채팅 서비스를 예로 들면 Query, Mutation, Subscription은 다음과 같이 역할을 나눌 수 있습니다.

#### 이전 메시지 조회

```graphql
query {
  messages(roomId: 10) {
    id
    senderName
    message
  }
}
```

```text
Query
→ 기존 채팅 메시지를 조회
```

#### 새로운 메시지 전송

```graphql
mutation {
  sendMessage(
    roomId: 10
    message: "안녕하세요."
  ) {
    id
    message
    createdAt
  }
}
```

```text
Mutation
→ 새로운 채팅 메시지를 서버에 저장
```

#### 새 메시지 실시간 수신

```graphql
subscription {
  messageAdded(roomId: 10) {
    id
    senderName
    message
    createdAt
  }
}
```

```text
Subscription
→ 다른 사용자가 보낸 새 메시지를 실시간으로 수신
```

전체 흐름은 다음과 같습니다.

```text
채팅방 입장
    ↓
Query로 이전 메시지 조회
    ↓
Subscription으로 새 메시지 구독
    ↓
Mutation으로 메시지 전송
    ↓
서버가 구독 중인 사용자에게 새 메시지 전달
```

---

### 4.5 한눈에 정리하기

| 구분           | 역할         |        실행 횟수 | 예시            |
| ------------ | ---------- | -----------: | ------------- |
| Query        | 데이터 조회     |   요청당 한 번 응답 | 사용자 조회, 상품 목록 |
| Mutation     | 데이터 변경     |   요청당 한 번 응답 | 게시글 생성·수정·삭제  |
| Subscription | 실시간 데이터 수신 | 연결 중 여러 번 응답 | 채팅, 알림, 상태 변경 |

가장 간단하게 기억하면 다음과 같습니다.

```text
Query
→ 보여주세요.

Mutation
→ 변경해주세요.

Subscription
→ 변경되면 계속 알려주세요.
```


## 5. Resolver

Resolver는 GraphQL 스키마에 정의된 필드의 실제 데이터를 가져오거나 작업을 실행하는 함수입니다.

```text
user(id: 1)
→ User Resolver가 사용자 조회

posts 필드
→ Post Resolver가 게시글 조회
```

예를 들어 다음 Query가 들어오면:

```graphql
query {
  user(id: 1) {
    name
    posts {
      title
    }
  }
}
```

서버는 사용자를 조회하는 Resolver와 해당 사용자의 게시글을 조회하는 Resolver를 순서대로 실행합니다.

Resolver를 사용하면 클라이언트가 요청한 필드에 맞춰 필요한 데이터를 조회할 수 있습니다. 하지만 목록에 포함된 관계 데이터를 각 Resolver가 개별적으로 조회하면 같은 형태의 DB Query가 반복되는 **N+1 문제**가 발생할 수 있습니다.

이 문제는 DataLoader를 이용한 일괄 조회, JOIN, ORM 관계 조회, 페이지네이션 등의 방법으로 해결할 수 있습니다.


## 6. 응답과 오류

```json
{
  "data": {
    "user": {
      "name": "홍길동"
    }
  },
  "errors": []
}
```

일부 필드에서 오류가 발생해도 성공한 데이터와 오류 정보가 함께 올 수 있습니다. 또한 HTTP 자체는 `200 OK`인데 GraphQL 응답의 `errors`에 업무 오류가 담기는 설계도 가능하므로, 클라이언트는 HTTP 상태 코드와 GraphQL 오류 구조를 모두 이해해야 합니다.

## 7. 장점과 한계

### 장점

- 화면에 필요한 필드만 요청할 수 있습니다.
- 연관 데이터를 하나의 질의로 표현할 수 있습니다.
- 강한 타입과 스키마 기반 도구를 활용할 수 있습니다.
- 여러 종류의 클라이언트가 각자 다른 데이터 모양을 요구할 때 유연합니다.

### 한계

- 서버의 실행과 성능 최적화가 복잡해질 수 있습니다.
- HTTP URL 단위의 기본 캐시를 그대로 활용하기 어렵습니다.
- 클라이언트가 지나치게 복잡하거나 비싼 질의를 보낼 수 있어 깊이·복잡도 제한이 필요합니다.
- 파일 업로드, 오류 정책, 권한 검사를 팀 규칙으로 잘 정해야 합니다.

## 8. REST를 완전히 대체할까?

반드시 그렇지는 않습니다. 단순 CRUD 서비스라면 REST가 더 이해하고 운영하기 쉬울 수 있습니다. 반대로 여러 화면과 클라이언트가 복잡한 관계 데이터를 각기 다른 모양으로 요구한다면 GraphQL의 장점이 커집니다.

한 서비스에서 외부 API는 REST, 프론트엔드 전용 집계 API는 GraphQL처럼 병행할 수도 있습니다.

## 9. 언제 고려하면 좋은가?

- 웹·모바일 등 클라이언트별 데이터 요구가 크게 다름
- 하나의 화면이 여러 연관 자원을 조합함
- API 변경 요청이 빈번하고 스키마 기반 협업이 중요함
- GraphQL 운영 복잡성을 감당할 팀과 도구가 있음

## 10. 핵심 정리

> GraphQL은 서버가 제공하는 타입과 관계 안에서 클라이언트가 필요한 데이터의 모양을 선언하는 API 방식이다.
