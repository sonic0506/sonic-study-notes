---
aliases: [GraphQL N+1 Problem and DataLoader, GraphQL N+1 문제와 DataLoader]
tags: [api, graphql, database]
prerequisites:
  - "[[graphql]]"
related: []
status: draft
created: 2026-10-01
---

# GraphQL N+1 문제와 DataLoader

## 1. N+1 문제란?

N+1 문제는 목록 데이터를 한 번 조회한 다음, 목록에 포함된 각 항목의 관계 데이터를 조회하기 위해 DB Query가 반복적으로 실행되는 문제입니다.

```text
목록 조회
→ 1번

목록에 포함된 각 항목의 관계 데이터 조회
→ N번

전체 조회
→ 1 + N번
```

예를 들어 게시글 10개와 각 게시글의 작성자를 조회한다고 가정하겠습니다.

```text
게시글 목록 조회
→ 1번

게시글 10개의 작성자 조회
→ 10번

전체 DB 조회
→ 11번
```

목록 개수가 증가할수록 DB Query 수도 함께 증가하기 때문에 응답 속도 저하와 DB 부하로 이어질 수 있습니다.

N+1 문제는 [[graphql|GraphQL]]에서만 발생하는 것은 아닙니다. REST API, ORM, 반복문 안에서 DB를 조회하는 코드에서도 발생할 수 있습니다.

GraphQL에서는 필드마다 Resolver를 별도로 구현할 수 있기 때문에 관계 데이터를 단순하게 조회하면 N+1 문제가 발생하기 쉽습니다.

---

## 2. N+1 문제가 발생하는 과정

게시글과 작성자를 나타내는 GraphQL 스키마가 있다고 가정하겠습니다.

```graphql
type Post {
  id: ID!
  title: String!
  author: User!
}

type User {
  id: ID!
  name: String!
}

type Query {
  posts: [Post!]!
}
```

클라이언트가 게시글 목록과 각 게시글의 작성자를 요청합니다.

```graphql
query GetPosts {
  posts {
    id
    title
    author {
      id
      name
    }
  }
}
```

서버는 먼저 `Query.posts` Resolver를 실행합니다.

```typescript
const resolvers = {
  Query: {
    posts: async () => {
      return postRepository.findAll();
    },
  },
};
```

DB에서는 게시글 목록을 조회하는 Query가 한 번 실행됩니다.

```sql
SELECT *
FROM posts;
```

게시글이 10개 조회되었다고 가정하겠습니다.

```text
Post 1
→ authorId: 10

Post 2
→ authorId: 20

Post 3
→ authorId: 30

...

Post 10
→ authorId: 100
```

GraphQL은 각 게시글의 `author` 필드를 채우기 위해 `Post.author` Resolver를 실행합니다.

```typescript
const resolvers = {
  Query: {
    posts: async () => {
      return postRepository.findAll();
    },
  },

  Post: {
    author: async (post) => {
      return userRepository.findById(post.authorId);
    },
  },
};
```

`Post.author` Resolver는 게시글마다 한 번씩 실행됩니다.

따라서 다음 SQL이 반복적으로 실행될 수 있습니다.

```sql
SELECT * FROM users WHERE id = 10;
SELECT * FROM users WHERE id = 20;
SELECT * FROM users WHERE id = 30;
SELECT * FROM users WHERE id = 40;
SELECT * FROM users WHERE id = 50;
SELECT * FROM users WHERE id = 60;
SELECT * FROM users WHERE id = 70;
SELECT * FROM users WHERE id = 80;
SELECT * FROM users WHERE id = 90;
SELECT * FROM users WHERE id = 100;
```

전체 조회 횟수는 다음과 같습니다.

```text
게시글 목록 조회
→ 1번

각 게시글의 작성자 조회
→ 10번

전체
→ 11번
```

이것이 가장 기본적인 N+1 문제입니다.

---

## 3. 데이터가 많아질 때의 문제

목록 개수가 많아질수록 DB Query 수도 증가합니다.

|  게시글 수 | 게시글 목록 조회 | 작성자 조회 |  전체 조회 |
| -----: | --------: | -----: | -----: |
|    10개 |        1번 |    10번 |    11번 |
|   100개 |        1번 |   100번 |   101번 |
| 1,000개 |        1번 | 1,000번 | 1,001번 |

