---
aliases: [Webhook, 웹훅]
tags: [api, http]
prerequisites:
  - "[[http]]"
related:
  - "[[polling]]"
status: draft
created: 2026-10-01
---

# 웹훅

## 1. 웹훅이란?

**웹훅(Webhook)**은 어떤 일이 일어났을 때 서버가 미리 등록된 URL로 [[http|HTTP]] 요청을 보내 알려 주는 방식입니다.

보통은 내가 서버에 요청을 보내고 응답을 받는데, 웹훅에서는 반대로 상대 서비스가 내 서버로 요청을 보냅니다.

```text
일반 API 호출
내 서버 ── 결제 상태 알려 주세요 ──> PG사

웹훅
내 서버 <── 입금이 완료되었습니다 ── PG사
```

택배로 비유하면, 배송 조회 페이지를 계속 확인하는 대신 "배송 완료되면 문자 주세요"라고 번호를 남겨 두는 것입니다.

## 2. 동작 흐름

```text
1. 등록   내 서버 → 서비스: "이벤트가 생기면 https://my.app/webhooks/pay 로 알려 주세요"
2. 발생   서비스 안에서 이벤트 발생 (입금 완료, 코드 push 등)
3. 전송   서비스 → 내 서버: POST https://my.app/webhooks/pay  { 이벤트 내용 }
4. 응답   내 서버 → 서비스: 200 OK
```

등록은 보통 서비스의 관리자 화면이나 API로 합니다. 이벤트 내용은 대부분 JSON으로 옵니다.

```http
POST /webhooks/pay HTTP/1.1
Content-Type: application/json
X-Signature: 3f9a...

{
  "id": "evt_123",
  "type": "payment.deposit_completed",
  "createdAt": "2026-10-01T10:00:00Z",
  "data": { "orderId": "order_456", "amount": 30000 }
}
```

## 3. 어디에 쓰이나?

| 서비스 | 이벤트 예시 |
|---|---|
| PG사 (결제) | 가상계좌 입금 완료, 결제 취소 |
| GitHub | push, Pull Request 생성 → CI 실행, 배포 |
| Slack | 외부 시스템이 채널에 메시지 보내기 (Incoming Webhook) |
| 배송·메시지 서비스 | 배송 상태 변경, 문자 발송 결과 |

공통점은 언제 일어날지 모르는 일이라는 것입니다. 가상계좌 입금은 몇 분 뒤일 수도, 며칠 뒤일 수도 있습니다. 이런 일을 기다리며 계속 물어보는 대신, 일어난 순간 알려 받습니다.

## 4. 받는 쪽 구현하기

웹훅 수신 엔드포인트는 외부에 공개된 URL이라 누구나 요청을 보낼 수 있습니다. 그래서 받는 쪽에서 지켜야 할 것이 많습니다.

### 4-1. 서명 검증

요청이 정말 그 서비스에서 왔는지 확인합니다. 대부분의 서비스는 공유한 비밀 키로 요청 본문의 HMAC 서명을 만들어 헤더에 넣어 보냅니다.

```text
서비스: signature = HMAC-SHA256(비밀 키, 요청 본문)  → 헤더에 담아 전송
내 서버: 같은 방법으로 계산한 값과 헤더 값이 같은지 비교
```

```js
import crypto from "node:crypto";
import express from "express";

const app = express();

// 서명은 원본 바이트 기준이므로 JSON 파싱 전 raw body를 받는다
app.post("/webhooks/pay", express.raw({ type: "application/json" }), (req, res) => {
  const expected = crypto
    .createHmac("sha256", process.env.WEBHOOK_SECRET)
    .update(req.body)
    .digest("hex");
  const received = req.get("X-Signature") ?? "";

  const ok =
    received.length === expected.length &&
    crypto.timingSafeEqual(Buffer.from(received), Buffer.from(expected));
  if (!ok) return res.status(401).end();

  const event = JSON.parse(req.body);
  // ... 처리
  res.status(200).end();
});
```

- 본문을 파싱한 뒤 다시 문자열로 만들면 공백이나 키 순서가 바뀌어 서명이 맞지 않습니다. 받은 그대로의 본문으로 계산합니다.
- 문자열 비교에 `===` 대신 `timingSafeEqual`을 써서 비교 시간으로 값을 추측하는 공격을 막습니다.
- 헤더 이름과 서명 방식은 서비스마다 다르므로 해당 서비스 문서를 따릅니다.

