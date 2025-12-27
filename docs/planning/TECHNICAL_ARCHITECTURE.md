# Quotah - Technical Architecture

**Last Updated:** December 27, 2025
**Version:** 1.0
**Status:** Design Phase

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Diagrams](#architecture-diagrams)
3. [Component Details](#component-details)
4. [Data Flow](#data-flow)
5. [Security Architecture](#security-architecture)
6. [Scalability & Performance](#scalability--performance)
7. [Technology Decisions](#technology-decisions)

---

## System Overview

### High-Level Architecture

Quotah follows a **modern 3-tier architecture** with a clear separation between presentation, business logic, and data layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Web App      │  │ Browser      │  │ Mobile (Future)      │  │
│  │ (React)      │  │ Extension    │  │ (React Native)       │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────────────┘  │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │ HTTPS / WSS
                             │
┌─────────────────────────────▼─────────────────────────────────┐
│                     APPLICATION LAYER                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              FastAPI Backend (Python)                   │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │ API Gateway (Rate Limiting, Auth, Logging)        │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │ Service Layer                                     │  │  │
│  │  │ - UnifiedAPIService (Multi-API orchestration)    │  │  │
│  │  │ - UserService (Auth, profile management)         │  │  │
│  │  │ - AlertService (Notifications, thresholds)       │  │  │
│  │  │ - AnalyticsService (Cost analysis, trends)       │  │  │
│  │  │ - RecommendationService (Smart routing, ML)      │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │ API Adapter Layer (Extensible)                    │  │  │
│  │  │ - ClaudeAdapter                                   │  │  │
│  │  │ - OpenAIAdapter                                   │  │  │
│  │  │ - GeminiAdapter                                   │  │  │
│  │  │ - CohereAdapter                                   │  │  │
│  │  │ - MistralAdapter                                  │  │  │
│  │  │ - [Custom API Adapter] (Plugin system)           │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │ Background Workers (Celery)                       │  │  │
│  │  │ - API polling (every 5 min)                       │  │  │
│  │  │ - Alert processing                                │  │  │
│  │  │ - Analytics aggregation                           │  │  │
│  │  │ - Report generation                               │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                        DATA LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ PostgreSQL   │  │ Redis Cache  │  │ S3 / Object Storage  │ │
│  │ (Primary DB) │  │ (Sessions,   │  │ (Logs, Backups,      │ │
│  │              │  │  Rate Limits)│  │  Export Files)       │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                   EXTERNAL SERVICES                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ AI APIs      │  │ Notification │  │ Monitoring           │ │
│  │ (Claude,     │  │ Services     │  │ (Sentry, Posthog)    │ │
│  │  OpenAI,     │  │ (SendGrid,   │  │                      │ │
│  │  Gemini...)  │  │  Slack, etc) │  │                      │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture Diagrams

### Component Interaction Diagram

```
User Request Flow (Example: Get Rate Limits)
─────────────────────────────────────────────

1. User → Web App (React)
2. Web App → FastAPI Backend (GET /api/multi/rate-limits)
3. FastAPI → Auth Middleware (Verify JWT token)
4. Auth → UserService (Get user context)
5. FastAPI → UnifiedAPIService.get_unified_rate_limit_status()
6. UnifiedAPIService → Load active adapters from DB
7. UnifiedAPIService → ClaudeAdapter.get_rate_limit_info()
8. ClaudeAdapter → Anthropic API (minimal request)
9. Anthropic API → ClaudeAdapter (response with headers)
10. ClaudeAdapter → Parse headers, return RateLimitInfo
11. UnifiedAPIService → OpenAIAdapter.get_rate_limit_info()
12. OpenAIAdapter → OpenAI API (minimal request)
13. OpenAI API → OpenAIAdapter (response with headers)
14. OpenAIAdapter → Parse headers, return RateLimitInfo
15. UnifiedAPIService → Aggregate data from all adapters
16. UnifiedAPIService → Cache results in Redis (60s TTL)
17. UnifiedAPIService → Return unified status
18. FastAPI → JSON response to Web App
19. Web App → Render dashboard
```

### Database Schema Relationships

```
┌─────────────┐        ┌──────────────┐        ┌─────────────────┐
│   users     │───────┤   api_keys   │───────┤ usage_events     │
│             │  1:N   │              │  1:N   │                 │
│ - id        │        │ - id         │        │ - id            │
│ - email     │        │ - user_id    │        │ - user_id       │
│ - created   │        │ - provider   │        │ - provider      │
│ - tier      │        │ - key_hash   │        │ - model         │
│ - settings  │        │ - is_active  │        │ - tokens        │
└─────────────┘        └──────────────┘        │ - cost_usd      │
       │                                        │ - timestamp     │
       │ 1:N                                    └─────────────────┘
       │
┌──────▼──────────┐        ┌─────────────────────┐
│  alert_configs  │        │ rate_limit_snapshots│
│                 │        │                     │
│ - id            │        │ - id                │
│ - user_id       │        │ - user_id           │
│ - threshold     │        │ - provider          │
│ - channels      │        │ - tokens_remaining  │
│ - enabled       │        │ - tokens_limit      │
└─────────────────┘        │ - reset_time        │
                           │ - timestamp         │
                           └─────────────────────┘
```

---

## Component Details

### 1. API Gateway Layer

**Purpose**: Entry point for all API requests, handles cross-cutting concerns

**Responsibilities**:
- Request validation
- Authentication & authorization (JWT)
- Rate limiting (per-user, per-endpoint)
- Request/response logging
- CORS handling
- API versioning (v1, v2, etc.)

**Key Middleware**:
```python
# middleware/auth.py
async def auth_middleware(request, call_next):
    """Verify JWT token and inject user context"""

# middleware/rate_limit.py
async def rate_limit_middleware(request, call_next):
    """Rate limit requests per user tier"""

# middleware/logging.py
async def logging_middleware(request, call_next):
    """Log all requests with timing and metadata"""
```

### 2. Service Layer

#### UnifiedAPIService

**Purpose**: Orchestrate monitoring across multiple AI APIs

**Key Methods**:
```python
class UnifiedAPIService:
    async def get_unified_rate_limit_status() -> Dict
    async def get_unified_cost_analysis(days: int) -> Dict
    async def get_smart_routing_recommendation(task_type: str) -> Dict
    async def register_api(provider: str, api_key: str) -> Dict
    async def health_check_all_apis() -> List[Dict]
```

**Design Pattern**: Facade pattern - provides unified interface to multiple subsystems

#### UserService

**Purpose**: Manage user accounts, authentication, subscriptions

**Key Methods**:
```python
class UserService:
    async def register(email: str, password: str) -> User
    async def authenticate(email: str, password: str) -> Token
    async def get_user_profile(user_id: str) -> User
    async def update_subscription(user_id: str, tier: str) -> User
    async def delete_account(user_id: str) -> bool
```

#### AlertService

**Purpose**: Manage alerts and notifications

**Key Methods**:
```python
class AlertService:
    async def create_alert_config(user_id: str, config: AlertConfig) -> AlertConfig
    async def check_thresholds(user_id: str) -> List[Alert]
    async def send_alert(user_id: str, alert: Alert) -> bool
    async def get_alert_history(user_id: str) -> List[Alert]
```

**Notification Channels**:
- Email (SendGrid)
- Slack (Webhooks)
- Discord (Webhooks)
- Push notifications (Firebase)
- SMS (Twilio) - Enterprise only

### 3. API Adapter Layer

**Purpose**: Abstract AI API interactions into a common interface

**Design Pattern**: Adapter pattern + Strategy pattern

**Base Interface**:
```python
class AIAPIAdapter(ABC):
    @abstractmethod
    async def get_rate_limit_info() -> RateLimitInfo

    @abstractmethod
    async def get_usage_history(days: int) -> List[UsageSnapshot]

    @abstractmethod
    async def get_model_pricing(model: str) -> ModelPricing

    @abstractmethod
    def parse_response_for_usage(response) -> UsageSnapshot

    @abstractmethod
    async def validate_api_key() -> bool
```

**Implementation Strategy**:
- Each AI provider gets dedicated adapter class
- Standardized data models (RateLimitInfo, UsageSnapshot, etc.)
- Easy to add new providers (1 file, ~200 lines of code)
- Isolated failure (one API down doesn't break others)

### 4. Background Workers (Celery)

**Purpose**: Handle long-running tasks asynchronously

**Tasks**:

1. **API Polling** (Every 5 minutes)
```python
@celery.task
def poll_rate_limits_for_all_users():
    """Poll AI APIs for all active users"""
    for user in active_users:
        poll_user_apis(user.id)
```

2. **Alert Processing** (Real-time)
```python
@celery.task
def process_alerts(user_id: str):
    """Check thresholds and send alerts"""
    alerts = AlertService.check_thresholds(user_id)
    for alert in alerts:
        AlertService.send_alert(user_id, alert)
```

3. **Analytics Aggregation** (Hourly)
```python
@celery.task
def aggregate_analytics():
    """Compute hourly/daily analytics"""
    compute_cost_trends()
    compute_usage_patterns()
    update_recommendations()
```

4. **Report Generation** (On-demand)
```python
@celery.task
def generate_report(user_id: str, report_type: str):
    """Generate PDF/CSV reports"""
    data = fetch_report_data(user_id, report_type)
    pdf = create_pdf_report(data)
    upload_to_s3(pdf)
    notify_user(user_id, pdf_url)
```

---

## Data Flow

### User Registration Flow

```
1. User submits registration form
2. Frontend validates input
3. POST /api/auth/register
4. UserService.register()
   - Hash password (bcrypt)
   - Create user record in DB
   - Send welcome email
   - Generate JWT token
5. Return JWT to frontend
6. Frontend stores JWT in localStorage
7. Redirect to dashboard
```

### API Key Addition Flow

```
1. User navigates to Settings → Add API
2. User enters API provider + API key
3. POST /api/api-keys
4. UnifiedAPIService.register_api()
   - Validate API key (call provider API)
   - Encrypt API key (Fernet encryption)
   - Store in DB (api_keys table)
   - Initialize adapter
5. Trigger immediate rate limit poll
6. Return success + current status
7. Frontend updates dashboard
```

### Rate Limit Monitoring Flow

```
Background Process (Every 5 minutes):
1. Celery task wakes up
2. Fetch all active users + API keys
3. For each user:
   a. Load API adapters
   b. Call get_rate_limit_info() for each API
   c. Store snapshot in DB
   d. Cache in Redis (1-hour TTL)
   e. Check alert thresholds
   f. Send alerts if needed

User Request (On-demand):
1. GET /api/multi/rate-limits
2. Check Redis cache first
3. If cache miss:
   - Fetch latest snapshots from DB
   - Or trigger immediate poll
4. Return data to frontend
5. Frontend renders dashboard
```

---

## Security Architecture

### Authentication & Authorization

**JWT-Based Authentication**:
- Access token: 15 minutes expiry
- Refresh token: 7 days expiry
- Stored in httpOnly cookies (not localStorage)
- RS256 signing algorithm

**Authorization Levels**:
```python
class UserRole(Enum):
    USER = "user"           # Basic user
    ADMIN = "admin"         # Team admin
    OWNER = "owner"         # Account owner
    SUPPORT = "support"     # Customer support
```

### API Key Security

**Encryption at Rest**:
```python
from cryptography.fernet import Fernet

# Encrypt API keys before storing
def encrypt_api_key(key: str) -> str:
    cipher = Fernet(settings.ENCRYPTION_KEY)
    return cipher.encrypt(key.encode()).decode()

# Decrypt when loading adapters
def decrypt_api_key(encrypted_key: str) -> str:
    cipher = Fernet(settings.ENCRYPTION_KEY)
    return cipher.decrypt(encrypted_key.encode()).decode()
```

**Key Storage**:
- Encryption key stored in environment variables
- Never committed to git
- Rotated every 90 days
- Different keys per environment (dev, staging, prod)

### Data Privacy

**PII Protection**:
- User emails hashed in logs
- API keys never logged
- Usage data anonymized for analytics
- GDPR-compliant data deletion

**Network Security**:
- HTTPS only (TLS 1.3)
- HSTS headers
- Rate limiting per IP
- DDoS protection (Cloudflare)

---

## Scalability & Performance

### Horizontal Scaling

**Stateless Backend**:
- All state in DB or Redis
- No in-memory sessions
- Load balancer friendly
- Can scale to N instances

**Database Scaling**:
```
Phase 1: Single PostgreSQL instance (< 10K users)
Phase 2: Read replicas (10K-100K users)
Phase 3: Sharding by user_id (100K+ users)
```

**Caching Strategy**:

| Data Type | Cache Duration | Storage |
|-----------|---------------|---------|
| Rate limits | 60 seconds | Redis |
| User profiles | 5 minutes | Redis |
| Cost analytics | 15 minutes | Redis |
| API pricing | 24 hours | Redis |
| Session data | 7 days | Redis |

### Performance Targets

**API Response Times**:
- p50: < 100ms
- p95: < 300ms
- p99: < 500ms

**Throughput**:
- 1,000 requests/second (per backend instance)
- 10,000+ concurrent WebSocket connections

**Database Performance**:
- Query p95: < 50ms
- Indexed lookups: < 10ms
- Background jobs: < 1 second

---

## Technology Decisions

### Why FastAPI?

**Pros**:
- ✅ Native async support (critical for I/O-bound workload)
- ✅ Auto-generated OpenAPI docs
- ✅ Type safety with Pydantic
- ✅ Fast (comparable to Node.js, Go)
- ✅ Modern Python ecosystem
- ✅ WebSocket support

**Cons**:
- ⚠️ Smaller ecosystem than Django
- ⚠️ Less built-in admin tools

**Verdict**: ✅ Best choice for API-first, async-heavy workload

### Why PostgreSQL?

**Pros**:
- ✅ ACID compliance
- ✅ JSON support (flexible schema)
- ✅ Excellent performance
- ✅ Mature ecosystem
- ✅ Easy backups/replication

**Cons**:
- ⚠️ Not as simple as SQLite
- ⚠️ Higher operational overhead

**Verdict**: ✅ Industry standard, scales to millions of users

### Why Redis?

**Pros**:
- ✅ Extremely fast (in-memory)
- ✅ Rich data structures
- ✅ TTL support (auto-expiration)
- ✅ Celery integration
- ✅ Pub/Sub for real-time features

**Cons**:
- ⚠️ Memory-limited
- ⚠️ Requires separate service

**Verdict**: ✅ Perfect for caching + background jobs

### Why React?

**Pros**:
- ✅ Largest ecosystem
- ✅ Component reusability
- ✅ TypeScript support
- ✅ Great tooling (Vite, etc.)
- ✅ Easy to hire developers

**Cons**:
- ⚠️ Boilerplate-heavy
- ⚠️ State management complexity

**Verdict**: ✅ Safe, proven choice for SaaS dashboards

---

## Monitoring & Observability

### Application Monitoring

**Sentry** (Error Tracking):
- All uncaught exceptions
- Performance monitoring
- User session replay
- Custom error context

**Posthog** (Product Analytics):
- User behavior tracking
- Feature usage metrics
- Conversion funnels
- A/B testing

### Infrastructure Monitoring

**Metrics** (Prometheus + Grafana):
- API response times
- Database query performance
- Cache hit rates
- Background job queue length
- Memory/CPU usage

**Logs** (CloudWatch / Loki):
- Structured JSON logs
- Centralized log aggregation
- Search and filtering
- Alerting on patterns

### Health Checks

```python
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "database": await check_db_health(),
        "redis": await check_redis_health(),
        "celery": await check_celery_health()
    }

@app.get("/readiness")
async def readiness_check():
    """Check if app is ready to receive traffic"""
    return {
        "ready": all([
            db_connected,
            redis_connected,
            migrations_applied
        ])
    }
```

---

## Disaster Recovery

### Backup Strategy

**Database Backups**:
- Automated daily snapshots (AWS RDS)
- 30-day retention
- Point-in-time recovery (5-minute granularity)
- Cross-region replication

**Redis Backups**:
- RDB snapshots every hour
- AOF (append-only file) for durability
- Can rebuild from PostgreSQL if needed

### Incident Response

**Playbooks**:
1. Database outage → Failover to read replica
2. Redis outage → Degrade gracefully (no cache)
3. API provider outage → Show cached data
4. DDoS attack → Activate Cloudflare protection

---

## Future Architecture Considerations

### Phase 2+ Enhancements

**Multi-Region Deployment**:
- Deploy in US-East, US-West, EU, APAC
- Route users to nearest region
- Data residency compliance

**Microservices (if needed)**:
- Billing service (separate)
- Analytics service (separate)
- Notification service (separate)

**Event-Driven Architecture**:
- Replace Celery with event streaming (Kafka/RabbitMQ)
- Better scalability for high-volume events

**GraphQL API**:
- Alternative to REST for frontend
- Reduce over-fetching
- Better developer experience

---

## Conclusion

This architecture is designed to:
- ✅ **Scale** from 100 to 100,000+ users
- ✅ **Perform** with sub-second response times
- ✅ **Secure** user data and API keys
- ✅ **Extend** easily with new AI APIs
- ✅ **Maintain** with clean separation of concerns

**Next Steps**:
1. Review and approve architecture
2. Set up development environment
3. Implement core backend structure
4. Create database migrations
5. Build first API adapter (Claude)

---

**Document Status**: ✅ Complete
**Reviewers**: Technical Team
**Last Updated**: December 27, 2025
