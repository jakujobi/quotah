# Quotah - Project Structure

**Last Updated:** December 27, 2025
**Type:** Monorepo (Backend + Frontend)

---

## Overview

Quotah uses a **monorepo structure** with separate backend (Python) and frontend (React) directories. This allows for:
- Independent deployment of frontend and backend
- Clear separation of concerns
- Easy local development
- Shared documentation

---

## Root Directory Structure

```
quotah/
├── backend/                    # Python FastAPI backend
├── frontend/                   # React TypeScript frontend
├── browser-extension/          # Chrome extension (Phase 2)
├── docs/                       # Documentation
│   ├── planning/               # Project planning docs
│   ├── research/               # Market research
│   ├── api/                    # API documentation
│   └── guides/                 # User guides
├── scripts/                    # Utility scripts
│   ├── setup.sh                # Initial setup
│   ├── deploy.sh               # Deployment script
│   └── backup.sh               # Database backup
├── .github/                    # GitHub workflows
│   ├── workflows/
│   │   ├── backend-ci.yml      # Backend CI/CD
│   │   ├── frontend-ci.yml     # Frontend CI/CD
│   │   └── deploy.yml          # Production deployment
│   └── ISSUE_TEMPLATE/
├── docker-compose.yml          # Local development services
├── .gitignore
├── README.md
└── LICENSE
```

---

## Backend Structure

```
backend/
├── app/                        # Main application code
│   ├── __init__.py
│   ├── main.py                 # FastAPI app initialization
│   │
│   ├── api/                    # API routes
│   │   ├── __init__.py
│   │   ├── deps.py             # Dependencies (DB session, auth, etc.)
│   │   ├── v1/                 # API version 1
│   │   │   ├── __init__.py
│   │   │   ├── auth.py         # Auth endpoints
│   │   │   ├── users.py        # User management
│   │   │   ├── api_keys.py     # API key management
│   │   │   ├── multi.py        # Multi-API monitoring
│   │   │   ├── analytics.py    # Analytics endpoints
│   │   │   ├── alerts.py       # Alert management
│   │   │   └── teams.py        # Team management
│   │   └── websocket.py        # WebSocket endpoints
│   │
│   ├── core/                   # Core application config
│   │   ├── __init__.py
│   │   ├── config.py           # Settings (from env vars)
│   │   ├── security.py         # JWT, password hashing
│   │   ├── logging.py          # Logging configuration
│   │   └── middleware.py       # Custom middleware
│   │
│   ├── models/                 # SQLAlchemy models
│   │   ├── __init__.py
│   │   ├── user.py             # User model
│   │   ├── api_key.py          # APIKey model
│   │   ├── usage_event.py      # UsageEvent model
│   │   ├── rate_limit.py       # RateLimitSnapshot model
│   │   ├── alert.py            # Alert, AlertConfig models
│   │   └── team.py             # Team, TeamMember models
│   │
│   ├── schemas/                # Pydantic schemas (request/response)
│   │   ├── __init__.py
│   │   ├── user.py             # User schemas
│   │   ├── api_key.py          # API key schemas
│   │   ├── rate_limit.py       # Rate limit schemas
│   │   ├── alert.py            # Alert schemas
│   │   ├── analytics.py        # Analytics schemas
│   │   └── team.py             # Team schemas
│   │
│   ├── services/               # Business logic
│   │   ├── __init__.py
│   │   ├── user_service.py     # User operations
│   │   ├── unified_api_service.py  # Multi-API orchestration
│   │   ├── alert_service.py    # Alert management
│   │   ├── analytics_service.py    # Analytics calculations
│   │   └── email_service.py    # Email notifications
│   │
│   ├── adapters/               # AI API adapters
│   │   ├── __init__.py
│   │   ├── base.py             # Abstract base adapter
│   │   ├── claude.py           # Claude adapter
│   │   ├── openai.py           # OpenAI adapter
│   │   ├── gemini.py           # Gemini adapter (Phase 2)
│   │   ├── cohere.py           # Cohere adapter (Phase 2)
│   │   └── models.py           # Shared data models
│   │
│   ├── workers/                # Background workers (Celery)
│   │   ├── __init__.py
│   │   ├── celery_app.py       # Celery configuration
│   │   ├── tasks.py            # Task definitions
│   │   └── schedules.py        # Periodic task schedules
│   │
│   └── utils/                  # Utility functions
│       ├── __init__.py
│       ├── encryption.py       # API key encryption
│       ├── validators.py       # Custom validators
│       └── helpers.py          # Helper functions
│
├── tests/                      # Test suite
│   ├── __init__.py
│   ├── conftest.py             # pytest fixtures
│   ├── test_api/               # API endpoint tests
│   │   ├── test_auth.py
│   │   ├── test_api_keys.py
│   │   ├── test_multi.py
│   │   └── test_analytics.py
│   ├── test_services/          # Service tests
│   │   ├── test_user_service.py
│   │   ├── test_unified_api_service.py
│   │   └── test_alert_service.py
│   ├── test_adapters/          # Adapter tests
│   │   ├── test_claude.py
│   │   └── test_openai.py
│   └── test_utils/             # Utility tests
│
├── alembic/                    # Database migrations
│   ├── versions/               # Migration files
│   ├── env.py
│   └── alembic.ini
│
├── .env.example                # Environment variables template
├── .env                        # Local environment variables (gitignored)
├── requirements.txt            # Python dependencies
├── requirements-dev.txt        # Development dependencies
├── Dockerfile                  # Docker image
├── pytest.ini                  # pytest configuration
├── mypy.ini                    # Type checking configuration
└── README.md                   # Backend documentation
```