서명 기능이 없는 서비스라면, 웹훅 내용을 그대로 믿지 말고 서비스 API로 상태를 다시 조회해서 확인합니다.

### 4-2. 빠르게 응답하기

보내는 쪽은 보통 몇 초 안에 2xx 응답이 오지 않으면 실패로 보고 다시 보냅니다. 오래 걸리는 처리는 응답 뒤로 미룹니다.

```text
요청 수신 → 서명 검증 → 이벤트 저장(또는 큐에 넣기) → 200 응답
                                     ↓
                         백그라운드 작업이 실제 처리
```

### 4-3. 중복 처리 막기 (멱등성)

웹훅은 같은 이벤트가 여러 번 올 수 있습니다. 내 서버가 처리는 했는데 응답이 네트워크에서 유실되면, 보내는 쪽은 실패로 알고 다시 보냅니다.

```text
입금 완료 웹훅 수신 → 포인트 적립 → 200 응답 (유실)
입금 완료 웹훅 재전송 → 포인트 또 적립 ✗
```

이벤트 ID를 저장해 두고, 이미 처리한 ID면 건너뜁니다.

```sql
CREATE TABLE processed_webhook_events (
  event_id TEXT PRIMARY KEY,
  processed_at TIMESTAMP NOT NULL
);
```

```text
event_id INSERT 성공  → 처리
event_id 이미 있음    → 처리하지 않고 200 응답
```

### 4-4. 순서를 믿지 않기

재시도 때문에 이벤트가 보낸 순서대로 도착한다는 보장이 없습니다. "결제 취소"가 "결제 완료"보다 먼저 올 수도 있습니다.

- 이벤트의 생성 시각이나 버전을 보고 오래된 이벤트는 무시합니다.
- 또는 웹훅을 "변경이 있었다"는 신호로만 쓰고, 최신 상태는 서비스 API로 조회합니다.

## 5. 보내는 쪽의 재시도

웹훅을 보내는 서비스는 실패하면 간격을 늘려 가며 다시 보냅니다(예: 1분, 5분, 30분, 2시간...). 일정 횟수가 넘으면 포기하고, 웹훅을 비활성화하는 서비스도 있습니다.

내 서버가 오래 멈춰 있었다면 그 사이 웹훅을 놓쳤을 수 있습니다. 그래서 중요한 데이터는 정기적으로 서비스 API와 대조하는 작업을 두어 빠진 것을 채웁니다.

```text
평소:      웹훅으로 즉시 반영
하루 한 번: 서비스 API로 최근 거래 조회 → 내 DB와 비교 → 빠진 건 보정
```

## 6. 로컬에서 개발하기

웹훅은 외부 서비스가 내 서버로 요청을 보내야 하므로, `localhost`로는 받을 수 없습니다.

- 터널링 도구: ngrok 같은 도구로 공개 URL을 받아 로컬 서버로 연결합니다.
- 서비스 제공 CLI: 일부 서비스는 웹훅을 로컬로 전달해 주는 CLI를 제공합니다.
- 테스트 전송: 대부분의 관리자 화면에 테스트 이벤트를 보내는 버튼이 있습니다.

```text
PG사 → https://abcd.ngrok.app/webhooks/pay → ngrok → localhost:3000/webhooks/pay
```

## 7. 장단점

**장점**
- 일이 일어난 즉시 알 수 있습니다.
- 변화가 없을 때는 요청이 오가지 않아 낭비가 없습니다.
- 언제 일어날지 모르는 이벤트(입금, 배송 완료)를 기다리기에 적합합니다.

**단점**
- 내 서버가 외부에서 접근 가능한 공개 URL을 가져야 합니다. 브라우저나 모바일 앱은 직접 받을 수 없습니다.
- 서명 검증, 중복 처리, 순서 문제, 누락 보정까지 받는 쪽이 챙겨야 합니다.
- 내 서버가 멈춰 있으면 놓칠 수 있고, 디버깅이 어렵습니다.

## 8. 정리

- 웹훅은 이벤트가 생기면 서비스가 내 서버의 URL로 HTTP 요청을 보내 주는 방식이다.
- 서버와 서버 사이의 알림이다. 사용자 브라우저로 직접 오지 않는다.
- 받는 쪽은 서명 검증, 빠른 응답, 중복 제거, 순서 무시, 정기 대조를 챙긴다.
- 웹훅만 믿지 말고, 중요한 상태는 서비스 API로 다시 확인한다.
