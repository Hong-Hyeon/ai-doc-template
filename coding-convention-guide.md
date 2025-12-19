# Commerce Platform - Coding Convention Guide

## 📋 목차

1. [개요](#개요)
2. [Python 코딩 컨벤션](#python-코딩-컨벤션)
3. [프로젝트 구조 컨벤션](#프로젝트-구조-컨벤션)
4. [Git 컨벤션](#git-컨벤션)
5. [테스트 컨벤션](#테스트-컨벤션)
6. [로깅 컨벤션](#로깅-컨벤션)
7. [문서화 컨벤션](#문서화-컨벤션)

---

## 개요

본 문서는 Commerce Platform 개발 시 일관된 코드 품질을 유지하기 위한 코딩 컨벤션을 정의합니다.

### 기본 원칙

1. **일관성**: 팀 전체가 동일한 스타일을 사용
2. **가독성**: 코드는 쓰는 것보다 읽히는 경우가 더 많음
3. **명확성**: 의도가 명확하게 드러나는 코드 작성
4. **단순성**: 불필요한 복잡성 제거

### 도구

| 도구 | 용도 | 설정 파일 |
|------|------|----------|
| Black | 코드 포맷팅 | pyproject.toml |
| isort | import 정렬 | pyproject.toml |
| Ruff | Linting | pyproject.toml |
| mypy | Type checking | pyproject.toml |
| pre-commit | Git hooks | .pre-commit-config.yaml |

---

## Python 코딩 컨벤션

### 1. 기본 스타일

[PEP 8](https://pep8.org/)을 기본으로 하며, Black 포맷터를 사용합니다.

```toml
# pyproject.toml
[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | generated
  | migrations
)/
'''

[tool.isort]
profile = "black"
line_length = 88
known_first_party = ["app", "core"]
sections = ["FUTURE", "STDLIB", "THIRDPARTY", "FIRSTPARTY", "LOCALFOLDER"]

[tool.ruff]
line-length = 88
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # Pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
]
ignore = ["E501"]  # line too long (Black handles this)

[tool.mypy]
python_version = "3.11"
strict = true
ignore_missing_imports = true
```

### 2. 네이밍 컨벤션

```python
# 모듈명: snake_case
# order_service.py
# user_repository.py

# 클래스명: PascalCase
class OrderService:
    pass

class SQLAlchemyOrderRepository:
    pass

# 함수/메서드명: snake_case
def create_order():
    pass

async def get_user_by_id():
    pass

# 변수명: snake_case
user_id = "123"
total_amount = 10000
is_active = True

# 상수: SCREAMING_SNAKE_CASE
MAX_RETRY_COUNT = 3
DEFAULT_PAGE_SIZE = 20
API_VERSION = "v1"

# Private: 단일 underscore prefix
class OrderService:
    def __init__(self):
        self._repository = None  # private
    
    def _validate_order(self):  # private method
        pass

# Protected: 사용하지 않음 (Python에서는 관례적으로 _로 통일)

# Dunder: 특수 메서드만 사용
def __init__(self):
    pass

def __str__(self):
    pass
```

### 3. Type Hints

모든 함수/메서드에 Type Hint를 사용합니다.

```python
from typing import Optional, List, Dict, Any, TypeVar, Generic
from uuid import UUID
from datetime import datetime
from collections.abc import Sequence

# 기본 타입
def get_user(user_id: UUID) -> Optional[User]:
    pass

def create_users(users: List[UserCreate]) -> List[User]:
    pass

# 복잡한 타입
from typing import TypedDict

class OrderDict(TypedDict):
    id: str
    user_id: str
    total_amount: int

def process_order(order: OrderDict) -> bool:
    pass

# Generic 타입
T = TypeVar("T")

class Repository(Generic[T]):
    async def get_by_id(self, id: UUID) -> Optional[T]:
        pass
    
    async def save(self, entity: T) -> T:
        pass

# Callable
from typing import Callable, Awaitable

EventHandler = Callable[[Dict[str, Any]], Awaitable[None]]

def register_handler(handler: EventHandler) -> None:
    pass

# 반환 타입이 없는 경우
def log_message(message: str) -> None:
    pass

# 여러 반환 타입 (Union 대신 | 사용)
def parse_value(value: str) -> int | float | None:
    pass
```

### 4. Import 순서

```python
# 1. Standard library
from __future__ import annotations

import asyncio
import json
from datetime import datetime
from typing import Optional, List
from uuid import UUID

# 2. Third-party packages
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy.ext.asyncio import AsyncSession
import grpc

# 3. Local application
from app.core.config import settings
from app.domain.models.order import Order, OrderStatus
from app.domain.services.order_service import OrderService
from app.infrastructure.database.repositories import OrderRepository
```

### 5. 클래스 구조

```python
from dataclasses import dataclass
from typing import Optional, List
from uuid import UUID
from datetime import datetime


@dataclass
class Order:
    """
    주문 도메인 모델
    
    Attributes:
        id: 주문 고유 식별자
        user_id: 주문자 ID
        items: 주문 상품 목록
        status: 주문 상태
    """
    
    # 클래스 변수 (상수)
    MAX_ITEMS = 100
    
    # 인스턴스 변수
    id: UUID
    user_id: UUID
    items: List["OrderItem"]
    status: "OrderStatus"
    created_at: datetime
    updated_at: datetime
    
    # Properties
    @property
    def total_amount(self) -> int:
        """총 주문 금액"""
        return sum(item.total_price for item in self.items)
    
    @property
    def is_cancellable(self) -> bool:
        """취소 가능 여부"""
        return self.status in [OrderStatus.PENDING, OrderStatus.CONFIRMED]
    
    # Public 메서드
    def cancel(self) -> None:
        """주문 취소"""
        if not self.is_cancellable:
            raise ValueError(f"Cannot cancel order in {self.status} status")
        self.status = OrderStatus.CANCELLED
        self._update_timestamp()
    
    def confirm(self) -> None:
        """주문 확정"""
        if self.status != OrderStatus.PENDING:
            raise ValueError(f"Cannot confirm order in {self.status} status")
        self.status = OrderStatus.CONFIRMED
        self._update_timestamp()
    
    # Private 메서드
    def _update_timestamp(self) -> None:
        """타임스탬프 업데이트"""
        self.updated_at = datetime.utcnow()
    
    # Class 메서드
    @classmethod
    def create(
        cls,
        user_id: UUID,
        items: List["OrderItem"],
        shipping_address: str,
    ) -> "Order":
        """새 주문 생성"""
        now = datetime.utcnow()
        return cls(
            id=uuid4(),
            user_id=user_id,
            items=items,
            status=OrderStatus.PENDING,
            created_at=now,
            updated_at=now,
        )
    
    # Static 메서드
    @staticmethod
    def validate_items(items: List["OrderItem"]) -> bool:
        """주문 상품 유효성 검증"""
        return 0 < len(items) <= Order.MAX_ITEMS
```

### 6. 비동기 코드

```python
import asyncio
from typing import List, Optional
from uuid import UUID


# async/await 사용
async def get_order(order_id: UUID) -> Optional[Order]:
    """단일 주문 조회"""
    return await order_repository.get_by_id(order_id)


# 병렬 실행
async def get_order_with_details(order_id: UUID) -> OrderWithDetails:
    """주문과 상세 정보 병렬 조회"""
    order, user, products = await asyncio.gather(
        order_repository.get_by_id(order_id),
        user_service.get_user(order.user_id),
        product_service.get_products([item.product_id for item in order.items]),
    )
    return OrderWithDetails(order=order, user=user, products=products)


# 타임아웃 처리
async def get_order_with_timeout(order_id: UUID) -> Optional[Order]:
    """타임아웃이 있는 주문 조회"""
    try:
        return await asyncio.wait_for(
            order_repository.get_by_id(order_id),
            timeout=5.0,
        )
    except asyncio.TimeoutError:
        logger.warning(f"Timeout getting order {order_id}")
        raise


# Context Manager
async def process_with_transaction():
    """트랜잭션 내에서 처리"""
    async with db_session.begin():
        await order_repository.save(order)
        await inventory_service.reserve_stock(order.items)
```

### 7. 예외 처리

```python
# 커스텀 예외 정의
class DomainException(Exception):
    """도메인 예외 기본 클래스"""
    
    def __init__(self, message: str, code: str = "DOMAIN_ERROR"):
        self.message = message
        self.code = code
        super().__init__(message)


class OrderNotFoundError(DomainException):
    """주문을 찾을 수 없음"""
    
    def __init__(self, order_id: UUID):
        super().__init__(
            message=f"Order {order_id} not found",
            code="ORDER_NOT_FOUND",
        )
        self.order_id = order_id


class InsufficientStockError(DomainException):
    """재고 부족"""
    
    def __init__(self, product_id: UUID, requested: int, available: int):
        super().__init__(
            message=f"Insufficient stock for product {product_id}",
            code="INSUFFICIENT_STOCK",
        )
        self.product_id = product_id
        self.requested = requested
        self.available = available


# 예외 처리 패턴
async def cancel_order(order_id: UUID, user_id: UUID) -> Order:
    """주문 취소"""
    try:
        order = await order_repository.get_by_id(order_id)
        if not order:
            raise OrderNotFoundError(order_id)
        
        if order.user_id != user_id:
            raise PermissionDeniedError("Cannot cancel other user's order")
        
        order.cancel()
        await order_repository.save(order)
        
        return order
        
    except DomainException:
        # 도메인 예외는 그대로 전파
        raise
    except Exception as e:
        # 예상치 못한 예외는 로깅 후 래핑
        logger.exception(f"Unexpected error cancelling order {order_id}")
        raise DomainException(
            message="Failed to cancel order",
            code="INTERNAL_ERROR",
        ) from e
```

### 8. Docstring

Google 스타일 Docstring을 사용합니다.

```python
def create_order(
    user_id: UUID,
    items: List[OrderItemInput],
    shipping_address_id: UUID,
    coupon_code: Optional[str] = None,
) -> Order:
    """주문을 생성합니다.
    
    사용자가 선택한 상품들로 새 주문을 생성하고,
    재고를 예약한 후 결제 대기 상태로 설정합니다.
    
    Args:
        user_id: 주문자의 사용자 ID
        items: 주문할 상품 목록
        shipping_address_id: 배송지 주소 ID
        coupon_code: 적용할 쿠폰 코드 (선택)
    
    Returns:
        생성된 주문 객체
    
    Raises:
        UserNotFoundError: 사용자를 찾을 수 없는 경우
        ProductNotFoundError: 상품을 찾을 수 없는 경우
        InsufficientStockError: 재고가 부족한 경우
        InvalidCouponError: 유효하지 않은 쿠폰인 경우
    
    Examples:
        >>> order = await create_order(
        ...     user_id=UUID("123..."),
        ...     items=[OrderItemInput(product_id=UUID("456..."), quantity=2)],
        ...     shipping_address_id=UUID("789..."),
        ... )
        >>> print(order.status)
        OrderStatus.PENDING
    """
    pass


class OrderService:
    """주문 도메인 서비스
    
    주문의 생성, 조회, 수정, 취소 등 주문 관련 비즈니스 로직을 처리합니다.
    
    Attributes:
        order_repository: 주문 저장소
        user_service: 사용자 서비스
        product_service: 상품 서비스
        inventory_service: 재고 서비스
        event_publisher: 이벤트 발행자
    
    Examples:
        >>> service = OrderService(
        ...     order_repository=repo,
        ...     user_service=user_svc,
        ...     ...
        ... )
        >>> order = await service.create_order(...)
    """
    pass
```

---

## 프로젝트 구조 컨벤션

### 1. 파일명 규칙

```
# 모듈 파일: snake_case
order_service.py
user_repository.py
kafka_producer.py

# 테스트 파일: test_ prefix
test_order_service.py
test_user_repository.py

# 설정 파일
config.py
settings.py

# __init__.py: 패키지 초기화 및 public API 정의
```

### 2. 디렉토리 구조 규칙

```
app/
├── domain/
│   ├── models/
│   │   ├── __init__.py     # from .order import Order, OrderItem 등
│   │   ├── order.py        # Order, OrderItem, OrderStatus
│   │   └── user.py         # User, UserRole
│   └── services/
│       ├── __init__.py     # from .order_service import OrderService 등
│       └── order_service.py
```

### 3. __init__.py 작성 규칙

```python
# app/domain/models/__init__.py
"""도메인 모델 모듈

이 모듈은 모든 도메인 모델을 포함합니다.
"""

from .order import Order, OrderItem, OrderStatus
from .user import User, UserRole
from .product import Product, ProductCategory

__all__ = [
    # Order
    "Order",
    "OrderItem", 
    "OrderStatus",
    # User
    "User",
    "UserRole",
    # Product
    "Product",
    "ProductCategory",
]
```

---

## Git 컨벤션

### 1. 브랜치 전략

```
main (production)
├── develop (development)
│   ├── feature/OD-123-add-order-creation
│   ├── feature/OD-124-add-payment-integration
│   └── feature/OD-125-add-inventory-check
├── release/v1.2.0
├── hotfix/OD-999-fix-payment-bug
```

#### 브랜치 네이밍

```
feature/{ticket-id}-{short-description}
bugfix/{ticket-id}-{short-description}
hotfix/{ticket-id}-{short-description}
release/{version}

예시:
feature/OD-123-add-order-cancellation
bugfix/OD-456-fix-stock-calculation
hotfix/OD-789-fix-payment-timeout
release/v1.2.0
```

### 2. Commit 메시지

[Conventional Commits](https://www.conventionalcommits.org/) 규칙을 따릅니다.

```
<type>(<scope>): <subject>

<body>

<footer>
```

#### Type

| Type | 설명 |
|------|------|
| feat | 새로운 기능 |
| fix | 버그 수정 |
| docs | 문서 변경 |
| style | 코드 포맷팅 (기능 변경 없음) |
| refactor | 리팩토링 (기능 변경 없음) |
| test | 테스트 추가/수정 |
| chore | 빌드, 설정 변경 |
| perf | 성능 개선 |

#### 예시

```
feat(order): add order cancellation feature

- Add cancel method to Order domain model
- Add cancel_order method to OrderService
- Add CancelOrder RPC to gRPC service
- Emit OrderCancelledEvent on cancellation

Closes OD-123

---

fix(inventory): fix stock calculation bug

Stock was not properly decremented when order contained
multiple items of the same product.

Fixes OD-456

---

refactor(order): extract order validation logic

Move validation logic from OrderService to Order domain model
to follow DDD principles.

No functional changes.
```

### 3. PR 규칙

#### PR 제목

```
[OD-123] feat(order): add order cancellation feature
[OD-456] fix(payment): fix timeout handling
```

#### PR 템플릿

```markdown
## 변경 사항
<!-- 무엇을 변경했는지 설명 -->

## 변경 이유
<!-- 왜 변경했는지 설명 -->

## 테스트
- [ ] 단위 테스트 추가/수정
- [ ] 통합 테스트 추가/수정
- [ ] 수동 테스트 완료

## 체크리스트
- [ ] 코드 리뷰 요청 준비 완료
- [ ] 문서 업데이트 완료 (필요한 경우)
- [ ] Breaking change 없음

## 스크린샷 (UI 변경 시)
<!-- 스크린샷 첨부 -->

## 관련 이슈
Closes #123
```

---

## 테스트 컨벤션

### 1. 테스트 구조

```
tests/
├── conftest.py              # 공통 fixtures
├── unit/                    # 단위 테스트
│   ├── domain/
│   │   ├── models/
│   │   │   └── test_order.py
│   │   └── services/
│   │       └── test_order_service.py
│   └── infrastructure/
│       └── test_order_repository.py
├── integration/             # 통합 테스트
│   ├── test_order_flow.py
│   └── test_payment_integration.py
└── e2e/                     # E2E 테스트
    └── test_checkout_flow.py
```

### 2. 테스트 네이밍

```python
# 파일명: test_{module_name}.py
# test_order_service.py

# 클래스명: Test{ClassName}
class TestOrderService:
    pass

# 메서드명: test_{method}_{scenario}_{expected_result}
class TestOrderService:
    async def test_create_order_with_valid_items_returns_order(self):
        pass
    
    async def test_create_order_with_empty_items_raises_error(self):
        pass
    
    async def test_cancel_order_when_pending_succeeds(self):
        pass
    
    async def test_cancel_order_when_shipped_raises_error(self):
        pass
```

### 3. 테스트 작성 패턴

```python
import pytest
from unittest.mock import AsyncMock, Mock
from uuid import uuid4

from app.domain.models.order import Order, OrderStatus
from app.domain.services.order_service import OrderService
from app.domain.exceptions import OrderNotFoundError, InsufficientStockError


class TestOrderService:
    """OrderService 테스트"""
    
    @pytest.fixture
    def order_repository(self):
        """Mock OrderRepository"""
        return AsyncMock()
    
    @pytest.fixture
    def user_service(self):
        """Mock UserService"""
        return AsyncMock()
    
    @pytest.fixture
    def inventory_service(self):
        """Mock InventoryService"""
        return AsyncMock()
    
    @pytest.fixture
    def order_service(
        self,
        order_repository,
        user_service,
        inventory_service,
    ):
        """OrderService 인스턴스"""
        return OrderService(
            order_repository=order_repository,
            user_service=user_service,
            inventory_service=inventory_service,
            event_publisher=AsyncMock(),
        )
    
    # Given-When-Then 패턴
    async def test_create_order_with_valid_items_returns_order(
        self,
        order_service: OrderService,
        user_service: AsyncMock,
        inventory_service: AsyncMock,
    ):
        """유효한 상품으로 주문 생성 시 주문이 반환되어야 함"""
        # Given
        user_id = uuid4()
        product_id = uuid4()
        
        user_service.get_user.return_value = Mock(id=user_id)
        inventory_service.check_stock.return_value = Mock(
            is_available=True,
            available_quantity=100,
        )
        
        items = [{"product_id": product_id, "quantity": 2}]
        
        # When
        order = await order_service.create_order(
            user_id=user_id,
            items=items,
            shipping_address="서울시 강남구",
        )
        
        # Then
        assert order is not None
        assert order.user_id == user_id
        assert order.status == OrderStatus.PENDING
        assert len(order.items) == 1
    
    async def test_create_order_with_insufficient_stock_raises_error(
        self,
        order_service: OrderService,
        user_service: AsyncMock,
        inventory_service: AsyncMock,
    ):
        """재고 부족 시 InsufficientStockError 발생해야 함"""
        # Given
        user_id = uuid4()
        product_id = uuid4()
        
        user_service.get_user.return_value = Mock(id=user_id)
        inventory_service.check_stock.return_value = Mock(
            is_available=False,
            available_quantity=1,
        )
        
        items = [{"product_id": product_id, "quantity": 10}]
        
        # When & Then
        with pytest.raises(InsufficientStockError) as exc_info:
            await order_service.create_order(
                user_id=user_id,
                items=items,
                shipping_address="서울시 강남구",
            )
        
        assert exc_info.value.product_id == product_id
    
    async def test_cancel_order_when_order_not_found_raises_error(
        self,
        order_service: OrderService,
        order_repository: AsyncMock,
    ):
        """존재하지 않는 주문 취소 시 OrderNotFoundError 발생해야 함"""
        # Given
        order_id = uuid4()
        user_id = uuid4()
        order_repository.get_by_id.return_value = None
        
        # When & Then
        with pytest.raises(OrderNotFoundError) as exc_info:
            await order_service.cancel_order(order_id, user_id)
        
        assert exc_info.value.order_id == order_id
```

### 4. Fixtures

```python
# tests/conftest.py
import pytest
import asyncio
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from testcontainers.postgres import PostgresContainer

from app.infrastructure.database.models import Base


@pytest.fixture(scope="session")
def event_loop():
    """이벤트 루프 설정"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session")
def postgres_container():
    """PostgreSQL 테스트 컨테이너"""
    with PostgresContainer("postgres:15") as postgres:
        yield postgres


@pytest.fixture(scope="session")
async def db_engine(postgres_container):
    """테스트용 DB 엔진"""
    url = postgres_container.get_connection_url().replace(
        "postgresql://", "postgresql+asyncpg://"
    )
    engine = create_async_engine(url)
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    yield engine
    
    await engine.dispose()


@pytest.fixture
async def db_session(db_engine) -> AsyncGenerator[AsyncSession, None]:
    """테스트용 DB 세션 (각 테스트마다 롤백)"""
    async with AsyncSession(db_engine) as session:
        yield session
        await session.rollback()


@pytest.fixture
def sample_user():
    """샘플 사용자 데이터"""
    return {
        "id": uuid4(),
        "email": "test@example.com",
        "name": "Test User",
    }


@pytest.fixture
def sample_product():
    """샘플 상품 데이터"""
    return {
        "id": uuid4(),
        "name": "Test Product",
        "price": 10000,
    }
```

---

## 로깅 컨벤션

### 1. 로깅 설정

```python
# app/core/logging.py
import logging
import sys
from typing import Any

import structlog
from structlog.types import Processor

from app.core.config import settings


def setup_logging() -> None:
    """로깅 설정"""
    
    # 공통 프로세서
    shared_processors: list[Processor] = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.dev.set_exc_info,
        structlog.processors.TimeStamper(fmt="iso"),
    ]
    
    if settings.ENVIRONMENT == "production":
        # Production: JSON 포맷
        processors = shared_processors + [
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),
        ]
    else:
        # Development: 컬러 콘솔 출력
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(colors=True),
        ]
    
    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.getLevelName(settings.LOG_LEVEL)
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )


# 로거 인스턴스
logger = structlog.get_logger()
```

### 2. 로깅 레벨 사용

```python
from app.core.logging import logger

# DEBUG: 개발 시 디버깅 정보
logger.debug("Processing order", order_id=order_id, items=items)

# INFO: 정상적인 비즈니스 이벤트
logger.info("Order created", order_id=order_id, user_id=user_id)

# WARNING: 비정상적이지만 처리 가능한 상황
logger.warning(
    "Retry attempt",
    order_id=order_id,
    attempt=attempt,
    max_attempts=max_attempts,
)

# ERROR: 에러 발생 (처리 실패)
logger.error(
    "Failed to create order",
    order_id=order_id,
    error=str(e),
    exc_info=True,
)

# CRITICAL: 시스템 레벨 심각한 에러
logger.critical(
    "Database connection lost",
    host=db_host,
    exc_info=True,
)
```

### 3. 구조화된 로깅

```python
# Context 바인딩
from structlog.contextvars import bind_contextvars, clear_contextvars

# 요청 시작 시 컨텍스트 설정
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid4()))
    
    bind_contextvars(
        request_id=request_id,
        path=request.url.path,
        method=request.method,
    )
    
    try:
        response = await call_next(request)
        return response
    finally:
        clear_contextvars()


# 서비스에서 추가 컨텍스트 바인딩
async def create_order(user_id: UUID, items: List[dict]) -> Order:
    bind_contextvars(user_id=str(user_id))
    
    logger.info("Creating order", item_count=len(items))
    
    # ... 비즈니스 로직 ...
    
    logger.info("Order created", order_id=str(order.id))
    return order
```

### 4. 로깅 출력 예시

```json
// Production (JSON)
{
  "event": "Order created",
  "level": "info",
  "timestamp": "2025-12-03T10:30:00.000Z",
  "request_id": "abc-123",
  "user_id": "user-456",
  "order_id": "order-789",
  "total_amount": 50000
}

// Development (Console)
2025-12-03T10:30:00.000Z [info     ] Order created    request_id=abc-123 user_id=user-456 order_id=order-789 total_amount=50000
```

---

## 문서화 컨벤션

### 1. 코드 주석

```python
# 단일 라인 주석: 코드 위에 작성
# 주문 상태가 PENDING인 경우에만 취소 가능
if order.status == OrderStatus.PENDING:
    order.cancel()

# TODO: 담당자와 함께 작성
# TODO(john): 결제 실패 시 재시도 로직 추가 필요

# FIXME: 수정이 필요한 부분
# FIXME: N+1 쿼리 문제 해결 필요

# NOTE: 중요한 참고 사항
# NOTE: 이 로직은 재고 서비스의 eventual consistency를 고려함

# HACK: 임시 해결책 (나중에 개선 필요)
# HACK: 외부 API 버그로 인한 우회 처리
```

### 2. README 구조

```markdown
# Service Name

간단한 서비스 설명

## 시작하기

### 요구사항

- Python 3.11+
- PostgreSQL 15+
- Redis 7+

### 설치

\`\`\`bash
poetry install
\`\`\`

### 환경 설정

\`\`\`bash
cp .env.example .env
# .env 파일 수정
\`\`\`

### 실행

\`\`\`bash
# 개발 서버
poetry run uvicorn app.main:app --reload

# gRPC 서버
poetry run python -m app.grpc.server
\`\`\`

## 프로젝트 구조

\`\`\`
app/
├── api/          # API 레이어
├── domain/       # 도메인 레이어
├── infrastructure/  # 인프라 레이어
└── core/         # 공통 모듈
\`\`\`

## API 문서

- GraphQL Playground: http://localhost:8000/graphql
- REST API Docs: http://localhost:8000/docs

## 테스트

\`\`\`bash
# 전체 테스트
poetry run pytest

# 커버리지
poetry run pytest --cov=app
\`\`\`

## 배포

배포 가이드는 [deployment-guide.md](./docs/deployment-guide.md) 참조
```

---

## 체크리스트

### 코드 작성 전

- [ ] 기능/버그에 대한 이슈/티켓 확인
- [ ] 관련 도메인 이해
- [ ] 기존 코드 스타일 확인

### 코드 작성 중

- [ ] Type Hint 작성
- [ ] Docstring 작성
- [ ] 적절한 예외 처리
- [ ] 로깅 추가

### 코드 작성 후

- [ ] 테스트 작성 및 통과
- [ ] Linter 통과 (ruff, mypy)
- [ ] 포맷팅 확인 (black, isort)
- [ ] Self 코드 리뷰

### PR 전

- [ ] Commit 메시지 컨벤션 확인
- [ ] PR 제목/설명 작성
- [ ] 리뷰어 지정

---

> 📅 **최종 업데이트**: 2025-12-03  
> ✍️ **작성자**: Backend Team

