# 01-requirements.md - 요구사항 명세 (현재 코드 기준)

연결 문서:
- 개요: [README.md](../../README.md)
- 시퀀스: [02-sequence-diagrams.md](02-sequence-diagrams.md)
- 클래스: [03-class-diagram.md](03-class-diagram.md)
- ERD: [04-erd.md](04-erd.md)

## 1. 문서 목적
- 이 문서는 `apps/commerce-api`의 **현재 구현 코드**를 기준으로 기능 요구사항을 정리한다.
- 기존 설계 초안이 아닌, 실제 컨트롤러/파사드/도메인/통합 테스트 동작을 기준으로 작성한다.

## 2. 시스템 범위

### 2.1 포함 범위
- 사용자 등록/조회 (`/api/v1/users`)
- 포인트 조회/충전 (`/api/v1/points`)
- 브랜드/상품 조회 (`/api/v1/brands`, `/api/v1/products`)
- 좋아요 등록/취소/목록 (`/api/v1/like/products`)
- 주문 생성/조회 (`/api/v1/orders`)
- 결제 콜백 처리 (`/api/v1/payments/callback`)
- 랭킹 조회 (`/api/v1/rankings/*`)

### 2.2 연계 범위
- `commerce-streamer`: Kafka `catalog-events` 소비 후 일간 랭킹 점수 누적
- `commerce-batch`: `product_metrics` 집계를 통해 주간/월간 랭킹 MV 생성

## 3. 공통 규칙

### 3.1 사용자 식별 (`X-USER-ID`)
- 선택 헤더: `GET /api/v1/products`, `GET /api/v1/products/{productId}`, `GET /api/v1/rankings/*`
- 필수 헤더: 좋아요/주문/포인트 API
- 필수 헤더 누락 시: `400 Bad Request` (`MissingRequestHeaderException`)

### 3.2 표준 응답 포맷
- 모든 API는 `ApiResponse<T>` 포맷 사용
- 성공: `meta.result=SUCCESS`, 실패: `meta.result=FAIL`
- 실패 시 `errorCode`, `message` 제공

### 3.3 예외/상태 코드 원칙
- 리소스 미존재: `404 Not Found` (`ErrorType.NOT_FOUND`)
- 요청 유효성 오류: `400 Bad Request`
- 권한 오류(타인 주문 조회): `403 Forbidden`
- 처리되지 않은 예외: `500 Internal Server Error`

### 3.4 일관성 모델
- 일부 로직은 `@TransactionalEventListener(AFTER_COMMIT)` + `@Async`로 처리
- 즉시 응답 값과 후속 비동기 반영 값(예: `like_count`, 주문 상태)이 짧은 시간 차이로 다를 수 있음

## 4. 도메인별 기능 요구사항

### 4.1 User

#### 4.1.1 사용자 등록
- `POST /api/v1/users/register`
- 요청: `loginId`, `email`, `birth`, `gender`
- 검증: 로그인 ID 형식/길이, 이메일 형식, 생년월일 과거 여부

#### 4.1.2 사용자 조회
- `GET /api/v1/users/{loginId}`
- 존재하지 않으면 `404`

### 4.2 Point

#### 4.2.1 포인트 조회
- `GET /api/v1/points`
- `X-USER-ID` 기준 조회
- 포인트 데이터가 없으면 `200 OK` + `data=null`

#### 4.2.2 포인트 충전
- `PATCH /api/v1/points/charge`
- 요청: `amount` (양수)
- 동시성 제어: 사용자 포인트 행 조회 시 락 사용

### 4.3 Brand

#### 4.3.1 브랜드 조회
- `GET /api/v1/brands/{brandId}`
- 응답: `brandId`, `name`, `description`

### 4.4 Product

#### 4.4.1 상품 목록 조회
- `GET /api/v1/products`
- 파라미터:
  - `brandId` (optional)
  - `sort`: `latest`, `price_asc`, `likes_desc`
  - `page` (default `0`), `size` (default `20`)
- 응답: 상품/브랜드/좋아요 수/사용자 좋아요 여부 + `pageInfo`

#### 4.4.2 상품 상세 조회
- `GET /api/v1/products/{productId}`
- 응답: 목록 필드 + `description`, `stock`, `rank`
- 상세 조회 시 `ProductViewedEvent` 발행

#### 4.4.3 캐시 정책
- 아래 조건에서만 목록 캐시 사용:
  - 비회원(`userId == null`)
  - `page=0`, `size=20`
  - 정렬 `latest` 또는 `likes_desc`

### 4.5 Like

