# 02-sequence-diagrams.md - 시퀀스 다이어그램 (현재 코드 기준)

연결 문서:
- 개요: [README.md](../../README.md)
- 요구사항: [01-requirements.md](01-requirements.md)
- 클래스: [03-class-diagram.md](03-class-diagram.md)
- ERD: [04-erd.md](04-erd.md)

## 목차
- [1. 상품 목록 조회](#1-상품-목록-조회)
- [2. 상품 상세 조회 + 조회 이벤트 기록](#2-상품-상세-조회--조회-이벤트-기록)
- [3. 좋아요 등록 (멱등)](#3-좋아요-등록-멱등)
- [4. 좋아요 취소 (멱등)](#4-좋아요-취소-멱등)
- [5. 주문 생성](#5-주문-생성)
- [6. 결제 콜백 처리](#6-결제-콜백-처리)
- [7. 결제 복구 스케줄러](#7-결제-복구-스케줄러)
- [8. Outbox 발행/Relay](#8-outbox-발행relay)
- [9. 일간 랭킹 조회](#9-일간-랭킹-조회)
- [10. 주간/월간 랭킹 조회](#10-주간월간-랭킹-조회)

## 1. 상품 목록 조회

```mermaid
sequenceDiagram
    participant Client
    participant ProductController
    participant ProductFacade
    participant CacheTemplate
    participant ProductService
    participant BrandService
    participant ProductLikeService

    Client->>+ProductController: GET /api/v1/products
    ProductController->>+ProductFacade: searchProducts(brandId, condition, pageable)

    alt 캐시 가능 조건 (비회원 + page0 + size20 + latest/likes_desc)
        ProductFacade->>+CacheTemplate: getOrLoad(cacheKey)
        CacheTemplate->>+ProductService: findProducts(brandId, pageable)
        ProductService-->>-CacheTemplate: Page<Product>
        CacheTemplate-->>-ProductFacade: cached/non-cached ProductList
    else 일반 조회
        ProductFacade->>+ProductService: findProducts(brandId, pageable)
        ProductService-->>-ProductFacade: Page<Product>
    end

    ProductFacade->>+BrandService: findByIdIn(brandIds)
    BrandService-->>-ProductFacade: List<Brand>

    ProductFacade->>+ProductLikeService: findLikeStatusByProductId(userId, productIds)
    ProductLikeService-->>-ProductFacade: Map<productId, liked>

    ProductFacade-->>-ProductController: Page<ProductDetail>
    ProductController-->>-Client: 200 OK (products + pageInfo)
```

## 2. 상품 상세 조회 + 조회 이벤트 기록

```mermaid
sequenceDiagram
    participant Client
    participant ProductController
    participant ProductFacade
    participant ProductService
    participant BrandService
    participant ProductLikeService
    participant RankingService
    participant EventPublisher as DomainEventPublisher
    participant OutboxWriter as OutboxEventWriter
    participant OutboxRepo as OutboxEventRepository

    Client->>+ProductController: GET /api/v1/products/{productId}
    ProductController->>+ProductFacade: retrieveProductDetail(productId, userId)

    ProductFacade->>+ProductService: getById(productId)
    ProductService-->>-ProductFacade: Product or empty
    ProductFacade->>+BrandService: getById(brandId)
    BrandService-->>-ProductFacade: Brand
    ProductFacade->>+ProductLikeService: isLiked(userId, productId)
    ProductLikeService-->>-ProductFacade: boolean
    ProductFacade->>+RankingService: getRankOrNull(today, productId)
    RankingService-->>-ProductFacade: rank or null

    ProductFacade->>+EventPublisher: publish(ProductViewedEvent)
    Note over EventPublisher,OutboxWriter: BEFORE_COMMIT
    OutboxWriter->>+OutboxRepo: save(outbox_event: NEW)
    OutboxRepo-->>-OutboxWriter: saved
    EventPublisher-->>-ProductFacade: published

    ProductFacade-->>-ProductController: ProductDetail
    ProductController-->>-Client: 200 OK
```

## 3. 좋아요 등록 (멱등)

```mermaid
sequenceDiagram
    participant Client
    participant LikeController
    participant LikeFacade
    participant ProductService
    participant ProductLikeService
    participant LikeRepo as ProductLikeRepository
    participant EventPublisher as DomainEventPublisher
    participant LikeEventHandler

    Client->>+LikeController: POST /api/v1/like/products/{productId}
    LikeController->>+LikeFacade: registerProductLike(userId, productId)

    LikeFacade->>+ProductService: getById(productId)
    ProductService-->>-LikeFacade: Product or empty

    LikeFacade->>+ProductLikeService: createLike(userId, productId)
    ProductLikeService->>+LikeRepo: saveIfNotExists(INSERT IGNORE)
    LikeRepo-->>-ProductLikeService: saved=true/false

    alt 신규 좋아요(saved=true)
        ProductLikeService->>+EventPublisher: publish(ProductLikedEvent)
        EventPublisher-->>-ProductLikeService: published
        Note over LikeEventHandler: AFTER_COMMIT + @Async
        LikeEventHandler->>ProductService: increaseLikeCount(productId)
    else 중복 좋아요(saved=false)
        Note over ProductLikeService: 이벤트 발행 없음 (멱등)
    end

    ProductLikeService-->>-LikeFacade: return
    LikeFacade-->>-LikeController: return
    LikeController-->>-Client: 200 OK
```

## 4. 좋아요 취소 (멱등)

```mermaid
sequenceDiagram
    participant Client
    participant LikeController
    participant LikeFacade
    participant ProductService
    participant ProductLikeService
    participant LikeRepo as ProductLikeRepository
    participant EventPublisher as DomainEventPublisher
    participant LikeEventHandler

    Client->>+LikeController: DELETE /api/v1/like/products/{productId}
    LikeController->>+LikeFacade: cancelProductLike(userId, productId)

    LikeFacade->>+ProductService: getById(productId)
    ProductService-->>-LikeFacade: Product or empty

    LikeFacade->>+ProductLikeService: deleteLike(userId, productId)
    ProductLikeService->>+LikeRepo: deleteByUserIdAndProductId
    LikeRepo-->>-ProductLikeService: deletedCount

    alt deletedCount > 0
        ProductLikeService->>+EventPublisher: publish(ProductUnlikedEvent)
        EventPublisher-->>-ProductLikeService: published
        Note over LikeEventHandler: AFTER_COMMIT + @Async
        LikeEventHandler->>ProductService: decreaseLikeCount(productId)
    else 이미 취소된 상태
        Note over ProductLikeService: 이벤트 발행 없음 (멱등)
    end

    ProductLikeService-->>-LikeFacade: return
    LikeFacade-->>-LikeController: return
    LikeController-->>-Client: 200 OK
```

## 5. 주문 생성

```mermaid
sequenceDiagram
    participant Client
    participant OrderController
    participant OrderFacade
    participant ProductService
    participant OrderCreateService
    participant PaymentCalc as OrderPaymentCalculator
    participant OrderService
    participant PaymentFacade
    participant PaymentService
    participant PgClient
    participant OrderEventHandler
    participant PointService
    participant CouponService

    Client->>+OrderController: POST /api/v1/orders
    OrderController->>+OrderFacade: createOrder(userId, items, couponId)

    OrderFacade->>+ProductService: findByIdsWithLock(productIds)
    ProductService-->>-OrderFacade: locked products
    OrderFacade->>+OrderCreateService: prepareOrder(items, productById)
    OrderCreateService-->>-OrderFacade: OrderPreparation
    OrderFacade->>+PaymentCalc: calculate(userId, couponId, totalAmount)
    PaymentCalc-->>-OrderFacade: discount/point/pg 금액

    OrderFacade->>+OrderService: create(OrderCreateCommand)
    Note over OrderService: status=PENDING 저장 + OrderCreatedEvent 발행
    OrderService-->>-OrderFacade: Order(PENDING)

    OrderFacade-->>-OrderController: Order
    OrderController->>+PaymentFacade: processPayment(order, paymentInfo)

    alt pgAmount == 0 (포인트 전액)
        PaymentFacade-->>OrderController: PG 요청 스킵
    else pgAmount > 0
        PaymentFacade->>+PaymentService: create(REQUESTED)
        PaymentService-->>-PaymentFacade: Payment
        PaymentFacade->>+PgClient: requestPayment
        PgClient-->>-PaymentFacade: PgPaymentResponse

        alt PG PENDING 응답
            PaymentFacade->>+PaymentService: toPending(paymentId, transactionKey)
            PaymentService-->>-PaymentFacade: updated
        else PG 요청 실패
            PaymentFacade->>+PaymentService: toRequestFailed(paymentId)
            PaymentService-->>-PaymentFacade: updated
            PaymentFacade-->>OrderController: exception
        end
    end

    OrderController-->>-Client: 201 Created (주문은 우선 PENDING)

    Note over OrderEventHandler: AFTER_COMMIT + @Async
    par 포인트 차감
        OrderEventHandler->>PointService: deduct(userId, pointAmount)
    and 쿠폰 사용
        OrderEventHandler->>CouponService: useCoupon(couponId)
    and 포인트 전액 결제 주문 완료 처리(pgAmount==0)
        OrderEventHandler->>ProductService: tryDecreaseStocks(orderItems)
        alt 재고 충분
            OrderEventHandler->>OrderService: completeOrder(orderId)
        else 재고 부족
            OrderEventHandler->>OrderService: failPaymentOrder(orderId)
            OrderEventHandler->>PointService: refund(userId, pointUsed)
            OrderEventHandler->>CouponService: restoreCoupon(couponId)
        end
    end
```

## 6. 결제 콜백 처리

```mermaid
sequenceDiagram
    participant PG
    participant PaymentCallbackController
    participant PaymentCallbackFacade
    participant PaymentService
    participant EventPublisher as DomainEventPublisher
    participant PaymentEventHandler
    participant OrderService
    participant ProductService
    participant PointService
    participant CouponService

    PG->>+PaymentCallbackController: POST /api/v1/payments/callback
    PaymentCallbackController->>+PaymentCallbackFacade: handleCallback(request)

    alt status=SUCCESS
        PaymentCallbackFacade->>+PaymentService: getByTransactionKeyWithLock(txKey)
        PaymentService-->>-PaymentCallbackFacade: Payment

        alt 이미 완료된 결제
            PaymentCallbackFacade-->>PaymentCallbackController: no-op
        else 미완료 결제
            PaymentCallbackFacade->>+PaymentService: toSuccess(payment, completedAt)
            Note over PaymentService,EventPublisher: PaymentSucceededEvent 발행
            PaymentService-->>-PaymentCallbackFacade: updated
        end
    else status=FAILED
        PaymentCallbackFacade->>+PaymentService: getByTransactionKeyWithLock(txKey)
        PaymentService-->>-PaymentCallbackFacade: Payment

        alt 이미 완료된 결제
            PaymentCallbackFacade-->>PaymentCallbackController: no-op
        else 미완료 결제
            PaymentCallbackFacade->>+PaymentService: toFailed(payment, reason, completedAt)
            Note over PaymentService,EventPublisher: PaymentFailedEvent 발행
            PaymentService-->>-PaymentCallbackFacade: updated
        end
    else 기타 상태
        PaymentCallbackFacade-->>PaymentCallbackController: no-op
    end

    PaymentCallbackController-->>-PG: 200 OK

    Note over PaymentEventHandler: AFTER_COMMIT + @Async
    alt PaymentSucceededEvent
        PaymentEventHandler->>OrderService: getWithItemsById(orderId)
        PaymentEventHandler->>ProductService: tryDecreaseStocks(orderItems)
        alt 재고 충분
            PaymentEventHandler->>OrderService: completeOrder(orderId)
        else 재고 부족
            PaymentEventHandler->>OrderService: failPaymentOrder(orderId)
            PaymentEventHandler->>PointService: refund(userId, pointUsed)
            PaymentEventHandler->>CouponService: restoreCoupon(couponId)
        end
    else PaymentFailedEvent
        PaymentEventHandler->>OrderService: failPaymentOrder(orderId)
        PaymentEventHandler->>PointService: refund(optional)
        PaymentEventHandler->>CouponService: restoreCoupon(optional)
    end
```

## 7. 결제 복구 스케줄러

```mermaid
sequenceDiagram
    participant Scheduler as PaymentRecoveryScheduler
    participant PaymentService
    participant PgClient
    participant CallbackFacade as PaymentCallbackFacade

    loop 60초 주기
        Scheduler->>+PaymentService: findPendingPaymentsBefore(now-1m)
        PaymentService-->>-Scheduler: List<Payment>

        loop 각 pending payment
            Scheduler->>+PgClient: getTransaction(userId, txKey)
            PgClient-->>-Scheduler: PgTransactionResponse

            alt status=SUCCESS
                Scheduler->>+CallbackFacade: handleSuccess(txKey)
                CallbackFacade-->>-Scheduler: done
            else status=FAILED
                Scheduler->>+CallbackFacade: handleFailed(txKey, reason)
                CallbackFacade-->>-Scheduler: done
            else status=PENDING
                Note over Scheduler: 다음 주기에 재확인
            end
        end
    end
```

## 8. Outbox 발행/Relay

```mermaid
sequenceDiagram
    participant DomainTx as Domain Transaction
    participant Writer as OutboxEventWriter
    participant Repo as OutboxEventRepository
    participant Updater as OutboxEventUpdater
    participant Relay as OutboxRelay
    participant Kafka

    DomainTx->>+Writer: TransactionalEventListener(BEFORE_COMMIT)
    Writer->>+Repo: save(outbox_event, status=NEW)
    Repo-->>-Writer: saved
    Writer-->>-DomainTx: done

    alt 이벤트가 ImmediatePublishEvent
        DomainTx->>+Writer: TransactionalEventListener(AFTER_COMMIT)
        Writer->>+Updater: updateStatusToSending(eventId)
        alt 선점 성공
            Writer->>Kafka: send(topic, aggregateId, envelope)
            alt 전송 성공
                Writer->>Updater: toSent(eventId)
            else 전송 실패
                Writer->>Updater: resetToNew(eventId)
            end
        else 선점 실패
            Note over Writer: Relay가 처리
        end
    end

    loop fixed-delay relay
        Relay->>Repo: recoverExpiredEvents()
        Relay->>Repo: findNewEventsReadyToSend(batch)
        Relay->>Repo: updateStatusToSending(eventId)
        Relay->>Kafka: send(...)
        alt 성공
            Relay->>Repo: save(status=SENT)
        else 실패
            Relay->>Repo: save(status=NEW 재시도 or DEAD)
        end
    end
```

## 9. 일간 랭킹 조회

```mermaid
sequenceDiagram
    participant Client
    participant RankingController
    participant RankingFacade
    participant RankingService
    participant RankingRepo as RankingRedisRepository
    participant ProductService
    participant BrandService
    participant ProductLikeService

    Client->>+RankingController: GET /api/v1/rankings/daily?date=yyyyMMdd
    RankingController->>+RankingFacade: getDailyRanking(date, page, size, userId)

    RankingFacade->>+RankingService: getTopN(date, page, size)
    RankingService->>+RankingRepo: getTopN(redisKey)
    RankingRepo-->>-RankingService: List<RankingEntry>
    RankingService-->>-RankingFacade: entries

    RankingFacade->>+ProductService: findByIds(productIds)
    ProductService-->>-RankingFacade: products
    RankingFacade->>+BrandService: findByIdIn(brandIds)
    BrandService-->>-RankingFacade: brands
    RankingFacade->>+ProductLikeService: findLikeStatusByProductId(userId, productIds)
    ProductLikeService-->>-RankingFacade: liked map

    RankingFacade-->>-RankingController: RankingResult
    RankingController-->>-Client: 200 OK
```

## 10. 주간/월간 랭킹 조회

```mermaid
sequenceDiagram
    participant Client
    participant RankingController
    participant RankingFacade
    participant RankingService
    participant WeeklyRepo as WeeklyProductRankRepository
    participant MonthlyRepo as MonthlyProductRankRepository

    Client->>+RankingController: GET /api/v1/rankings/weekly or monthly
    RankingController->>+RankingFacade: getWeeklyRanking/getMonthlyRanking

    alt weekly
        RankingFacade->>+RankingService: getWeeklyTopN(date, page, size)
        RankingService->>+WeeklyRepo: findByYearWeekOrderByScoreDesc
        WeeklyRepo-->>-RankingService: weekly rows
        RankingService-->>-RankingFacade: ranking entries
    else monthly
        RankingFacade->>+RankingService: getMonthlyTopN(date, page, size)
        RankingService->>+MonthlyRepo: findByYearMonthOrderByScoreDesc
        MonthlyRepo-->>-RankingService: monthly rows
        RankingService-->>-RankingFacade: ranking entries
    end

    Note over RankingFacade: 이후 Product/Brand/Like 조합은 일간과 동일
    RankingFacade-->>-RankingController: RankingResult
    RankingController-->>-Client: 200 OK
```