게시글 조회 한 번의 응답이 느리지 않더라도 수십 번 또는 수백 번 반복되면 전체 응답 시간이 길어질 수 있습니다.

또한 여러 사용자가 동시에 같은 API를 요청하면 DB 연결이 빠르게 사용될 수 있습니다.

```text
사용자 1명의 요청
→ DB Query 101번

사용자 100명이 동시에 요청
→ 최대 10,100번의 DB Query
```

실제 실행 횟수와 성능은 DB, ORM, 캐시, 동시 처리 방식에 따라 달라지지만, 목록 크기에 따라 Query 수가 증가하는 구조 자체가 문제가 됩니다.

---

## 4. 관계가 깊어질수록 커지는 문제

다음처럼 게시글, 작성자, 댓글, 댓글 작성자를 한 번에 요청할 수 있습니다.

```graphql
query GetPosts {
  posts {
    id
    title

    author {
      id
      name
    }

    comments {
      id
      content

      author {
        id
        name
      }
    }
  }
}
```

Resolver를 단순하게 구현하면 다음 조회가 발생할 수 있습니다.

```text
게시글 목록 조회
→ 1번

각 게시글의 작성자 조회
→ 게시글 수만큼

각 게시글의 댓글 목록 조회
→ 게시글 수만큼

각 댓글의 작성자 조회
→ 댓글 수만큼
```

게시글이 20개이고 각 게시글에 댓글이 10개씩 있다면 다음과 같은 조회 구조가 만들어질 수 있습니다.

```text
게시글 목록
→ 1번

게시글 작성자
→ 20번

게시글별 댓글
→ 20번

댓글 작성자
→ 200번

전체
→ 최대 241번
```

관계가 여러 단계로 연결될수록 N+1 문제가 반복되어 DB Query가 크게 증가할 수 있습니다.

---

## 5. N+1 문제를 발견하는 방법

N+1 문제는 실제 실행되는 SQL을 확인하는 것이 가장 정확합니다.

다음과 같이 ID만 변경된 동일한 SQL이 반복된다면 N+1 문제를 의심할 수 있습니다.

```sql
SELECT * FROM users WHERE id = 10;
SELECT * FROM users WHERE id = 20;
SELECT * FROM users WHERE id = 30;
SELECT * FROM users WHERE id = 40;
```

다음 항목을 확인합니다.

```text
GraphQL 요청 한 번에 DB Query가 몇 번 실행되는가?

동일한 형태의 SELECT가 ID만 바뀌면서 반복되는가?

목록 개수가 늘어날수록 Query 수도 함께 증가하는가?

Resolver 안의 findById()가 목록 항목마다 실행되는가?

관계가 깊어질수록 Query 수가 빠르게 증가하는가?
```

확인에 사용할 수 있는 도구는 다음과 같습니다.

```text
ORM Query 로그

DB Slow Query 로그

APM

GraphQL Resolver 추적

요청별 Query 개수 측정

분산 추적 시스템
```

코드에 DataLoader를 추가했다면 실제 SQL 로그에서 일괄 조회가 실행되는지 검증해야 합니다.

---

## 6. DataLoader란?

DataLoader는 여러 Resolver에서 발생한 개별 데이터 조회 요청을 모아 한 번에 조회하도록 도와주는 도구입니다.

핵심 기능은 다음 두 가지입니다.

```text
Batching
→ 여러 개별 조회를 하나의 일괄 조회로 합침

Caching
→ 같은 요청 안에서 동일한 데이터를 반복 조회하지 않음
```

기존 Resolver에서는 사용자 한 명을 직접 조회했습니다.

```typescript
const resolvers = {
  Post: {
    author: async (post) => {
      return userRepository.findById(post.authorId);
    },
  },
};
```

게시글이 10개라면 `findById()`가 최대 10번 실행됩니다.

DataLoader를 사용하면 Resolver는 사용자 ID만 Loader에 전달합니다.

```typescript
const resolvers = {
  Post: {
    author: async (post, _, context) => {
      return context.loaders.user.load(post.authorId);
    },
  },
};
```