#### 4.5.1 좋아요 등록
- `POST /api/v1/like/products/{productId}`
- 사전 검증: 상품 존재 확인
- 중복 등록은 멱등 처리 (`200 OK`)
- 구현: `INSERT IGNORE` + `UNIQUE(ref_user_id, ref_product_id)`

#### 4.5.2 좋아요 취소
- `DELETE /api/v1/like/products/{productId}`
- 이미 취소된 상태 재요청도 멱등 처리 (`200 OK`)

#### 4.5.3 좋아요 목록 조회
- `GET /api/v1/like/products`
- 정렬: `latest`, `product_name`, `price_asc`, `price_desc`
- 응답: 상품/브랜드/좋아요 수 + `pageInfo`

#### 4.5.4 좋아요 수 반영 방식
- 좋아요/취소 시 `ProductLikedEvent`, `ProductUnlikedEvent` 발행
- AFTER_COMMIT 비동기 핸들러가 `product.like_count`를 증감

### 4.6 Order / Payment / Coupon

#### 4.6.1 주문 생성
- `POST /api/v1/orders`
- 요청:
  - `items[]`: `{ productId, quantity }`
  - `paymentInfo` (카드 결제 필요 시 필수)
  - `couponId` (optional)
- 응답: `201 Created` + 주문 상세(`status`, 금액 분해, 아이템 목록)

#### 4.6.2 주문 생성 시 동기 처리
- 상품 행 비관적 락 조회 (`findByIdsWithLock`)
- 주문 아이템 생성 및 재고 사전 검증
- 할인/포인트/PG 결제금 계산
- `Order`를 `PENDING` 상태로 저장
- `OrderCreatedEvent` 발행
- PG 결제 필요 시 `Payment(REQUESTED)` 생성 후 PG 요청

#### 4.6.3 주문 생성 후 비동기 처리
- `OrderCreatedEvent` 기반:
  - 포인트 차감
  - 쿠폰 사용
  - 포인트 전액 결제 주문이면 재고 차감 후 주문 완료/실패 처리
- PG 결제 주문은 콜백 전까지 `PENDING` 유지 가능

#### 4.6.4 결제 콜백
- `POST /api/v1/payments/callback`
- `status=SUCCESS|FAILED`만 처리
- 성공:
  - `Payment -> SUCCESS`
  - `PaymentSucceededEvent` 발행
  - 이벤트 핸들러가 재고 차감 후 주문 `COMPLETED` 또는 `PAYMENT_FAILED`
- 실패:
  - `Payment -> FAILED`
  - 주문 `PAYMENT_FAILED`, 포인트/쿠폰 복구
- 같은 `transactionKey` 재호출 시 완료 결제는 멱등하게 무시

#### 4.6.5 결제 복구 스케줄러
- 1분 이상 `PENDING`인 결제를 주기적으로 조회
- PG 조회 API로 상태 재확인 후 성공/실패 처리 재진입

#### 4.6.6 주문 조회
- `GET /api/v1/orders`: 사용자 주문 목록 페이지 조회
- `GET /api/v1/orders/{orderId}`: 본인 주문만 조회 가능, 타인 주문은 `403`

### 4.7 Ranking

#### 4.7.1 일간 랭킹
- `GET /api/v1/rankings/daily?date=yyyyMMdd&page=0&size=10`
- Redis Sorted Set에서 점수/순위 조회

#### 4.7.2 주간/월간 랭킹
- `GET /api/v1/rankings/weekly`
- `GET /api/v1/rankings/monthly`
- MySQL MV(`mv_product_rank_weekly`, `mv_product_rank_monthly`) 조회

#### 4.7.3 응답 조합
- 랭킹 점수 + 상품명/가격/브랜드 + 현재 사용자 liked 여부

## 5. 비기능 요구사항

### 5.1 재고 정합성
- 주문 시 상품 행 비관적 락 사용
- 동시 주문 상황에서 재고 음수 방지

### 5.2 멱등성
- 좋아요 등록/취소 멱등
- 결제 콜백 중복 처리 멱등

### 5.3 이벤트 전달 신뢰성
- 도메인 이벤트를 Outbox 테이블에 기록 후 Kafka 전송
- Relay 재시도 및 `DEAD` 전이 지원

## 6. 상태 전이 요약

### 6.1 주문 상태 (`OrderStatus`)
- `PENDING`: 주문 생성 직후
- `COMPLETED`: 결제/재고 처리 완료
- `PAYMENT_FAILED`: 결제 실패 또는 후속 처리 실패

### 6.2 결제 상태 (`PaymentStatus`)
- `REQUESTED`: PG 요청 직전 생성
- `PENDING`: PG 처리 중
- `SUCCESS`: 콜백 성공
- `FAILED`: 콜백 실패
- `REQUEST_FAILED`: PG 요청 자체 실패
