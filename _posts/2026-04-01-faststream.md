---
title: "FastStream - 이벤트 기반 마이크로서비스 프레임워크"
date: 2026-04-01 17:00:00 +0900
categories: [Tech Exploration]
tags: [faststream, kafka, python, event-driven, microservice]
---

## FastStream이란?

FastStream은 Python으로 이벤트 기반 마이크로서비스를 구축하기 위한 프레임워크다. Kafka, RabbitMQ, NATS, Redis 등 다양한 메시지 브로커를 **통일된 API**로 다룰 수 있다.

FastAPI가 HTTP 요청을 데코레이터로 처리하듯, FastStream은 메시지 큐의 이벤트를 데코레이터로 처리한다.

## 왜 FastStream인가?

메시지 브로커를 직접 다루려면 consumer/producer 설정, 직렬화/역직렬화, 에러 핸들링, 재시도 로직 등을 모두 직접 구현해야 한다. FastStream은 이걸 프레임워크 수준에서 해결한다.

| 항목 | 직접 구현 (aiokafka 등) | FastStream |
|---|---|---|
| Consumer/Producer 설정 | 수동 | 데코레이터 선언 |
| 메시지 검증 | 직접 구현 | Pydantic 자동 검증 |
| 의존성 주입 | 없음 | `Depends` 지원 |
| 테스트 | 브로커 필요 | In-memory 테스트 |
| API 문서 | 없음 | AsyncAPI 자동 생성 |
| 브로커 교체 | 전면 재작성 | import 경로만 변경 |

## 아키텍처

```
┌─────────────────────────────────────────────┐
│ FastStream Application                      │
│                                             │
│   ┌─────────┐    ┌──────────────────────┐   │
│   │   App   │───▶│   Lifecycle Hooks    │   │
│   │         │    │  (startup/shutdown)  │   │
│   └────┬────┘    └──────────────────────┘   │
│        │                                    │
│   ┌────▼────┐    ┌──────────────────────┐   │
│   │ Broker  │───▶│    Middlewares       │   │
│   │ (Kafka) │    └──────────────────────┘   │
│   └────┬────┘                               │
│        │                                    │
│   ┌────▼──────────────────────────────┐     │
│   │         Message Router            │     │
│   │                                   │     │
│   │  ┌────────────┐  ┌────────────┐   │     │
│   │  │ Subscriber │  │ Subscriber │   │     │
│   │  │ (topic-a)  │  │ (topic-b)  │   │     │
│   │  └─────┬──────┘  └─────┬──────┘   │     │
│   │        │               │          │     │
│   │   ┌────▼────┐    ┌─────▼─────┐    │     │
│   │   │Pydantic │    │  Depends  │    │     │
│   │   │Validate │    │ Injection │    │     │
│   │   └────┬────┘    └─────┬─────┘    │     │
│   │        │               │          │     │
│   │   ┌────▼───────────────▼─────┐    │     │
│   │   │     Handler Function     │    │     │
│   │   └────────────┬─────────────┘    │     │
│   │                │                  │     │
│   │   ┌────────────▼─────────────┐    │     │
│   │   │  Publisher (out-topic)   │    │     │
│   │   └──────────────────────────┘    │     │
│   └───────────────────────────────────┘     │
└─────────────────────────────────────────────┘
```

### 핵심 구성 요소

| 컴포넌트 | 역할 |
|---|---|
| **App** | 애플리케이션 진입점, 라이프사이클 관리 |
| **Broker** | 메시지 브로커 연결 및 관리 |
| **Subscriber** | 토픽에서 메시지를 수신하는 핸들러 |
| **Publisher** | 처리 결과를 다른 토픽으로 발행 |
| **Depends** | FastAPI 스타일 의존성 주입 |
| **Middleware** | 메시지 처리 전후 공통 로직 |

## 기본 사용법

### 앱 + 브로커 설정

```python
from faststream import FastStream
from faststream.kafka import KafkaBroker

broker = KafkaBroker("localhost:9092")
app = FastStream(broker)
```

Kafka 외 다른 브로커를 사용하려면 import만 바꾸면 된다.

```python
# RabbitMQ
from faststream.rabbit import RabbitBroker
broker = RabbitBroker("amqp://guest:guest@localhost:5672/")

# NATS
from faststream.nats import NatsBroker
broker = NatsBroker("nats://localhost:4222")

# Redis
from faststream.redis import RedisBroker
broker = RedisBroker("redis://localhost:6379")
```