각 Resolver는 다음과 같이 개별 ID를 요청합니다.

```typescript
userLoader.load("10");
userLoader.load("20");
userLoader.load("30");
```

겉으로 보면 세 번 조회하는 것처럼 보이지만 DataLoader는 같은 실행 흐름에서 들어온 요청을 모읍니다.

```text
["10", "20", "30"]
```

그다음 하나의 일괄 Query로 조회합니다.

```sql
SELECT *
FROM users
WHERE id IN ('10', '20', '30');
```

결과적으로 다음과 같이 변경됩니다.

```text
DataLoader 사용 전

게시글 목록 조회
→ 1번

작성자 조회
→ 10번

전체
→ 11번
```

```text
DataLoader 사용 후

게시글 목록 조회
→ 1번

작성자 일괄 조회
→ 1번

전체
→ 2번
```

---

## 7. DataLoader 설치

Node.js 프로젝트에서는 `dataloader` 패키지를 사용할 수 있습니다.

```bash
npm install dataloader
```

또는 다음 명령어를 사용할 수 있습니다.

```bash
pnpm add dataloader
```

```bash
yarn add dataloader
```

---

## 8. 사용자 DataLoader 만들기

사용자 ID 여러 개를 한 번에 조회하는 DataLoader를 만들어 보겠습니다.

```typescript
import DataLoader from "dataloader";

type User = {
  id: string;
  name: string;
};

export function createUserLoader() {
  return new DataLoader<string, User | null>(
    async (userIds) => {
      const users = await userRepository.findByIds([
        ...userIds,
      ]);

      const userMap = new Map(
        users.map((user) => [user.id, user]),
      );

      return userIds.map((userId) => {
        return userMap.get(userId) ?? null;
      });
    },
  );
}
```

DataLoader의 Batch 함수는 다음 ID 배열을 전달받습니다.

```typescript
["10", "20", "30"]
```

Repository에서는 여러 사용자 ID를 한 번에 조회합니다.

```typescript
async function findByIds(userIds: string[]) {
  return database.user.findMany({
    where: {
      id: {
        in: userIds,
      },
    },
  });
}
```

실제 SQL은 다음과 비슷합니다.

```sql
SELECT *
FROM users
WHERE id IN ('10', '20', '30');
```

---

## 9. 결과 순서를 다시 맞추는 이유

DataLoader의 Batch 함수는 입력받은 ID와 같은 순서로 결과를 반환해야 합니다.

다음 순서로 ID를 전달받았다고 가정하겠습니다.

```typescript
["30", "10", "20"]
```

하지만 DB에서 반드시 같은 순서로 반환된다는 보장은 없습니다.

```typescript
[
  {
    id: "10",
    name: "홍길동",
  },
  {
    id: "20",
    name: "김철수",
  },
  {
    id: "30",
    name: "이영희",
  },
]
```

이 결과를 그대로 반환하면 요청한 ID와 사용자 데이터의 위치가 일치하지 않습니다.

따라서 사용자 ID를 기준으로 Map을 만듭니다.

```typescript
const userMap = new Map(
  users.map((user) => [user.id, user]),
);
```

그다음 요청받은 ID 순서대로 결과를 다시 구성합니다.

```typescript
return userIds.map((userId) => {
  return userMap.get(userId) ?? null;
});
```

최종 결과는 다음과 같습니다.

```typescript
[
  {
    id: "30",
    name: "이영희",
  },
  {
    id: "10",
    name: "홍길동",
  },
  {
    id: "20",
    name: "김철수",
  },
]
```

Batch 함수는 다음 조건을 지켜야 합니다.

```text
입력받은 ID 배열과 반환 배열의 길이가 같아야 함

각 ID와 반환 데이터의 배열 위치가 일치해야 함

해당 데이터가 없다면 그 위치에 null 또는 Error를 반환해야 함
```

---

## 10. Resolver에서 DataLoader 사용하기

DataLoader를 GraphQL Context에 넣습니다.

```typescript
function createContext() {
  return {
    loaders: {
      user: createUserLoader(),
    },
  };
}
```