### Backend File Examples

#### app/main.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings
from app.core.logging import setup_logging
from app.api.v1 import auth, users, api_keys, multi, analytics, alerts

setup_logging()

app = FastAPI(
    title="Quotah API",
    description="Multi-API cost monitoring platform",
    version="1.0.0"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])
app.include_router(api_keys.router, prefix="/api/v1/api-keys", tags=["api-keys"])
app.include_router(multi.router, prefix="/api/v1/multi", tags=["multi-api"])
app.include_router(analytics.router, prefix="/api/v1/analytics", tags=["analytics"])
app.include_router(alerts.router, prefix="/api/v1/alerts", tags=["alerts"])

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

#### app/core/config.py
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # App
    APP_NAME: str = "Quotah"
    DEBUG: bool = False

    # Database
    DATABASE_URL: str

    # Redis
    REDIS_URL: str

    # JWT
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15

    # Encryption
    ENCRYPTION_KEY: str

    # CORS
    CORS_ORIGINS: list[str] = ["http://localhost:5173"]

    # External Services
    SENDGRID_API_KEY: str | None = None
    SENTRY_DSN: str | None = None

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## Frontend Structure

```
frontend/
├── public/                     # Static assets
│   ├── favicon.ico
│   └── robots.txt
│
├── src/
│   ├── components/             # Reusable components
│   │   ├── ui/                 # Base UI components (shadcn/ui)
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Modal.tsx
│   │   │   └── ...
│   │   ├── layout/             # Layout components
│   │   │   ├── Navbar.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── Footer.tsx
│   │   ├── charts/             # Chart components
│   │   │   ├── RateLimitGauge.tsx
│   │   │   ├── CostChart.tsx
│   │   │   └── UsageTrend.tsx
│   │   └── shared/             # Shared components
│   │       ├── RateLimitCard.tsx
│   │       ├── AlertBadge.tsx
│   │       └── LoadingSpinner.tsx
│   │
│   ├── pages/                  # Page components
│   │   ├── LoginPage.tsx
│   │   ├── RegisterPage.tsx
│   │   ├── DashboardPage.tsx
│   │   ├── AnalyticsPage.tsx
│   │   ├── APIKeysPage.tsx
│   │   ├── AlertsPage.tsx
│   │   ├── SettingsPage.tsx
│   │   └── TeamPage.tsx
│   │
│   ├── services/               # API services
│   │   ├── api.ts              # Base API client (axios)
│   │   ├── auth.service.ts     # Auth API calls
│   │   ├── apiKeys.service.ts  # API key management
│   │   ├── multi.service.ts    # Multi-API monitoring
│   │   ├── analytics.service.ts    # Analytics
│   │   └── alerts.service.ts   # Alerts
│   │
│   ├── hooks/                  # Custom React hooks
│   │   ├── useAuth.ts          # Auth context
│   │   ├── useRateLimits.ts    # Rate limit data
│   │   ├── useAnalytics.ts     # Analytics data
│   │   ├── useAlerts.ts        # Alerts
│   │   └── useWebSocket.ts     # WebSocket connection
│   │
│   ├── store/                  # State management (Zustand)
│   │   ├── authStore.ts        # Auth state
│   │   ├── rateLimitStore.ts   # Rate limit state
│   │   └── settingsStore.ts    # User settings
│   │
│   ├── types/                  # TypeScript types
│   │   ├── auth.ts
│   │   ├── apiKey.ts
│   │   ├── rateLimit.ts
│   │   ├── analytics.ts
│   │   └── alert.ts
│   │
│   ├── utils/                  # Utility functions
│   │   ├── formatters.ts       # Date, currency formatting
│   │   ├── validators.ts       # Form validation
│   │   └── helpers.ts          # Helper functions
│   │
│   ├── styles/                 # Global styles
│   │   ├── globals.css         # Global CSS
│   │   └── tailwind.css        # Tailwind imports
│   │
│   ├── App.tsx                 # Root component
│   ├── main.tsx                # Entry point
│   └── routes.tsx              # Route definitions
│
├── .env.example                # Environment variables template
├── .env.local                  # Local environment variables (gitignored)
├── package.json
├── tsconfig.json               # TypeScript configuration
├── vite.config.ts              # Vite configuration
├── tailwind.config.js          # Tailwind configuration
├── postcss.config.js           # PostCSS configuration
└── README.md                   # Frontend documentation
```

