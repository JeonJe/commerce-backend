# ADR-002: ProductLike는 Hard Delete 전략 사용

## 결정사항
**ProductLike 삭제 시 Hard Delete(물리 삭제)를 사용한다.**

```java
// ProductLikeService.deleteLike()
@Transactional
public int deleteLike(Long userId, Long productId) {
  int deleted = productLikeRepository.deleteByUserIdAndProductId(userId, productId);

  if (deleted > 0) {
    LocalDateTime unlikedAt = LocalDateTime.now(clock);
    eventPublisher.publish(ProductUnlikedEvent.of(userId, productId, unlikedAt));
  }

  return deleted;
}
```

```java
// ProductLikeJpaRepository
@Modifying
@Query("DELETE FROM ProductLike pl WHERE pl.userId = :userId AND pl.productId = :productId")
int deleteByUserIdAndProductId(@Param("userId") Long userId, @Param("productId") Long productId);
```

## 상황
ProductLike 삭제 전략 선택 시 고려사항:
- 좋아요는 선호도 표시이며 법적 보관 의무가 없음
- 대규모 트래픽 환경 가정
- 사용자가 언제든지 재등록 가능한 데이터
- 조회 성능 최적화 필요

## 선택 근거

### 1. 법적 보관 의무 없음
- 좋아요는 금전 거래 아님
- 세법, 전자상거래법 적용 대상 아님
- Order/Payment와 달리 이력 보관 불필요

### 2. 성능 최적화
**Soft Delete 시 문제점**:
- 모든 조회 쿼리에 `deleted_at IS NULL` 필터 필수
- 삭제된 데이터가 인덱스에 계속 남음
- 테이블 크기 증가 -> 조회 성능 저하

**Hard Delete 장점**:
- 테이블 크기 최소화 -> 인덱스 효율 증가
- 조회 쿼리가 단순함 (필터링 불필요)

### 3. 비즈니스 의미 일치
- 좋아요 취소 = 데이터 삭제 (직관적)
- 재등록 가능하므로 복구 불필요

### 4. 멱등성 보장

좋아요 등록은 `INSERT IGNORE` + `UNIQUE(ref_user_id, ref_product_id)`로 멱등 처리한다:

```java
// ProductLikeJpaRepository
@Modifying
@Query(nativeQuery = true,
    value = "INSERT IGNORE INTO product_like (ref_user_id, ref_product_id, liked_at, created_at, updated_at) " +
            "VALUES (:userId, :productId, :likedAt, NOW(), NOW())")
int insertIgnore(@Param("userId") Long userId,
                 @Param("productId") Long productId,
                 @Param("likedAt") LocalDateTime likedAt);
```

좋아요 취소는 DELETE 반환값으로 멱등 처리한다:
- `deleted > 0`: 실제 삭제 발생 -> ProductUnlikedEvent 발행
- `deleted == 0`: 이미 취소된 상태 -> 이벤트 발행 없이 200 OK

### 5. likeCount 반영은 이벤트 기반 비동기 처리

likeCount 증감은 트랜잭션 커밋 후 비동기로 처리한다:

```java
// LikeEventHandler
@Async
@TransactionalEventListener(phase = AFTER_COMMIT)
public void handleProductUnliked(ProductUnlikedEvent event) {
  productService.decreaseLikeCount(event.productId());
}
```

```java
// ProductJpaRepository - 벌크 SQL로 원자적 업데이트
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Product p SET p.likeCount = p.likeCount - 1 WHERE p.id = :productId AND p.likeCount > 0")
int decrementLikeCount(@Param("productId") Long productId);
```

음수 방어: `WHERE p.likeCount > 0` 조건으로 DB 레벨에서 보장한다.

## 이력 추적: 이벤트 파이프라인

좋아요 이력은 메인 DB가 아닌 이벤트 파이프라인으로 관리한다:

```
ProductLikedEvent -> Outbox -> Kafka -> commerce-streamer -> product_metrics 테이블
```

Streamer가 `ProductLikedStrategy`에서 날짜별 좋아요 수를 `product_metrics`에 적재하므로,
메인 DB에서 Hard Delete해도 집계 데이터는 유지된다.

## 트레이드오프

### 장점
- 조회 성능 향상 (테이블 크기 최소화)
- 코드 단순성 (필터링 로직 불필요)
- 비즈니스 의미 일치
- 멱등성 보장 (INSERT IGNORE + DELETE 반환값)
- 이벤트 파이프라인으로 집계 데이터 보존

### 단점
- 메인 DB에서 즉시 복구 불가
- 개별 좋아요 이력은 메인 DB에서 추적 불가 (이벤트 스토어에서만 가능)

## 참고
- 요구사항: `docs/design/01-requirements.md` (4.5 Like)
- ERD: `docs/design/04-erd.md`