Resolver에서는 Repository를 직접 호출하지 않고 Loader를 사용합니다.

```typescript
const resolvers = {
  Query: {
    posts: async () => {
      return postRepository.findAll();
    },
  },

  Post: {
    author: async (post, _, context) => {
      return context.loaders.user.load(post.authorId);
    },
  },
};
```

전체 흐름은 다음과 같습니다.

```text
클라이언트가 게시글과 작성자를 요청
    ↓
Query.posts Resolver가 게시글 목록 조회
    ↓
각 Post.author Resolver가 userLoader.load() 호출
    ↓
DataLoader가 작성자 ID를 모음
    ↓
사용자를 WHERE IN으로 일괄 조회
    ↓
각 게시글에 작성자 데이터 배분
    ↓
GraphQL 응답 반환
```

---

## 11. DataLoader의 요청 단위 캐시

여러 게시글을 같은 사용자가 작성할 수 있습니다.

```text
Post 1
→ authorId 10

Post 2
→ authorId 10

Post 3
→ authorId 10
```

DataLoader를 사용하지 않으면 동일한 사용자를 반복해서 조회할 수 있습니다.

```sql
SELECT * FROM users WHERE id = 10;
SELECT * FROM users WHERE id = 10;
SELECT * FROM users WHERE id = 10;
```

DataLoader는 같은 요청 안에서 이미 조회한 ID의 결과를 기억합니다.

```typescript
userLoader.load("10");
userLoader.load("10");
userLoader.load("10");
```

따라서 동일한 ID를 반복해서 조회하지 않습니다.

```sql
SELECT *
FROM users
WHERE id IN ('10');
```

하지만 DataLoader의 캐시는 Redis처럼 서버 전체에서 공유하는 캐시가 아닙니다.

```text
Redis 또는 공용 캐시
→ 여러 요청과 사용자 사이에서 데이터 공유

DataLoader 캐시
→ 하나의 요청 안에서만 중복 조회 방지
```

---

## 12. DataLoader를 요청마다 생성해야 하는 이유

DataLoader는 일반적으로 GraphQL 요청마다 새로 만들어야 합니다.

```typescript
function createContext() {
  return {
    loaders: {
      user: createUserLoader(),
      postsByUser: createPostsByUserLoader(),
      commentsByPost: createCommentsByPostLoader(),
    },
  };
}
```

하나의 GraphQL 요청이 끝나면 해당 Loader와 캐시도 더 이상 사용하지 않습니다.

```text
GraphQL 요청 시작
    ↓
DataLoader 생성
    ↓
해당 요청에서 Batching과 Caching
    ↓
GraphQL 응답 완료
    ↓
DataLoader 캐시 제거
```

DataLoader를 서버 전체의 Singleton으로 만들면 다음 문제가 발생할 수 있습니다.

```text
서로 다른 사용자의 데이터가 같은 캐시에 저장될 수 있음

권한이 다른 사용자에게 잘못된 데이터가 반환될 수 있음

수정 전 데이터가 캐시에 남을 수 있음

캐시가 계속 증가해 메모리를 사용할 수 있음
```

잘못된 예시는 다음과 같습니다.

```typescript
const userLoader = createUserLoader();

const context = {
  loaders: {
    user: userLoader,
  },
};
```

이 Loader를 모든 요청에서 공유하면 요청별 인증과 권한을 안전하게 반영하기 어려울 수 있습니다.

다음처럼 요청이 들어올 때마다 생성하는 편이 안전합니다.

```typescript
async function createContext({ request }) {
  const currentUser = await authenticate(request);

  return {
    currentUser,

    loaders: {
      user: createUserLoader(currentUser),
    },
  };
}
```

---

## 13. 일대다 관계 DataLoader

작성자처럼 ID 하나로 데이터 하나를 조회하는 관계뿐 아니라, 게시글별 댓글처럼 데이터 목록을 조회할 때도 DataLoader를 사용할 수 있습니다.

다음 Query를 예로 들어보겠습니다.

```graphql
query {
  posts {
    id
    title
    comments {
      id
      content
    }
  }
}
```

단순한 Resolver는 다음과 같습니다.