### Subscriber + Publisher

```python
from pydantic import BaseModel


class OrderEvent(BaseModel):
    order_id: str
    user_id: int
    amount: float


class NotificationEvent(BaseModel):
    user_id: int
    message: str


@broker.subscriber("orders")
@broker.publisher("notifications")
async def handle_order(event: OrderEvent) -> NotificationEvent:
    # Pydantic이 메시지를 자동으로 검증/파싱한다
    return NotificationEvent(
        user_id=event.user_id,
        message=f"주문 {event.order_id} 완료 (₩{event.amount:,.0f})"
    )
```

`@broker.subscriber`로 토픽을 구독하고, `@broker.publisher`로 처리 결과를 다른 토픽에 발행한다. 함수의 반환값이 자동으로 publisher 토픽에 전송된다.

### 메시지 직접 발행

```python
@app.after_startup
async def publish_on_start():
    await broker.publish(
        OrderEvent(order_id="ORD-001", user_id=42, amount=15000),
        "orders"
    )
```

## Dependency Injection

FastAPI의 `Depends`와 동일한 패턴이다. 핸들러에 공통 로직을 주입할 수 있다.

```python
from typing import Annotated
from faststream import Depends, Logger


async def get_db_session():
    session = await create_session()
    try:
        yield session
    finally:
        await session.close()


async def verify_user(user_id: int) -> bool:
    # 사용자 검증 로직
    return user_id > 0


@broker.subscriber("orders")
async def handle_order(
    event: OrderEvent,
    logger: Logger,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    is_valid: Annotated[bool, Depends(verify_user)],
):
    if not is_valid:
        logger.warning(f"Invalid user: {event.user_id}")
        return

    logger.info(f"Processing order: {event.order_id}")
    # session으로 DB 작업 수행
```

의존성 체이닝도 가능하다. 의존성 안에서 다른 의존성을 주입받을 수 있다.

```python
async def get_config() -> dict:
    return {"retry_count": 3}


async def get_service(config: dict = Depends(get_config)) -> MyService:
    return MyService(**config)


@broker.subscriber("events")
async def handler(msg: str, svc: MyService = Depends(get_service)):
    await svc.process(msg)
```

## Batch Consumer

대량 메시지를 한 번에 처리할 때는 `batch=True`를 사용한다.

```python
from pydantic import BaseModel


class MetricEvent(BaseModel):
    name: str
    value: float


@broker.subscriber("metrics", batch=True)
async def handle_metrics(events: list[MetricEvent]):
    # 여러 메시지를 한 번에 처리
    total = sum(e.value for e in events)
    print(f"Batch size: {len(events)}, Total: {total}")
```

## Middleware

메시지 처리 전후에 공통 로직을 끼울 수 있다.

```python
from faststream import BaseMiddleware


class LoggingMiddleware(BaseMiddleware):
    async def on_receive(self):
        print(f"Received: {self.msg}")
        return await super().on_receive()

    async def after_processed(self, exc_type=None, exc_val=None, exc_tb=None):
        if exc_type:
            print(f"Error: {exc_val}")
        else:
            print("Processed successfully")
        return await super().after_processed(exc_type, exc_val, exc_tb)


# 브로커 전체에 적용
broker = KafkaBroker("localhost:9092", middlewares=[LoggingMiddleware])

# 또는 특정 핸들러에만 적용
@broker.subscriber("orders", middlewares=[LoggingMiddleware])
async def handle_order(event: OrderEvent):
    ...
```

## Lifecycle Hooks

애플리케이션 시작/종료 시점에 초기화/정리 로직을 실행한다.

```python
@app.on_startup
async def on_startup():
    print("Initializing resources...")
    # DB 커넥션 풀, 캐시 등 초기화

@app.on_shutdown
async def on_shutdown():
    print("Cleaning up resources...")
    # 리소스 정리

@app.after_startup
async def after_startup():
    # 브로커 연결 완료 후 실행
    await broker.publish("Service started", "system-events")
```

## 실전 예시: 주문 처리 파이프라인

```
orders ──▶ [validate] ──▶ validated-orders ──▶ [process] ──▶ notifications
                │                                                │
                ▼                                                ▼
          dead-letter                                     Kafka Topic
```

