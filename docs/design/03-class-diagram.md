# 03-class-diagram.md - 클래스 다이어그램 (현재 코드 기준)

연결 문서:
- 개요: [README.md](../../README.md)
- 요구사항: [01-requirements.md](01-requirements.md)
- 시퀀스: [02-sequence-diagrams.md](02-sequence-diagrams.md)
- ERD: [04-erd.md](04-erd.md)

## 1. 레이어 구조

```mermaid
flowchart LR
    A[interfaces/api\nController + DTO] --> B[application\nFacade + EventHandler]
    B --> C[domain\nEntity/VO/Service/Repository Interface]
    C --> D[infrastructure\nRepository Impl/외부 연동/Outbox]
```

## 2. 핵심 도메인 모델

```mermaid
classDiagram
    class User {
        +Long id
        +String loginId
        +String email
        +LocalDate birth
        +Gender gender
    }

    class Point {
        +Long id
        +Long userId
        -PointAmount amount
        +charge(amount)
        +deduct(Money)
        +calculateDeduction(Money)
    }

    class Brand {
        +Long id
        +String name
        +String description
    }

    class Product {
        +Long id
        +String name
        -Money price
        -Stock stock
        +Long likeCount
        +Long brandId
        +decreaseStock(amount)
        +increaseLikeCount()
        +decreaseLikeCount()
    }

    class ProductLike {
        +Long id
        +Long userId
        +Long productId
        +LocalDateTime likedAt
    }

    class Order {
        +Long id
        +Long userId
        +OrderStatus status
        -Money totalAmount
        -Money pointUsedAmount
        -Money pgAmount
        +Long couponId
        -Money discountAmount
        +LocalDateTime orderedAt
        +addItem(OrderItem)
        +complete(completedAt)
        +failPayment()
        +retryComplete()
    }

    class OrderItem {
        +Long id
        +Long productId
        +String productName
        -Quantity quantity
        -OrderPrice orderPrice
        +assignOrder(Order)
    }

    class Coupon {
        +Long id
        +Long userId
        +CouponStatus status
        +Long usedOrderId
        +toUsed(orderId, usedAt)
        +toAvailable()
        +calculateDiscount(Money)
    }

    class CouponPolicy {
        +Long id
        +CouponDiscountType discountType
        -Money discountAmount
        -BigDecimal discountRate
        +calculateDiscount(Money)
    }

    class Payment {
        +Long id
        +Long orderId
        +Long userId
        +String transactionKey
        +PaymentStatus status
        -Money amount
        +CardType cardType
        +toPending(transactionKey)
        +toSuccess(completedAt)
        +toFailed(reason, completedAt)
        +toRequestFailed()
    }

    class OutboxEvent {
        +String eventId
        +String topic
        +String eventType
        +String aggregateId
        +OutboxStatus status
        +Integer retryCount
        +toSent()
        +toDead()
        +scheduleRetry(...)
    }

    class Money {
        <<ValueObject>>
        +Long value
        +add(Money)
        +subtract(Money)
    }

    class Stock {
        <<ValueObject>>
        +Long value
        +increase(amount)
        +decrease(amount)
    }

    class Quantity {
        <<ValueObject>>
        +Long value
    }

    class PointAmount {
        <<ValueObject>>
        +Long value
        +add(amount)
        +subtract(amount)
    }

    class OrderPrice {
        <<ValueObject>>
        +Long value
    }

    Point --> User : ref_user_id
    Product --> Brand : ref_brand_id
    ProductLike --> User : ref_user_id
    ProductLike --> Product : ref_product_id
    Order --> User : ref_user_id
    Order "1" *-- "N" OrderItem
    OrderItem --> Product : ref_product_id
    Coupon --> User : ref_user_id
    Coupon --> CouponPolicy : ref_coupon_policy_id
    Payment --> Order : ref_order_id
    Payment --> User : ref_user_id

    Product *-- Money
    Product *-- Stock
    Point *-- PointAmount
    Order *-- Money
    OrderItem *-- Quantity
    OrderItem *-- OrderPrice
```

## 3. 애플리케이션 오케스트레이션

```mermaid
classDiagram
    class ProductFacade
    class LikeFacade
    class OrderFacade
    class PaymentFacade
    class PaymentCallbackFacade
    class RankingFacade

    class ProductService
    class ProductLikeService
    class OrderService
    class OrderCreateService
    class OrderPaymentCalculator
    class PaymentService
    class PointService
    class CouponService
    class RankingService

    class LikeEventHandler
    class OrderEventHandler
    class PaymentEventHandler

    class OutboxEventWriter
    class OutboxRelay
    class OutboxEventUpdater

    ProductFacade --> ProductService
    ProductFacade --> ProductLikeService
    ProductFacade --> RankingService

    LikeFacade --> ProductService
    LikeFacade --> ProductLikeService
    LikeEventHandler --> ProductService

    OrderFacade --> ProductService
    OrderFacade --> OrderCreateService
    OrderFacade --> OrderPaymentCalculator
    OrderFacade --> OrderService

    PaymentFacade --> PaymentService
    PaymentCallbackFacade --> PaymentService
    PaymentCallbackFacade --> OrderService

    OrderEventHandler --> PointService
    OrderEventHandler --> CouponService
    OrderEventHandler --> ProductService
    OrderEventHandler --> OrderService

    PaymentEventHandler --> OrderService
    PaymentEventHandler --> ProductService
    PaymentEventHandler --> PointService
    PaymentEventHandler --> CouponService

    RankingFacade --> RankingService
    RankingFacade --> ProductService
    RankingFacade --> ProductLikeService

    OutboxEventWriter --> OutboxEventUpdater
    OutboxRelay --> OutboxEventUpdater
```

## 4. 상태 다이어그램

### 4.1 주문 상태 (`OrderStatus`)

```mermaid
stateDiagram-v2
    [*] --> PENDING: Order 생성
    PENDING --> COMPLETED: 결제 성공 + 재고 차감 성공
    PENDING --> PAYMENT_FAILED: 결제 실패 또는 후속 처리 실패
    PAYMENT_FAILED --> COMPLETED: retryComplete()
    COMPLETED --> [*]
    PAYMENT_FAILED --> [*]
```

### 4.2 결제 상태 (`PaymentStatus`)

```mermaid
stateDiagram-v2
    [*] --> REQUESTED: Payment 생성
    REQUESTED --> PENDING: PG 요청 성공(transactionKey 수신)
    REQUESTED --> REQUEST_FAILED: PG 요청 자체 실패
    PENDING --> SUCCESS: 콜백/복구에서 성공 확인
    PENDING --> FAILED: 콜백/복구에서 실패 확인
    SUCCESS --> [*]
    FAILED --> [*]
    REQUEST_FAILED --> [*]
```

## 5. 설계 포인트
- 주문은 생성 시 항상 `PENDING`으로 저장되고, 완료/실패는 이벤트 핸들러가 후속 반영한다.
- 좋아요 수(`product.like_count`)는 이벤트 기반 비동기 반영이므로 짧은 지연이 발생할 수 있다.
- `OrderItem`은 `Order`와 양방향 객체 참조(`@OneToMany`/`@ManyToOne`)로 Aggregate 일관성을 유지한다.
- 외부 전송은 Outbox 패턴으로 분리되어 트랜잭션 커밋과 메시지 발행 신뢰성을 동시에 확보한다.