```typescript
const resolvers = {
  Post: {
    comments: async (post) => {
      return commentRepository.findByPostId(post.id);
    },
  },
};
```

게시글이 10개라면 댓글 조회도 최대 10번 실행됩니다.

```sql
SELECT * FROM comments WHERE post_id = 1;
SELECT * FROM comments WHERE post_id = 2;
SELECT * FROM comments WHERE post_id = 3;
```

게시글 ID별 댓글을 일괄 조회하는 DataLoader를 만들 수 있습니다.

```typescript
import DataLoader from "dataloader";

export function createCommentsByPostLoader() {
  return new DataLoader<string, Comment[]>(
    async (postIds) => {
      const comments =
        await commentRepository.findByPostIds([
          ...postIds,
        ]);

      const commentsByPostId =
        new Map<string, Comment[]>();

      for (const comment of comments) {
        const currentComments =
          commentsByPostId.get(comment.postId) ?? [];

        currentComments.push(comment);

        commentsByPostId.set(
          comment.postId,
          currentComments,
        );
      }

      return postIds.map((postId) => {
        return commentsByPostId.get(postId) ?? [];
      });
    },
  );
}
```

Repository는 다음과 같은 SQL을 실행합니다.

```sql
SELECT *
FROM comments
WHERE post_id IN ('1', '2', '3');
```

Resolver에서는 게시글 ID를 Loader에 전달합니다.

```typescript
const resolvers = {
  Post: {
    comments: async (post, _, context) => {
      return context.loaders.commentsByPost.load(
        post.id,
      );
    },
  },
};
```

여러 게시글의 댓글을 한 번에 조회한 다음 `post_id` 기준으로 분류하여 각 게시글에 전달합니다.

---

## 14. JOIN을 사용한 해결 방법

DataLoader를 사용하지 않고 JOIN으로 관계 데이터를 한 번에 조회할 수도 있습니다.

```sql
SELECT
  posts.id,
  posts.title,
  users.id AS author_id,
  users.name AS author_name
FROM posts
JOIN users
  ON users.id = posts.author_id;
```

DB Query 한 번으로 게시글과 작성자를 함께 가져올 수 있습니다.

```text
게시글 목록
+ 작성자 정보
→ 한 번의 JOIN Query
```

관계 데이터가 항상 필요하다면 JOIN이 더 단순하고 효율적일 수 있습니다.

하지만 다음 Query처럼 작성자가 필요하지 않을 수도 있습니다.

```graphql
query {
  posts {
    id
    title
  }
}
```

클라이언트가 작성자를 요청하지 않았는데도 항상 JOIN을 실행하면 불필요한 데이터를 조회하게 됩니다.

따라서 다음 기준으로 판단할 수 있습니다.

```text
관계 데이터가 항상 필요함
→ JOIN 고려

관계 데이터가 선택적으로 요청됨
→ DataLoader 고려
```

---

## 15. ORM 관계 조회

ORM에서 제공하는 관계 조회 기능을 사용할 수도 있습니다.

Prisma 예시는 다음과 같습니다.

```typescript
const posts = await prisma.post.findMany({
  include: {
    author: true,
  },
});
```

JPA에서는 Fetch Join을 사용할 수 있습니다.

```java
@Query("""
    SELECT p
    FROM Post p
    JOIN FETCH p.author
    """)
List<Post> findAllWithAuthor();
```

ORM의 관계 조회 기능을 사용하더라도 실제 실행되는 SQL을 확인해야 합니다.

다음과 같은 상황이 있을 수 있기 때문입니다.

```text
코드에서는 관계 한 번 조회
→ 실제로는 JOIN 한 번 실행

또는

코드에서는 관계 한 번 조회
→ 내부적으로 항목별 SELECT 반복
```

ORM 설정만 보고 N+1 문제가 해결됐다고 판단하지 말고 Query 로그를 함께 확인해야 합니다.

---

## 16. 직접 일괄 조회

Resolver나 Service에서 ID를 직접 모아 일괄 조회할 수도 있습니다.

```typescript
const posts = await postRepository.findAll();

const authorIds = [
  ...new Set(
    posts.map((post) => post.authorId),
  ),
];

const authors =
  await userRepository.findByIds(authorIds);
```