```python
from faststream import FastStream, Depends, Logger
from faststream.kafka import KafkaBroker
from pydantic import BaseModel, field_validator


broker = KafkaBroker("localhost:9092")
app = FastStream(broker)


# --- Models ---

class OrderEvent(BaseModel):
    order_id: str
    user_id: int
    items: list[str]
    amount: float

    @field_validator("amount")
    @classmethod
    def amount_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError("amount must be positive")
        return v


class ValidatedOrder(BaseModel):
    order_id: str
    user_id: int
    amount: float
    discount: float = 0.0


class Notification(BaseModel):
    user_id: int
    title: str
    body: str


# --- Dependencies ---

async def calculate_discount(event: OrderEvent) -> float:
    if event.amount >= 50000:
        return event.amount * 0.1
    return 0.0


# --- Handlers ---

@broker.subscriber("orders")
@broker.publisher("validated-orders")
async def validate_order(
    event: OrderEvent,
    logger: Logger,
    discount: float = Depends(calculate_discount),
) -> ValidatedOrder:
    logger.info(f"Validating order {event.order_id}")
    return ValidatedOrder(
        order_id=event.order_id,
        user_id=event.user_id,
        amount=event.amount,
        discount=discount,
    )


@broker.subscriber("validated-orders")
@broker.publisher("notifications")
async def process_order(
    order: ValidatedOrder,
    logger: Logger,
) -> Notification:
    final_amount = order.amount - order.discount
    logger.info(f"Order {order.order_id} processed: ₩{final_amount:,.0f}")
    return Notification(
        user_id=order.user_id,
        title="주문 완료",
        body=f"주문 {order.order_id} - ₩{final_amount:,.0f}",
    )


@broker.subscriber("notifications")
async def send_notification(event: Notification, logger: Logger):
    logger.info(f"Sending to user {event.user_id}: {event.title}")
    # 실제 알림 전송 로직
```

## In-Memory 테스트

실제 Kafka 없이 핸들러를 테스트할 수 있다. `TestBroker`가 메시지를 메모리에서 라우팅한다.

```python
import pytest
from faststream.kafka import TestKafkaBroker


@pytest.mark.asyncio
async def test_validate_order():
    async with TestKafkaBroker(broker) as br:
        await br.publish(
            OrderEvent(
                order_id="ORD-001",
                user_id=1,
                items=["item-a"],
                amount=60000,
            ),
            "orders",
        )

        # validate_order 핸들러가 호출됐는지 확인
        validate_order.mock.assert_called_once()

        # publisher를 통해 validated-orders에 메시지가 발행됐는지 확인
        process_order.mock.assert_called_once()
```

## FastAPI 통합

FastStream은 FastAPI와 함께 사용할 수 있다. HTTP와 이벤트를 하나의 서비스에서 처리한다.

```python
from fastapi import FastAPI
from faststream.kafka.fastapi import KafkaRouter

router = KafkaRouter("localhost:9092")

@router.subscriber("orders")
@router.publisher("notifications")
async def handle_order(event: OrderEvent) -> Notification:
    return Notification(user_id=event.user_id, title="완료", body="처리됨")

app = FastAPI()
app.include_router(router)

@app.get("/publish")
async def publish_order():
    await router.broker.publish(
        OrderEvent(order_id="ORD-001", user_id=1, items=["a"], amount=10000),
        "orders",
    )
    return {"status": "published"}
```

## 실행

```bash
faststream run main:app
```

`--reload` 옵션으로 개발 중 자동 리로드도 지원한다.

```bash
faststream run main:app --reload
```

## 정리

| 항목 | 설명 |
|---|---|
| **통일 API** | Kafka, RabbitMQ, NATS, Redis를 동일한 인터페이스로 |
| **Pydantic 검증** | 메시지 스키마를 타입으로 선언, 자동 검증 |
| **DI** | FastAPI와 동일한 `Depends` 패턴 |
| **테스트** | 브로커 없이 In-Memory 테스트 |
| **문서화** | AsyncAPI 스펙 자동 생성 |
| **FastAPI 통합** | HTTP + Event를 한 서비스에서 처리 |

## References

- [FastStream Documentation](https://faststream.airt.ai/latest/)
- [FastStream - Getting Started](https://faststream.airt.ai/latest/getting-started/)
- [FastStream - Kafka Broker](https://faststream.airt.ai/latest/kafka/)
- [FastStream - Dependency Injection](https://faststream.airt.ai/latest/getting-started/dependencies/)
- [FastStream - Middleware](https://faststream.airt.ai/latest/getting-started/middlewares/)
- [FastStream GitHub Repository](https://github.com/ag2ai/faststream)
- [AsyncAPI Specification](https://www.asyncapi.com/)
