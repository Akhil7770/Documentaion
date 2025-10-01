# Cost Estimator Calc Service - Technical Documentation

## Executive Summary

The **Cost Estimator Calc Service** is a FastAPI-based microservice designed to calculate healthcare cost estimates for members based on their insurance benefits, accumulators, and negotiated provider rates. The service implements a sophisticated calculation engine using the Chain of Responsibility pattern to handle complex insurance scenarios including deductibles, copays, coinsurance, out-of-pocket maximums, and benefit limitations.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Core Components](#core-components)
4. [Request Flow](#request-flow)
5. [Calculation Chain](#calculation-chain)
6. [Key Features](#key-features)
7. [External Integrations](#external-integrations)
8. [Database Layer](#database-layer)
9. [API Endpoints](#api-endpoints)
10. [Error Handling](#error-handling)
11. [Testing Strategy](#testing-strategy)
12. [Deployment](#deployment)

---

## 1. System Overview

### Purpose
The Cost Estimator Service calculates the estimated out-of-pocket cost a member will pay for a healthcare service by:
- Fetching member benefit information from the Benefit Service
- Retrieving accumulator data (deductibles, OOP max, limits) from the Accumulator Service
- Looking up negotiated rates from Google Cloud Spanner database
- Applying complex insurance calculation logic through a handler chain
- Returning the highest member pay amount among all applicable benefits

### Technology Stack
- **Framework**: FastAPI (Python 3.10+)
- **Database**: Google Cloud Spanner
- **External Services**: Benefit API, Accumulator API
- **Security**: OAuth 2.0 Token-based authentication
- **Deployment**: Docker/Kubernetes
- **Testing**: Pytest, Behave (BDD)

---

## 2. Architecture

### High-Level Architecture

```
┌──────────────┐
│   Client     │
│  Application │
└──────┬───────┘
       │
       │ POST /api/v1/rate
       ▼
┌─────────────────────────────────────────────────────┐
│              FastAPI Application                     │
│  ┌─────────────────────────────────────────────┐   │
│  │         Cost Estimation Service              │   │
│  │                                              │   │
│  │  ┌──────────────┐  ┌──────────────┐        │   │
│  │  │   Benefit    │  │ Accumulator  │        │   │
│  │  │   Service    │  │   Service    │        │   │
│  │  └──────────────┘  └──────────────┘        │   │
│  │                                              │   │
│  │  ┌────────────────────────────────────┐    │   │
│  │  │    Calculation Service             │    │   │
│  │  │  (Chain of Responsibility)         │    │   │
│  │  └────────────────────────────────────┘    │   │
│  │                                              │   │
│  │  ┌────────────────────────────────────┐    │   │
│  │  │    Repository Layer                │    │   │
│  │  └────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
       │                    │                  │
       ▼                    ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Benefit    │  │ Accumulator  │  │   Spanner    │
│     API      │  │     API      │  │   Database   │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Layer Architecture

**1. API Layer (`app/api/`)**
- Handles HTTP requests/responses
- Request validation using Pydantic
- Header processing and logging
- Route management

**2. Service Layer (`app/services/`)**
- Business logic orchestration
- Parallel API calls to external services
- Calculation coordination
- Token management

**3. Handler Layer (`app/services/handlers/`)**
- Chain of Responsibility pattern
- Insurance calculation logic
- Benefit evaluation
- Accumulator processing

**4. Repository Layer (`app/repository/`)**
- Database access abstraction
- Rate lookups
- Query execution
- Circuit breaker implementation

**5. Data Layer (`app/schemas/`, `app/models/`)**
- Request/Response models
- Data validation
- Type definitions

---

## 3. Core Components

### 3.1 Main Application (`app/main.py`)

**Responsibilities:**
- FastAPI application initialization
- Router registration
- Exception handling setup
- Background tasks for cache refresh and token renewal

**Key Features:**
```python
# Token refresh every hour
async def refresh_token_every_hour():
    while True:
        await get_token()
        await asyncio.sleep(3540)  # 59 minutes

# Cache refresh every 24 hours
async def refresh_cache_every_24_hours():
    while True:
        await cost_estimation_service.load_payment_method_hierarchy()
        await cost_estimation_service.load_pcp_specialty_codes()
        await asyncio.sleep(86400)  # 24 hours
```

### 3.2 Cost Estimation Service

**File:** `app/services/impl/cost_estimation_service_impl.py`

**Main Method: `estimate_cost()`**
```python
async def estimate_cost(self, request: CostEstimatorRequest) -> dict:
    1. Map request to benefit request and rate criteria
    2. Parallel execution:
       - Call Benefit Service
       - Call Accumulator Service
       - Fetch rate from database
    3. Match benefits with accumulators
    4. Run calculation chain on matched benefits
    5. Return highest member pay result
```

**Parallel Execution:**
```python
# Uses asyncio.gather for concurrent API calls
benefit_response, accumulator_response, rate = await asyncio.gather(
    benefit_service.get_benefit(benefit_request),
    accumulator_service.get_accumulator(request),
    repository.get_rate(rate_criteria)
)
```

### 3.3 Calculation Service

**File:** `app/services/impl/calculation_service_impl.py`

**Purpose:** Processes multiple benefits through the calculation chain and finds the one with the highest member pay.

**Algorithm:**
```python
def find_highest_member_pay(service_amount, benefits):
    results = []
    
    for benefit in benefits:
        # Create insurance context from benefit
        context = InsuranceContext().populate_from_benefit(benefit, service_amount)
        
        # Process through handler chain
        processed_context = chain.handle(context)
        results.append(processed_context)
    
    # Return context with highest member_pays
    return max(results, key=lambda ctx: ctx.member_pays)
```

### 3.4 Insurance Context (`app/core/base.py`)

**Purpose:** Container for all data needed during calculation.

**Key Fields:**
```python
@dataclass
class InsuranceContext:
    # Input
    service_amount: float
    is_service_covered: bool
    
    # Benefit data
    cost_share_copay: float
    cost_share_coinsurance: float
    copay_applies_oop: bool
    coins_applies_oop: bool
    deductible_applies_oop: bool
    is_deductible_before_copay: bool
    
    # Accumulator data
    oopmax_individual_calculated: float
    oopmax_family_calculated: float
    deductible_individual_calculated: float
    deductible_family_calculated: float
    limit_calculated: float
    
    # Results
    member_pays: float
    calculation_complete: bool
```

---

## 4. Request Flow

### Complete Request-Response Flow

```
1. Client sends POST /api/v1/rate request
   ↓
2. FastAPI validates request with Pydantic
   ↓
3. Request handler calls CostEstimationService.estimate_cost()
   ↓
4. Service maps request to:
   - BenefitRequest (for Benefit API)
   - RateCriteria (for database lookup)
   ↓
5. **PARALLEL EXECUTION** (asyncio.gather):
   ├── Benefit Service → External Benefit API
   ├── Accumulator Service → External Accumulator API
   └── Repository → Spanner Database
   ↓
6. Match benefits with accumulators
   ↓
7. For each matched benefit:
   ├── Create InsuranceContext
   ├── Populate with benefit + accumulator data
   └── Process through Calculation Chain
   ↓
8. Find context with highest member_pays
   ↓
9. Build response with:
   - Rate
   - Member pays amount
   - Benefit response
   - Accumulator response
   - Calculation details
   ↓
10. Return JSON response to client
```

### Example Request
```json
{
  "membershipId": "5~186103331+10+7+20240101+793854+8A+829",
  "zipCode": "85305",
  "benefitProductType": "Medical",
  "languageCode": "11",
  "service": {
    "code": "99214",
    "type": "CPT4",
    "description": "Adult Office visit",
    "placeOfService": {"code": "11"}
  },
  "providerInfo": [{
    "serviceLocation": "000761071",
    "providerType": "HO",
    "specialty": {"code": "91017"},
    "providerNetworks": {"networkID": "58921"},
    "providerIdentificationNumber": "0004000317"
  }]
}
```

---

## 5. Calculation Chain

### Chain of Responsibility Pattern

The calculation engine uses the **Chain of Responsibility** design pattern, where each handler:
1. Processes specific calculation logic
2. Decides whether to continue or stop
3. Passes context to the next handler if needed

### Handler Hierarchy

```
ServiceCoverageHandler
  └─► Is service covered?
      ├─ No → member_pays = service_amount, STOP
      └─ Yes → Continue
          │
          └─► BenefitLimitationHandler
              └─► Has benefit limit?
                  ├─ Yes → Apply limit logic, STOP/CONTINUE
                  └─ No → Continue
                      │
                      └─► OOPMaxHandler
                          └─► Is OOPMax met?
                              ├─ Yes → Apply OOPMax logic, STOP
                              └─ No → Continue
                                  │
                                  └─► DeductibleHandler
                                      └─► Is deductible met?
                                          ├─ No → Apply deductible, STOP
                                          └─ Yes → Continue
                                              │
                                              └─► CostShareCoPayHandler
                                                  └─► Apply copay/coinsurance
```

### 10 Handlers in the Chain

#### 1. **ServiceCoverageHandler**
```python
# First check: Is the service covered?
if not is_service_covered:
    member_pays = service_amount
    insurance_pays = 0
    STOP
```

#### 2. **BenefitLimitationHandler**
```python
# Check if benefit has usage limits (e.g., 1 visit per year)
if has_limit:
    if limit_reached:
        member_pays = service_amount  # Member pays all
    elif service_amount > remaining_limit:
        member_pays = service_amount - remaining_limit  # Partial coverage
    else:
        insurance_pays = service_amount  # Within limit
```

#### 3. **OOPMaxHandler**
```python
# Check if Out-of-Pocket Maximum is met
min_oopmax = min(oopmax_individual, oopmax_family)
if min_oopmax <= 0:
    member_pays = 0  # OOPMax met, insurance covers all
    STOP
```

#### 4. **OOPMaxCopayHandler**
```python
# Handle copay when OOPMax is close to being met
if copay_continues_after_oopmax_met:
    if remaining_oopmax < copay:
        member_pays = remaining_oopmax + remaining_copay
    else:
        member_pays = copay
```

#### 5. **DeductibleHandler**
```python
# Check if deductible is met
min_deductible = min(deductible_individual, deductible_family)
if min_deductible > 0:
    # Deductible not met
    if service_amount <= min_deductible:
        member_pays = service_amount  # Member pays all towards deductible
    else:
        member_pays = min_deductible  # Member pays remaining deductible
    STOP or CONTINUE based on is_deductible_before_copay
```

#### 6. **DeductibleOOPMaxHandler**
```python
# Handle scenario where both deductible and OOPMax apply
if service_amount <= min_deductible:
    member_pays = service_amount
    update_deductible_and_oopmax()
else:
    member_pays = min_deductible
    update_deductible_and_oopmax()
    # Continue to copay/coinsurance
```

#### 7. **DeductibleCostShareCoPayHandler**
```python
# Apply cost sharing when deductible needs to be considered
if is_deductible_before_copay:
    # Apply deductible first, then copay
    pass_to_deductible_handler()
else:
    # Apply copay/coinsurance directly
    pass_to_cost_share_handler()
```

#### 8. **CostShareCoPayHandler**
```python
# Apply copay and/or coinsurance after deductible is met
if has_copay and has_coinsurance:
    member_pays = copay + (service_amount - copay) * coinsurance%
elif has_copay:
    member_pays = min(copay, service_amount)
elif has_coinsurance:
    member_pays = service_amount * coinsurance%
```

#### 9. **DeductibleCoPayHandler**
```python
# Handle copay in deductible scenarios
if copay_counts_to_deductible:
    apply_copay_to_deductible()
if copay_continues_when_deductible_met:
    continue_charging_copay()
```

#### 10. **DeductibleCoInsuranceHandler**
```python
# Apply coinsurance in deductible scenarios
coinsurance_amount = remaining_service_amount * coinsurance%
member_pays += coinsurance_amount
update_oopmax_if_applies()
```

### Calculation Flow Decision Tree

```
Start
  │
  ├─ Service Not Covered? → YES → member_pays = service_amount → END
  │                       → NO ↓
  │
  ├─ Has Benefit Limit? → YES → Check limit remaining → Apply limit logic
  │                     → NO ↓
  │
  ├─ OOPMax Met? → YES → member_pays = 0 or minimal → END
  │              → NO ↓
  │
  ├─ Deductible Met? → NO → member_pays += deductible_amount → Check if continues
  │                  → YES ↓
  │
  ├─ Has Copay? → YES → member_pays += copay_amount
  │             → NO ↓
  │
  ├─ Has Coinsurance? → YES → member_pays += service * coinsurance%
  │                   → NO ↓
  │
  └─ Calculate Final Amount → Apply OOPMax updates → END
```

---

## 6. Key Features

### 6.1 Parallel Processing
```python
# Concurrent calls to external services
async with asyncio.TaskGroup() as tg:
    benefit_task = tg.create_task(benefit_service.get_benefit())
    accumulator_task = tg.create_task(accumulator_service.get_accumulator())
    rate_task = tg.create_task(repository.get_rate())
```

**Benefits:**
- Reduces total API response time
- Improves user experience
- Efficient resource utilization

### 6.2 Token Management
```python
# Automatic token refresh every hour
class SessionManager:
    _token = None
    
    @classmethod
    def set_token(cls, token_data):
        cls._token = token_data
    
    @classmethod
    def get_token(cls):
        return cls._token
```

### 6.3 Circuit Breaker Pattern
```python
@circuit_breaker("rate_repository")
@retry(stop=stop_after_attempt(3), 
       wait=wait_exponential(multiplier=1, min=4, max=10))
async def get_rate(self, rate_criteria):
    # Database call with automatic retry and circuit breaking
```

**Benefits:**
- Prevents cascade failures
- Automatic retry with exponential backoff
- System resilience

### 6.4 Benefit-Accumulator Matching
```python
def match_benefits_with_accumulators(benefits, accumulators):
    matched_benefits = []
    
    for benefit in benefits:
        # Match by network, code, level
        matched_accums = find_matching_accumulators(
            benefit, 
            accumulators, 
            network=benefit.networkCategory
        )
        
        if matched_accums:
            benefit.coverage.matchedAccumulators = matched_accums
            matched_benefits.append(benefit)
    
    return matched_benefits
```

### 6.5 Caching
```python
# Payment method hierarchy cache (refreshed every 24 hours)
async def load_payment_method_hierarchy(self):
    query = "SELECT * FROM payment_method_hierarchy"
    results = await self.db.execute_query(query)
    self._cache[PAYMENT_METHOD_HIERARCHY_CACHE_KEY] = results
```

---

## 7. External Integrations

### 7.1 Benefit Service Integration

**Endpoint:** `POST https://benefit-api.com/servicesbenefits/retrieve`

**Purpose:** Fetch member benefit information

**Request Mapping:**
```python
def to_benefit_request(cost_request):
    return BenefitRequest(
        benefitProductType=cost_request.benefitProductType,
        membershipID=cost_request.membershipId,
        planIdentifier="3~",
        serviceInfo=[{
            "serviceCodeInfo": {
                "code": cost_request.service.code,
                "type": cost_request.service.type,
                "providerType": [{"code": provider.providerType}],
                "placeOfService": [{"code": cost_request.service.placeOfService.code}]
            }
        }]
    )
```

**Response Data:**
- Benefit details (copay, coinsurance, deductible)
- Coverage information
- Network category
- Service provider details

### 7.2 Accumulator Service Integration

**Endpoint:** `GET https://accumulator-api.com/memberships/{id}/accumulators`

**Purpose:** Retrieve current accumulator values

**Query Parameters:**
```python
params = {
    "benefitProductType": "Medical",
    "networkType": "InNetwork"
}
```

**Response Data:**
- Deductible (Individual/Family)
- Out-of-Pocket Maximum (Individual/Family)
- Benefit Limits
- Current values and remaining amounts

### 7.3 Authentication

**OAuth 2.0 Token Flow:**
```python
class TokenService:
    async def get_new_token(self):
        # Call token endpoint
        response = await http_client.post(
            TOKEN_URL,
            data={
                "grant_type": "client_credentials",
                "client_id": CLIENT_ID,
                "client_secret": CLIENT_SECRET
            }
        )
        
        return response.json()  # access_token, expires_in
```

---

## 8. Database Layer

### 8.1 Google Cloud Spanner

**Configuration:**
```python
class SpannerConfig:
    project_id: str = os.getenv("SPANNER_PROJECT_ID")
    instance_id: str = os.getenv("SPANNER_INSTANCE_ID")
    database_id: str = os.getenv("SPANNER_DATABASE_ID")
```

### 8.2 Rate Lookup Strategy

**Hierarchical Rate Lookup:**

```
1. Claim-Based Rate (highest priority)
   ↓ (if not found)
2. Provider Information Lookup
   ↓
3. Contract Type Determination
   ↓
4. Standard Rate or Non-Standard Rate
   ↓ (if not found)
5. Default Rate (last resort)
```

**Query Example:**
```sql
SELECT 
    rate_amount, 
    rate_type, 
    payment_method_code
FROM 
    negotiated_rates
WHERE 
    provider_id = @provider_id
    AND service_code = @service_code
    AND network_id = @network_id
    AND effective_date <= @service_date
    AND (expiration_date IS NULL OR expiration_date >= @service_date)
ORDER BY 
    rate_priority DESC
LIMIT 1
```

### 8.3 Rate Types

**1. Percentage Rate:**
```python
if payment_method_code in PERCENTAGE_PAYMENT_METHOD_CODES:
    # Rate is a percentage of billed amount
    actual_rate = service_amount * (rate_amount / 100)
```

**2. Amount Rate:**
```python
else:
    # Rate is a fixed dollar amount
    actual_rate = rate_amount
```

---

## 9. API Endpoints

### 9.1 POST /api/v1/rate

**Purpose:** Calculate cost estimate

**Request Headers:**
```
Content-Type: application/json
x-global-transaction-id: <correlation-id>
x-clientrefid: <client-reference-id>
```

**Response Structure:**
```json
{
  "status": "success",
  "rate": 150.00,
  "member_pays": 50.00,
  "insurance_pays": 100.00,
  "benefit_response": { /* Benefit API response */ },
  "accumulator_response": { /* Accumulator API response */ },
  "calculation_details": {
    "benefit_id": "BEN123",
    "copay": 25.00,
    "coinsurance": 0.20,
    "deductible_applied": 25.00,
    "oopmax_remaining": 2500.00
  }
}
```

### 9.2 POST /api/v1/rate-only

**Purpose:** Get negotiated rate without calculation

**Response:**
```json
{
  "rates": [
    {
      "provider_id": "0004000317",
      "service_code": "99214",
      "rate": 150.00,
      "rate_type": "Amount",
      "contract_type": "Standard"
    }
  ]
}
```

### 9.3 GET /health

**Purpose:** Health check endpoint

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2025-10-01T12:00:00Z"
}
```

---

## 10. Error Handling

### Exception Hierarchy

```python
class CostEstimatorException(Exception):
    """Base exception"""

class BenefitServiceException(CostEstimatorException):
    """Benefit API errors"""

class AccumulatorServiceException(CostEstimatorException):
    """Accumulator API errors"""

class DatabaseException(CostEstimatorException):
    """Database errors"""

class ValidationException(CostEstimatorException):
    """Request validation errors"""
```

### Global Exception Handler

```python
@app.exception_handler(CostEstimatorException)
async def custom_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "status": "error",
            "error_code": exc.error_code,
            "message": exc.message,
            "timestamp": datetime.now().isoformat()
        }
    )
```

### HTTP Status Codes

| Code | Description | When Used |
|------|-------------|-----------|
| 200 | OK | Successful calculation |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Authentication failure |
| 422 | Unprocessable Entity | Validation error |
| 500 | Internal Server Error | Unexpected error |
| 502 | Bad Gateway | External service failure |
| 503 | Service Unavailable | System overload |
| 504 | Gateway Timeout | External service timeout |

---

## 11. Testing Strategy

### 11.1 Unit Tests

**Location:** `tests/services/`, `tests/handlers/`

**Example:**
```python
def test_service_coverage_handler():
    context = InsuranceContext(
        service_amount=100.0,
        is_service_covered=False
    )
    
    handler = ServiceCoverageHandler()
    result = handler.handle(context)
    
    assert result.member_pays == 100.0
    assert result.calculation_complete == True
```

### 11.2 Integration Tests

**Location:** `tests/api/`

**Example:**
```python
@pytest.mark.asyncio
async def test_estimate_cost_endpoint():
    request = CostEstimatorRequest(...)
    
    with mock.patch('benefit_service.get_benefit'):
        with mock.patch('accumulator_service.get_accumulator'):
            response = await client.post("/api/v1/rate", json=request.dict())
            
    assert response.status_code == 200
    assert "rate" in response.json()
```

### 11.3 BDD Tests (Behave)

**Location:** `tests/features/`

**Example Feature:**
```gherkin
Feature: Copay Calculation

  Scenario: Apply copay when deductible is met
    Given a member with medical benefits
    And the member has met their deductible
    And the copay is $25.00
    When a service amount of $150 is processed
    Then the member pays $25.00
    And the insurance pays $125.00
```

### 11.4 Scenario-Based Testing

**CSV-Driven Test Framework:**
- 49 test scenarios covering all calculation paths
- Scenarios defined in CSV files by category:
  - Service Coverage (2 scenarios)
  - Benefit Limitations (5 scenarios)
  - OOPMax (6 scenarios)
  - Deductible (19 scenarios)
  - Copay (11 scenarios)
  - Coinsurance (6 scenarios)

**Dynamic Test Generation:**
```python
# Tests are automatically generated from CSV
for scenario in scenarios:
    test_func = create_test_function(scenario)
    globals()[f"test_scenario_{scenario.id}"] = test_func
```

---

## 12. Deployment

### 12.1 Docker

**Dockerfile:**
```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY Pipfile Pipfile.lock ./
RUN pip install pipenv && pipenv install --system --deploy

COPY app/ ./app/

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 12.2 Kubernetes

**Deployment Configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cost-estimator-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cost-estimator
  template:
    spec:
      containers:
      - name: cost-estimator
        image: cost-estimator:latest
        ports:
        - containerPort: 8000
        env:
        - name: SPANNER_PROJECT_ID
          valueFrom:
            secretKeyRef:
              name: spanner-config
              key: project-id
```

### 12.3 CI/CD Pipeline

**Jenkinsfile:**
```groovy
pipeline {
    stages {
        stage('Test') {
            steps {
                sh 'pipenv run pytest'
                sh 'pipenv run behave'
            }
        }
        stage('Build') {
            steps {
                sh 'docker build -t cost-estimator:${BUILD_NUMBER} .'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}
```

---

## Appendix A: Key Calculation Scenarios

### Scenario 1: No Deductible, Simple Copay
```
Service Amount: $900
Deductible: $0 (met)
Copay: $25
Coinsurance: 0%

Result: Member Pays = $25
```

### Scenario 2: Deductible Not Met
```
Service Amount: $900
Remaining Deductible: $500
Copay: $25
Coinsurance: 20%

Calculation:
1. Apply deductible: $500
2. Remaining: $400
3. Apply coinsurance: $400 * 20% = $80

Result: Member Pays = $580
```

### Scenario 3: OOPMax Met
```
Service Amount: $900
OOPMax Family: $0 (met)
Copay: $100

Result: Member Pays = $0 (OOPMax met, no copay)
```

### Scenario 4: Benefit Limit Partial
```
Service Amount: $900
Benefit Limit Remaining: $600
Copay: $25

Calculation:
1. Insurance covers: $600 (limit)
2. Member pays excess: $900 - $600 = $300

Result: Member Pays = $300
```

### Scenario 5: Complex - Deductible + Copay + OOPMax
```
Service Amount: $900
Remaining Deductible: $500
Copay: $100
Remaining OOPMax: $70
Copay continues after OOPMax: No

Calculation:
1. Apply deductible: $500 → member_pays = $500
2. Remaining amount: $400
3. Copay = $100, but OOPMax remaining = $70
4. Member pays: min(copay, remaining_oopmax) = $70

Result: Member Pays = $570 ($500 deductible + $70 OOPMax)
```

---

## Appendix B: Project Structure

```
cost-estimator-calc-service/
├── app/
│   ├── api/
│   │   ├── router.py                    # Main API router
│   │   ├── routes/
│   │   │   └── index.py                 # Health check routes
│   │   └── v1/
│   │       └── routes/
│   │           └── requests.py          # Cost estimation endpoints
│   ├── config/
│   │   ├── database_config.py           # Spanner configuration
│   │   ├── queries.py                   # SQL query definitions
│   │   └── queries/                     # SQL files
│   ├── core/
│   │   ├── base.py                      # Handler base class, InsuranceContext
│   │   ├── constants.py                 # Application constants
│   │   ├── logger.py                    # Logging configuration
│   │   ├── session_manager.py           # Token management
│   │   ├── service_factory.py           # Dependency injection
│   │   └── circuitbreaker/
│   │       └── circuit_breaker.py       # Circuit breaker implementation
│   ├── database/
│   │   └── spanner_client.py            # Spanner database client
│   ├── exception/
│   │   ├── exceptions.py                # Custom exceptions
│   │   └── exception_handler.py         # Global exception handling
│   ├── mappers/
│   │   └── cost_estimator_mapper.py     # Request/Response mapping
│   ├── models/
│   │   ├── rate_criteria.py             # Rate lookup models
│   │   └── selected_benefit.py          # Benefit models
│   ├── repository/
│   │   ├── cost_estimator_repository.py # Repository interface
│   │   └── impl/
│   │       └── cost_estimator_repository_impl.py  # Repository implementation
│   ├── schemas/
│   │   ├── cost_estimator_request.py    # Request schema
│   │   ├── benefit_request.py           # Benefit API request
│   │   ├── benefit_response.py          # Benefit API response
│   │   ├── accumulator_request.py       # Accumulator API request
│   │   └── accumulator_response.py      # Accumulator API response
│   ├── services/
│   │   ├── cost_estimation_service.py   # Service interface
│   │   ├── calculation_service.py       # Calculation interface
│   │   ├── benefit_service.py           # Benefit service interface
│   │   ├── accumulator_service.py       # Accumulator service interface
│   │   ├── token_service.py             # Token service
│   │   ├── benefit_accumulator_matcher_service.py  # Matching logic
│   │   ├── handlers/                    # Calculation handlers
│   │   │   ├── service_coverage_handler.py
│   │   │   ├── benefit_limitation_handler.py
│   │   │   ├── oopmax_handler.py
│   │   │   ├── oopmax_co_pay_handler.py
│   │   │   ├── deductible_handler.py
│   │   │   ├── deductible_oopmax_handler.py
│   │   │   ├── deductible_co_pay_handler.py
│   │   │   ├── deductible_co_insurance_handler.py
│   │   │   ├── deductible_cost_share_co_pay.py
│   │   │   └── cost_share_co_pay_handler.py
│   │   └── impl/                        # Service implementations
│   │       ├── cost_estimation_service_impl.py
│   │       ├── calculation_service_impl.py
│   │       ├── benefit_service_impl.py
│   │       ├── accumulator_service_impl.py
│   │       └── benefit_accumulator_matcher_service_impl.py
│   └── main.py                          # FastAPI application entry point
├── tests/
│   ├── api/                             # API integration tests
│   ├── features/                        # BDD tests (Behave)
│   │   ├── handlers/                    # Feature files
│   │   └── steps/                       # Step definitions
│   ├── mock-data/                       # Test data
│   │   └── mock-api-test-data/
│   │       ├── csv-files/               # Scenario definitions
│   │       └── responses/               # Expected responses
│   ├── services/                        # Service unit tests
│   ├── repository/                      # Repository tests
│   └── conftest.py                      # Pytest configuration
├── pipeline/
│   ├── values.dev.yaml                  # Dev environment config
│   ├── values.test.yaml                 # Test environment config
│   └── values.prod.yaml                 # Prod environment config
├── Dockerfile                            # Docker image definition
├── Pipfile                               # Python dependencies
├── pytest.ini                            # Pytest configuration
├── behave.ini                            # Behave configuration
├── jenkinsfile.yaml                      # CI/CD pipeline
├── README.md                             # Project documentation
└── .gitignore                            # Git ignore rules
```

---

## Glossary

**Accumulator:** A running total of healthcare expenses or services used during a benefit period.

**Benefit:** The specific coverage provided by an insurance plan for a healthcare service.

**Coinsurance:** The percentage of costs that a member pays after the deductible is met (e.g., 20%).

**Copay:** A fixed amount that a member pays for a covered service (e.g., $25).

**Deductible:** The amount a member must pay out-of-pocket before insurance coverage begins.

**OOP Max (Out-of-Pocket Maximum):** The maximum amount a member pays during a benefit period before the plan pays 100%.

**Negotiated Rate:** The agreed-upon price between the insurance company and healthcare provider.

**Network:** The group of healthcare providers contracted with an insurance plan (In-Network or Out-of-Network).

**Service Amount:** The billed charge for a healthcare service.

**Member Pays:** The calculated amount the member is responsible for paying.

**Insurance Pays:** The amount the insurance company will cover.

---

## Document Information

**Version:** 1.0  
**Last Updated:** October 1, 2025  
**Author:** Technical Documentation Team  
**Status:** Final

---

**End of Document**
