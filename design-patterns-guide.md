# Commerce Platform - Design Patterns Guide

## 📋 목차

1. [개요](#개요)
2. [아키텍처 패턴](#아키텍처-패턴)
3. [도메인 주도 설계 (DDD)](#도메인-주도-설계-ddd)
4. [생성 패턴](#생성-패턴)
5. [구조 패턴](#구조-패턴)
6. [행동 패턴](#행동-패턴)
7. [분산 시스템 패턴](#분산-시스템-패턴)
8. [데이터베이스 패턴](#데이터베이스-패턴)
9. [적용 가이드](#적용-가이드)

---

## 개요

본 문서는 Commerce Platform 개발에 사용되는 디자인 패턴을 정의합니다.

### 패턴 적용 원칙

1. **문제에 맞는 패턴 선택**: 패턴을 위한 패턴 적용 금지
2. **일관성 유지**: 팀 전체가 동일한 패턴 사용
3. **단순함 우선**: 복잡한 패턴보다 단순한 해결책 선호
4. **점진적 도입**: 필요에 따라 단계적으로 도입

---

## 아키텍처 패턴

### 1. Layered Architecture (계층형 아키텍처)

우리 서비스의 기본 아키텍처입니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        API Layer (Presentation)                      │
│               GraphQL Resolvers, gRPC Servicers, REST Routers       │
├─────────────────────────────────────────────────────────────────────┤
│                        Service Layer (Application)                   │
│                    비즈니스 로직, Use Cases, Facades                  │
├─────────────────────────────────────────────────────────────────────┤
│                        Domain Layer (Business)                       │
│                 도메인 모델, 도메인 서비스, 비즈니스 규칙               │
├─────────────────────────────────────────────────────────────────────┤
│                    Infrastructure Layer (Persistence)                │
│              Repository 구현, 외부 서비스 연동, DB 접근               │
└─────────────────────────────────────────────────────────────────────┘
```

#### 레이어 규칙

| 레이어 | 의존 가능 대상 | 의존 불가 대상 |
|--------|---------------|---------------|
| API | Service, Domain | Infrastructure |
| Service | Domain, Infrastructure(interface) | - |
| Domain | 없음 (순수) | API, Service, Infrastructure |
| Infrastructure | Domain(interface) | API, Service |

### 2. Clean Architecture

Domain Layer를 중심으로 의존성이 안쪽으로만 향합니다.

```
                    ┌──────────────────────────────────┐
                    │         API / Frameworks          │
                    │    (FastAPI, gRPC, GraphQL)       │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │      Interface Adapters           │
                    │  (Controllers, Presenters, Gateways)│
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │       Application Services        │
                    │        (Use Cases)                │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │           Entities                │
                    │    (Domain Models, Business Rules) │
                    └──────────────────────────────────┘
```

### 3. Hexagonal Architecture (Ports & Adapters)

외부 시스템과의 연동을 추상화합니다.

```python
# Port (Interface) - Domain Layer
class OrderRepository(ABC):
    """주문 저장소 인터페이스 (Port)"""
    
    @abstractmethod
    async def save(self, order: Order) -> Order:
        pass
    
    @abstractmethod
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        pass


# Adapter - Infrastructure Layer
class PostgresOrderRepository(OrderRepository):
    """PostgreSQL 주문 저장소 구현 (Adapter)"""
    
    def __init__(self, session: AsyncSession):
        self._session = session
    
    async def save(self, order: Order) -> Order:
        db_order = self._to_model(order)
        self._session.add(db_order)
        await self._session.commit()
        return self._to_domain(db_order)
    
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        result = await self._session.execute(
            select(OrderModel).where(OrderModel.id == order_id)
        )
        db_order = result.scalar_one_or_none()
        return self._to_domain(db_order) if db_order else None


# 다른 Adapter로 교체 가능
class MongoOrderRepository(OrderRepository):
    """MongoDB 주문 저장소 구현 (Adapter)"""
    pass
```

---

## 도메인 주도 설계 (DDD)

### 핵심 개념

#### 1. Entity (엔티티)

고유 식별자를 가지며, 생명주기 동안 상태가 변할 수 있는 객체입니다.

```python
from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4


@dataclass
class Order:
    """주문 엔티티"""
    
    id: UUID = field(default_factory=uuid4)
    user_id: UUID
    items: list["OrderItem"] = field(default_factory=list)
    status: "OrderStatus" = OrderStatus.PENDING
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)
    
    def __eq__(self, other):
        """엔티티는 ID로 동등성 비교"""
        if not isinstance(other, Order):
            return False
        return self.id == other.id
    
    def __hash__(self):
        return hash(self.id)
```

#### 2. Value Object (값 객체)

식별자 없이 속성 값으로만 정의되는 불변 객체입니다.

```python
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)  # 불변
class Money:
    """금액 값 객체"""
    
    amount: int  # 센트/원 단위
    currency: str = "KRW"
    
    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")
    
    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)
    
    def multiply(self, factor: int) -> "Money":
        return Money(self.amount * factor, self.currency)


@dataclass(frozen=True)
class Address:
    """주소 값 객체"""
    
    street: str
    city: str
    postal_code: str
    country: str = "KR"
    detail: Optional[str] = None
    
    @property
    def full_address(self) -> str:
        parts = [self.street, self.city, self.postal_code, self.country]
        if self.detail:
            parts.insert(0, self.detail)
        return ", ".join(parts)
```

#### 3. Aggregate (집합체)

관련된 엔티티와 값 객체를 하나의 단위로 묶은 것입니다.

```python
@dataclass
class Order:
    """주문 Aggregate Root"""
    
    id: UUID
    user_id: UUID
    items: list["OrderItem"]  # Aggregate 내부 엔티티
    shipping_address: Address  # Value Object
    status: OrderStatus
    total_amount: Money  # Value Object
    
    # Aggregate 내부 비즈니스 로직
    def add_item(self, item: "OrderItem") -> None:
        """상품 추가"""
        if self.status != OrderStatus.PENDING:
            raise InvalidOperationError("Cannot add items to non-pending order")
        
        existing = next(
            (i for i in self.items if i.product_id == item.product_id),
            None
        )
        if existing:
            existing.increase_quantity(item.quantity)
        else:
            self.items.append(item)
        
        self._recalculate_total()
    
    def remove_item(self, product_id: UUID) -> None:
        """상품 제거"""
        if self.status != OrderStatus.PENDING:
            raise InvalidOperationError("Cannot remove items from non-pending order")
        
        self.items = [i for i in self.items if i.product_id != product_id]
        self._recalculate_total()
    
    def confirm(self) -> None:
        """주문 확정"""
        if self.status != OrderStatus.PENDING:
            raise InvalidOperationError(f"Cannot confirm order in {self.status} status")
        if not self.items:
            raise InvalidOperationError("Cannot confirm empty order")
        
        self.status = OrderStatus.CONFIRMED
        self._update_timestamp()
    
    def _recalculate_total(self) -> None:
        """총액 재계산"""
        total = sum(item.total_price.amount for item in self.items)
        self.total_amount = Money(total)
    
    def _update_timestamp(self) -> None:
        self.updated_at = datetime.utcnow()
```

#### 4. Domain Service (도메인 서비스)

엔티티나 값 객체에 속하지 않는 도메인 로직을 처리합니다.

```python
class OrderPricingService:
    """주문 가격 계산 도메인 서비스"""
    
    def calculate_final_price(
        self,
        order: Order,
        coupon: Optional[Coupon] = None,
        membership: Optional[Membership] = None,
    ) -> Money:
        """최종 가격 계산"""
        base_price = order.total_amount
        
        # 쿠폰 할인 적용
        if coupon and coupon.is_applicable(order):
            base_price = coupon.apply_discount(base_price)
        
        # 멤버십 할인 적용
        if membership:
            base_price = membership.apply_discount(base_price)
        
        # 배송비 추가
        shipping_fee = self._calculate_shipping_fee(order)
        
        return base_price.add(shipping_fee)
    
    def _calculate_shipping_fee(self, order: Order) -> Money:
        """배송비 계산"""
        if order.total_amount.amount >= 50000:  # 5만원 이상 무료배송
            return Money(0)
        return Money(3000)
```

#### 5. Domain Event (도메인 이벤트)

도메인에서 발생한 중요한 사건을 나타냅니다.

```python
from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4
from abc import ABC


@dataclass
class DomainEvent(ABC):
    """도메인 이벤트 기본 클래스"""
    
    event_id: UUID = field(default_factory=uuid4)
    timestamp: datetime = field(default_factory=datetime.utcnow)
    
    def get_topic(self) -> str:
        """Kafka 토픽 이름"""
        raise NotImplementedError
    
    def get_partition_key(self) -> str:
        """파티션 키"""
        raise NotImplementedError


@dataclass
class OrderCreatedEvent(DomainEvent):
    """주문 생성 이벤트"""
    
    order_id: UUID
    user_id: UUID
    total_amount: int
    items: list[dict]
    
    def get_topic(self) -> str:
        return "order.order.created"
    
    def get_partition_key(self) -> str:
        return str(self.order_id)


@dataclass
class OrderCancelledEvent(DomainEvent):
    """주문 취소 이벤트"""
    
    order_id: UUID
    user_id: UUID
    reason: str
    
    def get_topic(self) -> str:
        return "order.order.cancelled"
    
    def get_partition_key(self) -> str:
        return str(self.order_id)
```

---

## 생성 패턴

### 1. Factory Pattern (팩토리 패턴)

복잡한 객체 생성 로직을 캡슐화합니다.

```python
class OrderFactory:
    """주문 생성 팩토리"""
    
    @staticmethod
    def create(
        user_id: UUID,
        items: list[dict],
        shipping_address: dict,
    ) -> Order:
        """새 주문 생성"""
        order_items = [
            OrderItem(
                product_id=item["product_id"],
                product_name=item["product_name"],
                quantity=item["quantity"],
                unit_price=Money(item["price"]),
            )
            for item in items
        ]
        
        address = Address(
            street=shipping_address["street"],
            city=shipping_address["city"],
            postal_code=shipping_address["postal_code"],
            detail=shipping_address.get("detail"),
        )
        
        total = sum(i.total_price.amount for i in order_items)
        
        return Order(
            id=uuid4(),
            user_id=user_id,
            items=order_items,
            shipping_address=address,
            status=OrderStatus.PENDING,
            total_amount=Money(total),
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
        )
    
    @staticmethod
    def create_from_cart(user_id: UUID, cart: Cart, address: Address) -> Order:
        """장바구니에서 주문 생성"""
        order_items = [
            OrderItem(
                product_id=item.product_id,
                product_name=item.product_name,
                quantity=item.quantity,
                unit_price=item.unit_price,
            )
            for item in cart.items
        ]
        
        return Order(
            id=uuid4(),
            user_id=user_id,
            items=order_items,
            shipping_address=address,
            status=OrderStatus.PENDING,
            total_amount=cart.total_amount,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
        )
```

### 2. Builder Pattern (빌더 패턴)

복잡한 객체를 단계별로 구성합니다.

```python
class OrderBuilder:
    """주문 빌더"""
    
    def __init__(self):
        self._user_id: Optional[UUID] = None
        self._items: list[OrderItem] = []
        self._shipping_address: Optional[Address] = None
        self._coupon: Optional[Coupon] = None
    
    def for_user(self, user_id: UUID) -> "OrderBuilder":
        self._user_id = user_id
        return self
    
    def add_item(
        self,
        product_id: UUID,
        product_name: str,
        quantity: int,
        unit_price: int,
    ) -> "OrderBuilder":
        self._items.append(
            OrderItem(
                product_id=product_id,
                product_name=product_name,
                quantity=quantity,
                unit_price=Money(unit_price),
            )
        )
        return self
    
    def with_shipping_address(
        self,
        street: str,
        city: str,
        postal_code: str,
        detail: Optional[str] = None,
    ) -> "OrderBuilder":
        self._shipping_address = Address(
            street=street,
            city=city,
            postal_code=postal_code,
            detail=detail,
        )
        return self
    
    def with_coupon(self, coupon: Coupon) -> "OrderBuilder":
        self._coupon = coupon
        return self
    
    def build(self) -> Order:
        if not self._user_id:
            raise ValueError("User ID is required")
        if not self._items:
            raise ValueError("At least one item is required")
        if not self._shipping_address:
            raise ValueError("Shipping address is required")
        
        total = sum(i.total_price.amount for i in self._items)
        
        return Order(
            id=uuid4(),
            user_id=self._user_id,
            items=self._items,
            shipping_address=self._shipping_address,
            status=OrderStatus.PENDING,
            total_amount=Money(total),
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
        )


# 사용 예시
order = (
    OrderBuilder()
    .for_user(user_id)
    .add_item(product_id=..., product_name="상품A", quantity=2, unit_price=10000)
    .add_item(product_id=..., product_name="상품B", quantity=1, unit_price=20000)
    .with_shipping_address(
        street="테헤란로 123",
        city="서울시 강남구",
        postal_code="06234",
    )
    .build()
)
```

### 3. Singleton Pattern (싱글톤 패턴)

애플리케이션 전체에서 하나의 인스턴스만 사용합니다.

```python
from functools import lru_cache


class Settings:
    """설정 싱글톤"""
    
    _instance: Optional["Settings"] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance


# 또는 lru_cache 사용 (권장)
@lru_cache()
def get_settings() -> Settings:
    return Settings()


# 또는 module-level singleton
# config.py
settings = Settings()
```

---

## 구조 패턴

### 1. Repository Pattern (저장소 패턴)

데이터 접근 로직을 캡슐화합니다.

```python
# Interface (Domain Layer)
class OrderRepository(ABC):
    """주문 저장소 인터페이스"""
    
    @abstractmethod
    async def save(self, order: Order) -> Order:
        pass
    
    @abstractmethod
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        pass
    
    @abstractmethod
    async def get_by_user_id(
        self,
        user_id: UUID,
        status: Optional[OrderStatus] = None,
        limit: int = 20,
        offset: int = 0,
    ) -> list[Order]:
        pass
    
    @abstractmethod
    async def delete(self, order_id: UUID) -> bool:
        pass


# Implementation (Infrastructure Layer)
class SQLAlchemyOrderRepository(OrderRepository):
    """SQLAlchemy 주문 저장소"""
    
    def __init__(self, session: AsyncSession):
        self._session = session
    
    async def save(self, order: Order) -> Order:
        db_order = OrderMapper.to_model(order)
        self._session.add(db_order)
        await self._session.flush()
        return OrderMapper.to_domain(db_order)
    
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        result = await self._session.execute(
            select(OrderModel)
            .options(selectinload(OrderModel.items))
            .where(OrderModel.id == order_id)
        )
        db_order = result.scalar_one_or_none()
        return OrderMapper.to_domain(db_order) if db_order else None
```

### 2. Facade Pattern (파사드 패턴)

복잡한 서브시스템을 단순한 인터페이스로 제공합니다.

```python
class OrderFacade:
    """주문 파사드 - 주문 관련 복잡한 작업을 단순화"""
    
    def __init__(
        self,
        order_service: OrderService,
        payment_service: PaymentService,
        inventory_service: InventoryService,
        notification_service: NotificationService,
    ):
        self._order_service = order_service
        self._payment_service = payment_service
        self._inventory_service = inventory_service
        self._notification_service = notification_service
    
    async def place_order(
        self,
        user_id: UUID,
        cart_id: UUID,
        payment_method_id: UUID,
        shipping_address_id: UUID,
    ) -> OrderResult:
        """주문 접수 - 복잡한 프로세스를 하나의 메서드로 단순화"""
        
        # 1. 주문 생성
        order = await self._order_service.create_from_cart(
            user_id=user_id,
            cart_id=cart_id,
            shipping_address_id=shipping_address_id,
        )
        
        try:
            # 2. 재고 예약
            await self._inventory_service.reserve_stock(order)
            
            # 3. 결제 처리
            payment = await self._payment_service.process_payment(
                order_id=order.id,
                payment_method_id=payment_method_id,
                amount=order.total_amount,
            )
            
            # 4. 주문 확정
            order = await self._order_service.confirm(order.id)
            
            # 5. 알림 발송
            await self._notification_service.send_order_confirmation(order)
            
            return OrderResult(success=True, order=order, payment=payment)
            
        except PaymentFailedError as e:
            # 보상 트랜잭션
            await self._inventory_service.release_stock(order)
            await self._order_service.cancel(order.id, reason=str(e))
            
            return OrderResult(success=False, error=str(e))
```

### 3. Adapter Pattern (어댑터 패턴)

호환되지 않는 인터페이스를 연결합니다.

```python
# 외부 결제 API (호환되지 않는 인터페이스)
class ExternalPaymentAPI:
    def charge(self, card_token: str, amount_in_cents: int) -> dict:
        # 외부 API 호출
        pass


# 우리 도메인의 인터페이스
class PaymentGateway(ABC):
    @abstractmethod
    async def process_payment(
        self,
        payment_method_id: UUID,
        amount: Money,
    ) -> PaymentResult:
        pass


# Adapter
class ExternalPaymentAdapter(PaymentGateway):
    """외부 결제 API 어댑터"""
    
    def __init__(self, api: ExternalPaymentAPI, token_service: TokenService):
        self._api = api
        self._token_service = token_service
    
    async def process_payment(
        self,
        payment_method_id: UUID,
        amount: Money,
    ) -> PaymentResult:
        # 1. 우리 시스템의 payment_method_id를 외부 API의 card_token으로 변환
        card_token = await self._token_service.get_card_token(payment_method_id)
        
        # 2. 금액 단위 변환
        amount_in_cents = amount.to_cents()
        
        # 3. 외부 API 호출
        result = self._api.charge(card_token, amount_in_cents)
        
        # 4. 응답을 우리 도메인 객체로 변환
        return PaymentResult(
            transaction_id=result["transaction_id"],
            status=self._map_status(result["status"]),
            amount=amount,
        )
    
    def _map_status(self, external_status: str) -> PaymentStatus:
        mapping = {
            "success": PaymentStatus.COMPLETED,
            "pending": PaymentStatus.PENDING,
            "failed": PaymentStatus.FAILED,
        }
        return mapping.get(external_status, PaymentStatus.UNKNOWN)
```

### 4. Decorator Pattern (데코레이터 패턴)

기존 객체에 동적으로 기능을 추가합니다.

```python
# 기본 인터페이스
class OrderRepository(ABC):
    @abstractmethod
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        pass


# 기본 구현
class PostgresOrderRepository(OrderRepository):
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        # DB 조회
        pass


# 캐싱 데코레이터
class CachedOrderRepository(OrderRepository):
    """캐싱 기능이 추가된 주문 저장소"""
    
    def __init__(self, repository: OrderRepository, cache: Redis):
        self._repository = repository
        self._cache = cache
    
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        # 1. 캐시 확인
        cache_key = f"order:{order_id}"
        cached = await self._cache.get(cache_key)
        
        if cached:
            return Order.from_dict(json.loads(cached))
        
        # 2. DB 조회
        order = await self._repository.get_by_id(order_id)
        
        # 3. 캐시 저장
        if order:
            await self._cache.set(
                cache_key,
                json.dumps(order.to_dict()),
                ex=3600,  # 1시간
            )
        
        return order


# 로깅 데코레이터
class LoggingOrderRepository(OrderRepository):
    """로깅 기능이 추가된 주문 저장소"""
    
    def __init__(self, repository: OrderRepository, logger: Logger):
        self._repository = repository
        self._logger = logger
    
    async def get_by_id(self, order_id: UUID) -> Optional[Order]:
        self._logger.info(f"Getting order: {order_id}")
        start = time.time()
        
        result = await self._repository.get_by_id(order_id)
        
        elapsed = time.time() - start
        self._logger.info(
            f"Got order: {order_id}, found={result is not None}, elapsed={elapsed:.3f}s"
        )
        
        return result


# 조합하여 사용
repository = LoggingOrderRepository(
    CachedOrderRepository(
        PostgresOrderRepository(session),
        redis,
    ),
    logger,
)
```

---

## 행동 패턴

### 1. Strategy Pattern (전략 패턴)

알고리즘을 런타임에 교체할 수 있게 합니다.

```python
# 전략 인터페이스
class DiscountStrategy(ABC):
    """할인 전략 인터페이스"""
    
    @abstractmethod
    def calculate_discount(self, order: Order) -> Money:
        pass


# 구체적인 전략들
class PercentageDiscount(DiscountStrategy):
    """퍼센트 할인"""
    
    def __init__(self, percentage: int):
        self._percentage = percentage
    
    def calculate_discount(self, order: Order) -> Money:
        discount = order.total_amount.amount * self._percentage // 100
        return Money(discount)


class FixedAmountDiscount(DiscountStrategy):
    """정액 할인"""
    
    def __init__(self, amount: int):
        self._amount = amount
    
    def calculate_discount(self, order: Order) -> Money:
        return Money(min(self._amount, order.total_amount.amount))


class TieredDiscount(DiscountStrategy):
    """구간별 할인"""
    
    def __init__(self, tiers: list[tuple[int, int]]):
        # [(50000, 5), (100000, 10), (200000, 15)]
        # 5만원 이상 5%, 10만원 이상 10%, 20만원 이상 15%
        self._tiers = sorted(tiers, reverse=True)
    
    def calculate_discount(self, order: Order) -> Money:
        amount = order.total_amount.amount
        for threshold, percentage in self._tiers:
            if amount >= threshold:
                return Money(amount * percentage // 100)
        return Money(0)


# Context
class Coupon:
    """쿠폰 (전략 사용)"""
    
    def __init__(self, code: str, strategy: DiscountStrategy):
        self.code = code
        self._strategy = strategy
    
    def apply(self, order: Order) -> Money:
        return self._strategy.calculate_discount(order)


# 사용 예시
coupon = Coupon("SUMMER20", PercentageDiscount(20))
discount = coupon.apply(order)
```

### 2. Observer Pattern (옵저버 패턴)

이벤트 기반으로 객체 간 통신합니다.

```python
from typing import Callable, Awaitable

EventHandler = Callable[[DomainEvent], Awaitable[None]]


class EventBus:
    """이벤트 버스"""
    
    def __init__(self):
        self._handlers: dict[type, list[EventHandler]] = defaultdict(list)
    
    def subscribe(self, event_type: type, handler: EventHandler) -> None:
        """이벤트 구독"""
        self._handlers[event_type].append(handler)
    
    def unsubscribe(self, event_type: type, handler: EventHandler) -> None:
        """구독 취소"""
        self._handlers[event_type].remove(handler)
    
    async def publish(self, event: DomainEvent) -> None:
        """이벤트 발행"""
        handlers = self._handlers.get(type(event), [])
        await asyncio.gather(
            *[handler(event) for handler in handlers]
        )


# 핸들러
async def send_order_confirmation_email(event: OrderCreatedEvent) -> None:
    """주문 생성 시 확인 이메일 발송"""
    await email_service.send(
        to=event.user_email,
        subject="주문이 완료되었습니다",
        body=f"주문번호: {event.order_id}",
    )


async def update_inventory(event: OrderCreatedEvent) -> None:
    """주문 생성 시 재고 차감"""
    for item in event.items:
        await inventory_service.decrease(
            product_id=item["product_id"],
            quantity=item["quantity"],
        )


# 등록
event_bus = EventBus()
event_bus.subscribe(OrderCreatedEvent, send_order_confirmation_email)
event_bus.subscribe(OrderCreatedEvent, update_inventory)

# 발행
await event_bus.publish(OrderCreatedEvent(...))
```

### 3. Command Pattern (커맨드 패턴)

요청을 객체로 캡슐화합니다.

```python
from dataclasses import dataclass
from abc import ABC, abstractmethod


# Command Interface
class Command(ABC):
    @abstractmethod
    async def execute(self) -> Any:
        pass


# Concrete Commands
@dataclass
class CreateOrderCommand(Command):
    """주문 생성 커맨드"""
    
    user_id: UUID
    items: list[dict]
    shipping_address_id: UUID
    
    async def execute(self) -> Order:
        # 주문 생성 로직
        pass


@dataclass
class CancelOrderCommand(Command):
    """주문 취소 커맨드"""
    
    order_id: UUID
    user_id: UUID
    reason: str
    
    async def execute(self) -> Order:
        # 주문 취소 로직
        pass


# Command Handler
class CommandHandler:
    """커맨드 핸들러"""
    
    def __init__(self):
        self._handlers: dict[type, Callable] = {}
    
    def register(self, command_type: type, handler: Callable) -> None:
        self._handlers[command_type] = handler
    
    async def handle(self, command: Command) -> Any:
        handler = self._handlers.get(type(command))
        if not handler:
            raise ValueError(f"No handler for {type(command)}")
        return await handler(command)


# 사용
handler = CommandHandler()
handler.register(CreateOrderCommand, create_order_handler)
handler.register(CancelOrderCommand, cancel_order_handler)

result = await handler.handle(CreateOrderCommand(...))
```

### 4. Chain of Responsibility (책임 연쇄 패턴)

요청을 처리할 수 있는 핸들러 체인을 구성합니다.

```python
class OrderValidationHandler(ABC):
    """주문 검증 핸들러"""
    
    def __init__(self):
        self._next: Optional["OrderValidationHandler"] = None
    
    def set_next(self, handler: "OrderValidationHandler") -> "OrderValidationHandler":
        self._next = handler
        return handler
    
    async def handle(self, order: Order) -> ValidationResult:
        result = await self._validate(order)
        
        if not result.is_valid:
            return result
        
        if self._next:
            return await self._next.handle(order)
        
        return result
    
    @abstractmethod
    async def _validate(self, order: Order) -> ValidationResult:
        pass


class StockValidationHandler(OrderValidationHandler):
    """재고 검증"""
    
    async def _validate(self, order: Order) -> ValidationResult:
        for item in order.items:
            stock = await inventory_service.check_stock(item.product_id)
            if stock < item.quantity:
                return ValidationResult(
                    is_valid=False,
                    error=f"재고 부족: {item.product_name}",
                )
        return ValidationResult(is_valid=True)


class PaymentMethodValidationHandler(OrderValidationHandler):
    """결제 수단 검증"""
    
    async def _validate(self, order: Order) -> ValidationResult:
        payment_method = await payment_service.get_method(order.payment_method_id)
        if not payment_method or not payment_method.is_active:
            return ValidationResult(
                is_valid=False,
                error="유효하지 않은 결제 수단입니다",
            )
        return ValidationResult(is_valid=True)


class AddressValidationHandler(OrderValidationHandler):
    """배송 주소 검증"""
    
    async def _validate(self, order: Order) -> ValidationResult:
        if not order.shipping_address.is_deliverable:
            return ValidationResult(
                is_valid=False,
                error="배송 불가 지역입니다",
            )
        return ValidationResult(is_valid=True)


# 체인 구성
stock_handler = StockValidationHandler()
payment_handler = PaymentMethodValidationHandler()
address_handler = AddressValidationHandler()

stock_handler.set_next(payment_handler).set_next(address_handler)

# 검증 실행
result = await stock_handler.handle(order)
```

---

## 분산 시스템 패턴

### 1. Saga Pattern

분산 트랜잭션을 관리합니다.

```python
from enum import Enum
from dataclasses import dataclass


class SagaState(Enum):
    STARTED = "STARTED"
    STEP_COMPLETED = "STEP_COMPLETED"
    COMPLETED = "COMPLETED"
    COMPENSATING = "COMPENSATING"
    COMPENSATED = "COMPENSATED"
    FAILED = "FAILED"


@dataclass
class SagaStep:
    """Saga 단계"""
    name: str
    action: Callable
    compensation: Callable


class OrderSaga:
    """주문 Saga"""
    
    def __init__(
        self,
        order_service: OrderService,
        inventory_service: InventoryService,
        payment_service: PaymentService,
    ):
        self._steps = [
            SagaStep(
                name="create_order",
                action=self._create_order,
                compensation=self._cancel_order,
            ),
            SagaStep(
                name="reserve_inventory",
                action=self._reserve_inventory,
                compensation=self._release_inventory,
            ),
            SagaStep(
                name="process_payment",
                action=self._process_payment,
                compensation=self._refund_payment,
            ),
            SagaStep(
                name="confirm_order",
                action=self._confirm_order,
                compensation=lambda: None,  # 마지막 단계는 보상 불필요
            ),
        ]
        self._completed_steps: list[SagaStep] = []
    
    async def execute(self, order_data: dict) -> SagaResult:
        """Saga 실행"""
        context = {"order_data": order_data}
        
        for step in self._steps:
            try:
                result = await step.action(context)
                context.update(result)
                self._completed_steps.append(step)
                
            except Exception as e:
                # 보상 트랜잭션 실행
                await self._compensate(context)
                return SagaResult(success=False, error=str(e))
        
        return SagaResult(success=True, order=context["order"])
    
    async def _compensate(self, context: dict) -> None:
        """보상 트랜잭션 - 역순으로 실행"""
        for step in reversed(self._completed_steps):
            try:
                await step.compensation(context)
            except Exception as e:
                logger.error(f"Compensation failed for {step.name}: {e}")
    
    async def _create_order(self, context: dict) -> dict:
        order = await self._order_service.create(context["order_data"])
        return {"order": order}
    
    async def _cancel_order(self, context: dict) -> None:
        await self._order_service.cancel(context["order"].id)
    
    # ... 나머지 단계들
```

### 2. Circuit Breaker Pattern

장애 전파를 방지합니다.

```python
import asyncio
from datetime import datetime, timedelta
from enum import Enum


class CircuitState(Enum):
    CLOSED = "CLOSED"      # 정상 - 요청 허용
    OPEN = "OPEN"          # 차단 - 요청 거부
    HALF_OPEN = "HALF_OPEN"  # 테스트 - 일부 요청 허용


class CircuitBreaker:
    """서킷 브레이커"""
    
    def __init__(
        self,
        failure_threshold: int = 5,
        success_threshold: int = 3,
        timeout: float = 30.0,
    ):
        self._failure_threshold = failure_threshold
        self._success_threshold = success_threshold
        self._timeout = timeout
        
        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._success_count = 0
        self._last_failure_time: Optional[datetime] = None
    
    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """보호된 함수 호출"""
        if self._state == CircuitState.OPEN:
            if self._should_try_reset():
                self._state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenError("Circuit is open")
        
        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
            
        except Exception as e:
            self._on_failure()
            raise
    
    def _should_try_reset(self) -> bool:
        """타임아웃 후 half-open 상태로 전환할지 확인"""
        if self._last_failure_time is None:
            return False
        return datetime.utcnow() - self._last_failure_time > timedelta(seconds=self._timeout)
    
    def _on_success(self) -> None:
        """성공 시 처리"""
        if self._state == CircuitState.HALF_OPEN:
            self._success_count += 1
            if self._success_count >= self._success_threshold:
                self._state = CircuitState.CLOSED
                self._reset_counts()
        else:
            self._reset_counts()
    
    def _on_failure(self) -> None:
        """실패 시 처리"""
        self._failure_count += 1
        self._last_failure_time = datetime.utcnow()
        
        if self._state == CircuitState.HALF_OPEN:
            self._state = CircuitState.OPEN
            self._reset_counts()
        elif self._failure_count >= self._failure_threshold:
            self._state = CircuitState.OPEN
    
    def _reset_counts(self) -> None:
        self._failure_count = 0
        self._success_count = 0


# 사용 예시
payment_circuit = CircuitBreaker(failure_threshold=5, timeout=30)

async def process_payment(order: Order) -> PaymentResult:
    return await payment_circuit.call(
        payment_gateway.charge,
        order.payment_method_id,
        order.total_amount,
    )
```

### 3. Retry Pattern

일시적인 오류에 대해 재시도합니다.

```python
import asyncio
import random
from typing import Type


class RetryConfig:
    """재시도 설정"""
    
    def __init__(
        self,
        max_retries: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        exponential_base: float = 2.0,
        jitter: bool = True,
        retryable_exceptions: tuple[Type[Exception], ...] = (Exception,),
    ):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter
        self.retryable_exceptions = retryable_exceptions


async def retry_with_backoff(
    func: Callable,
    config: RetryConfig = RetryConfig(),
    *args,
    **kwargs,
) -> Any:
    """지수 백오프로 재시도"""
    
    last_exception = None
    
    for attempt in range(config.max_retries + 1):
        try:
            return await func(*args, **kwargs)
            
        except config.retryable_exceptions as e:
            last_exception = e
            
            if attempt == config.max_retries:
                break
            
            # 지수 백오프 딜레이 계산
            delay = min(
                config.base_delay * (config.exponential_base ** attempt),
                config.max_delay,
            )
            
            # 지터 추가 (동시 재시도 방지)
            if config.jitter:
                delay *= (0.5 + random.random())
            
            logger.warning(
                f"Retry {attempt + 1}/{config.max_retries}",
                delay=delay,
                error=str(e),
            )
            
            await asyncio.sleep(delay)
    
    raise last_exception


# 데코레이터 버전
def with_retry(config: RetryConfig = RetryConfig()):
    def decorator(func: Callable):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            return await retry_with_backoff(func, config, *args, **kwargs)
        return wrapper
    return decorator


# 사용
@with_retry(RetryConfig(max_retries=3, retryable_exceptions=(ConnectionError,)))
async def call_external_api():
    pass
```

---

## 데이터베이스 패턴

### Database per Service (서비스별 데이터베이스)

MSA에서 권장되는 패턴으로, 각 서비스가 자신의 데이터베이스를 독립적으로 소유합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Commerce Platform                             │
├─────────────┬─────────────┬─────────────┬─────────────┬────────────┤
│   User      │   Product   │   Order     │   Payment   │  Inventory │
│  Service    │  Service    │  Service    │  Service    │  Service   │
├─────────────┼─────────────┼─────────────┼─────────────┼────────────┤
│  PostgreSQL │  PostgreSQL │  PostgreSQL │  PostgreSQL │   Redis    │
│  (users)    │  (products) │  (orders)   │  (payments) │  (stock)   │
│             │             │             │             │            │
│  + Redis    │  + ES       │  + Redis    │             │  + Postgres│
│  (session)  │  (search)   │  (cache)    │             │  (history) │
└─────────────┴─────────────┴─────────────┴─────────────┴────────────┘
```

#### 장점

| 장점 | 설명 |
|------|------|
| **서비스 독립성** | 각 서비스가 독립적으로 개발, 배포, 확장 가능 |
| **기술 선택의 자유** | 서비스별로 최적의 DB 선택 가능 (PostgreSQL, MongoDB, Redis 등) |
| **장애 격리** | 한 DB 장애가 다른 서비스에 영향 없음 |
| **확장성** | 서비스별 독립 확장 가능 |
| **스키마 자유** | 다른 서비스 영향 없이 스키마 변경 가능 |

#### 단점 및 해결책

| 단점 | 해결책 |
|------|--------|
| **분산 트랜잭션** | Saga 패턴 사용 |
| **데이터 일관성** | Eventual Consistency 허용 |
| **조인 불가** | API Composition 또는 CQRS |
| **운영 복잡도** | IaC, 자동화된 운영 도구 |

#### 서비스별 권장 DB

| 서비스 | 주 DB | 보조 DB | 이유 |
|--------|-------|---------|------|
| User Service | PostgreSQL | Redis | 관계형 데이터, 세션 캐싱 |
| Product Service | PostgreSQL | Elasticsearch | 상품 정보, 전문 검색 |
| Order Service | PostgreSQL | Redis | 트랜잭션 중요, 캐싱 |
| Payment Service | PostgreSQL | - | 트랜잭션, 감사 로그 중요 |
| Inventory Service | Redis | PostgreSQL | 빠른 재고 확인, 이력 보관 |
| Search Service | Elasticsearch | - | 전문 검색 특화 |
| Notification Service | MongoDB | Redis | 유연한 스키마, 큐 |

### 데이터 동기화 전략

```python
# 이벤트 기반 데이터 동기화
class ProductUpdatedEventHandler:
    """상품 업데이트 이벤트 처리 - Search Service"""
    
    def __init__(self, es_client: AsyncElasticsearch):
        self._es = es_client
    
    async def handle(self, event: ProductUpdatedEvent) -> None:
        """상품 정보를 Elasticsearch에 동기화"""
        await self._es.update(
            index="products",
            id=str(event.product_id),
            body={
                "doc": {
                    "name": event.name,
                    "description": event.description,
                    "price": event.price,
                    "updated_at": event.timestamp.isoformat(),
                }
            },
        )
```

---

## 적용 가이드

### 패턴 선택 매트릭스

| 상황 | 권장 패턴 |
|------|----------|
| 객체 생성이 복잡함 | Factory, Builder |
| 데이터 접근 추상화 | Repository |
| 외부 API 연동 | Adapter |
| 알고리즘 교체 필요 | Strategy |
| 이벤트 기반 통신 | Observer |
| 분산 트랜잭션 | Saga |
| 장애 전파 방지 | Circuit Breaker |
| 일시적 오류 처리 | Retry |

### 레이어별 패턴 적용

| 레이어 | 적용 패턴 |
|--------|----------|
| API | Facade, Adapter |
| Service | Strategy, Command, Observer |
| Domain | Factory, Builder, Entity, Value Object, Aggregate |
| Infrastructure | Repository, Adapter, Decorator |

---

## 참고 자료

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/)
- [Microservices Patterns by Chris Richardson](https://microservices.io/patterns/)
- [Design Patterns: GoF](https://en.wikipedia.org/wiki/Design_Patterns)

---

> 📅 **최종 업데이트**: 2025-12-03  
> ✍️ **작성자**: Backend Team