조회한 작성자를 Map으로 만듭니다.

```typescript
const authorMap = new Map(
  authors.map((author) => [author.id, author]),
);
```

각 게시글에 작성자를 연결합니다.

```typescript
const result = posts.map((post) => {
  return {
    ...post,
    author: authorMap.get(post.authorId),
  };
});
```

조회 구조가 단순하고 고정되어 있다면 이 방법도 충분히 사용할 수 있습니다.

하지만 관계 필드가 많아지면 다음 코드가 반복될 수 있습니다.

```text
ID 모으기

일괄 조회하기

Map 만들기

원래 데이터에 다시 연결하기
```

여러 Resolver에서 같은 패턴이 반복된다면 DataLoader로 공통화하는 것이 관리하기 쉽습니다.

---

## 17. 페이지네이션과 함께 사용하기

DataLoader를 사용해도 한 번에 지나치게 많은 데이터를 요청하면 서버와 DB에 부담이 됩니다.

다음 Query는 모든 게시글과 작성자를 가져올 수 있습니다.

```graphql
query {
  posts {
    id
    title
    author {
      id
      name
    }
  }
}
```

게시글이 10,000개라면 DataLoader가 작성자 ID를 한 번에 모으더라도 매우 큰 `IN` Query가 만들어질 수 있습니다.

```sql
SELECT *
FROM users
WHERE id IN (
  '1',
  '2',
  '3',
  ...
  '10000'
);
```

따라서 목록 데이터에는 페이지네이션을 함께 적용하는 것이 좋습니다.

```graphql
query {
  posts(page: 1, size: 20) {
    items {
      id
      title

      author {
        id
        name
      }
    }

    page
    size
    totalCount
  }
}
```

```text
게시글 20개 조회
→ 작성자 ID 최대 20개 수집
→ 작성자 일괄 조회
```

DataLoader는 Query 횟수를 줄이고, 페이지네이션은 한 번에 처리하는 데이터 양을 제한합니다.

두 방법은 서로 다른 문제를 해결하므로 함께 사용하는 것이 좋습니다.

---

## 18. DB 인덱스

DataLoader로 DB Query 횟수를 줄여도 개별 Query 실행 자체가 느리면 성능 문제는 남아 있을 수 있습니다.

관계 조회에 자주 사용하는 컬럼에는 적절한 인덱스가 필요합니다.

게시글의 작성자 ID를 자주 조회한다면 다음 인덱스를 고려할 수 있습니다.

```sql
CREATE INDEX idx_posts_author_id
ON posts(author_id);
```

게시글 ID로 댓글을 자주 조회한다면 다음 인덱스를 고려할 수 있습니다.

```sql
CREATE INDEX idx_comments_post_id
ON comments(post_id);
```

댓글 작성자를 자주 조회한다면 다음 인덱스를 고려할 수 있습니다.

```sql
CREATE INDEX idx_comments_author_id
ON comments(author_id);
```

DataLoader와 인덱스의 역할은 다릅니다.

```text
DataLoader
→ DB Query가 반복되는 횟수를 줄임

DB 인덱스
→ 각 Query가 데이터를 더 빠르게 찾도록 도움
```

---

## 19. Query 복잡도 제한

GraphQL에서는 클라이언트가 관계를 깊게 요청할 수 있습니다.

```graphql
query {
  users {
    posts {
      comments {
        author {
          posts {
            comments {
              author {
                name
              }
            }
          }
        }
      }
    }
  }
}
```

DataLoader를 사용하더라도 이처럼 지나치게 깊고 많은 데이터를 요청하면 서버에 부담이 됩니다.

다음과 같은 제한을 함께 사용할 수 있습니다.

```text
Query 최대 깊이 제한

Query 복잡도 점수 제한

목록 최대 개수 제한

페이지네이션 필수 적용

요청 Timeout 설정

한 번에 조회할 수 있는 필드 제한
```

DataLoader는 N+1 문제를 줄여줄 뿐이므로, 지나치게 큰 GraphQL Query는 이런 제한으로 따로 막아야 합니다.

---

## 20. 관리자 회원 상세 페이지 예시

