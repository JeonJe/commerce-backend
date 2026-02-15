# ADR-003: 주문 재고 차감은 비관적 락(Pessimistic Lock) 사용

## 결정사항
**주문 생성 시 재고 차감은 비관적 락(SELECT FOR UPDATE)을 사용한다.**

```java
// ProductService
@Transactional
public List<Product> findByIdsWithLock(List<Long> productIds) {
  Objects.requireNonNull(productIds, "상품 ID 목록은 null일 수 없습니다");
  List<Long> distinctIds = productIds.stream().distinct().toList();
  return productRepository.findAllByIdWithLock(distinctIds);
}

// ProductJpaRepository
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id IN :ids ORDER BY p.id ASC")
List<Product> findAllByIdWithLock(@Param("ids") List<Long> ids);
```

## 상황
동시 주문 처리 시 재고 관리 문제:
- 여러 사용자가 동시에 같은 상품 주문
- 재고 확인과 차감 사이 경쟁 조건(race condition) 발생 가능
- 재고가 음수가 되면 비즈니스 규칙 위반

## 선택 근거

### 1. 낙관적 락(Optimistic Lock) vs 비관적 락(Pessimistic Lock)

**낙관적 락 (@Version)**:
- 충돌 발생 시 예외 발생 -> 재시도 필요
- 충돌 빈도가 낮을 때 효율적
- 사용자에게 "재고 부족" 에러 반환 -> UX 저하

**비관적 락 (SELECT FOR UPDATE)**:
- 트랜잭션 시작 시 즉시 락 획득
- 충돌 원천 차단 -> 재시도 불필요
- 선착순 정확성 보장

### 2. 재고 관리의 특성

**재고는 선착순 확정이 필수**:
- "재고 1개" 상황에서 동시 주문 10건 -> 정확히 1건만 성공
- 낙관적 락: 9건이 "충돌" 에러 -> 재시도 -> 다시 실패 (반복)
- 비관적 락: 9건이 대기 -> 1건 성공 후 즉시 "재고 부족" 에러

### 3. 데드락 방지: ORDER BY p.id ASC

여러 상품을 동시에 주문할 때, 트랜잭션 간 락 획득 순서가 다르면 데드락이 발생할 수 있다.

```
// 데드락 시나리오 (ORDER BY 없을 때)
TX1: LOCK(상품A) -> LOCK(상품B)  // 대기
TX2: LOCK(상품B) -> LOCK(상품A)  // 대기 -> 데드락!
```

`ORDER BY p.id ASC`로 모든 트랜잭션이 동일한 순서로 락을 획득하여 데드락을 방지한다.

```java
// OrderFacade.getProductByIdWithLocks()
List<Long> productIds = commands.stream()
    .map(OrderItemCommand::productId)
    .distinct()
    .sorted()   // 클라이언트 측도 정렬
    .toList();

return productService.findByIdsWithLock(productIds).stream()
    .collect(Collectors.toMap(Product::getId, Function.identity()));
```

## 재고 차감 흐름

주문 생성과 재고 차감은 두 시점에서 발생한다:

### 주문 생성 시: 재고 검증만 수행

```java
// OrderFacade.createOrder() - @Transactional
Map<Long, Product> productById = getProductByIdWithLocks(commands);    // 비관적 락 획득
OrderPreparation result = orderCreateService.prepareOrder(commands, productById);
// prepareOrder 내부에서 orderItems.validateStock(productById) 호출
```

### 결제 완료 후: 실제 재고 차감

```java
// ProductService.tryDecreaseStocks()
@Transactional
public StockDecreaseResult tryDecreaseStocks(List<OrderItem> orderItems, Long orderId) {
  Map<Long, Long> quantityByProductId = sumQuantityByProductId(orderItems);

  List<Product> products = productRepository.findAllByIdWithLock(    // 다시 비관적 락 획득
      new ArrayList<>(quantityByProductId.keySet()));

  // 상품 존재 여부 + 재고 충분성 검증
  List<Long> insufficientProductIds = findInsufficientProducts(products, quantityByProductId);
  if (!insufficientProductIds.isEmpty()) {
    return StockDecreaseResult.failure(insufficientProductIds);
  }

  // 재고 차감 + 판매/품절 이벤트 발행
  decreaseAndPublishEvents(products, quantityByProductId, orderId);
  return StockDecreaseResult.success();
}
```

`StockDecreaseResult`로 성공/실패를 반환하며, 실패 시 호출 측에서 주문 실패 + 포인트/쿠폰 복구를 처리한다.

## 동시성 테스트 검증

```java
@Test
@DisplayName("재고 1개 상품에 2명이 동시 주문 시 1개만 성공하고 재고가 0이 된다")
void concurrentOrders_onlyOneSucceeds() {
  // given
  Product product = productRepository.save(
      Product.of("재고1개상품", Money.of(10000L), "재고 1개 상품", Stock.of(1L), BRAND_ID));

  // when: CompletableFuture로 2명 동시 주문
  futures.add(asyncExecuteWithOrderId(
      () -> orderFacade.createOrder(user1.getId(), commands), orderIds));
  futures.add(asyncExecuteWithOrderId(
      () -> orderFacade.createOrder(user2.getId(), commands), orderIds));

  // then: 비동기 이벤트 핸들러 완료 대기 (awaitility)
  await().atMost(ASYNC_EVENT_TIMEOUT_SECONDS, SECONDS).untilAsserted(() -> {
    assertThat(updatedProduct.getStockValue()).isZero();
    assertThat(completedCount).isEqualTo(1);
  });
}
```

추가 테스트 시나리오:
- **재고 5개, 10명 동시 주문**: 정확히 5개만 성공, 재고 0
- **재고 3개, 5명 동시 주문**: 재고 >= 0, 최대 3명만 성공
- **동일 유저, 서로 다른 상품 동시 주문**: 포인트 정확성 검증
- **여러 상품 역순 동시 주문**: ORDER BY로 데드락 미발생 검증

## 트레이드오프

### 장점
- **정확성 보장**: 재고 음수 불가능, 선착순 정확
- **단순성**: 재시도 로직 불필요
- **예측 가능성**: 일관된 에러 처리 (대기 -> 실패)
- **데드락 방지**: ORDER BY p.id ASC로 일관된 락 순서 보장

### 단점
- **대기 시간**: 락 대기로 인한 응답 지연 가능
- **처리량 제한**: 동시 처리 수 제한
- **확장성**: 단일 DB 락에 의존

## 향후 고려사항

### 트래픽 증가 시 대안

**1. 분산 락**

**2. 이벤트 기반 아키텍처**
```
주문 요청 -> Queue -> 재고 처리 Worker (순차 처리)
- 장점: 높은 처리량, 확장성
- 단점: 복잡도 증가, 즉시성 감소
```

## 참고
- 요구사항: `docs/design/01-requirements.md` (4.6 Order, 5.1 재고 정합성)
- ERD: `docs/design/04-erd.md`
- 테스트: `OrderFacadeRaceConditionTest.java`