### Frontend File Examples

#### src/App.tsx
```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { useAuth } from './hooks/useAuth';
import LoginPage from './pages/LoginPage';
import DashboardPage from './pages/DashboardPage';
import AnalyticsPage from './pages/AnalyticsPage';
import APIKeysPage from './pages/APIKeysPage';
import Layout from './components/layout/Layout';

function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? <>{children}</> : <Navigate to="/login" />;
}

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<LoginPage />} />
        <Route path="/register" element={<RegisterPage />} />

        <Route path="/" element={<ProtectedRoute><Layout /></ProtectedRoute>}>
          <Route index element={<DashboardPage />} />
          <Route path="analytics" element={<AnalyticsPage />} />
          <Route path="api-keys" element={<APIKeysPage />} />
          <Route path="alerts" element={<AlertsPage />} />
          <Route path="settings" element={<SettingsPage />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

#### src/services/api.ts
```tsx
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000/api/v1',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle token expiration
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('access_token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

---

## Browser Extension Structure (Phase 2)

```
browser-extension/
├── src/
│   ├── background/             # Background script
│   │   └── service-worker.ts
│   ├── content/                # Content scripts
│   │   └── inject.ts
│   ├── popup/                  # Extension popup
│   │   ├── Popup.tsx
│   │   └── popup.html
│   ├── options/                # Options page
│   │   ├── Options.tsx
│   │   └── options.html
│   └── shared/                 # Shared utilities
│       ├── api.ts
│       └── storage.ts
├── public/
│   ├── icons/
│   │   ├── icon16.png
│   │   ├── icon48.png
│   │   └── icon128.png
│   └── manifest.json           # Extension manifest
├── package.json
└── README.md
```

---

## Documentation Structure

```
docs/
├── planning/                   # Project planning
│   ├── PROJECT_OVERVIEW.md
│   ├── TECHNICAL_ARCHITECTURE.md
│   ├── DATABASE_SCHEMA.md
│   ├── API_SPECIFICATIONS.md
│   ├── SPRINT_PLAN_12_WEEKS.md
│   ├── PROJECT_STRUCTURE.md
│   ├── TESTING_STRATEGY.md
│   └── DEPLOYMENT_PLAN.md
│
├── research/                   # Market research
│   ├── MASTER_STARTUP_RESEARCH_DOCUMENT.md
│   ├── Complete_Research_Compilation.md
│   └── Multi_API_Expansion_Strategy.md
│
├── api/                        # API documentation
│   ├── authentication.md
│   ├── endpoints.md
│   └── websockets.md
│
└── guides/                     # User guides
    ├── getting-started.md
    ├── adding-api-keys.md
    ├── setting-up-alerts.md
    └── understanding-analytics.md
```

---

## Configuration Files

### docker-compose.yml
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: quotah
      POSTGRES_USER: quotah
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  backend:
    build: ./backend
    command: uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://quotah:dev_password@postgres/quotah
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  celery_worker:
    build: ./backend
    command: celery -A app.workers.celery_app worker --loglevel=info
    volumes:
      - ./backend:/app
    environment:
      DATABASE_URL: postgresql://quotah:dev_password@postgres/quotah
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  celery_beat:
    build: ./backend
    command: celery -A app.workers.celery_app beat --loglevel=info
    volumes:
      - ./backend:/app
    environment:
      DATABASE_URL: postgresql://quotah:dev_password@postgres/quotah
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
```

---

## Naming Conventions

### Python (Backend)

- **Files**: snake_case (e.g., `user_service.py`)
- **Classes**: PascalCase (e.g., `UserService`)
- **Functions**: snake_case (e.g., `get_user_by_id`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_API_KEYS`)
- **Private**: Prefix with `_` (e.g., `_encrypt_key`)