관리자 웹의 회원 상세 페이지에서 다음 데이터를 요청한다고 가정하겠습니다.

```graphql
query GetUserDetail($userId: ID!) {
  user(id: $userId) {
    id
    name
    email

    posts {
      id
      title

      comments {
        id
        content

        author {
          id
          name
        }
      }
    }

    vehicles {
      id
      manufacturer
      model
    }
  }
}
```

관계 구조는 다음과 같습니다.

```text
사용자
├─ 작성한 게시글
│   └─ 게시글별 댓글
│       └─ 댓글 작성자
│
└─ 소유한 차량
```

단순 Resolver를 사용하면 다음과 같은 조회가 반복될 수 있습니다.

```text
사용자 조회
→ 1번

사용자의 게시글 조회
→ 1번

게시글별 댓글 조회
→ 게시글 수만큼

댓글별 작성자 조회
→ 댓글 수만큼

사용자 차량 조회
→ 1번
```

관계별 DataLoader를 다음과 같이 구성할 수 있습니다.

```text
userLoader
→ 사용자 ID 여러 개 일괄 조회

postsByUserLoader
→ 사용자 ID 여러 개로 게시글 일괄 조회

commentsByPostLoader
→ 게시글 ID 여러 개로 댓글 일괄 조회

vehiclesByUserLoader
→ 사용자 ID 여러 개로 차량 일괄 조회
```

GraphQL Context에는 다음 Loader를 넣습니다.

```typescript
function createContext() {
  return {
    loaders: {
      user: createUserLoader(),
      postsByUser: createPostsByUserLoader(),
      commentsByPost: createCommentsByPostLoader(),
      vehiclesByUser: createVehiclesByUserLoader(),
    },
  };
}
```

각 Resolver는 자신에게 필요한 Loader를 사용합니다.

```typescript
const resolvers = {
  User: {
    posts: async (user, _, context) => {
      return context.loaders.postsByUser.load(
        user.id,
      );
    },

    vehicles: async (user, _, context) => {
      return context.loaders.vehiclesByUser.load(
        user.id,
      );
    },
  },

  Post: {
    comments: async (post, _, context) => {
      return context.loaders.commentsByPost.load(
        post.id,
      );
    },
  },

  Comment: {
    author: async (comment, _, context) => {
      return context.loaders.user.load(
        comment.authorId,
      );
    },
  },
};
```

다만 관리자 화면에서 게시글, 댓글, 차량 목록에 각각 페이지네이션이 필요하다면 하나의 깊은 Query로 모두 조회하기보다 목록 Query를 분리하는 것도 고려해야 합니다.

```graphql
query GetUser($userId: ID!) {
  user(id: $userId) {
    id
    name
    email
  }
}
```

```graphql
query GetUserPosts(
  $userId: ID!
  $page: Int!
  $size: Int!
) {
  userPosts(
    userId: $userId
    page: $page
    size: $size
  ) {
    items {
      id
      title
    }

    totalCount
  }
}
```

```graphql
query GetUserVehicles(
  $userId: ID!
  $page: Int!
  $size: Int!
) {
  userVehicles(
    userId: $userId
    page: $page
    size: $size
  ) {
    items {
      id
      manufacturer
      model
    }

    totalCount
  }
}
```

GraphQL이라고 해서 반드시 모든 관계를 하나의 Query에 깊게 포함해야 하는 것은 아닙니다.

데이터 크기, 페이지네이션, 캐시, 화면 구조를 고려하여 Query를 적절하게 분리해야 합니다.

---

## 21. 해결 방법 선택 기준

| 상황                       | 우선 고려할 방법             |
| ------------------------ | --------------------- |
| 관계 데이터가 항상 필요함           | JOIN 또는 ORM 관계 조회     |
| 관계 필드가 선택적으로 요청됨         | DataLoader            |
| 여러 Resolver가 같은 데이터를 조회함 | DataLoader            |
| 같은 ID가 요청 안에서 반복됨        | DataLoader 요청 단위 캐시   |
| 조회 구조가 단순하고 고정됨          | 직접 일괄 조회              |
| 목록 데이터가 많음               | 페이지네이션                |
| 관계 컬럼 조회가 느림             | DB 인덱스                |
| Query 관계가 지나치게 깊음        | Query 깊이·복잡도 제한       |
| DataLoader Batch가 너무 큼   | 페이지네이션 또는 Batch 크기 제한 |

