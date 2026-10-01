---
aliases: [SSE, Server-Sent Events]
tags: [network, realtime]
prerequisites:
  - "[[http]]"
related: []
status: draft
created: 2026-10-01
---

# SSE

## 1. SSE란?

**SSE(Server-Sent Events)**는 서버가 하나의 [[http|HTTP]] 연결을 유지하면서 클라이언트로 이벤트를 계속 보내는 방식입니다.

```text
클라이언트 ── 연결 요청 ──> 서버
클라이언트 <─ 이벤트 1 ─── 서버
클라이언트 <─ 이벤트 2 ─── 서버
클라이언트 <─ 이벤트 3 ─── 서버
```

통신 방향은 주로 서버에서 클라이언트로 한 방향입니다. 클라이언트가 데이터를 보내야 하면 일반 HTTP 요청을 함께 사용합니다.

## 2. 데이터 형식

서버는 `text/event-stream` 형식으로 응답합니다.

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

event: message
id: 101
data: {"text":"안녕하세요"}

```

각 이벤트는 빈 줄로 구분됩니다.

- `event`: 이벤트 종류
- `id`: 이벤트 식별자
- `data`: 실제 데이터
- `retry`: 재연결 대기 시간 힌트

## 3. 브라우저에서 사용하기

```javascript
const source = new EventSource('/api/events');

source.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(data);
};

source.onerror = () => {
  console.log('연결 오류');
};
```

브라우저의 `EventSource`는 연결이 끊기면 자동 재연결을 지원합니다. 마지막 이벤트 ID를 활용하면 서버가 누락된 다음 이벤트부터 다시 보낼 수 있습니다.

POST 요청에 대한 응답 본문을 스트리밍하는 AI API는 `fetch()`와 ReadableStream을 쓰기도 합니다. 이를 넓게 SSE 스타일 스트리밍이라 부르는 경우가 있지만, 브라우저 `EventSource` 사용 방식과는 구분해서 보는 것이 정확합니다.

## 4. 어디에 적합한가?

- AI 답변 토큰 스트리밍
- 서버 로그 실시간 표시
- 작업 진행률
- 뉴스·알림 피드
- 주기적으로 변하는 모니터링 값
- 빌드·배포 상태 변경

클라이언트가 보낼 데이터는 REST API로 처리하고, 진행 상황만 SSE로 받을 수 있습니다.

```text
POST /jobs       → 작업 생성
GET /jobs/1/events → 진행 상태 SSE 수신
```

## 5. 연결과 복구

SSE도 지속 연결이므로 프록시와 로드밸런서의 Timeout, 서버의 버퍼링, 브라우저 연결 제한을 확인해야 합니다. 중간 장치가 응답을 모아서 한 번에 전달하면 실시간처럼 보이지 않을 수 있어 버퍼링 비활성화나 주기적인 Heartbeat가 필요할 수 있습니다.

이벤트 ID와 재연결 정책을 이용하면 연결이 잠시 끊겨도 이어받는 구조를 만들 수 있습니다.

## 6. 인증

같은 출처의 쿠키 인증은 비교적 자연스럽습니다. 하지만 기본 `EventSource` API에서는 임의의 Authorization 헤더 설정이 제한적이므로 쿠키 사용, URL에 짧은 수명의 연결 토큰 사용, 또는 `fetch` 스트리밍 같은 대안을 검토할 수 있습니다. 민감한 장기 토큰을 URL에 그대로 넣으면 로그나 기록에 남을 수 있으므로 피해야 합니다.

## 7. 장점과 한계

### 장점

- 일반 HTTP 기반이라 구조가 단순합니다.
- 브라우저에서 자동 재연결을 지원합니다.
- 텍스트 이벤트를 순차적으로 전달하기 좋습니다.
- 서버 → 클라이언트만 필요한 기능에서는 WebSocket보다 구현 부담이 작습니다.

### 한계

- 기본적으로 단방향입니다.
- 바이너리 데이터에는 적합하지 않습니다.
- 브라우저·프록시의 연결 정책 영향을 받습니다.
- 양방향 이벤트가 빈번하면 REST 요청과 SSE를 함께 관리해야 합니다.

## 8. 핵심 정리

> SSE는 HTTP 연결을 유지하며 서버가 클라이언트로 텍스트 이벤트를 연속해서 보내는 단방향 스트리밍 방식이다.