### TypeScript (Frontend)

- **Files**: PascalCase for components (e.g., `DashboardPage.tsx`), camelCase for utilities (e.g., `formatters.ts`)
- **Components**: PascalCase (e.g., `RateLimitCard`)
- **Functions**: camelCase (e.g., `fetchRateLimits`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `API_BASE_URL`)
- **Types/Interfaces**: PascalCase (e.g., `RateLimitInfo`)

### Database

- **Tables**: plural snake_case (e.g., `usage_events`)
- **Columns**: snake_case (e.g., `created_at`)
- **Indexes**: `idx_<table>_<column>` (e.g., `idx_users_email`)
- **Foreign keys**: `fk_<table>_<column>` (e.g., `fk_api_keys_user_id`)

---

## Git Workflow

### Branch Naming

- **Features**: `feature/description` (e.g., `feature/claude-adapter`)
- **Bugs**: `fix/description` (e.g., `fix/login-timeout`)
- **Chores**: `chore/description` (e.g., `chore/update-deps`)
- **Releases**: `release/version` (e.g., `release/1.0.0`)

### Commit Messages

Follow Conventional Commits:

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting)
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Maintenance tasks

**Examples**:
```
feat(api): add Claude adapter for rate limit monitoring
fix(auth): resolve JWT token expiration issue
docs(readme): update setup instructions
```

---

## Conclusion

This project structure is designed to:
- ✅ Scale from MVP to enterprise product
- ✅ Support multiple developers working concurrently
- ✅ Maintain clear separation of concerns
- ✅ Enable fast iteration and deployment
- ✅ Provide excellent developer experience

**Next Steps**:
1. Review and approve structure
2. Generate directory structure
3. Create template files
4. Initialize git repository
5. Set up CI/CD pipeline

---

**Document Status**: ✅ Complete
**Last Updated**: December 27, 2025