실무에서는 보통 다음과 같이 여러 방법을 조합합니다.

```text
DataLoader
+ 페이지네이션
+ 필요한 관계의 JOIN
+ DB 인덱스
+ Query 복잡도 제한
+ SQL 모니터링
```

---

## 22. 흔히 하는 실수

### DataLoader를 서버 전체에서 공유

```typescript
const userLoader = createUserLoader();
```

이 Loader를 모든 요청에서 공유하면 다른 사용자의 캐시와 권한이 섞일 수 있습니다.

DataLoader는 요청마다 생성하는 것이 일반적입니다.

### DataLoader를 추가한 뒤 SQL을 확인하지 않음

Resolver에서 `load()`를 사용하더라도 Batch 함수 내부에서 다시 ID별 Query를 실행하면 N+1 문제가 해결되지 않습니다.

잘못된 예시는 다음과 같습니다.

```typescript
async function batchUsers(userIds: string[]) {
  return Promise.all(
    userIds.map((userId) => {
      return userRepository.findById(userId);
    }),
  );
}
```

이 코드는 DataLoader를 사용하지만 내부적으로 ID 개수만큼 Query를 실행합니다.

일괄 조회를 사용해야 합니다.

```typescript
async function batchUsers(userIds: string[]) {
  return userRepository.findByIds(userIds);
}
```

### 결과 순서를 맞추지 않음

DB가 반환한 배열을 그대로 반환하면 요청 ID와 데이터 위치가 일치하지 않을 수 있습니다.

반드시 요청된 키 순서대로 결과를 다시 구성해야 합니다.

### 페이지네이션 없이 모든 데이터를 조회

DataLoader가 있다고 해서 무제한 목록 조회가 안전해지는 것은 아닙니다.

목록에는 페이지네이션과 최대 개수 제한을 적용해야 합니다.

### 모든 관계를 무조건 JOIN

관계 데이터가 필요하지 않은 Query에서도 JOIN이 실행되면 불필요한 DB 작업이 발생할 수 있습니다.

항상 필요한 관계인지, 선택적으로 필요한 관계인지 구분해야 합니다.

---

## 23. 최종 정리

N+1 문제는 다음과 같은 구조에서 발생합니다.

```text
목록 조회
→ 1번

각 목록 항목의 관계 데이터 조회
→ N번

전체
→ N+1번
```

DataLoader는 개별 Resolver의 조회 요청을 모아 일괄 처리합니다.

```text
각 Resolver가 ID를 Loader에 전달
    ↓
DataLoader가 여러 ID를 수집
    ↓
WHERE IN으로 한 번에 조회
    ↓
결과를 ID별로 다시 배분
    ↓
같은 요청 안의 중복 데이터는 캐시
```

DataLoader를 적용하면 다음과 같이 줄일 수 있습니다.

```text
적용 전
→ 게시글 1번 + 작성자 N번

적용 후
→ 게시글 1번 + 작성자 일괄 조회 1번
```

하지만 DataLoader 하나만으로 모든 성능 문제가 해결되는 것은 아닙니다.

```text
DataLoader
→ 반복 조회 횟수 감소

페이지네이션
→ 한 번에 처리하는 데이터 양 제한

DB 인덱스
→ 개별 Query 조회 속도 개선

JOIN
→ 항상 필요한 관계 데이터를 한 번에 조회

Query 복잡도 제한
→ 지나치게 깊거나 큰 요청 방지

SQL 모니터링
→ 실제 최적화 여부 확인
```

N+1 문제를 의심할 기준은 다음과 같습니다.

> 목록 Resolver 내부에서 각 항목마다 `findById()` 또는 `findByParentId()`가 실행되고 있다면 N+1 문제를 의심해야 한다.

그리고 해결책을 적용한 뒤에는 반드시 실제 SQL 로그를 확인하여 반복 Query가 일괄 Query로 변경되었는지 검증해야 합니다.
