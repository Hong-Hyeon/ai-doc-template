# Commerce Platform - API Design Guide

## 📋 목차

1. [개요](#개요)
2. [GraphQL API 설계](#graphql-api-설계)
3. [gRPC API 설계](#grpc-api-설계)
4. [공통 설계 원칙](#공통-설계-원칙)
5. [에러 처리](#에러-처리)
6. [버전 관리](#버전-관리)
7. [문서화](#문서화)

---

## 개요

본 문서는 Commerce Platform의 API 설계 가이드라인을 정의합니다.

### API 통신 매트릭스

| 통신 구간 | 프로토콜 | 용도 |
|----------|----------|------|
| Client → Backend | GraphQL | Frontend 데이터 요청 |
| Backend ↔ Backend | gRPC | 서비스 간 동기 통신 |
| Backend ↔ Backend | Kafka | 서비스 간 비동기 통신 |

---

## GraphQL API 설계

### 1. 스키마 설계 원칙

#### 네이밍 컨벤션

```graphql
# ✅ Good - PascalCase for types
type Product {
  id: ID!
  name: String!
  # camelCase for fields
  createdAt: DateTime!
  updatedAt: DateTime!
}

# ✅ Good - PascalCase for input types with "Input" suffix
input CreateProductInput {
  name: String!
  price: Int!
  categoryId: ID!
}

# ✅ Good - PascalCase for payload types with "Payload" suffix
type CreateProductPayload {
  product: Product
  errors: [UserError!]
}

# ✅ Good - SCREAMING_SNAKE_CASE for enums
enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}
```

#### Type 설계

```graphql
# 기본 타입
type Product {
  id: ID!
  name: String!
  description: String
  price: Int!                    # 가격은 센트/원 단위 정수로
  status: ProductStatus!
  category: Category!            # 연관 관계
  images: [ProductImage!]!       # 리스트는 non-null 리스트 + non-null 아이템
  createdAt: DateTime!
  updatedAt: DateTime!
}

# 페이지네이션을 위한 Connection 타입 (Relay 스펙)
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ProductEdge {
  cursor: String!
  node: Product!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### 2. Query 설계

```graphql
type Query {
  # 단일 조회 - nullable (없으면 null 반환)
  product(id: ID!): Product
  order(id: ID!): Order
  user(id: ID!): User
  
  # 리스트 조회 - Connection 타입 사용
  products(
    filter: ProductFilter
    sort: ProductSort
    first: Int
    after: String
    last: Int
    before: String
  ): ProductConnection!
  
  orders(
    filter: OrderFilter
    first: Int
    after: String
  ): OrderConnection!
  
  # 검색
  searchProducts(
    query: String!
    filter: ProductFilter
    first: Int
    after: String
  ): ProductSearchConnection!
  
  # 현재 사용자 정보
  me: User
}

# 필터 Input
input ProductFilter {
  categoryId: ID
  status: ProductStatus
  minPrice: Int
  maxPrice: Int
  brandIds: [ID!]
}

# 정렬 Input
input ProductSort {
  field: ProductSortField!
  direction: SortDirection!
}

enum ProductSortField {
  CREATED_AT
  PRICE
  NAME
  POPULARITY
}

enum SortDirection {
  ASC
  DESC
}
```

### 3. Mutation 설계

```graphql
type Mutation {
  # Create
  createProduct(input: CreateProductInput!): CreateProductPayload!
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  
  # Update
  updateProduct(id: ID!, input: UpdateProductInput!): UpdateProductPayload!
  updateOrderStatus(id: ID!, status: OrderStatus!): UpdateOrderStatusPayload!
  
  # Delete
  deleteProduct(id: ID!): DeleteProductPayload!
  
  # 특정 액션
  cancelOrder(id: ID!, reason: String): CancelOrderPayload!
  addToCart(input: AddToCartInput!): AddToCartPayload!
  checkout(input: CheckoutInput!): CheckoutPayload!
}

# Input 타입
input CreateOrderInput {
  items: [OrderItemInput!]!
  shippingAddressId: ID!
  paymentMethodId: ID!
  couponCode: String
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
  options: [OrderItemOptionInput!]
}

# Payload 타입 - 항상 성공/실패 모두 처리
type CreateOrderPayload {
  order: Order                    # 성공 시 반환
  errors: [UserError!]!           # 실패 시 에러 목록
}

# 사용자 친화적 에러 타입
type UserError {
  field: String                   # 에러가 발생한 필드
  code: ErrorCode!                # 에러 코드
  message: String!                # 사용자에게 보여줄 메시지
}

enum ErrorCode {
  INVALID_INPUT
  NOT_FOUND
  UNAUTHORIZED
  FORBIDDEN
  INSUFFICIENT_STOCK
  INVALID_COUPON
  PAYMENT_FAILED
}
```

### 4. Subscription 설계

```graphql
type Subscription {
  # 주문 상태 변경 구독
  orderStatusChanged(orderId: ID!): OrderStatusEvent!
  
  # 새 알림 구독
  notificationReceived: Notification!
  
  # 장바구니 업데이트 구독
  cartUpdated: Cart!
}

type OrderStatusEvent {
  orderId: ID!
  previousStatus: OrderStatus!
  currentStatus: OrderStatus!
  timestamp: DateTime!
}
```

### 5. 인증/인가

```graphql
# Directive를 사용한 인가
directive @auth(requires: Role = USER) on FIELD_DEFINITION

enum Role {
  USER
  SELLER
  ADMIN
}

type Query {
  # 인증 필요 없음
  products: ProductConnection!
  
  # 인증 필요
  me: User @auth
  
  # 특정 역할 필요
  adminDashboard: Dashboard @auth(requires: ADMIN)
}

type Mutation {
  # 모든 Mutation은 기본적으로 인증 필요
  createOrder(input: CreateOrderInput!): CreateOrderPayload! @auth
  
  # 판매자 전용
  createProduct(input: CreateProductInput!): CreateProductPayload! @auth(requires: SELLER)
}
```

### 6. N+1 문제 해결

```python
# DataLoader 패턴 사용
from strawberry.dataloader import DataLoader

async def load_products(keys: list[str]) -> list[Product]:
    """배치로 상품 조회"""
    products = await product_repo.get_by_ids(keys)
    product_map = {str(p.id): p for p in products}
    return [product_map.get(key) for key in keys]

# Context에 DataLoader 주입
class Context:
    def __init__(self):
        self.product_loader = DataLoader(load_fn=load_products)
        self.user_loader = DataLoader(load_fn=load_users)
        self.category_loader = DataLoader(load_fn=load_categories)

# Resolver에서 사용
@strawberry.type
class Order:
    id: strawberry.ID
    
    @strawberry.field
    async def items(self, info: Info) -> list[OrderItem]:
        # DataLoader를 통해 배치 조회
        return await info.context.order_item_loader.load(self.id)
```

---

## gRPC API 설계

### 1. Proto 파일 구조

```
protos/
├── common/
│   ├── pagination.proto      # 공통 페이지네이션
│   ├── error.proto           # 공통 에러 타입
│   └── timestamp.proto       # 타임스탬프 (google/protobuf 사용)
├── user/
│   └── user_service.proto
├── product/
│   └── product_service.proto
├── order/
│   └── order_service.proto
├── payment/
│   └── payment_service.proto
└── inventory/
    └── inventory_service.proto
```

### 2. 네이밍 컨벤션

```protobuf
syntax = "proto3";

// 패키지명: 소문자, 도트 구분
package commerce.order.v1;

// Go 패키지 옵션
option go_package = "github.com/company/protos/order/v1;orderv1";

// 서비스명: PascalCase + "Service" 접미사
service OrderService {
  // 메서드명: PascalCase
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
}

// 메시지명: PascalCase
message Order {
  // 필드명: snake_case
  string id = 1;
  string user_id = 2;
  OrderStatus status = 3;
  int64 total_amount = 4;
  google.protobuf.Timestamp created_at = 5;
}

// Enum: PascalCase, 값은 SCREAMING_SNAKE_CASE
// 첫 번째 값은 반드시 0이고 UNSPECIFIED
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}
```

### 3. Request/Response 패턴

```protobuf
// Get (단일 조회)
message GetOrderRequest {
  string order_id = 1;
}

message GetOrderResponse {
  Order order = 1;
}

// List (목록 조회) - 페이지네이션 포함
message ListOrdersRequest {
  string user_id = 1;
  OrderFilter filter = 2;
  Pagination pagination = 3;
}

message ListOrdersResponse {
  repeated Order orders = 1;
  PaginationInfo pagination_info = 2;
}

// Create
message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItemInput items = 2;
  string shipping_address_id = 3;
}

message CreateOrderResponse {
  Order order = 1;
}

// Update
message UpdateOrderStatusRequest {
  string order_id = 1;
  OrderStatus status = 2;
  string reason = 3;  // optional
}

message UpdateOrderStatusResponse {
  Order order = 1;
}

// Delete
message DeleteOrderRequest {
  string order_id = 1;
}

message DeleteOrderResponse {
  bool success = 1;
}
```

### 4. 공통 메시지

```protobuf
// common/pagination.proto
syntax = "proto3";
package commerce.common;

message Pagination {
  int32 page_size = 1;          // 페이지 크기 (기본: 20, 최대: 100)
  string page_token = 2;         // 다음 페이지 토큰
}

message PaginationInfo {
  int32 total_count = 1;         // 전체 개수
  bool has_next_page = 2;        // 다음 페이지 존재 여부
  string next_page_token = 3;    // 다음 페이지 토큰
}
```

```protobuf
// common/error.proto
syntax = "proto3";
package commerce.common;

message ErrorDetail {
  string code = 1;              // 에러 코드
  string message = 2;           // 에러 메시지
  string field = 3;             // 관련 필드 (optional)
  map<string, string> metadata = 4;  // 추가 메타데이터
}
```

### 5. 서비스 정의 예시

```protobuf
// order/order_service.proto
syntax = "proto3";

package commerce.order.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "common/pagination.proto";

service OrderService {
  // 주문 조회
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  
  // 주문 목록 조회
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
  
  // 주문 생성
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  
  // 주문 상태 업데이트
  rpc UpdateOrderStatus(UpdateOrderStatusRequest) returns (UpdateOrderStatusResponse);
  
  // 주문 취소
  rpc CancelOrder(CancelOrderRequest) returns (CancelOrderResponse);
  
  // 스트리밍: 주문 상태 실시간 조회
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusEvent);
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  int64 total_amount = 5;          // 센트/원 단위
  int64 discount_amount = 6;
  int64 shipping_fee = 7;
  ShippingAddress shipping_address = 8;
  google.protobuf.Timestamp created_at = 9;
  google.protobuf.Timestamp updated_at = 10;
}

message OrderItem {
  string id = 1;
  string product_id = 2;
  string product_name = 3;
  int32 quantity = 4;
  int64 unit_price = 5;
  int64 total_price = 6;
}

message ShippingAddress {
  string recipient_name = 1;
  string phone = 2;
  string address_line1 = 3;
  string address_line2 = 4;
  string city = 5;
  string postal_code = 6;
}

// Request/Response 메시지들...
```

### 6. 에러 처리

```protobuf
// gRPC 표준 에러 코드 사용
// https://grpc.github.io/grpc/core/md_doc_statuscodes.html

// Python에서의 에러 반환
import grpc

async def GetOrder(self, request, context):
    order = await self.order_service.get_order(request.order_id)
    
    if not order:
        context.set_code(grpc.StatusCode.NOT_FOUND)
        context.set_details(f"Order {request.order_id} not found")
        return GetOrderResponse()
    
    return GetOrderResponse(order=self._to_proto(order))
```

#### 에러 코드 매핑

| 상황 | gRPC Status | HTTP Status |
|------|-------------|-------------|
| 리소스 없음 | NOT_FOUND | 404 |
| 잘못된 요청 | INVALID_ARGUMENT | 400 |
| 인증 실패 | UNAUTHENTICATED | 401 |
| 권한 없음 | PERMISSION_DENIED | 403 |
| 중복 리소스 | ALREADY_EXISTS | 409 |
| 비즈니스 규칙 위반 | FAILED_PRECONDITION | 412 |
| 서버 에러 | INTERNAL | 500 |
| 서비스 불가 | UNAVAILABLE | 503 |

### 7. 인터셉터

```python
# auth_interceptor.py
import grpc
from grpc import aio


class AuthInterceptor(aio.ServerInterceptor):
    """인증 인터셉터"""
    
    # 인증이 필요 없는 메서드
    EXCLUDED_METHODS = [
        "/grpc.health.v1.Health/Check",
        "/grpc.reflection.v1alpha.ServerReflection/ServerReflectionInfo",
    ]
    
    async def intercept_service(self, continuation, handler_call_details):
        method = handler_call_details.method
        
        if method in self.EXCLUDED_METHODS:
            return await continuation(handler_call_details)
        
        # 메타데이터에서 토큰 추출
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get("authorization", "")
        
        if not token.startswith("Bearer "):
            return self._unauthenticated_handler
        
        # 토큰 검증 로직
        try:
            user_info = verify_token(token[7:])
            # Context에 사용자 정보 추가
            handler_call_details.user_info = user_info
        except Exception:
            return self._unauthenticated_handler
        
        return await continuation(handler_call_details)
    
    @property
    def _unauthenticated_handler(self):
        async def handler(request, context):
            context.set_code(grpc.StatusCode.UNAUTHENTICATED)
            context.set_details("Invalid or missing authentication token")
            return None
        return grpc.unary_unary_rpc_method_handler(handler)
```

```python
# logging_interceptor.py
import time
import grpc
from grpc import aio

from app.core.logging import logger


class LoggingInterceptor(aio.ServerInterceptor):
    """로깅 인터셉터"""
    
    async def intercept_service(self, continuation, handler_call_details):
        method = handler_call_details.method
        start_time = time.time()
        
        try:
            response = await continuation(handler_call_details)
            elapsed = time.time() - start_time
            
            logger.info(
                "gRPC request",
                method=method,
                elapsed_ms=round(elapsed * 1000, 2),
                status="OK",
            )
            
            return response
            
        except Exception as e:
            elapsed = time.time() - start_time
            
            logger.error(
                "gRPC request failed",
                method=method,
                elapsed_ms=round(elapsed * 1000, 2),
                error=str(e),
            )
            raise
```

---

## 공통 설계 원칙

### 1. ID 설계

```
# UUID v4 사용
- 분산 환경에서 충돌 없이 생성 가능
- 예측 불가능 (보안)

# 형식
- GraphQL: ID 스칼라 타입 (문자열)
- gRPC: string 타입
- DB: UUID 타입 또는 CHAR(36)
```

### 2. 날짜/시간

```
# ISO 8601 형식 사용
- GraphQL: DateTime 커스텀 스칼라
- gRPC: google.protobuf.Timestamp
- JSON: "2025-12-03T10:30:00Z"

# 항상 UTC 기준
- 클라이언트에서 로컬 시간으로 변환
```

### 3. 금액/가격

```
# 정수 사용 (센트/원 단위)
- 부동소수점 오차 방지
- 10000 = 10,000원 또는 $100.00

# 타입
- GraphQL: Int
- gRPC: int64
```

### 4. 페이지네이션

```
# Cursor 기반 (권장)
- 대용량 데이터에 효율적
- 실시간 데이터에서 중복/누락 방지

# Offset 기반 (단순한 경우)
- 구현이 간단
- 작은 데이터셋에 적합
```

---

## 에러 처리

### GraphQL 에러 응답

```json
{
  "data": {
    "createOrder": {
      "order": null,
      "errors": [
        {
          "field": "items[0].quantity",
          "code": "INSUFFICIENT_STOCK",
          "message": "상품 재고가 부족합니다. 현재 재고: 5개"
        }
      ]
    }
  }
}
```

### gRPC 에러 상세 정보

```python
from google.rpc import status_pb2, error_details_pb2
from grpc_status import rpc_status

# 상세 에러 정보 포함
detail = error_details_pb2.BadRequest()
violation = detail.field_violations.add()
violation.field = "quantity"
violation.description = "Insufficient stock"

status = status_pb2.Status(
    code=grpc.StatusCode.FAILED_PRECONDITION.value[0],
    message="Insufficient stock for product",
    details=[any_pb2.Any(value=detail.SerializeToString())]
)

context.abort_with_status(rpc_status.to_status(status))
```

---

## 버전 관리

### GraphQL 버전 관리

```graphql
# Deprecation 사용 (필드 레벨)
type Product {
  id: ID!
  name: String!
  
  # 기존 필드 deprecated
  price: Int! @deprecated(reason: "Use 'pricing' field instead")
  
  # 새로운 필드
  pricing: ProductPricing!
}

type ProductPricing {
  amount: Int!
  currency: Currency!
  discountedAmount: Int
}
```

### gRPC 버전 관리

```protobuf
// 패키지에 버전 포함
package commerce.order.v1;
package commerce.order.v2;

// 하위 호환성 유지
// - 필드 번호 변경 금지
// - 필드 삭제 대신 reserved 사용
// - 필드 타입 변경 금지

message Order {
  string id = 1;
  string user_id = 2;
  
  // 삭제된 필드
  reserved 3;
  reserved "old_field_name";
  
  // 새 필드는 새 번호로
  string new_field = 10;
}
```

---

## 문서화

### GraphQL 문서화

```graphql
"""
상품 정보를 나타내는 타입입니다.
"""
type Product {
  """
  상품의 고유 식별자
  """
  id: ID!
  
  """
  상품명
  최소 1자, 최대 200자
  """
  name: String!
  
  """
  상품 가격 (원 단위)
  음수 불가
  """
  price: Int!
}

"""
상품 생성을 위한 입력 타입
"""
input CreateProductInput {
  """
  상품명 (필수)
  - 최소 1자, 최대 200자
  - 특수문자 사용 가능
  """
  name: String!
  
  """
  상품 가격 (필수)
  - 원 단위 정수
  - 0 이상
  """
  price: Int!
}
```

### gRPC 문서화

```protobuf
// 서비스 설명
// 주문 관련 모든 기능을 제공하는 서비스입니다.
service OrderService {
  // 주문 조회
  // 
  // 주문 ID로 단일 주문을 조회합니다.
  // 존재하지 않는 주문은 NOT_FOUND 에러를 반환합니다.
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
}

message GetOrderRequest {
  // 조회할 주문의 ID (UUID 형식)
  string order_id = 1;
}
```

---

## 체크리스트

### GraphQL API 체크리스트

- [ ] 모든 Query는 nullable 반환 (단일 조회)
- [ ] 모든 리스트는 Connection 타입 사용
- [ ] 모든 Mutation은 Payload 타입 반환
- [ ] UserError 포함하여 에러 처리
- [ ] DataLoader로 N+1 문제 해결
- [ ] 적절한 인증/인가 directive 적용
- [ ] 모든 타입에 description 추가

### gRPC API 체크리스트

- [ ] 패키지에 버전 포함 (v1, v2...)
- [ ] 모든 RPC는 Request/Response 쌍으로 정의
- [ ] Enum 첫 번째 값은 UNSPECIFIED
- [ ] 페이지네이션 메시지 표준화
- [ ] 적절한 gRPC 에러 코드 사용
- [ ] 인터셉터로 공통 로직 처리
- [ ] Proto 파일에 주석 추가

---

> 📅 **최종 업데이트**: 2025-12-03  
> ✍️ **작성자**: Backend Team

