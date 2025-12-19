# Debugging Guide

## 📋 목차

1. [개요](#개요)
2. [Import 관점 디버깅](#import-관점-디버깅)
3. [로직 관점 디버깅](#로직-관점-디버깅)
4. [Endpoint 관점 디버깅](#endpoint-관점-디버깅)
5. [아키텍처적 관점 디버깅](#아키텍처적-관점-디버깅)
6. [데이터베이스 관점 디버깅](#데이터베이스-관점-디버깅)
7. [비동기/동시성 관점 디버깅](#비동기동시성-관점-디버깅)
8. [환경/설정 관점 디버깅](#환경설정-관점-디버깅)
9. [캐시 관점 디버깅](#캐시-관점-디버깅)
10. [디버깅 도구 및 기법](#디버깅-도구-및-기법)
11. [체크리스트](#체크리스트)

---

## 개요

본 문서는 소프트웨어 개발 시 발생하는 문제를 체계적으로 디버깅하기 위한 가이드입니다. 
다양한 관점에서 문제를 접근하여 효율적으로 원인을 파악하고 해결할 수 있도록 구성되었습니다.

### 디버깅 기본 원칙

1. **체계적 접근**: 문제를 여러 관점에서 분석
2. **가설 설정**: 가능한 원인을 가설로 설정하고 검증
3. **증거 수집**: 로그, 에러 메시지, 상태 정보 등을 체계적으로 수집
4. **재현 가능성**: 문제를 재현할 수 있는 환경과 시나리오 확보
5. **점진적 축소**: 문제 범위를 점진적으로 좁혀가며 원인 파악

### 디버깅 프로세스

```
1. 문제 인식 및 재현
   ↓
2. 증상 관찰 및 정보 수집
   ↓
3. 가설 설정 (Import → 로직 → Endpoint → 아키텍처 → DB → 비동기 → 환경 → 캐시 순서)
   ↓
4. 검증 및 테스트
   ↓
5. 해결 및 검증
   ↓
6. 문서화 및 예방 조치
```

---

## Import 관점 디버깅

Import 문제는 코드 실행 전 단계에서 발생하는 문제로, 가장 먼저 확인해야 할 부분입니다.

### 1. Import 에러 유형

#### 1.1 ModuleNotFoundError

**증상**
```python
ModuleNotFoundError: No module named 'some_module'
```

**체크리스트**
- [ ] 모듈이 실제로 존재하는가?
- [ ] 경로가 올바른가? (상대 경로 vs 절대 경로)
- [ ] `__init__.py` 파일이 필요한 위치에 있는가?
- [ ] Python 경로(PYTHONPATH)에 모듈이 포함되어 있는가?
- [ ] 가상 환경이 활성화되어 있고 필요한 패키지가 설치되어 있는가?

**디버깅 방법**

```python
# 1. 모듈 경로 확인
import sys
print(sys.path)

# 2. 모듈 존재 여부 확인
import importlib.util
spec = importlib.util.find_spec("some_module")
print(spec.origin if spec else "Module not found")

# 3. 상대 경로 문제 확인
# 잘못된 예
from ..parent import something  # 상위 디렉토리로 이동 불가능한 경우

# 올바른 예
from app.parent import something  # 절대 경로 사용
```

#### 1.2 ImportError (순환 참조)

**증상**
```python
ImportError: cannot import name 'X' from partially initialized module 'Y'
```

**원인**
- 모듈 A가 모듈 B를 import하고, 모듈 B가 다시 모듈 A를 import하는 경우
- 모듈 레벨에서 서로 의존하는 코드 실행

**해결 방법**

```python
# 문제 코드
# module_a.py
from module_b import ClassB

class ClassA:
    def use_b(self):
        return ClassB()

# module_b.py
from module_a import ClassA  # 순환 참조!

class ClassB:
    def use_a(self):
        return ClassA()

# 해결 방법 1: 지연 import (Lazy Import)
# module_b.py
class ClassB:
    def use_a(self):
        from module_a import ClassA  # 함수 내부에서 import
        return ClassA()

# 해결 방법 2: 타입 힌트만 사용 (Python 3.7+)
# module_b.py
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from module_a import ClassA

class ClassB:
    def use_a(self) -> 'ClassA':
        from module_a import ClassA
        return ClassA()

# 해결 방법 3: 아키텍처 재설계
# 공통 인터페이스를 별도 모듈로 분리
```

#### 1.3 AttributeError (Import 후 속성 접근 실패)

**증상**
```python
AttributeError: module 'X' has no attribute 'Y'
```

**체크리스트**
- [ ] 모듈이 올바르게 import되었는가?
- [ ] 해당 속성/함수/클래스가 실제로 존재하는가?
- [ ] 이름이 정확히 일치하는가? (대소문자, 언더스코어 등)
- [ ] `__all__`에 의해 export가 제한되지 않았는가?

**디버깅 방법**

```python
# 1. 모듈 내용 확인
import some_module
print(dir(some_module))  # 모듈의 모든 속성 확인
print(some_module.__dict__)  # 모듈 딕셔너리 확인

# 2. __all__ 확인
print(some_module.__all__)  # export된 항목 확인

# 3. 타입 확인
import inspect
print(inspect.getmembers(some_module))
```

#### 1.4 패키지 버전 충돌

**증상**
- 예상과 다른 동작
- 타입 불일치
- 메서드/속성이 없다는 에러

**체크리스트**
- [ ] 여러 버전의 동일 패키지가 설치되어 있는가?
- [ ] 의존성 트리에 충돌이 있는가?
- [ ] 가상 환경이 격리되어 있는가?

**디버깅 방법**

```python
# 1. 패키지 버전 확인
import some_package
print(some_package.__version__)

# 2. 패키지 위치 확인
import some_package
print(some_package.__file__)

# 3. 설치된 패키지 목록 확인 (터미널)
# pip list | grep package_name
# pip show package_name
```

### 2. Import 디버깅 체크리스트

#### 2.1 기본 확인 사항

```
□ Python 버전이 요구사항과 일치하는가?
□ 가상 환경이 활성화되어 있는가?
□ requirements.txt / pyproject.toml의 패키지가 모두 설치되었는가?
□ IDE의 Python 인터프리터가 올바르게 설정되었는가?
```

#### 2.2 경로 관련 확인 사항

```
□ 프로젝트 루트가 PYTHONPATH에 포함되어 있는가?
□ 상대 경로가 올바른가? (.., . 의 사용)
□ __init__.py 파일이 필요한 모든 디렉토리에 있는가?
□ 패키지 구조가 올바른가?
```

#### 2.3 의존성 관련 확인 사항

```
□ 순환 참조가 없는가?
□ 의존성 버전이 호환되는가?
□ 의존성 충돌이 없는가?
□ 선택적 의존성(optional dependencies)이 필요한가?
```

### 3. Import 디버깅 도구

#### 3.1 Python 내장 도구

```python
# importlib를 사용한 동적 import 디버깅
import importlib
import importlib.util

# 모듈 로드 확인
try:
    module = importlib.import_module('some_module')
    print(f"Module loaded: {module.__file__}")
except ImportError as e:
    print(f"Import failed: {e}")

# 모듈 재로드 (개발 중 유용)
importlib.reload(module)
```

#### 3.2 외부 도구

```bash
# pipdeptree: 의존성 트리 확인
pip install pipdeptree
pipdeptree

# pip-check: 업데이트 가능한 패키지 확인
pip install pip-check
pip-check

# import-linter: import 규칙 검사
pip install import-linter
lint-imports
```

---

## 로직 관점 디버깅

로직 관점 디버깅은 코드의 실행 흐름과 비즈니스 로직에서 발생하는 문제를 찾는 것입니다.

### 1. 로직 에러 유형

#### 1.1 제어 흐름 문제

**증상**
- 예상과 다른 분기로 진입
- 루프가 예상과 다르게 동작
- 조건문이 항상 True/False

**디버깅 방법**

```python
# 문제 코드 예시
def process_order(order):
    if order.status == "pending":  # 항상 False?
        return handle_pending(order)
    elif order.status == "completed":
        return handle_completed(order)
    else:
        return handle_unknown(order)

# 디버깅 코드 추가
def process_order(order):
    # 1. 입력 값 확인
    print(f"DEBUG: order.status = {order.status}")
    print(f"DEBUG: type(order.status) = {type(order.status)}")
    print(f"DEBUG: order.status == 'pending' = {order.status == 'pending'}")
    
    # 2. 조건문 단계별 확인
    if order.status == "pending":
        print("DEBUG: Entering pending branch")
        return handle_pending(order)
    elif order.status == "completed":
        print("DEBUG: Entering completed branch")
        return handle_completed(order)
    else:
        print(f"DEBUG: Entering unknown branch with status: {order.status}")
        return handle_unknown(order)

# 개선된 버전: 타입 안전성 확보
def process_order(order):
    status = str(order.status).lower().strip()  # 정규화
    print(f"DEBUG: normalized status = '{status}'")
    
    if status == "pending":
        return handle_pending(order)
    elif status == "completed":
        return handle_completed(order)
    else:
        raise ValueError(f"Unknown order status: {order.status}")
```

#### 1.2 데이터 변환 문제

**증상**
- 데이터 타입 불일치
- None 값 처리 실패
- 데이터 포맷 오류

**디버깅 방법**

```python
# 문제 코드 예시
def calculate_total(items):
    total = 0
    for item in items:
        total += item.price * item.quantity
    return total

# 디버깅 코드 추가
def calculate_total(items):
    print(f"DEBUG: items type = {type(items)}")
    print(f"DEBUG: items = {items}")
    
    total = 0
    for idx, item in enumerate(items):
        print(f"DEBUG: item[{idx}] = {item}")
        print(f"DEBUG: item type = {type(item)}")
        print(f"DEBUG: item.price = {item.price}, type = {type(item.price)}")
        print(f"DEBUG: item.quantity = {item.quantity}, type = {type(item.quantity)}")
        
        # None 체크
        if item.price is None:
            print(f"WARNING: item[{idx}].price is None")
            continue
        if item.quantity is None:
            print(f"WARNING: item[{idx}].quantity is None")
            continue
        
        item_total = item.price * item.quantity
        print(f"DEBUG: item[{idx}] total = {item_total}")
        total += item_total
    
    print(f"DEBUG: final total = {total}")
    return total

# 개선된 버전: 방어적 프로그래밍
def calculate_total(items):
    if not items:
        return 0
    
    total = 0
    for item in items:
        try:
            price = float(item.price) if item.price is not None else 0
            quantity = int(item.quantity) if item.quantity is not None else 0
            total += price * quantity
        except (ValueError, TypeError) as e:
            logger.warning(f"Invalid item data: {item}, error: {e}")
            continue
    
    return total
```

#### 1.3 상태 관리 문제

**증상**
- 상태가 예상과 다름
- 상태 변경이 반영되지 않음
- 동시성 문제

**디버깅 방법**

```python
# 문제 코드 예시
class OrderProcessor:
    def __init__(self):
        self.processing = False
    
    def process(self, order):
        if self.processing:
            return "Already processing"
        self.processing = True
        # ... 처리 로직 ...
        self.processing = False

# 디버깅 코드 추가
class OrderProcessor:
    def __init__(self):
        self.processing = False
        self._state_history = []  # 상태 변경 이력 추적
    
    def _log_state(self, action, value):
        import time
        self._state_history.append({
            'time': time.time(),
            'action': action,
            'value': value,
            'stack': traceback.format_stack()
        })
        print(f"DEBUG: State change - {action}: {value}")
    
    def process(self, order):
        self._log_state("check_processing", self.processing)
        
        if self.processing:
            print("DEBUG: Already processing, returning early")
            return "Already processing"
        
        self._log_state("set_processing", True)
        self.processing = True
        
        try:
            # ... 처리 로직 ...
            result = self._do_process(order)
            return result
        finally:
            self._log_state("set_processing", False)
            self.processing = False
```

#### 1.4 예외 처리 문제

**증상**
- 예외가 잡히지 않음
- 잘못된 예외 처리로 인한 데이터 손실
- 예외 정보 손실

**디버깅 방법**

```python
# 문제 코드 예시
def process_data(data):
    try:
        result = risky_operation(data)
        return result
    except:
        return None  # 어떤 에러인지 알 수 없음

# 개선된 버전
def process_data(data):
    try:
        result = risky_operation(data)
        return result
    except SpecificError as e:
        logger.error(f"Specific error in process_data: {e}", exc_info=True)
        raise
    except ValueError as e:
        logger.warning(f"Invalid data format: {data}, error: {e}")
        return None
    except Exception as e:
        logger.error(f"Unexpected error in process_data: {e}", exc_info=True)
        raise RuntimeError(f"Failed to process data: {e}") from e
```

### 2. 로직 디버깅 전략

#### 2.1 단계별 실행 (Step-by-step)

```python
# 복잡한 로직을 단계별로 분리
def complex_operation(data):
    # Step 1: 입력 검증
    validated_data = validate_input(data)
    print(f"Step 1 - Validated: {validated_data}")
    
    # Step 2: 데이터 변환
    transformed_data = transform_data(validated_data)
    print(f"Step 2 - Transformed: {transformed_data}")
    
    # Step 3: 비즈니스 로직
    result = apply_business_logic(transformed_data)
    print(f"Step 3 - Result: {result}")
    
    # Step 4: 결과 검증
    validated_result = validate_result(result)
    print(f"Step 4 - Final: {validated_result}")
    
    return validated_result
```

#### 2.2 단위 테스트 작성

```python
# 문제를 재현하는 최소한의 테스트 작성
def test_problematic_case():
    # 문제가 발생하는 입력
    problematic_input = {...}
    
    # 예상 결과
    expected_result = {...}
    
    # 실제 실행
    actual_result = function_under_test(problematic_input)
    
    # 비교
    assert actual_result == expected_result, \
        f"Expected {expected_result}, got {actual_result}"
```

#### 2.3 로그 전략

```python
import logging

# 로그 레벨별 사용
logger = logging.getLogger(__name__)

def business_logic(data):
    logger.debug(f"Input data: {data}")  # 상세 정보
    
    if not data:
        logger.warning("Empty data received")  # 경고
        return None
    
    try:
        result = process(data)
        logger.info(f"Processed successfully: {result}")  # 정상 흐름
        return result
    except Exception as e:
        logger.error(f"Processing failed: {e}", exc_info=True)  # 에러
        raise
```

### 3. 로직 디버깅 체크리스트

```
□ 입력 값이 예상한 형식/범위인가?
□ 모든 분기 경로가 테스트되었는가?
□ 엣지 케이스(빈 값, None, 최대/최소 값)가 처리되는가?
□ 루프의 종료 조건이 올바른가?
□ 상태 변경이 예상대로 일어나는가?
□ 예외 상황이 적절히 처리되는가?
□ 동시성 문제가 없는가? (멀티스레드/비동기 환경)
□ 메모리 누수 가능성이 없는가?
```

---

## Endpoint 관점 디버깅

Endpoint 디버깅은 API 엔드포인트에서 발생하는 문제를 찾는 것입니다. 
요청부터 응답까지의 전체 흐름을 추적해야 합니다.

### 1. Endpoint 에러 유형

#### 1.1 요청 검증 실패

**증상**
- 400 Bad Request
- 422 Unprocessable Entity
- 검증 에러 메시지

**디버깅 방법**

```python
# FastAPI 예시
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
import logging

logger = logging.getLogger(__name__)

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    # 요청 정보 로깅
    logger.error(f"Validation error on {request.method} {request.url}")
    logger.error(f"Headers: {dict(request.headers)}")
    logger.error(f"Query params: {dict(request.query_params)}")
    
    # 요청 본문 로깅 (보안 주의)
    try:
        body = await request.body()
        logger.error(f"Request body: {body.decode()}")
    except Exception as e:
        logger.error(f"Could not read body: {e}")
    
    # 검증 에러 상세 정보
    logger.error(f"Validation errors: {exc.errors()}")
    
    return JSONResponse(
        status_code=422,
        content={"detail": exc.errors(), "body": body.decode() if body else None}
    )

# 요청 로깅 미들웨어
@app.middleware("http")
async def log_requests(request: Request, call_next):
    import time
    start_time = time.time()
    
    # 요청 정보 로깅
    logger.info(f"Request: {request.method} {request.url.path}")
    logger.debug(f"Query params: {dict(request.query_params)}")
    logger.debug(f"Headers: {dict(request.headers)}")
    
    response = await call_next(request)
    
    process_time = time.time() - start_time
    logger.info(f"Response: {response.status_code} - {process_time:.3f}s")
    
    return response
```

#### 1.2 인증/인가 실패

**증상**
- 401 Unauthorized
- 403 Forbidden
- 토큰 만료/무효

**디버깅 방법**

```python
# 인증 디버깅
def authenticate_request(request: Request):
    # 1. 헤더 확인
    auth_header = request.headers.get("Authorization")
    logger.debug(f"Auth header present: {auth_header is not None}")
    
    if not auth_header:
        logger.warning("No Authorization header")
        raise HTTPException(status_code=401, detail="Missing authorization")
    
    # 2. 토큰 형식 확인
    try:
        scheme, token = auth_header.split(" ", 1)
        logger.debug(f"Auth scheme: {scheme}, token length: {len(token)}")
    except ValueError:
        logger.error(f"Invalid auth header format: {auth_header}")
        raise HTTPException(status_code=401, detail="Invalid authorization format")
    
    # 3. 토큰 검증
    try:
        payload = verify_token(token)
        logger.debug(f"Token payload: {payload}")
        return payload
    except TokenExpiredError as e:
        logger.warning(f"Token expired: {e}")
        raise HTTPException(status_code=401, detail="Token expired")
    except InvalidTokenError as e:
        logger.error(f"Invalid token: {e}")
        raise HTTPException(status_code=401, detail="Invalid token")
```

#### 1.3 비즈니스 로직 에러

**증상**
- 500 Internal Server Error
- 예상과 다른 응답
- 데이터 불일치

**디버깅 방법**

```python
# 엔드포인트 전체 흐름 추적
@app.post("/orders")
async def create_order(order_data: OrderCreate, request: Request):
    request_id = request.state.request_id  # 요청 ID 추적
    
    logger.info(f"[{request_id}] Creating order: {order_data}")
    
    try:
        # Step 1: 입력 검증
        logger.debug(f"[{request_id}] Step 1: Validating input")
        validated_data = validate_order_data(order_data)
        logger.debug(f"[{request_id}] Validation passed: {validated_data}")
        
        # Step 2: 비즈니스 규칙 검증
        logger.debug(f"[{request_id}] Step 2: Checking business rules")
        business_errors = check_business_rules(validated_data)
        if business_errors:
            logger.warning(f"[{request_id}] Business rule violations: {business_errors}")
            raise HTTPException(status_code=400, detail=business_errors)
        
        # Step 3: 데이터베이스 작업
        logger.debug(f"[{request_id}] Step 3: Saving to database")
        order = await save_order(validated_data)
        logger.info(f"[{request_id}] Order created: {order.id}")
        
        # Step 4: 부가 작업 (이벤트 발행 등)
        logger.debug(f"[{request_id}] Step 4: Publishing events")
        await publish_order_created_event(order)
        
        return order
        
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"[{request_id}] Unexpected error: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")
```

#### 1.4 응답 형식 문제

**증상**
- 직렬화 에러
- 타입 불일치
- 누락된 필드

**디버깅 방법**

```python
# 응답 직렬화 디버깅
def serialize_response(data):
    try:
        # Pydantic 모델인 경우
        if hasattr(data, 'dict'):
            serialized = data.dict()
            logger.debug(f"Serialized data: {serialized}")
            return serialized
        
        # 일반 딕셔너리인 경우
        if isinstance(data, dict):
            # None 값 확인
            none_keys = [k for k, v in data.items() if v is None]
            if none_keys:
                logger.warning(f"None values in response: {none_keys}")
            return data
        
        # 기타 타입
        logger.warning(f"Unexpected response type: {type(data)}")
        return str(data)
        
    except Exception as e:
        logger.error(f"Serialization error: {e}", exc_info=True)
        raise
```

### 2. Endpoint 디버깅 전략

#### 2.1 요청/응답 로깅

```python
# 상세한 요청/응답 로깅 미들웨어
@app.middleware("http")
async def detailed_logging(request: Request, call_next):
    import uuid
    import json
    import time
    
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id
    
    start_time = time.time()
    
    # 요청 로깅
    log_data = {
        "request_id": request_id,
        "method": request.method,
        "url": str(request.url),
        "path": request.url.path,
        "query_params": dict(request.query_params),
        "headers": dict(request.headers),
    }
    
    # 본문 로깅 (크기 제한)
    try:
        body = await request.body()
        if len(body) < 10000:  # 10KB 제한
            log_data["body"] = body.decode()
        else:
            log_data["body_size"] = len(body)
    except Exception:
        pass
    
    logger.info(f"Request: {json.dumps(log_data, indent=2)}")
    
    # 응답 처리
    response = await call_next(request)
    
    process_time = time.time() - start_time
    
    # 응답 로깅
    response_log = {
        "request_id": request_id,
        "status_code": response.status_code,
        "process_time": f"{process_time:.3f}s",
    }
    
    logger.info(f"Response: {json.dumps(response_log, indent=2)}")
    
    return response
```

#### 2.2 에러 핸들링

```python
# 전역 에러 핸들러
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    request_id = getattr(request.state, 'request_id', 'unknown')
    
    logger.error(
        f"[{request_id}] Unhandled exception: {exc}",
        exc_info=True,
        extra={
            "request_id": request_id,
            "method": request.method,
            "url": str(request.url),
        }
    )
    
    # 프로덕션에서는 상세 정보 숨김
    if settings.DEBUG:
        return JSONResponse(
            status_code=500,
            content={
                "error": str(exc),
                "type": type(exc).__name__,
                "request_id": request_id,
            }
        )
    else:
        return JSONResponse(
            status_code=500,
            content={
                "error": "Internal server error",
                "request_id": request_id,
            }
        )
```

#### 2.3 성능 모니터링

```python
# 성능 추적 데코레이터
import time
from functools import wraps

def track_performance(threshold_seconds=1.0):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            start_time = time.time()
            
            try:
                result = await func(*args, **kwargs)
                return result
            finally:
                elapsed = time.time() - start_time
                
                if elapsed > threshold_seconds:
                    logger.warning(
                        f"Slow endpoint: {func.__name__} took {elapsed:.3f}s "
                        f"(threshold: {threshold_seconds}s)"
                    )
                else:
                    logger.debug(f"{func.__name__} took {elapsed:.3f}s")
        
        return wrapper
    return decorator

# 사용 예시
@app.get("/slow-endpoint")
@track_performance(threshold_seconds=0.5)
async def slow_endpoint():
    # ... 로직 ...
    pass
```

### 3. Endpoint 디버깅 체크리스트

```
□ 요청이 올바른 HTTP 메서드로 전송되었는가?
□ 요청 헤더가 올바른가? (Content-Type, Authorization 등)
□ 요청 본문이 올바른 형식인가? (JSON, Form 등)
□ 쿼리 파라미터가 올바른가?
□ 경로 파라미터가 올바른가?
□ 인증 토큰이 유효한가?
□ 권한이 충분한가?
□ 입력 검증이 통과했는가?
□ 비즈니스 규칙 검증이 통과했는가?
□ 데이터베이스 작업이 성공했는가?
□ 외부 서비스 호출이 성공했는가?
□ 응답 직렬화가 성공했는가?
□ 응답 상태 코드가 올바른가?
□ CORS 설정이 올바른가?
□ Rate limiting에 걸리지 않았는가?
```

### 4. API 테스트 도구 활용

#### 4.1 cURL을 사용한 디버깅

```bash
# 기본 요청
curl -X POST http://localhost:8000/api/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token" \
  -d '{"item_id": 123, "quantity": 2}'

# 상세 정보 출력
curl -v -X POST http://localhost:8000/api/orders \
  -H "Content-Type: application/json" \
  -d '{"item_id": 123}'

# 응답 헤더만 확인
curl -I http://localhost:8000/api/orders

# 타임아웃 설정
curl --max-time 10 http://localhost:8000/api/orders
```

#### 4.2 HTTPie를 사용한 디버깅

```bash
# 기본 요청
http POST localhost:8000/api/orders \
  Authorization:"Bearer token" \
  item_id=123 quantity=2

# 상세 출력
http -v POST localhost:8000/api/orders item_id=123

# 응답 헤더 확인
http --headers GET localhost:8000/api/orders
```

---

## 아키텍처적 관점 디버깅

아키텍처적 관점 디버깅은 시스템 전체의 구조와 컴포넌트 간 상호작용에서 발생하는 문제를 찾는 것입니다.

### 1. 아키텍처 문제 유형

#### 1.1 서비스 간 통신 문제

**증상**
- 타임아웃
- 연결 실패
- 데이터 불일치

**디버깅 방법**

```python
# 서비스 간 통신 추적
import time
import logging
from typing import Optional

logger = logging.getLogger(__name__)

class ServiceClient:
    def __init__(self, service_name: str, base_url: str):
        self.service_name = service_name
        self.base_url = base_url
    
    async def call(self, endpoint: str, method: str = "GET", **kwargs):
        url = f"{self.base_url}/{endpoint}"
        request_id = kwargs.pop('request_id', None)
        
        logger.info(
            f"[{request_id}] Calling {self.service_name}: {method} {url}",
            extra={
                "service": self.service_name,
                "method": method,
                "url": url,
                "request_id": request_id,
            }
        )
        
        start_time = time.time()
        
        try:
            # 요청 전 상태 확인
            logger.debug(f"[{request_id}] Request params: {kwargs}")
            
            # 실제 호출
            response = await self._make_request(method, url, **kwargs)
            
            elapsed = time.time() - start_time
            
            logger.info(
                f"[{request_id}] {self.service_name} responded: "
                f"{response.status_code} in {elapsed:.3f}s"
            )
            
            return response
            
        except TimeoutError as e:
            elapsed = time.time() - start_time
            logger.error(
                f"[{request_id}] {self.service_name} timeout after {elapsed:.3f}s: {e}",
                extra={
                    "service": self.service_name,
                    "error": "timeout",
                    "duration": elapsed,
                }
            )
            raise
            
        except ConnectionError as e:
            logger.error(
                f"[{request_id}] {self.service_name} connection error: {e}",
                extra={
                    "service": self.service_name,
                    "error": "connection_error",
                }
            )
            raise
            
        except Exception as e:
            elapsed = time.time() - start_time
            logger.error(
                f"[{request_id}] {self.service_name} error after {elapsed:.3f}s: {e}",
                exc_info=True,
                extra={
                    "service": self.service_name,
                    "error": type(e).__name__,
                    "duration": elapsed,
                }
            )
            raise
```

#### 1.2 데이터 일관성 문제

**증상**
- 여러 서비스 간 데이터 불일치
- 캐시와 DB 불일치
- 이벤트 순서 문제

**디버깅 방법**

```python
# 분산 트레이싱
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter

tracer = trace.get_tracer(__name__)

async def process_order_with_tracing(order_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order_id", order_id)
        
        # Step 1: 주문 조회
        with tracer.start_as_current_span("fetch_order") as fetch_span:
            order = await fetch_order(order_id)
            fetch_span.set_attribute("order.status", order.status)
            span.add_event("Order fetched", {"order_id": order_id})
        
        # Step 2: 재고 확인
        with tracer.start_as_current_span("check_inventory") as inv_span:
            inventory = await check_inventory(order.items)
            inv_span.set_attribute("inventory.available", inventory.available)
            span.add_event("Inventory checked")
        
        # Step 3: 결제 처리
        with tracer.start_as_current_span("process_payment") as pay_span:
            payment = await process_payment(order)
            pay_span.set_attribute("payment.status", payment.status)
            span.add_event("Payment processed", {"payment_id": payment.id})
        
        # Step 4: 주문 상태 업데이트
        with tracer.start_as_current_span("update_order") as update_span:
            await update_order_status(order_id, "confirmed")
            update_span.set_attribute("new_status", "confirmed")
            span.add_event("Order confirmed")
        
        return order
```

#### 1.3 리소스 경합 문제

**증상**
- 데드락
- 성능 저하
- 리소스 부족

**디버깅 방법**

```python
# 리소스 사용 모니터링
import psutil
import threading
import time

class ResourceMonitor:
    def __init__(self):
        self.monitoring = False
        self.thread = None
    
    def start(self):
        self.monitoring = True
        self.thread = threading.Thread(target=self._monitor, daemon=True)
        self.thread.start()
    
    def stop(self):
        self.monitoring = False
        if self.thread:
            self.thread.join()
    
    def _monitor(self):
        while self.monitoring:
            # CPU 사용률
            cpu_percent = psutil.cpu_percent(interval=1)
            
            # 메모리 사용량
            memory = psutil.virtual_memory()
            
            # 디스크 I/O
            disk_io = psutil.disk_io_counters()
            
            # 네트워크 I/O
            net_io = psutil.net_io_counters()
            
            logger.debug(
                f"Resource usage - "
                f"CPU: {cpu_percent}%, "
                f"Memory: {memory.percent}% ({memory.used / 1024**3:.2f}GB / {memory.total / 1024**3:.2f}GB), "
                f"Disk read: {disk_io.read_bytes / 1024**2:.2f}MB, "
                f"Disk write: {disk_io.write_bytes / 1024**2:.2f}MB"
            )
            
            # 임계값 초과 시 경고
            if cpu_percent > 80:
                logger.warning(f"High CPU usage: {cpu_percent}%")
            if memory.percent > 80:
                logger.warning(f"High memory usage: {memory.percent}%")
            
            time.sleep(5)
```

#### 1.4 이벤트/메시지 처리 문제

**증상**
- 메시지 손실
- 중복 처리
- 순서 문제

**디버깅 방법**

```python
# 메시지 처리 추적
class MessageProcessor:
    def __init__(self):
        self.processed_messages = set()  # 중복 방지
        self.message_order = []  # 순서 추적
    
    async def process_message(self, message_id: str, message: dict):
        # 중복 확인
        if message_id in self.processed_messages:
            logger.warning(f"Duplicate message detected: {message_id}")
            return
        
        logger.info(f"Processing message: {message_id}")
        
        try:
            # 메시지 처리
            result = await self._handle_message(message)
            
            # 처리 완료 표시
            self.processed_messages.add(message_id)
            self.message_order.append({
                'id': message_id,
                'timestamp': time.time(),
                'status': 'success'
            })
            
            logger.info(f"Message processed successfully: {message_id}")
            return result
            
        except Exception as e:
            logger.error(
                f"Message processing failed: {message_id}, error: {e}",
                exc_info=True
            )
            self.message_order.append({
                'id': message_id,
                'timestamp': time.time(),
                'status': 'failed',
                'error': str(e)
            })
            raise
```

### 2. 아키텍처 디버깅 전략

#### 2.1 분산 추적 (Distributed Tracing)

```python
# OpenTelemetry를 사용한 분산 추적 설정
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger import JaegerExporter

def setup_tracing(service_name: str):
    trace.set_tracer_provider(TracerProvider())
    
    # Jaeger로 추적 데이터 전송
    jaeger_exporter = JaegerExporter(
        agent_host_name="localhost",
        agent_port=6831,
    )
    
    span_processor = BatchSpanProcessor(jaeger_exporter)
    trace.get_tracer_provider().add_span_processor(span_processor)
    
    return trace.get_tracer(service_name)

# 사용 예시
tracer = setup_tracing("order-service")

@app.post("/orders")
async def create_order(order_data: OrderCreate):
    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("order.items_count", len(order_data.items))
        
        # 다른 서비스 호출 시 컨텍스트 전파
        with tracer.start_as_current_span("call_inventory_service"):
            inventory_result = await inventory_service.check(order_data.items)
        
        return result
```

#### 2.2 헬스 체크 및 모니터링

```python
# 종합 헬스 체크 엔드포인트
@app.get("/health")
async def health_check():
    health_status = {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "checks": {}
    }
    
    # 데이터베이스 체크
    try:
        await db.execute("SELECT 1")
        health_status["checks"]["database"] = "healthy"
    except Exception as e:
        health_status["checks"]["database"] = f"unhealthy: {e}"
        health_status["status"] = "unhealthy"
    
    # 캐시 체크
    try:
        await cache.ping()
        health_status["checks"]["cache"] = "healthy"
    except Exception as e:
        health_status["checks"]["cache"] = f"unhealthy: {e}"
        health_status["status"] = "degraded"
    
    # 외부 서비스 체크
    try:
        response = await external_service.health_check()
        health_status["checks"]["external_service"] = "healthy"
    except Exception as e:
        health_status["checks"]["external_service"] = f"unhealthy: {e}"
        health_status["status"] = "degraded"
    
    status_code = 200 if health_status["status"] == "healthy" else 503
    return JSONResponse(content=health_status, status_code=status_code)
```

#### 2.3 의존성 그래프 분석

```python
# 서비스 의존성 추적
class DependencyTracker:
    def __init__(self):
        self.dependencies = {}
        self.call_graph = []
    
    def track_call(self, caller: str, callee: str, duration: float, success: bool):
        self.call_graph.append({
            'caller': caller,
            'callee': callee,
            'duration': duration,
            'success': success,
            'timestamp': time.time()
        })
        
        if caller not in self.dependencies:
            self.dependencies[caller] = []
        
        if callee not in self.dependencies[caller]:
            self.dependencies[caller].append(callee)
    
    def get_dependency_graph(self):
        return self.dependencies
    
    def analyze_circular_dependencies(self):
        # 순환 의존성 탐지
        visited = set()
        rec_stack = set()
        cycles = []
        
        def dfs(node, path):
            visited.add(node)
            rec_stack.add(node)
            path.append(node)
            
            for neighbor in self.dependencies.get(node, []):
                if neighbor not in visited:
                    dfs(neighbor, path)
                elif neighbor in rec_stack:
                    # 순환 발견
                    cycle_start = path.index(neighbor)
                    cycles.append(path[cycle_start:] + [neighbor])
            
            rec_stack.remove(node)
            path.pop()
        
        for node in self.dependencies:
            if node not in visited:
                dfs(node, [])
        
        return cycles
```

### 3. 아키텍처 디버깅 체크리스트

```
□ 서비스 간 통신이 정상인가?
□ 네트워크 지연이 허용 범위 내인가?
□ 타임아웃 설정이 적절한가?
□ 재시도 로직이 올바르게 동작하는가?
□ 서킷 브레이커가 필요할 만큼 실패가 많은가?
□ 데이터 일관성이 유지되는가?
□ 트랜잭션이 올바르게 처리되는가?
□ 분산 트랜잭션이 필요한가?
□ 이벤트 순서가 보장되는가?
□ 메시지가 중복 처리되지 않는가?
□ 메시지가 손실되지 않는가?
□ 캐시와 DB 간 일관성이 유지되는가?
□ 리소스 경합이 없는가?
□ 데드락 가능성이 없는가?
□ 리소스 사용량이 정상 범위인가?
□ 확장성 문제가 없는가?
□ 단일 장애점(Single Point of Failure)이 없는가?
```

---

## 데이터베이스 관점 디버깅

데이터베이스 관련 문제는 애플리케이션의 핵심 기능에 직접적인 영향을 미칩니다. 
쿼리 성능, 트랜잭션 관리, 데이터 일관성 등을 체계적으로 디버깅해야 합니다.

### 1. 데이터베이스 에러 유형

#### 1.1 쿼리 성능 문제

**증상**
- 느린 응답 시간
- 타임아웃 발생
- 높은 CPU/메모리 사용률

**디버깅 방법**

```python
# 쿼리 실행 시간 측정
import time
from sqlalchemy import event
from sqlalchemy.engine import Engine

# SQLAlchemy 쿼리 로깅
@event.listens_for(Engine, "before_cursor_execute")
def receive_before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    conn.info.setdefault('query_start_time', []).append(time.time())
    logger.debug(f"Query: {statement}")
    logger.debug(f"Parameters: {parameters}")

@event.listens_for(Engine, "after_cursor_execute")
def receive_after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - conn.info['query_start_time'].pop(-1)
    logger.info(f"Query executed in {total:.3f}s")
    
    # 느린 쿼리 경고
    if total > 1.0:  # 1초 이상
        logger.warning(f"Slow query detected ({total:.3f}s): {statement}")

# Django ORM 쿼리 디버깅
from django.db import connection
from django.conf import settings

if settings.DEBUG:
    # 모든 쿼리 로깅
    for query in connection.queries:
        logger.debug(f"Query: {query['sql']}, Time: {query['time']}s")
    
    # 느린 쿼리 확인
    slow_queries = [q for q in connection.queries if float(q['time']) > 1.0]
    if slow_queries:
        logger.warning(f"Found {len(slow_queries)} slow queries")
```

#### 1.2 트랜잭션 문제

**증상**
- 데이터 불일치
- 데드락 발생
- 롤백 실패

**디버깅 방법**

```python
# 트랜잭션 추적
class TransactionTracker:
    def __init__(self):
        self.active_transactions = {}
        self.transaction_history = []
    
    def start_transaction(self, tx_id: str, operation: str):
        self.active_transactions[tx_id] = {
            'start_time': time.time(),
            'operation': operation,
            'queries': []
        }
        logger.info(f"Transaction started: {tx_id}, operation: {operation}")
    
    def log_query(self, tx_id: str, query: str):
        if tx_id in self.active_transactions:
            self.active_transactions[tx_id]['queries'].append({
                'query': query,
                'time': time.time()
            })
    
    def commit_transaction(self, tx_id: str):
        if tx_id in self.active_transactions:
            tx_info = self.active_transactions.pop(tx_id)
            duration = time.time() - tx_info['start_time']
            
            self.transaction_history.append({
                'tx_id': tx_id,
                'duration': duration,
                'queries_count': len(tx_info['queries']),
                'operation': tx_info['operation']
            })
            
            logger.info(
                f"Transaction committed: {tx_id}, "
                f"duration: {duration:.3f}s, "
                f"queries: {len(tx_info['queries'])}"
            )
    
    def rollback_transaction(self, tx_id: str, reason: str):
        if tx_id in self.active_transactions:
            tx_info = self.active_transactions.pop(tx_id)
            duration = time.time() - tx_info['start_time']
            
            logger.error(
                f"Transaction rolled back: {tx_id}, "
                f"reason: {reason}, "
                f"duration: {duration:.3f}s"
            )

# 트랜잭션 데코레이터
def transactional(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        tx_id = str(uuid.uuid4())
        tracker = TransactionTracker()
        
        tracker.start_transaction(tx_id, func.__name__)
        
        async with db.begin():  # 트랜잭션 시작
            try:
                result = await func(*args, **kwargs)
                tracker.commit_transaction(tx_id)
                return result
            except Exception as e:
                tracker.rollback_transaction(tx_id, str(e))
                raise
    
    return wrapper
```

#### 1.3 연결 풀 문제

**증상**
- 연결 부족 에러
- 연결 누수
- 타임아웃

**디버깅 방법**

```python
# 연결 풀 모니터링
from sqlalchemy import create_engine, event
from sqlalchemy.pool import Pool

@event.listens_for(Pool, "connect")
def receive_connect(dbapi_conn, connection_record):
    logger.debug("New database connection created")
    connection_record.info['created_at'] = time.time()

@event.listens_for(Pool, "checkout")
def receive_checkout(dbapi_conn, connection_record, connection_proxy):
    logger.debug("Connection checked out from pool")
    connection_record.info['checked_out_at'] = time.time()

@event.listens_for(Pool, "checkin")
def receive_checkin(dbapi_conn, connection_record):
    logger.debug("Connection returned to pool")
    if 'checked_out_at' in connection_record.info:
        duration = time.time() - connection_record.info['checked_out_at']
        logger.debug(f"Connection was used for {duration:.3f}s")

@event.listens_for(Pool, "invalidate")
def receive_invalidate(dbapi_conn, connection_record, exception):
    logger.warning(f"Connection invalidated: {exception}")

# 연결 풀 상태 확인
def check_pool_status(engine):
    pool = engine.pool
    logger.info(
        f"Pool status - "
        f"Size: {pool.size()}, "
        f"Checked out: {pool.checkedout()}, "
        f"Overflow: {pool.overflow()}, "
        f"Checked in: {pool.checkedin()}"
    )
    
    if pool.checkedout() > pool.size() * 0.8:
        logger.warning("High connection pool usage detected")
```

#### 1.4 N+1 쿼리 문제

**증상**
- 예상보다 많은 쿼리 실행
- 느린 응답 시간

**디버깅 방법**

```python
# N+1 쿼리 탐지
class QueryCounter:
    def __init__(self):
        self.query_count = 0
        self.queries = []
    
    def count_query(self, query: str):
        self.query_count += 1
        self.queries.append({
            'number': self.query_count,
            'query': query,
            'time': time.time()
        })
    
    def analyze_n_plus_one(self, threshold: int = 10):
        if self.query_count > threshold:
            logger.warning(
                f"Possible N+1 query detected: {self.query_count} queries executed"
            )
            
            # 유사한 쿼리 패턴 찾기
            query_patterns = {}
            for q in self.queries:
                # 쿼리에서 테이블명 추출
                table_match = re.search(r'FROM\s+(\w+)', q['query'], re.IGNORECASE)
                if table_match:
                    table = table_match.group(1)
                    query_patterns[table] = query_patterns.get(table, 0) + 1
            
            for table, count in query_patterns.items():
                if count > threshold:
                    logger.warning(
                        f"Table {table} queried {count} times - "
                        f"consider using eager loading or batch loading"
                    )

# 사용 예시
query_counter = QueryCounter()

# 쿼리 실행 전
query_counter.count_query("SELECT * FROM users WHERE id = 1")

# 결과 분석
query_counter.analyze_n_plus_one(threshold=5)

# 해결 방법: Eager Loading
# SQLAlchemy
users = session.query(User).options(joinedload(User.orders)).all()

# Django ORM
users = User.objects.select_related('profile').prefetch_related('orders').all()
```

### 2. 데이터베이스 디버깅 체크리스트

```
□ 쿼리가 인덱스를 사용하는가?
□ 쿼리 실행 계획을 확인했는가? (EXPLAIN)
□ N+1 쿼리 문제가 없는가?
□ 트랜잭션이 올바르게 관리되는가?
□ 트랜잭션 격리 수준이 적절한가?
□ 데드락 가능성이 없는가?
□ 연결 풀이 적절히 설정되었는가?
□ 연결이 제대로 반환되는가? (연결 누수 확인)
□ 쿼리 타임아웃이 설정되어 있는가?
□ 슬로우 쿼리 로그를 확인했는가?
□ 데이터베이스 락이 적절히 사용되는가?
□ 배치 작업이 적절히 처리되는가?
```

---

## 비동기/동시성 관점 디버깅

비동기 프로그래밍과 동시성은 현대 애플리케이션에서 필수적이지만, 
디버깅이 어려운 영역입니다. 레이스 컨디션, 데드락, 동시성 버그 등을 주의 깊게 확인해야 합니다.

### 1. 비동기/동시성 에러 유형

#### 1.1 레이스 컨디션 (Race Condition)

**증상**
- 비결정적 동작
- 데이터 불일치
- 간헐적 실패

**디버깅 방법**

```python
# 레이스 컨디션 추적
import asyncio
from collections import defaultdict

class RaceConditionTracker:
    def __init__(self):
        self.shared_resources = defaultdict(list)
        self.access_log = []
    
    def track_access(self, resource_id: str, operation: str, task_id: str):
        timestamp = time.time()
        self.access_log.append({
            'timestamp': timestamp,
            'resource_id': resource_id,
            'operation': operation,
            'task_id': task_id
        })
        
        self.shared_resources[resource_id].append({
            'timestamp': timestamp,
            'operation': operation,
            'task_id': task_id
        })
    
    def detect_race_conditions(self):
        for resource_id, accesses in self.shared_resources.items():
            if len(accesses) < 2:
                continue
            
            # 동시 접근 탐지 (0.1초 이내)
            for i in range(len(accesses) - 1):
                time_diff = accesses[i+1]['timestamp'] - accesses[i]['timestamp']
                if time_diff < 0.1 and accesses[i]['operation'] != accesses[i+1]['operation']:
                    logger.warning(
                        f"Possible race condition detected on {resource_id}: "
                        f"{accesses[i]['operation']} and {accesses[i+1]['operation']} "
                        f"within {time_diff:.3f}s"
                    )

# 문제 코드 예시
async def update_balance(account_id: str, amount: float):
    # ❌ 레이스 컨디션 발생 가능
    account = await get_account(account_id)
    account.balance += amount
    await save_account(account)

# 해결 방법 1: 락 사용
async def update_balance_with_lock(account_id: str, amount: float):
    async with account_lock:  # 비동기 락
        account = await get_account(account_id)
        account.balance += amount
        await save_account(account)

# 해결 방법 2: 데이터베이스 레벨 락
async def update_balance_with_db_lock(account_id: str, amount: float):
    async with db.begin():
        account = await db.execute(
            select(Account).where(Account.id == account_id).with_for_update()
        )
        account.balance += amount
        await db.commit()
```

#### 1.2 데드락 (Deadlock)

**증상**
- 프로그램이 멈춤
- 타임아웃 발생
- 리소스가 해제되지 않음

**디버깅 방법**

```python
# 데드락 탐지
import asyncio
from asyncio import Lock

class DeadlockDetector:
    def __init__(self):
        self.lock_acquisitions = {}
        self.wait_graph = defaultdict(set)
    
    def track_lock_acquisition(self, task_id: str, lock_id: str):
        if task_id in self.lock_acquisitions:
            # 이미 다른 락을 보유하고 있음
            held_lock = self.lock_acquisitions[task_id]
            self.wait_graph[held_lock].add(lock_id)
        
        self.lock_acquisitions[task_id] = lock_id
    
    def detect_deadlock(self):
        # 사이클 탐지 (간단한 DFS)
        visited = set()
        rec_stack = set()
        
        def has_cycle(node):
            visited.add(node)
            rec_stack.add(node)
            
            for neighbor in self.wait_graph.get(node, []):
                if neighbor not in visited:
                    if has_cycle(neighbor):
                        return True
                elif neighbor in rec_stack:
                    logger.error(f"Deadlock detected: cycle involving {node} and {neighbor}")
                    return True
            
            rec_stack.remove(node)
            return False
        
        for node in self.wait_graph:
            if node not in visited:
                if has_cycle(node):
                    return True
        return False

# 락 순서 통일 (데드락 방지)
# ❌ 잘못된 예: 락 순서가 일관되지 않음
async def transfer_funds(account1, account2, amount):
    async with account1.lock:
        async with account2.lock:  # 다른 곳에서 반대 순서로 잠그면 데드락
            # ...

# ✅ 올바른 예: 항상 ID 순서로 락 획득
async def transfer_funds_safe(account1, account2, amount):
    first, second = sorted([account1, account2], key=lambda a: a.id)
    async with first.lock:
        async with second.lock:
            # ...
```

#### 1.3 비동기 컨텍스트 문제

**증상**
- 컨텍스트 변수 손실
- 비동기 함수가 동기적으로 실행됨
- 이벤트 루프 블로킹

**디버깅 방법**

```python
# 비동기 컨텍스트 추적
import contextvars

request_id_var = contextvars.ContextVar('request_id')

class AsyncContextTracker:
    def __init__(self):
        self.context_history = []
    
    def track_context(self, operation: str):
        context_data = {
            'operation': operation,
            'request_id': request_id_var.get(None),
            'task_name': asyncio.current_task().get_name() if asyncio.current_task() else None,
            'timestamp': time.time()
        }
        self.context_history.append(context_data)
        logger.debug(f"Context: {context_data}")
    
    def verify_context_preservation(self):
        # 컨텍스트가 유지되는지 확인
        for i in range(len(self.context_history) - 1):
            if self.context_history[i]['request_id'] != self.context_history[i+1]['request_id']:
                logger.warning(
                    f"Context lost between {self.context_history[i]['operation']} "
                    f"and {self.context_history[i+1]['operation']}"
                )

# 이벤트 루프 블로킹 탐지
def detect_blocking_operations():
    loop = asyncio.get_event_loop()
    
    # 블로킹 작업 탐지
    import sys
    if sys.platform != 'win32':
        import select
        
        def check_blocking():
            # 이벤트 루프가 블로킹되어 있는지 확인
            ready, _, _ = select.select([], [], [], 0)
            if ready:
                logger.warning("Event loop may be blocked")
        
        # 주기적으로 확인
        asyncio.create_task(periodic_check(check_blocking, interval=1.0))
```

#### 1.4 동시성 제어 문제

**증상**
- 공유 자원 접근 충돌
- 데이터 손실
- 예상과 다른 결과

**디버깅 방법**

```python
# 동시 실행 추적
class ConcurrencyTracker:
    def __init__(self):
        self.concurrent_operations = defaultdict(int)
        self.operation_timeline = []
    
    def track_operation(self, operation: str, start: bool):
        timestamp = time.time()
        
        if start:
            self.concurrent_operations[operation] += 1
            self.operation_timeline.append({
                'timestamp': timestamp,
                'operation': operation,
                'action': 'start',
                'concurrent_count': sum(self.concurrent_operations.values())
            })
        else:
            self.concurrent_operations[operation] -= 1
            if self.concurrent_operations[operation] == 0:
                del self.concurrent_operations[operation]
            
            self.operation_timeline.append({
                'timestamp': timestamp,
                'operation': operation,
                'action': 'end',
                'concurrent_count': sum(self.concurrent_operations.values())
            })
        
        logger.debug(
            f"Operation {operation} {'started' if start else 'ended'}, "
            f"concurrent: {sum(self.concurrent_operations.values())}"
        )
    
    def analyze_concurrency(self):
        max_concurrent = max(
            (item['concurrent_count'] for item in self.operation_timeline),
            default=0
        )
        logger.info(f"Maximum concurrent operations: {max_concurrent}")
        
        # 동시 실행이 많은 구간 찾기
        high_concurrency_periods = [
            item for item in self.operation_timeline
            if item['concurrent_count'] > 10
        ]
        if high_concurrency_periods:
            logger.warning(
                f"High concurrency detected: {len(high_concurrency_periods)} periods "
                f"with >10 concurrent operations"
            )

# 세마포어를 사용한 동시성 제어
class RateLimiter:
    def __init__(self, max_concurrent: int):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.active_count = 0
    
    async def acquire(self):
        await self.semaphore.acquire()
        self.active_count += 1
        logger.debug(f"Semaphore acquired, active: {self.active_count}")
    
    def release(self):
        self.semaphore.release()
        self.active_count -= 1
        logger.debug(f"Semaphore released, active: {self.active_count}")

# 사용 예시
rate_limiter = RateLimiter(max_concurrent=5)

async def limited_operation():
    await rate_limiter.acquire()
    try:
        # 작업 수행
        await do_work()
    finally:
        rate_limiter.release()
```

### 2. 비동기/동시성 디버깅 체크리스트

```
□ 비동기 함수가 await 없이 호출되지 않았는가?
□ 동기 블로킹 작업이 이벤트 루프를 막지 않는가?
□ 공유 자원 접근이 적절히 보호되는가?
□ 락 순서가 일관되는가? (데드락 방지)
□ 비동기 컨텍스트가 유지되는가?
□ 예외가 적절히 처리되고 전파되는가?
□ 타임아웃이 설정되어 있는가?
□ 동시 실행 수가 제한되어 있는가?
□ 리소스가 적절히 해제되는가?
□ 레이스 컨디션이 없는가?
```

---

## 환경/설정 관점 디버깅

환경 변수, 설정 파일, 의존성 버전 등은 애플리케이션의 동작에 큰 영향을 미칩니다. 
환경별 차이로 인한 문제를 체계적으로 디버깅해야 합니다.

### 1. 환경/설정 에러 유형

#### 1.1 환경 변수 문제

**증상**
- 설정 값이 None 또는 기본값
- 환경별로 다른 동작
- 민감한 정보 노출

**디버깅 방법**

```python
# 환경 변수 검증
import os
from typing import Optional

class EnvironmentValidator:
    def __init__(self):
        self.required_vars = []
        self.optional_vars = []
        self.validated_vars = {}
    
    def require(self, var_name: str, var_type: type = str, default: Optional[any] = None):
        self.required_vars.append({
            'name': var_name,
            'type': var_type,
            'default': default
        })
    
    def optional(self, var_name: str, var_type: type = str, default: Optional[any] = None):
        self.optional_vars.append({
            'name': var_name,
            'type': var_type,
            'default': default
        })
    
    def validate(self) -> dict:
        missing_vars = []
        
        # 필수 변수 확인
        for var_spec in self.required_vars:
            var_name = var_spec['name']
            var_type = var_spec['type']
            default = var_spec['default']
            
            value = os.getenv(var_name, default)
            
            if value is None:
                missing_vars.append(var_name)
                logger.error(f"Required environment variable missing: {var_name}")
            else:
                try:
                    # 타입 변환
                    if var_type == bool:
                        value = value.lower() in ('true', '1', 'yes', 'on')
                    elif var_type == int:
                        value = int(value)
                    elif var_type == float:
                        value = float(value)
                    
                    self.validated_vars[var_name] = value
                    logger.debug(f"Environment variable validated: {var_name}")
                except (ValueError, TypeError) as e:
                    logger.error(
                        f"Invalid type for {var_name}: expected {var_type.__name__}, "
                        f"got {type(value).__name__}, error: {e}"
                    )
                    missing_vars.append(var_name)
        
        # 선택적 변수 확인
        for var_spec in self.optional_vars:
            var_name = var_spec['name']
            var_type = var_spec['type']
            default = var_spec['default']
            
            value = os.getenv(var_name, default)
            
            if value is not None:
                try:
                    if var_type == bool:
                        value = value.lower() in ('true', '1', 'yes', 'on')
                    elif var_type == int:
                        value = int(value)
                    elif var_type == float:
                        value = float(value)
                    
                    self.validated_vars[var_name] = value
                except (ValueError, TypeError) as e:
                    logger.warning(
                        f"Invalid type for optional {var_name}: {e}, using default"
                    )
                    self.validated_vars[var_name] = default
        
        if missing_vars:
            raise ValueError(f"Missing required environment variables: {missing_vars}")
        
        return self.validated_vars

# 사용 예시
validator = EnvironmentValidator()
validator.require('DATABASE_URL', str)
validator.require('REDIS_URL', str)
validator.require('DEBUG', bool, default=False)
validator.optional('LOG_LEVEL', str, default='INFO')

config = validator.validate()
```

#### 1.2 설정 파일 문제

**증상**
- 설정 파일을 찾을 수 없음
- 설정 형식 오류
- 환경별 설정 불일치

**디버깅 방법**

```python
# 설정 파일 로딩 및 검증
import json
import yaml
from pathlib import Path
from typing import Dict, Any

class ConfigLoader:
    def __init__(self, config_path: str):
        self.config_path = Path(config_path)
        self.config = {}
        self.loaded_files = []
    
    def load(self) -> Dict[str, Any]:
        if not self.config_path.exists():
            raise FileNotFoundError(f"Config file not found: {self.config_path}")
        
        logger.info(f"Loading config from: {self.config_path}")
        
        # 파일 확장자에 따라 로더 선택
        if self.config_path.suffix == '.json':
            with open(self.config_path) as f:
                self.config = json.load(f)
        elif self.config_path.suffix in ['.yaml', '.yml']:
            with open(self.config_path) as f:
                self.config = yaml.safe_load(f)
        else:
            raise ValueError(f"Unsupported config file format: {self.config_path.suffix}")
        
        self.loaded_files.append(str(self.config_path))
        logger.debug(f"Config loaded: {len(self.config)} keys")
        
        return self.config
    
    def validate(self, schema: Dict[str, Any]):
        """설정 스키마 검증"""
        missing_keys = []
        invalid_types = []
        
        def validate_recursive(config: Dict, schema: Dict, path: str = ""):
            for key, expected_type in schema.items():
                current_path = f"{path}.{key}" if path else key
                
                if key not in config:
                    missing_keys.append(current_path)
                    continue
                
                value = config[key]
                
                if isinstance(expected_type, dict):
                    # 중첩된 딕셔너리
                    if not isinstance(value, dict):
                        invalid_types.append(f"{current_path}: expected dict, got {type(value).__name__}")
                    else:
                        validate_recursive(value, expected_type, current_path)
                elif isinstance(expected_type, type):
                    # 타입 검증
                    if not isinstance(value, expected_type):
                        invalid_types.append(
                            f"{current_path}: expected {expected_type.__name__}, "
                            f"got {type(value).__name__}"
                        )
        
        validate_recursive(self.config, schema)
        
        if missing_keys:
            logger.error(f"Missing config keys: {missing_keys}")
        if invalid_types:
            logger.error(f"Invalid config types: {invalid_types}")
        
        if missing_keys or invalid_types:
            raise ValueError("Config validation failed")
        
        logger.info("Config validation passed")
    
    def get(self, key: str, default: Any = None) -> Any:
        """점 표기법으로 중첩된 키 접근"""
        keys = key.split('.')
        value = self.config
        
        for k in keys:
            if isinstance(value, dict) and k in value:
                value = value[k]
            else:
                return default
        
        return value

# 환경별 설정 로딩
def load_environment_config(env: str = None) -> Dict[str, Any]:
    if env is None:
        env = os.getenv('ENVIRONMENT', 'development')
    
    base_config_path = Path('config') / 'base.yaml'
    env_config_path = Path('config') / f'{env}.yaml'
    
    config = {}
    
    # 기본 설정 로드
    if base_config_path.exists():
        base_loader = ConfigLoader(base_config_path)
        config.update(base_loader.load())
        logger.info(f"Loaded base config: {base_config_path}")
    
    # 환경별 설정 로드 (기본 설정을 덮어씀)
    if env_config_path.exists():
        env_loader = ConfigLoader(env_config_path)
        env_config = env_loader.load()
        config = merge_config(config, env_config)
        logger.info(f"Loaded environment config: {env_config_path}")
    else:
        logger.warning(f"Environment config not found: {env_config_path}")
    
    return config
```

#### 1.3 의존성 버전 문제

**증상**
- 패키지 버전 불일치
- 호환성 문제
- 기능이 예상과 다르게 동작

**디버깅 방법**

```python
# 의존성 버전 확인
import importlib.metadata
from packaging import version

class DependencyChecker:
    def __init__(self):
        self.installed_packages = {}
        self.required_packages = {}
    
    def check_installed(self, package_name: str) -> Optional[str]:
        """설치된 패키지 버전 확인"""
        try:
            dist = importlib.metadata.distribution(package_name)
            version = dist.version
            self.installed_packages[package_name] = version
            return version
        except importlib.metadata.PackageNotFoundError:
            logger.warning(f"Package not installed: {package_name}")
            return None
    
    def require(self, package_name: str, min_version: str = None, max_version: str = None):
        """필수 패키지 및 버전 요구사항 등록"""
        self.required_packages[package_name] = {
            'min': min_version,
            'max': max_version
        }
    
    def validate_all(self) -> Dict[str, bool]:
        """모든 의존성 검증"""
        results = {}
        
        for package_name, requirements in self.required_packages.items():
            installed_version = self.check_installed(package_name)
            
            if installed_version is None:
                logger.error(f"Required package not installed: {package_name}")
                results[package_name] = False
                continue
            
            # 버전 검증
            is_valid = True
            
            if requirements['min']:
                if version.parse(installed_version) < version.parse(requirements['min']):
                    logger.error(
                        f"{package_name}: installed {installed_version}, "
                        f"required >= {requirements['min']}"
                    )
                    is_valid = False
            
            if requirements['max']:
                if version.parse(installed_version) > version.parse(requirements['max']):
                    logger.error(
                        f"{package_name}: installed {installed_version}, "
                        f"required <= {requirements['max']}"
                    )
                    is_valid = False
            
            if is_valid:
                logger.info(
                    f"{package_name}: {installed_version} ✓ "
                    f"(required: {requirements['min'] or 'any'} - {requirements['max'] or 'any'})"
                )
            
            results[package_name] = is_valid
        
        return results

# 사용 예시
checker = DependencyChecker()
checker.require('fastapi', min_version='0.100.0')
checker.require('sqlalchemy', min_version='2.0.0', max_version='2.1.0')
checker.require('pydantic', min_version='2.0.0')

results = checker.validate_all()
if not all(results.values()):
    raise RuntimeError("Dependency validation failed")
```

### 2. 환경/설정 디버깅 체크리스트

```
□ 필수 환경 변수가 모두 설정되어 있는가?
□ 환경 변수 타입이 올바른가?
□ 설정 파일이 올바른 위치에 있는가?
□ 설정 파일 형식이 올바른가?
□ 환경별 설정이 올바르게 로드되는가?
□ 의존성 버전이 호환되는가?
□ 개발/스테이징/프로덕션 환경이 올바르게 구분되는가?
□ 민감한 정보가 코드에 하드코딩되지 않았는가?
□ 기본값이 적절히 설정되어 있는가?
□ 설정 변경 시 애플리케이션이 재시작되는가?
```

---

## 캐시 관점 디버깅

캐시는 성능 향상을 위해 필수적이지만, 잘못 사용하면 데이터 일관성 문제를 일으킬 수 있습니다. 
캐시 무효화, TTL, 일관성 등을 체계적으로 디버깅해야 합니다.

### 1. 캐시 에러 유형

#### 1.1 캐시 무효화 문제

**증상**
- 오래된 데이터 표시
- 업데이트가 반영되지 않음
- 데이터 불일치

**디버깅 방법**

```python
# 캐시 추적
class CacheTracker:
    def __init__(self):
        self.cache_operations = []
        self.cache_keys = {}
    
    def track_get(self, key: str, hit: bool, value: Any = None):
        operation = {
            'type': 'get',
            'key': key,
            'hit': hit,
            'timestamp': time.time(),
            'value_hash': hash(str(value)) if value else None
        }
        self.cache_operations.append(operation)
        
        if hit:
            logger.debug(f"Cache HIT: {key}")
        else:
            logger.debug(f"Cache MISS: {key}")
    
    def track_set(self, key: str, value: Any, ttl: int = None):
        operation = {
            'type': 'set',
            'key': key,
            'ttl': ttl,
            'timestamp': time.time(),
            'value_hash': hash(str(value))
        }
        self.cache_operations.append(operation)
        self.cache_keys[key] = {
            'set_at': time.time(),
            'ttl': ttl,
            'expires_at': time.time() + ttl if ttl else None
        }
        logger.debug(f"Cache SET: {key}, TTL: {ttl}")
    
    def track_delete(self, key: str):
        operation = {
            'type': 'delete',
            'key': key,
            'timestamp': time.time()
        }
        self.cache_operations.append(operation)
        
        if key in self.cache_keys:
            del self.cache_keys[key]
        
        logger.debug(f"Cache DELETE: {key}")
    
    def analyze_invalidation(self):
        """캐시 무효화 패턴 분석"""
        # SET 후 DELETE가 없는 키 찾기
        set_keys = set()
        delete_keys = set()
        
        for op in self.cache_operations:
            if op['type'] == 'set':
                set_keys.add(op['key'])
            elif op['type'] == 'delete':
                delete_keys.add(op['key'])
        
        never_invalidated = set_keys - delete_keys
        if never_invalidated:
            logger.warning(
                f"Cache keys never invalidated: {never_invalidated}. "
                f"Consider adding invalidation logic."
            )
        
        # GET 후 SET가 없는 키 찾기 (캐시 미스 후 업데이트 안 됨)
        get_keys = {}
        for op in self.cache_operations:
            if op['type'] == 'get' and not op['hit']:
                if op['key'] not in get_keys:
                    get_keys[op['key']] = []
                get_keys[op['key']].append(op['timestamp'])
        
        for key, miss_times in get_keys.items():
            # MISS 후 SET이 있는지 확인
            has_set_after_miss = any(
                op['type'] == 'set' and op['key'] == key and op['timestamp'] > miss_time
                for op in self.cache_operations
                for miss_time in miss_times
            )
            
            if not has_set_after_miss and len(miss_times) > 5:
                logger.warning(
                    f"Cache key {key} has {len(miss_times)} misses but no subsequent SET. "
                    f"Consider implementing cache-aside pattern."
                )

# 캐시 무효화 전략
class CacheInvalidator:
    def __init__(self, cache_client):
        self.cache = cache_client
        self.invalidation_rules = {}
    
    def register_invalidation_rule(self, pattern: str, invalidate_keys: callable):
        """무효화 규칙 등록"""
        self.invalidation_rules[pattern] = invalidate_keys
    
    def invalidate_on_update(self, entity_type: str, entity_id: str):
        """엔티티 업데이트 시 관련 캐시 무효화"""
        keys_to_invalidate = []
        
        # 패턴 기반 무효화
        for pattern, get_keys_func in self.invalidation_rules.items():
            if pattern in entity_type:
                keys = get_keys_func(entity_type, entity_id)
                keys_to_invalidate.extend(keys)
        
        # 일반적인 키 패턴
        keys_to_invalidate.extend([
            f"{entity_type}:{entity_id}",
            f"{entity_type}:{entity_id}:*",  # 와일드카드
        ])
        
        for key in keys_to_invalidate:
            if '*' in key:
                # 와일드카드 삭제 (Redis의 경우)
                self.cache.delete_pattern(key)
            else:
                self.cache.delete(key)
            
            logger.info(f"Cache invalidated: {key}")
```

#### 1.2 TTL 문제

**증상**
- 캐시가 너무 빨리 만료됨
- 캐시가 너무 오래 유지됨
- 예상과 다른 만료 시간

**디버깅 방법**

```python
# TTL 모니터링
class TTLMonitor:
    def __init__(self):
        self.key_ttls = {}
        self.expiration_events = []
    
    def track_ttl(self, key: str, ttl: int):
        expires_at = time.time() + ttl
        self.key_ttls[key] = {
            'ttl': ttl,
            'set_at': time.time(),
            'expires_at': expires_at
        }
        
        self.expiration_events.append({
            'key': key,
            'expires_at': expires_at
        })
        
        logger.debug(f"TTL set for {key}: {ttl}s, expires at {expires_at}")
    
    def check_expired_keys(self):
        """만료된 키 확인"""
        current_time = time.time()
        expired_keys = [
            key for key, info in self.key_ttls.items()
            if info['expires_at'] < current_time
        ]
        
        if expired_keys:
            logger.info(f"Expired cache keys: {expired_keys}")
        
        return expired_keys
    
    def analyze_ttl_patterns(self):
        """TTL 패턴 분석"""
        ttls = [info['ttl'] for info in self.key_ttls.values()]
        
        if not ttls:
            return
        
        avg_ttl = sum(ttls) / len(ttls)
        min_ttl = min(ttls)
        max_ttl = max(ttls)
        
        logger.info(
            f"TTL statistics - "
            f"Average: {avg_ttl:.1f}s, "
            f"Min: {min_ttl}s, "
            f"Max: {max_ttl}s"
        )
        
        # 매우 짧은 TTL 경고
        very_short_ttls = [ttl for ttl in ttls if ttl < 60]
        if very_short_ttls:
            logger.warning(
                f"Very short TTLs detected ({len(very_short_ttls)} keys < 60s). "
                f"Consider increasing TTL for better cache hit rate."
            )
        
        # 매우 긴 TTL 경고
        very_long_ttls = [ttl for ttl in ttls if ttl > 86400]  # 24시간
        if very_long_ttls:
            logger.warning(
                f"Very long TTLs detected ({len(very_long_ttls)} keys > 24h). "
                f"Consider shorter TTL for better data freshness."
            )
```

#### 1.3 캐시 일관성 문제

**증상**
- 캐시와 DB 데이터 불일치
- 동시 업데이트 시 데이터 손실
- 읽기/쓰기 불일치

**디버깅 방법**

```python
# 캐시 일관성 검증
class CacheConsistencyChecker:
    def __init__(self, cache_client, db_client):
        self.cache = cache_client
        self.db = db_client
        self.inconsistencies = []
    
    async def verify_consistency(self, key: str, db_fetch_func: callable):
        """캐시와 DB 일관성 확인"""
        cache_value = await self.cache.get(key)
        db_value = await db_fetch_func()
        
        if cache_value != db_value:
            inconsistency = {
                'key': key,
                'cache_value': cache_value,
                'db_value': db_value,
                'timestamp': time.time()
            }
            self.inconsistencies.append(inconsistency)
            
            logger.error(
                f"Cache inconsistency detected for {key}: "
                f"cache={cache_value}, db={db_value}"
            )
            
            return False
        
        return True
    
    def get_inconsistencies(self):
        return self.inconsistencies
    
    def clear_inconsistencies(self):
        self.inconsistencies.clear()

# 캐시 전략 선택
class CacheStrategy:
    """적절한 캐시 전략 선택"""
    
    @staticmethod
    def cache_aside(key: str, fetch_func: callable, ttl: int = 3600):
        """Cache-Aside 패턴"""
        # 1. 캐시에서 조회
        value = cache.get(key)
        if value is not None:
            return value
        
        # 2. 캐시 미스 시 DB에서 조회
        value = fetch_func()
        
        # 3. 캐시에 저장
        cache.set(key, value, ttl=ttl)
        
        return value
    
    @staticmethod
    def write_through(key: str, value: Any, save_func: callable, ttl: int = 3600):
        """Write-Through 패턴"""
        # 1. DB에 저장
        save_func(value)
        
        # 2. 캐시에 저장
        cache.set(key, value, ttl=ttl)
    
    @staticmethod
    def write_back(key: str, value: Any, save_func: callable, ttl: int = 3600):
        """Write-Back 패턴 (주의: 데이터 손실 가능)"""
        # 1. 캐시에만 저장 (지연 쓰기)
        cache.set(key, value, ttl=ttl)
        
        # 2. 백그라운드에서 DB에 저장
        async def background_save():
            await asyncio.sleep(1)  # 짧은 지연
            save_func(value)
        
        asyncio.create_task(background_save())
```

### 2. 캐시 디버깅 체크리스트

```
□ 캐시 키가 일관된 형식인가?
□ 캐시 무효화가 적절한 시점에 이루어지는가?
□ TTL이 적절히 설정되어 있는가?
□ 캐시와 DB 간 일관성이 유지되는가?
□ 캐시 전략이 적절한가? (Cache-Aside, Write-Through 등)
□ 캐시 히트율이 적절한가?
□ 메모리 사용량이 적절한가?
□ 캐시 키 충돌이 없는가?
□ 분산 환경에서 캐시 일관성이 유지되는가?
□ 캐시 장애 시 폴백 전략이 있는가?
```

---

## 디버깅 도구 및 기법

### 1. 로깅 전략

#### 1.1 구조화된 로깅

```python
import logging
import json
from datetime import datetime

class StructuredLogger:
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
    
    def log(self, level: str, message: str, **kwargs):
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": level,
            "message": message,
            **kwargs
        }
        
        log_message = json.dumps(log_data)
        
        if level == "DEBUG":
            self.logger.debug(log_message)
        elif level == "INFO":
            self.logger.info(log_message)
        elif level == "WARNING":
            self.logger.warning(log_message)
        elif level == "ERROR":
            self.logger.error(log_message)
        elif level == "CRITICAL":
            self.logger.critical(log_message)

# 사용 예시
logger = StructuredLogger(__name__)
logger.log("INFO", "Order created", order_id="123", user_id="456", amount=100.0)
```

#### 1.2 로그 레벨 전략

```python
# 로그 레벨별 사용 가이드

# DEBUG: 상세한 디버깅 정보
logger.debug(f"Processing item: {item}, with context: {context}")

# INFO: 일반적인 정보성 메시지
logger.info(f"Order {order_id} created successfully")

# WARNING: 잠재적 문제 (기능은 동작하지만 주의 필요)
logger.warning(f"High memory usage: {memory_percent}%")

# ERROR: 에러 발생 (기능 실패)
logger.error(f"Failed to process order: {order_id}", exc_info=True)

# CRITICAL: 심각한 문제 (시스템 중단 가능성)
logger.critical(f"Database connection lost")
```

### 2. 디버거 사용

#### 2.1 Python 디버거 (pdb)

```python
# 코드에 브레이크포인트 설정
import pdb

def problematic_function(data):
    pdb.set_trace()  # 여기서 실행이 멈춤
    # 디버거 명령어:
    # n (next): 다음 줄
    # s (step): 함수 내부로 들어가기
    # c (continue): 계속 실행
    # p variable: 변수 값 출력
    # l (list): 현재 코드 보기
    # q (quit): 종료
    
    result = process(data)
    return result
```

#### 2.2 IPython 디버거 (ipdb)

```python
# 더 강력한 디버거
import ipdb

def problematic_function(data):
    ipdb.set_trace()  # IPython 기반 디버거
    # 추가 기능:
    # 탭 자동완성
    # 색상 출력
    # 더 나은 변수 탐색
```

#### 2.3 VS Code / PyCharm 디버거

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: FastAPI",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/main.py",
            "console": "integratedTerminal",
            "env": {
                "PYTHONPATH": "${workspaceFolder}"
            },
            "justMyCode": false
        }
    ]
}
```

### 3. 프로파일링

#### 3.1 cProfile 사용

```python
import cProfile
import pstats

# 프로파일링 실행
profiler = cProfile.Profile()
profiler.enable()

# 프로파일링할 코드 실행
your_function()

profiler.disable()

# 결과 저장
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # 상위 20개 함수
```

#### 3.2 line_profiler 사용

```python
# 함수에 데코레이터 추가
@profile
def slow_function():
    # 각 줄의 실행 시간 측정
    result = expensive_operation()
    return result

# 실행: kernprof -l -v script.py
```

### 4. 메모리 디버깅

#### 4.1 memory_profiler 사용

```python
from memory_profiler import profile

@profile
def memory_intensive_function():
    large_list = [i for i in range(1000000)]
    return sum(large_list)

# 실행: python -m memory_profiler script.py
```

#### 4.2 tracemalloc 사용

```python
import tracemalloc

tracemalloc.start()

# 코드 실행
your_code()

# 메모리 사용량 확인
current, peak = tracemalloc.get_traced_memory()
print(f"Current memory usage: {current / 1024 / 1024:.2f} MB")
print(f"Peak memory usage: {peak / 1024 / 1024:.2f} MB")

# 상위 메모리 사용자 확인
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

for stat in top_stats[:10]:
    print(stat)
```

### 5. 네트워크 디버깅

#### 5.1 HTTP 요청 추적

```python
import httpx
import logging

# HTTP 클라이언트에 로깅 추가
logging.basicConfig(level=logging.DEBUG)

# httpx는 자동으로 요청/응답을 로깅
client = httpx.Client()
response = client.get("https://api.example.com/data")
```

#### 5.2 gRPC 디버깅

```python
import grpc
from grpc_interceptor import ServerInterceptor

class LoggingInterceptor(ServerInterceptor):
    def intercept(self, method, request, context, method_name):
        logger.info(f"gRPC call: {method_name}")
        logger.debug(f"Request: {request}")
        
        try:
            response = method(request, context)
            logger.debug(f"Response: {response}")
            return response
        except Exception as e:
            logger.error(f"gRPC error in {method_name}: {e}", exc_info=True)
            raise
```

---

## 체크리스트

### 문제 발생 시 체크리스트

#### 1단계: Import 확인
```
□ 모듈이 존재하는가?
□ 경로가 올바른가?
□ 의존성이 설치되어 있는가?
□ 순환 참조가 없는가?
□ Python 경로가 올바른가?
```

#### 2단계: 로직 확인
```
□ 입력 값이 올바른가?
□ 모든 분기가 테스트되었는가?
□ 예외 처리가 적절한가?
□ 상태 관리가 올바른가?
□ 데이터 변환이 올바른가?
```

#### 3단계: Endpoint 확인
```
□ 요청 형식이 올바른가?
□ 인증/인가가 통과했는가?
□ 검증이 통과했는가?
□ 비즈니스 규칙이 통과했는가?
□ 응답 형식이 올바른가?
```

#### 4단계: 아키텍처 확인
```
□ 서비스 간 통신이 정상인가?
□ 데이터 일관성이 유지되는가?
□ 리소스 사용량이 정상인가?
□ 이벤트/메시지가 올바르게 처리되는가?
□ 의존성이 올바른가?
```

#### 5단계: 데이터베이스 확인
```
□ 쿼리가 인덱스를 사용하는가?
□ N+1 쿼리 문제가 없는가?
□ 트랜잭션이 올바르게 관리되는가?
□ 연결 풀이 적절히 설정되었는가?
□ 슬로우 쿼리가 없는가?
```

#### 6단계: 비동기/동시성 확인
```
□ 레이스 컨디션이 없는가?
□ 데드락 가능성이 없는가?
□ 비동기 컨텍스트가 유지되는가?
□ 동시 실행 수가 제한되어 있는가?
```

#### 7단계: 환경/설정 확인
```
□ 필수 환경 변수가 설정되어 있는가?
□ 설정 파일이 올바른가?
□ 의존성 버전이 호환되는가?
```

#### 8단계: 캐시 확인
```
□ 캐시 무효화가 적절한가?
□ TTL이 적절히 설정되어 있는가?
□ 캐시와 DB 간 일관성이 유지되는가?
```

### 디버깅 세션 체크리스트

```
□ 문제를 재현할 수 있는가?
□ 재현 단계가 문서화되었는가?
□ 관련 로그를 수집했는가?
□ 에러 메시지를 기록했는가?
□ 환경 정보를 기록했는가? (OS, Python 버전, 패키지 버전 등)
□ 가설을 설정하고 검증했는가?
□ 해결 방법을 문서화했는가?
□ 유사 문제 예방 방안을 수립했는가?
```

---

## 결론

디버깅은 체계적인 접근이 필요합니다. 
Import → 로직 → Endpoint → 아키텍처 → 데이터베이스 → 비동기/동시성 → 환경/설정 → 캐시 순서로 문제를 좁혀가며, 
각 단계에서 적절한 도구와 기법을 활용하여 효율적으로 문제를 해결할 수 있습니다.

### 핵심 원칙

1. **체계적 접근**: 문제를 여러 관점에서 분석
2. **증거 수집**: 로그, 에러, 상태 정보를 체계적으로 수집
3. **가설 검증**: 가능한 원인을 가설로 설정하고 검증
4. **도구 활용**: 적절한 디버깅 도구를 활용
5. **문서화**: 문제와 해결 방법을 문서화하여 재발 방지

### 추가 리소스

- [Python Debugging Guide](https://docs.python.org/3/library/pdb.html)
- [Logging Best Practices](https://docs.python.org/3/howto/logging.html)
- [Distributed Tracing](https://opentelemetry.io/)
- [Performance Profiling](https://docs.python.org/3/library/profile.html)

