# Quotah - API Specifications

**Last Updated:** December 27, 2025
**API Version:** v1
**Base URL**: `https://api.quotah.com/v1`

---

## Table of Contents

1. [Authentication](#authentication)
2. [Core Endpoints](#core-endpoints)
3. [Multi-API Monitoring](#multi-api-monitoring)
4. [Analytics & Reporting](#analytics--reporting)
5. [Alerts & Notifications](#alerts--notifications)
6. [Team Management](#team-management)
7. [WebSocket API](#websocket-api)
8. [Error Handling](#error-handling)

---

## Authentication

### JWT Token-Based Authentication

**Authentication Flow**:
1. User logs in with email/password
2. Server returns JWT access token (15 min expiry) + refresh token (7 days)
3. Client includes access token in Authorization header
4. When access token expires, use refresh token to get new access token

**Headers**:
```http
Authorization: Bearer <access_token>
Content-Type: application/json
```

### Endpoints

#### POST /auth/register

Register a new user account.

**Request**:
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "name": "John Doe"
}
```

**Response** (201 Created):
```json
{
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "name": "John Doe",
    "tier": "free",
    "created_at": "2025-12-27T10:00:00Z"
  },
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_in": 900
}
```

**Errors**:
- `400` - Invalid email format
- `409` - Email already exists
- `422` - Password too weak

#### POST /auth/login

Authenticate user and receive tokens.

**Request**:
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response** (200 OK):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_in": 900,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "tier": "pro"
  }
}
```

**Errors**:
- `401` - Invalid credentials
- `403` - Account suspended

#### POST /auth/refresh

Refresh access token using refresh token.

**Request**:
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response** (200 OK):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_in": 900
}
```

#### POST /auth/logout

Invalidate refresh token.

**Request**:
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response** (204 No Content)

---

## Core Endpoints

### User Management

#### GET /users/me

Get current user profile.

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "name": "John Doe",
  "tier": "pro",
  "created_at": "2025-12-27T10:00:00Z",
  "settings": {
    "notifications": {
      "email": true,
      "slack": false
    },
    "dashboard": {
      "default_view": "overview",
      "currency": "USD"
    }
  },
  "usage": {
    "api_keys_count": 3,
    "api_keys_limit": 5,
    "checks_this_month": 45000,
    "checks_limit": 100000
  }
}
```

#### PATCH /users/me

Update user profile.

**Request**:
```json
{
  "name": "John Smith",
  "settings": {
    "notifications": {
      "email": true,
      "slack": true
    }
  }
}
```

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "name": "John Smith",
  "settings": {
    "notifications": {
      "email": true,
      "slack": true
    }
  }
}
```

### API Key Management

#### GET /api-keys

List all API keys for current user.

**Query Parameters**:
- `provider` (optional): Filter by provider (claude, openai, etc.)
- `is_active` (optional): Filter by active status

**Response** (200 OK):
```json
{
  "api_keys": [
    {
      "id": "key-1",
      "provider": "claude",
      "key_name": "Production",
      "is_active": true,
      "created_at": "2025-12-27T10:00:00Z",
      "last_used_at": "2025-12-27T14:30:00Z",
      "status": "healthy"
    },
    {
      "id": "key-2",
      "provider": "openai",
      "key_name": "Testing",
      "is_active": true,
      "created_at": "2025-12-26T09:00:00Z",
      "last_used_at": "2025-12-27T13:00:00Z",
      "status": "healthy"
    }
  ],
  "total": 2
}
```

#### POST /api-keys

Add a new API key.

**Request**:
```json
{
  "provider": "claude",
  "key_name": "Production",
  "api_key": "sk-ant-api03-..."
}
```

**Response** (201 Created):
```json
{
  "id": "key-1",
  "provider": "claude",
  "key_name": "Production",
  "is_active": true,
  "created_at": "2025-12-27T10:00:00Z",
  "validation": {
    "status": "valid",
    "tier": "tier_2",
    "current_limits": {
      "rpm": 1000,
      "tpm": 100000
    }
  }
}
```

**Errors**:
- `400` - Invalid API key format
- `401` - API key validation failed
- `403` - User has reached API key limit
- `409` - API key already exists

#### DELETE /api-keys/{key_id}

Remove an API key.

**Response** (204 No Content)

**Errors**:
- `404` - API key not found
- `403` - Not authorized to delete this key

---

## Multi-API Monitoring

### GET /multi/rate-limits

Get unified rate limit status across all connected APIs.

**Query Parameters**:
- `providers` (optional): Comma-separated list of providers

**Response** (200 OK):
```json
{
  "timestamp": "2025-12-27T15:00:00Z",
  "providers": {
    "claude": {
      "status": "healthy",
      "percentage_used": 35.2,
      "tokens_remaining": 648000,
      "tokens_limit": 1000000,
      "requests_remaining": 9500,
      "requests_limit": 10000,
      "reset_time": "2025-12-27T15:01:00Z",
      "reset_type": "rolling_window",
      "recommendation": "✓ Claude healthy with 648,000 tokens remaining"
    },
    "openai": {
      "status": "warning",
      "percentage_used": 82.5,
      "tokens_remaining": 35000,
      "tokens_limit": 200000,
      "requests_remaining": 500,
      "requests_limit": 3500,
      "reset_time": "2025-12-27T15:05:00Z",
      "reset_type": "rolling_window",
      "recommendation": "⚠️ WARNING: Approaching OpenAI limit. Reduce usage or switch APIs."
    }
  },
  "summary": {
    "healthy": 1,
    "warning": 1,
    "critical": 0,
    "error": 0
  }
}
```

### GET /multi/cost-analysis

Get cost analysis across all APIs.

**Query Parameters**:
- `days` (optional, default: 7): Number of days to analyze
- `group_by` (optional, default: provider): Group by provider, model, or day

**Response** (200 OK):
```json
{
  "timestamp": "2025-12-27T15:00:00Z",
  "period_days": 7,
  "total_cost": 45.67,
  "by_provider": {
    "claude": {
      "cost": 25.30,
      "tokens": 8500000,
      "requests": 1250,
      "cost_per_1m_tokens": 2.98
    },
    "openai": {
      "cost": 20.37,
      "tokens": 5200000,
      "requests": 850,
      "cost_per_1m_tokens": 3.92
    }
  },
  "cost_per_provider": [
    {
      "provider": "claude",
      "cost": 25.30,
      "tokens": 8500000,
      "requests": 1250,
      "cost_per_1m_tokens": 2.98
    },
    {
      "provider": "openai",
      "cost": 20.37,
      "tokens": 5200000,
      "requests": 850,
      "cost_per_1m_tokens": 3.92
    }
  ],
  "cheapest_provider": {
    "provider": "claude",
    "cost_per_1m_tokens": 2.98
  },
  "most_expensive_provider": {
    "provider": "openai",
    "cost_per_1m_tokens": 3.92
  },
  "optimization_opportunities": [
    {
      "type": "model_downgrade",
      "priority": "high",
      "current_model": "gpt-4-turbo",
      "recommendation": "Switch to GPT-4o-mini for simple tasks",
      "estimated_savings": 8.50,
      "reason": "GPT-4 Turbo is 20x more expensive but only marginally better for your use case"
    },
    {
      "type": "batch_processing",
      "priority": "medium",
      "recommendation": "Batch similar requests to reduce API calls",
      "potential_savings": "15-30%"
    }
  ]
}
```

### GET /multi/recommendations

Get smart routing recommendations.

**Query Parameters**:
- `task_type` (optional, default: balanced): cost_optimization, balanced, quality_first, speed_first

**Response** (200 OK):
```json
{
  "task_type": "balanced",
  "timestamp": "2025-12-27T15:00:00Z",
  "recommendations": [
    {
      "rank": 1,
      "provider": "claude",
      "model": "claude-3-5-sonnet-20241022",
      "estimated_cost_per_request": 0.0045,
      "estimated_tokens_per_request": 1500,
      "quality_rating": 8.8,
      "speed_rating": 8.5,
      "context_window": 200000,
      "supports_batching": true,
      "recommendation_score": 0.87
    },
    {
      "rank": 2,
      "provider": "openai",
      "model": "gpt-4o",
      "estimated_cost_per_request": 0.0052,
      "estimated_tokens_per_request": 1600,
      "quality_rating": 9.0,
      "speed_rating": 8.0,
      "context_window": 128000,
      "supports_batching": true,
      "recommendation_score": 0.84
    }
  ]
}
```

---

## Analytics & Reporting

### GET /analytics/usage

Get detailed usage analytics.

**Query Parameters**:
- `start_date` (required): ISO 8601 date
- `end_date` (required): ISO 8601 date
- `provider` (optional): Filter by provider
- `granularity` (optional, default: day): hour, day, week, month

**Response** (200 OK):
```json
{
  "period": {
    "start": "2025-12-20T00:00:00Z",
    "end": "2025-12-27T00:00:00Z"
  },
  "granularity": "day",
  "data": [
    {
      "date": "2025-12-27",
      "providers": {
        "claude": {
          "tokens": 1250000,
          "cost": 3.75,
          "requests": 150
        },
        "openai": {
          "tokens": 850000,
          "cost": 4.20,
          "requests": 100
        }
      },
      "total_tokens": 2100000,
      "total_cost": 7.95,
      "total_requests": 250
    }
  ],
  "summary": {
    "total_cost": 45.67,
    "total_tokens": 13700000,
    "total_requests": 1850,
    "average_cost_per_day": 6.52,
    "projected_monthly_cost": 195.60
  }
}
```

### GET /analytics/trends

Get usage trends and predictions.

**Response** (200 OK):
```json
{
  "timestamp": "2025-12-27T15:00:00Z",
  "trends": {
    "cost": {
      "current_week": 45.67,
      "previous_week": 38.20,
      "change_percent": 19.55,
      "trend": "increasing"
    },
    "tokens": {
      "current_week": 13700000,
      "previous_week": 11200000,
      "change_percent": 22.32,
      "trend": "increasing"
    }
  },
  "predictions": {
    "next_7_days_cost": 52.30,
    "next_30_days_cost": 198.50,
    "confidence": 0.85
  },
  "alerts": [
    {
      "type": "budget_warning",
      "message": "Projected to exceed monthly budget of $150 by $48.50",
      "severity": "warning"
    }
  ]
}
```

### POST /analytics/export

Generate and download usage report.

**Request**:
```json
{
  "format": "csv",
  "start_date": "2025-12-01",
  "end_date": "2025-12-31",
  "include": ["usage_events", "cost_summary", "rate_limits"]
}
```

**Response** (202 Accepted):
```json
{
  "job_id": "export-123",
  "status": "processing",
  "estimated_completion": "2025-12-27T15:05:00Z"
}
```

**Poll for completion**:
```
GET /analytics/export/{job_id}
```

**Response when ready** (200 OK):
```json
{
  "job_id": "export-123",
  "status": "completed",
  "download_url": "https://quotah-exports.s3.amazonaws.com/...",
  "expires_at": "2025-12-28T15:00:00Z"
}
```

---

## Alerts & Notifications

### GET /alerts/configs

Get alert configurations.

**Response** (200 OK):
```json
{
  "configs": [
    {
      "id": "alert-1",
      "alert_type": "rate_limit",
      "threshold_percent": 80,
      "enabled": true,
      "channels": [
        {
          "type": "email",
          "address": "user@example.com",
          "enabled": true
        },
        {
          "type": "slack",
          "webhook_url": "https://hooks.slack.com/...",
          "channel": "#ai-monitoring",
          "enabled": true
        }
      ],
      "created_at": "2025-12-27T10:00:00Z"
    }
  ]
}
```

### POST /alerts/configs

Create alert configuration.

**Request**:
```json
{
  "alert_type": "cost",
  "threshold_percent": 90,
  "enabled": true,
  "channels": [
    {
      "type": "email",
      "address": "user@example.com"
    }
  ]
}
```

**Response** (201 Created):
```json
{
  "id": "alert-2",
  "alert_type": "cost",
  "threshold_percent": 90,
  "enabled": true,
  "channels": [
    {
      "type": "email",
      "address": "user@example.com",
      "enabled": true
    }
  ],
  "created_at": "2025-12-27T15:00:00Z"
}
```

### GET /alerts/history

Get alert history.

**Query Parameters**:
- `limit` (optional, default: 50)
- `severity` (optional): info, warning, critical
- `acknowledged` (optional): true, false

**Response** (200 OK):
```json
{
  "alerts": [
    {
      "id": "alert-log-1",
      "alert_type": "rate_limit",
      "severity": "warning",
      "message": "OpenAI rate limit at 85% - 30K tokens remaining",
      "provider": "openai",
      "sent_at": "2025-12-27T14:30:00Z",
      "acknowledged_at": null,
      "metadata": {
        "percentage_used": 85,
        "tokens_remaining": 30000
      }
    }
  ],
  "total": 1,
  "unacknowledged_count": 1
}
```

### POST /alerts/{alert_id}/acknowledge

Mark alert as acknowledged.

**Response** (200 OK):
```json
{
  "id": "alert-log-1",
  "acknowledged_at": "2025-12-27T15:00:00Z"
}
```

---

## Team Management

### GET /teams

List teams for current user.

**Response** (200 OK):
```json
{
  "teams": [
    {
      "id": "team-1",
      "name": "Engineering Team",
      "role": "owner",
      "tier": "enterprise",
      "member_count": 12,
      "created_at": "2025-10-01T00:00:00Z"
    }
  ]
}
```

### POST /teams

Create a new team.

**Request**:
```json
{
  "name": "Engineering Team",
  "billing_email": "billing@company.com"
}
```

**Response** (201 Created):
```json
{
  "id": "team-1",
  "name": "Engineering Team",
  "owner_id": "550e8400-e29b-41d4-a716-446655440000",
  "tier": "team",
  "billing_email": "billing@company.com",
  "created_at": "2025-12-27T15:00:00Z"
}
```

### GET /teams/{team_id}/members

Get team members.

**Response** (200 OK):
```json
{
  "members": [
    {
      "id": "member-1",
      "user_id": "user-1",
      "email": "john@company.com",
      "name": "John Doe",
      "role": "owner",
      "joined_at": "2025-10-01T00:00:00Z",
      "usage_last_30_days": {
        "cost": 125.50,
        "tokens": 42000000,
        "requests": 5500
      }
    }
  ],
  "total": 1
}
```

### POST /teams/{team_id}/members

Invite a team member.

**Request**:
```json
{
  "email": "newmember@company.com",
  "role": "member"
}
```

**Response** (201 Created):
```json
{
  "id": "member-2",
  "email": "newmember@company.com",
  "role": "member",
  "invite_sent_at": "2025-12-27T15:00:00Z",
  "status": "pending"
}
```

---

## WebSocket API

### Real-Time Updates

Connect to WebSocket for real-time rate limit updates.

**URL**: `wss://api.quotah.com/v1/ws`

**Authentication**:
```json
{
  "type": "auth",
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Subscribe to rate limits**:
```json
{
  "type": "subscribe",
  "channel": "rate_limits"
}
```

**Receive updates**:
```json
{
  "type": "rate_limit_update",
  "provider": "claude",
  "data": {
    "percentage_used": 36.5,
    "tokens_remaining": 635000,
    "timestamp": "2025-12-27T15:05:00Z"
  }
}
```

---

## Error Handling

### Standard Error Response

```json
{
  "error": {
    "code": "invalid_api_key",
    "message": "The provided API key is invalid or has been revoked",
    "details": {
      "provider": "claude",
      "key_name": "Production"
    },
    "request_id": "req-12345",
    "timestamp": "2025-12-27T15:00:00Z"
  }
}
```

### Error Codes

| HTTP Code | Error Code | Description |
|-----------|-----------|-------------|
| 400 | `invalid_request` | Malformed request body |
| 401 | `unauthorized` | Missing or invalid auth token |
| 403 | `forbidden` | Insufficient permissions |
| 404 | `not_found` | Resource not found |
| 409 | `conflict` | Resource already exists |
| 422 | `validation_error` | Request validation failed |
| 429 | `rate_limited` | Too many requests |
| 500 | `internal_error` | Server error |
| 503 | `service_unavailable` | Service temporarily unavailable |

---

## Rate Limiting

**Per-tier limits** (requests per minute):

| Tier | Rate Limit |
|------|-----------|
| Free | 60 req/min |
| Pro | 300 req/min |
| Team | 1000 req/min |
| Enterprise | Custom |

**Rate limit headers**:
```http
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 250
X-RateLimit-Reset: 1703692800
```

---

**Document Status**: ✅ Complete
**Last Updated**: December 27, 2025
