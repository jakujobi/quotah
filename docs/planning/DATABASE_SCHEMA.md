# Quotah - Database Schema Design

**Last Updated:** December 27, 2025
**Version:** 1.0
**Database**: PostgreSQL 15+

---

## Table of Contents

1. [Schema Overview](#schema-overview)
2. [Table Definitions](#table-definitions)
3. [Indexes & Constraints](#indexes--constraints)
4. [Relationships](#relationships)
5. [Migration Strategy](#migration-strategy)
6. [Sample Queries](#sample-queries)

---

## Schema Overview

### Entity Relationship Diagram (ERD)

```
┌──────────────────┐
│     users        │
│──────────────────│
│ PK id            │────────┐
│    email         │        │
│    password_hash │        │
│    created_at    │        │
│    updated_at    │        │
│    tier          │        │
│    settings      │        │
│    is_active     │        │
└──────────────────┘        │
         │                  │
         │ 1:N              │ 1:N
         │                  │
         ▼                  ▼
┌──────────────────┐  ┌──────────────────────┐
│   api_keys       │  │  alert_configs       │
│──────────────────│  │──────────────────────│
│ PK id            │  │ PK id                │
│ FK user_id       │  │ FK user_id           │
│    provider      │  │    alert_type        │
│    key_name      │  │    threshold_percent │
│    key_hash      │  │    enabled           │
│    is_active     │  │    channels          │
│    created_at    │  │    created_at        │
└──────────────────┘  └──────────────────────┘
         │
         │ 1:N
         │
         ▼
┌──────────────────────────┐
│   usage_events           │
│──────────────────────────│
│ PK id                    │
│ FK user_id               │
│ FK api_key_id            │
│    provider              │
│    model                 │
│    input_tokens          │
│    output_tokens         │
│    total_tokens          │
│    cost_usd              │
│    request_id            │
│    timestamp             │
│    metadata              │
└──────────────────────────┘

┌──────────────────────────┐
│  rate_limit_snapshots    │
│──────────────────────────│
│ PK id                    │
│ FK user_id               │
│ FK api_key_id            │
│    provider              │
│    rpm_limit             │
│    rpm_remaining         │
│    rpm_reset             │
│    tpm_limit             │
│    tpm_remaining         │
│    tpm_reset             │
│    percentage_used       │
│    timestamp             │
└──────────────────────────┘

┌──────────────────────────┐
│  teams                   │
│──────────────────────────│
│ PK id                    │
│    name                  │
│    owner_id (FK users)   │
│    tier                  │
│    billing_email         │
│    created_at            │
└──────────────────────────┘
         │
         │ 1:N
         │
         ▼
┌──────────────────────────┐
│  team_members            │
│──────────────────────────│
│ PK id                    │
│ FK team_id               │
│ FK user_id               │
│    role                  │
│    joined_at             │
└──────────────────────────┘

┌──────────────────────────┐
│  alerts                  │
│──────────────────────────│
│ PK id                    │
│ FK user_id               │
│ FK alert_config_id       │
│    alert_type            │
│    severity              │
│    message               │
│    provider              │
│    metadata              │
│    sent_at               │
│    acknowledged_at       │
│    created_at            │
└──────────────────────────┘
```

---

## Table Definitions

### 1. users

Core user accounts table.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    tier VARCHAR(50) DEFAULT 'free',  -- 'free', 'pro', 'team', 'enterprise'
    settings JSONB DEFAULT '{}',
    is_active BOOLEAN DEFAULT true,
    email_verified BOOLEAN DEFAULT false,
    last_login_at TIMESTAMP WITH TIME ZONE,
    deleted_at TIMESTAMP WITH TIME ZONE  -- Soft delete
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_tier ON users(tier);
CREATE INDEX idx_users_created_at ON users(created_at);
```

**Fields**:
- `id`: Unique user identifier (UUID for privacy)
- `email`: User email (unique constraint)
- `password_hash`: bcrypt hashed password
- `tier`: Subscription level
- `settings`: User preferences (JSON)
  ```json
  {
    "notifications": {
      "email": true,
      "slack": false,
      "discord": false
    },
    "dashboard": {
      "default_view": "overview",
      "currency": "USD"
    },
    "alerts": {
      "default_threshold": 80
    }
  }
  ```

### 2. api_keys

Stores encrypted API keys for external services.

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,  -- 'claude', 'openai', 'gemini', etc.
    key_name VARCHAR(100) DEFAULT 'default',
    key_hash TEXT NOT NULL,  -- Encrypted API key (Fernet)
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP WITH TIME ZONE,
    metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_api_keys_provider ON api_keys(provider);
CREATE INDEX idx_api_keys_user_provider ON api_keys(user_id, provider);
CREATE UNIQUE INDEX idx_api_keys_unique ON api_keys(user_id, provider, key_name)
    WHERE deleted_at IS NULL;
```

**Fields**:
- `provider`: Which AI API this key belongs to
- `key_name`: User-friendly name ("Production", "Testing", etc.)
- `key_hash`: Encrypted API key (never stored plaintext)
- `metadata`: Additional provider-specific data
  ```json
  {
    "api_version": "v1",
    "region": "us-east-1",
    "rate_limit_tier": "tier_2"
  }
  ```

### 3. usage_events

Tracks every API call made through monitored APIs.

```sql
CREATE TABLE usage_events (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    api_key_id UUID NOT NULL REFERENCES api_keys(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,
    model VARCHAR(100) NOT NULL,
    input_tokens INTEGER NOT NULL,
    output_tokens INTEGER NOT NULL,
    total_tokens INTEGER NOT NULL,
    cost_usd DECIMAL(10, 6) NOT NULL,
    request_id VARCHAR(255),
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}'
);

-- Partitioning by month for better performance
CREATE TABLE usage_events_2025_01 PARTITION OF usage_events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE INDEX idx_usage_events_user_id ON usage_events(user_id);
CREATE INDEX idx_usage_events_timestamp ON usage_events(timestamp);
CREATE INDEX idx_usage_events_provider ON usage_events(provider);
CREATE INDEX idx_usage_events_user_timestamp ON usage_events(user_id, timestamp DESC);
CREATE INDEX idx_usage_events_user_provider ON usage_events(user_id, provider);
```

**Fields**:
- `input_tokens`, `output_tokens`: Token usage breakdown
- `cost_usd`: Calculated cost based on provider pricing
- `request_id`: Provider's request ID (for debugging)
- `metadata`: Additional context
  ```json
  {
    "endpoint": "/v1/messages",
    "response_time_ms": 1234,
    "cache_hit": false,
    "user_agent": "quotah-monitor/1.0"
  }
  ```

**Partitioning Strategy**: Monthly partitions for better query performance on large datasets.

### 4. rate_limit_snapshots

Periodic snapshots of rate limit status.

```sql
CREATE TABLE rate_limit_snapshots (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    api_key_id UUID NOT NULL REFERENCES api_keys(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,
    rpm_limit INTEGER,
    rpm_remaining INTEGER,
    rpm_reset TIMESTAMP WITH TIME ZONE,
    tpm_limit BIGINT,
    tpm_remaining BIGINT,
    tpm_reset TIMESTAMP WITH TIME ZONE,
    itpm_limit BIGINT,  -- Input tokens per minute (Claude)
    itpm_remaining BIGINT,
    otpm_limit BIGINT,  -- Output tokens per minute (Claude)
    otpm_remaining BIGINT,
    percentage_used DECIMAL(5, 2),
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_rate_limits_user_id ON rate_limit_snapshots(user_id);
CREATE INDEX idx_rate_limits_timestamp ON rate_limit_snapshots(timestamp);
CREATE INDEX idx_rate_limits_user_provider ON rate_limit_snapshots(user_id, provider);
CREATE INDEX idx_rate_limits_latest ON rate_limit_snapshots(user_id, provider, timestamp DESC);
```

**Query Pattern**: Typically fetch latest snapshot per provider:
```sql
SELECT DISTINCT ON (provider)
    provider, rpm_remaining, tpm_remaining, percentage_used, timestamp
FROM rate_limit_snapshots
WHERE user_id = :user_id
ORDER BY provider, timestamp DESC;
```

### 5. alert_configs

User-configured alert thresholds and preferences.

```sql
CREATE TABLE alert_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    alert_type VARCHAR(50) NOT NULL,  -- 'rate_limit', 'cost', 'usage_spike'
    threshold_percent INTEGER DEFAULT 80,
    enabled BOOLEAN DEFAULT true,
    channels JSONB DEFAULT '[]',  -- ['email', 'slack', 'discord']
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_alert_configs_user_id ON alert_configs(user_id);
CREATE INDEX idx_alert_configs_enabled ON alert_configs(enabled) WHERE enabled = true;
```

**Example channels JSON**:
```json
[
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
]
```

### 6. alerts

Log of all alerts sent to users.

```sql
CREATE TABLE alerts (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    alert_config_id UUID REFERENCES alert_configs(id) ON DELETE SET NULL,
    alert_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL,  -- 'info', 'warning', 'critical'
    message TEXT NOT NULL,
    provider VARCHAR(50),
    metadata JSONB DEFAULT '{}',
    sent_at TIMESTAMP WITH TIME ZONE,
    acknowledged_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_alerts_user_id ON alerts(user_id);
CREATE INDEX idx_alerts_created_at ON alerts(created_at);
CREATE INDEX idx_alerts_severity ON alerts(severity);
CREATE INDEX idx_alerts_acknowledged ON alerts(acknowledged_at) WHERE acknowledged_at IS NULL;
```

**Example alert**:
```json
{
  "alert_type": "rate_limit",
  "severity": "warning",
  "message": "OpenAI rate limit at 85% - 15K tokens remaining",
  "provider": "openai",
  "metadata": {
    "percentage_used": 85,
    "tokens_remaining": 15000,
    "reset_time": "2025-12-27T15:30:00Z"
  }
}
```

### 7. teams

Team accounts for multi-user collaboration.

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tier VARCHAR(50) DEFAULT 'team',  -- 'team', 'enterprise'
    billing_email VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    settings JSONB DEFAULT '{}'
);

CREATE INDEX idx_teams_owner_id ON teams(owner_id);
```

### 8. team_members

Team membership and roles.

```sql
CREATE TABLE team_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) DEFAULT 'member',  -- 'owner', 'admin', 'member', 'viewer'
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(team_id, user_id)
);

CREATE INDEX idx_team_members_team_id ON team_members(team_id);
CREATE INDEX idx_team_members_user_id ON team_members(user_id);
```

---

## Indexes & Constraints

### Primary Indexes (Auto-created)

All tables have primary key indexes automatically created.

### Secondary Indexes

**Performance-critical indexes**:

1. **User lookups** (Authentication):
   - `users.email` (unique) - for login
   - `users.tier` - for feature gating

2. **API key lookups**:
   - `api_keys(user_id, provider)` - composite for fast user+provider queries

3. **Usage queries**:
   - `usage_events(user_id, timestamp DESC)` - time-series queries
   - `usage_events(provider)` - provider-specific analytics

4. **Rate limit queries**:
   - `rate_limit_snapshots(user_id, provider, timestamp DESC)` - latest snapshot per provider

### Constraints

**Foreign Keys**:
- All `user_id` fields reference `users(id)` with `ON DELETE CASCADE`
- Ensures orphaned data is cleaned up when user deletes account

**Unique Constraints**:
- `users.email` - one account per email
- `api_keys(user_id, provider, key_name)` - unique key names per provider
- `team_members(team_id, user_id)` - user can only join team once

**Check Constraints**:
```sql
ALTER TABLE users
ADD CONSTRAINT check_tier
CHECK (tier IN ('free', 'pro', 'team', 'enterprise'));

ALTER TABLE usage_events
ADD CONSTRAINT check_tokens_positive
CHECK (input_tokens >= 0 AND output_tokens >= 0);

ALTER TABLE rate_limit_snapshots
ADD CONSTRAINT check_percentage
CHECK (percentage_used >= 0 AND percentage_used <= 100);
```

---

## Relationships

### One-to-Many (1:N)

- `users` → `api_keys` (one user has many API keys)
- `users` → `usage_events` (one user has many usage events)
- `api_keys` → `usage_events` (one key tracks many events)
- `users` → `alert_configs` (one user has many alert configs)
- `teams` → `team_members` (one team has many members)

### Many-to-Many (M:N)

- `users` ↔ `teams` (via `team_members` join table)

---

## Migration Strategy

### Alembic Migrations

Using Alembic for database migrations:

```bash
# Initialize migrations
alembic init migrations

# Create migration
alembic revision --autogenerate -m "Initial schema"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1
```

### Migration Files

**001_initial_schema.py**:
```python
def upgrade():
    # Create users table
    op.create_table('users', ...)

    # Create api_keys table
    op.create_table('api_keys', ...)

    # Create indexes
    op.create_index('idx_users_email', 'users', ['email'])

def downgrade():
    op.drop_table('users')
    op.drop_table('api_keys')
```

### Data Migration Strategy

**Adding new columns**:
```sql
-- Add column with default value (no downtime)
ALTER TABLE users ADD COLUMN phone VARCHAR(20) DEFAULT NULL;

-- Backfill data if needed
UPDATE users SET phone = extract_phone(email) WHERE phone IS NULL;
```

**Renaming columns**:
```sql
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN new_name VARCHAR(255);

-- Step 2: Copy data
UPDATE users SET new_name = old_name;

-- Step 3: Deploy code using new column
-- (zero-downtime deployment)

-- Step 4: Drop old column
ALTER TABLE users DROP COLUMN old_name;
```

---

## Sample Queries

### 1. Get user's total cost last 30 days

```sql
SELECT
    provider,
    SUM(cost_usd) as total_cost,
    SUM(total_tokens) as total_tokens,
    COUNT(*) as request_count
FROM usage_events
WHERE user_id = :user_id
  AND timestamp >= NOW() - INTERVAL '30 days'
GROUP BY provider
ORDER BY total_cost DESC;
```

### 2. Get current rate limit status for all APIs

```sql
SELECT DISTINCT ON (provider)
    provider,
    rpm_remaining,
    rpm_limit,
    tpm_remaining,
    tpm_limit,
    percentage_used,
    timestamp
FROM rate_limit_snapshots
WHERE user_id = :user_id
  AND timestamp >= NOW() - INTERVAL '1 hour'
ORDER BY provider, timestamp DESC;
```

### 3. Get hourly usage trend (last 24 hours)

```sql
SELECT
    DATE_TRUNC('hour', timestamp) as hour,
    provider,
    SUM(total_tokens) as tokens,
    SUM(cost_usd) as cost,
    COUNT(*) as requests
FROM usage_events
WHERE user_id = :user_id
  AND timestamp >= NOW() - INTERVAL '24 hours'
GROUP BY DATE_TRUNC('hour', timestamp), provider
ORDER BY hour DESC, provider;
```

### 4. Get unacknowledged alerts

```sql
SELECT
    id,
    alert_type,
    severity,
    message,
    provider,
    created_at
FROM alerts
WHERE user_id = :user_id
  AND acknowledged_at IS NULL
ORDER BY severity DESC, created_at DESC
LIMIT 20;
```

### 5. Get team usage breakdown

```sql
SELECT
    u.email,
    u.id as user_id,
    SUM(ue.cost_usd) as total_cost,
    SUM(ue.total_tokens) as total_tokens,
    COUNT(*) as request_count
FROM team_members tm
JOIN users u ON u.id = tm.user_id
LEFT JOIN usage_events ue ON ue.user_id = u.id
    AND ue.timestamp >= NOW() - INTERVAL '30 days'
WHERE tm.team_id = :team_id
GROUP BY u.id, u.email
ORDER BY total_cost DESC;
```

---

## Performance Optimization

### Query Optimization Tips

1. **Use EXPLAIN ANALYZE**:
```sql
EXPLAIN ANALYZE
SELECT * FROM usage_events WHERE user_id = '...' AND timestamp > '...';
```

2. **Partition large tables**:
```sql
-- Partition usage_events by month
CREATE TABLE usage_events_2025_01 PARTITION OF usage_events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

3. **Use covering indexes**:
```sql
-- Include frequently queried columns in index
CREATE INDEX idx_usage_events_cost_analysis
ON usage_events(user_id, timestamp DESC)
INCLUDE (provider, cost_usd, total_tokens);
```

4. **Materialized views for analytics**:
```sql
CREATE MATERIALIZED VIEW daily_usage_summary AS
SELECT
    user_id,
    DATE(timestamp) as date,
    provider,
    SUM(cost_usd) as total_cost,
    SUM(total_tokens) as total_tokens,
    COUNT(*) as request_count
FROM usage_events
GROUP BY user_id, DATE(timestamp), provider;

-- Refresh daily
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_usage_summary;
```

### Connection Pooling

```python
# SQLAlchemy connection pool settings
engine = create_engine(
    DATABASE_URL,
    pool_size=20,          # Max connections in pool
    max_overflow=10,       # Extra connections when pool full
    pool_pre_ping=True,    # Check connection health
    pool_recycle=3600      # Recycle connections every hour
)
```

---

## Data Retention Policy

### Automatic Cleanup

```sql
-- Delete old usage events (keep 1 year for free tier, unlimited for paid)
DELETE FROM usage_events
WHERE timestamp < NOW() - INTERVAL '1 year'
  AND user_id IN (SELECT id FROM users WHERE tier = 'free');

-- Delete old rate limit snapshots (keep 30 days)
DELETE FROM rate_limit_snapshots
WHERE timestamp < NOW() - INTERVAL '30 days';

-- Archive old alerts (keep 90 days)
DELETE FROM alerts
WHERE created_at < NOW() - INTERVAL '90 days'
  AND acknowledged_at IS NOT NULL;
```

**Scheduled cleanup job** (runs daily):
```python
@celery.task
def cleanup_old_data():
    # Delete old snapshots
    db.execute("""
        DELETE FROM rate_limit_snapshots
        WHERE timestamp < NOW() - INTERVAL '30 days'
    """)

    # Delete old usage events for free users
    db.execute("""
        DELETE FROM usage_events
        WHERE timestamp < NOW() - INTERVAL '1 year'
          AND user_id IN (SELECT id FROM users WHERE tier = 'free')
    """)
```

---

## Backup & Recovery

### Automated Backups

**AWS RDS Automated Backups**:
- Daily snapshots at 3 AM UTC
- 30-day retention
- Point-in-time recovery (5-minute granularity)
- Cross-region replication to US-West-2

### Manual Backup Script

```bash
#!/bin/bash
# backup_db.sh

pg_dump -h $DB_HOST -U $DB_USER -d quotah \
  --format=custom \
  --file="backup_$(date +%Y%m%d_%H%M%S).dump"

# Upload to S3
aws s3 cp backup_*.dump s3://quotah-backups/
```

### Restore Procedure

```bash
# Restore from backup
pg_restore -h $DB_HOST -U $DB_USER -d quotah backup_20251227.dump

# Restore to point-in-time (AWS RDS)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier quotah-prod \
  --target-db-instance-identifier quotah-recovery \
  --restore-time 2025-12-27T12:00:00Z
```

---

## Conclusion

This database schema is designed to:
- ✅ **Scale** to millions of usage events
- ✅ **Perform** with optimized indexes and partitioning
- ✅ **Maintain** data integrity with foreign keys and constraints
- ✅ **Support** multi-tenancy (teams + individual users)
- ✅ **Enable** rich analytics and reporting

**Next Steps**:
1. Review schema design
2. Create initial Alembic migration
3. Set up development database
4. Implement SQLAlchemy models
5. Write database seeder for testing

---

**Document Status**: ✅ Complete
**Last Updated**: December 27, 2025
