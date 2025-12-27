# Quotah - 12-Week Sprint Plan (MVP Development)

**Start Date:** Week of January 1, 2026
**End Date:** Week of March 23, 2026
**Goal:** Launch production-ready MVP with Claude + OpenAI support

---

## Overview

### MVP Scope

**Must-Have Features**:
- ✅ User authentication (email/password + JWT)
- ✅ Claude API adapter (rate limits + cost tracking)
- ✅ OpenAI API adapter (rate limits + cost tracking)
- ✅ Web dashboard (React)
- ✅ Basic cost analytics
- ✅ Email alerts (when approaching rate limits)
- ✅ API key management
- ✅ Production deployment

**Nice-to-Have** (if time permits):
- ⚠️ Browser extension
- ⚠️ Slack integration
- ⚠️ Discord integration
- ⚠️ Advanced analytics

**Post-MVP** (Phase 2):
- ❌ Gemini adapter
- ❌ Cohere adapter
- ❌ Team management
- ❌ Smart routing recommendations

---

## Sprint Breakdown

### Week 1-2: Foundation & Setup

**Sprint Goal**: Set up development environment and core backend structure

#### Week 1: Project Setup

**Backend Tasks**:
- [x] Initialize Git repository
- [ ] Set up Python virtual environment
- [ ] Install core dependencies (FastAPI, SQLAlchemy, Alembic, etc.)
- [ ] Configure project structure
  ```
  backend/
  ├── app/
  │   ├── __init__.py
  │   ├── main.py
  │   ├── api/
  │   ├── core/
  │   ├── models/
  │   ├── services/
  │   ├── adapters/
  │   └── utils/
  ├── tests/
  ├── alembic/
  ├── requirements.txt
  └── .env.example
  ```
- [ ] Set up logging configuration
- [ ] Create Docker Compose for local development (PostgreSQL + Redis)
- [ ] Set up pre-commit hooks (black, flake8, mypy)

**Frontend Tasks**:
- [ ] Initialize React + TypeScript project (Vite)
- [ ] Install UI dependencies (Tailwind, shadcn/ui, Recharts)
- [ ] Configure project structure
  ```
  frontend/
  ├── src/
  │   ├── components/
  │   ├── pages/
  │   ├── services/
  │   ├── hooks/
  │   ├── utils/
  │   └── App.tsx
  ├── public/
  └── package.json
  ```
- [ ] Set up Tailwind CSS
- [ ] Create design tokens (colors, typography, spacing)

**DevOps**:
- [ ] Set up GitHub repository
- [ ] Configure GitHub Actions for CI
- [ ] Set up development database (PostgreSQL in Docker)
- [ ] Create `.env.example` files

**Deliverables**:
- ✅ Working local dev environment
- ✅ Backend serves "Hello World" at localhost:8000
- ✅ Frontend renders at localhost:5173
- ✅ CI pipeline runs successfully

#### Week 2: Database & Auth Foundation

**Backend Tasks**:
- [ ] Define SQLAlchemy models (users, api_keys, usage_events)
- [ ] Create initial Alembic migration
- [ ] Run migrations on dev database
- [ ] Implement user registration endpoint
- [ ] Implement user login endpoint (JWT generation)
- [ ] Implement JWT token validation middleware
- [ ] Implement password hashing (bcrypt)
- [ ] Write unit tests for auth endpoints

**Frontend Tasks**:
- [ ] Create login page UI
- [ ] Create registration page UI
- [ ] Implement form validation (Zod)
- [ ] Create auth service (API client)
- [ ] Implement JWT token storage (localStorage)
- [ ] Create protected route wrapper
- [ ] Implement logout functionality

**Database**:
- [ ] Create `users` table
- [ ] Create `api_keys` table
- [ ] Seed test data

**Deliverables**:
- ✅ Users can register
- ✅ Users can login and receive JWT
- ✅ Protected routes work
- ✅ Database migrations work

---

### Week 3-4: API Adapters

**Sprint Goal**: Build Claude and OpenAI adapters for rate limit monitoring

#### Week 3: Claude Adapter

**Backend Tasks**:
- [ ] Create base `AIAPIAdapter` abstract class
- [ ] Define data models (RateLimitInfo, UsageSnapshot, ModelPricing)
- [ ] Implement `ClaudeAdapter` class
  - [ ] `get_rate_limit_info()` method
  - [ ] `validate_api_key()` method
  - [ ] `get_model_pricing()` method
  - [ ] `parse_response_for_usage()` method
- [ ] Write unit tests for ClaudeAdapter (mocked responses)
- [ ] Write integration tests (with real Anthropic API key)
- [ ] Create endpoint: `POST /api/api-keys` (add API key)
- [ ] Create endpoint: `GET /api/api-keys` (list keys)
- [ ] Create endpoint: `DELETE /api/api-keys/{id}` (remove key)

**Frontend Tasks**:
- [ ] Create API Keys page
- [ ] Create "Add API Key" modal
- [ ] Implement API key form (provider selection, key input)
- [ ] Display list of connected API keys
- [ ] Show API key status (healthy, error, last used)
- [ ] Implement delete API key functionality

**Testing**:
- [ ] Test adding Claude API key
- [ ] Test key validation
- [ ] Test rate limit fetching

**Deliverables**:
- ✅ Users can add Claude API keys
- ✅ System can fetch Claude rate limits
- ✅ Rate limit data stored in database

#### Week 4: OpenAI Adapter

**Backend Tasks**:
- [ ] Implement `OpenAIAdapter` class
  - [ ] `get_rate_limit_info()` method
  - [ ] `validate_api_key()` method
  - [ ] `get_model_pricing()` method
  - [ ] `parse_response_for_usage()` method
- [ ] Write unit tests for OpenAIAdapter
- [ ] Write integration tests
- [ ] Create `UnifiedAPIService` class
  - [ ] `get_unified_rate_limit_status()` method
  - [ ] `register_api()` method
  - [ ] Load adapters dynamically based on user's API keys
- [ ] Create endpoint: `GET /multi/rate-limits`

**Frontend Tasks**:
- [ ] Update "Add API Key" modal to support OpenAI
- [ ] Display OpenAI keys in API keys list
- [ ] Create basic dashboard layout
- [ ] Create rate limit cards component
- [ ] Display Claude rate limits on dashboard
- [ ] Display OpenAI rate limits on dashboard

**Testing**:
- [ ] Test adding OpenAI API key
- [ ] Test fetching multi-API rate limits
- [ ] End-to-end test: add both APIs, view dashboard

**Deliverables**:
- ✅ Users can add OpenAI API keys
- ✅ Dashboard shows rate limits for both Claude & OpenAI
- ✅ Unified API service works correctly

---

### Week 5-6: Dashboard & Analytics

**Sprint Goal**: Build functional dashboard with cost analytics

#### Week 5: Dashboard UI

**Frontend Tasks**:
- [ ] Create dashboard home page layout
- [ ] Create rate limit cards (with gauge charts)
- [ ] Color-code status (healthy, warning, critical)
- [ ] Display reset time countdown
- [ ] Create navigation sidebar
- [ ] Create top nav bar (user menu, logout)
- [ ] Implement real-time updates (poll every 60 seconds)
- [ ] Add loading states
- [ ] Add error states (API down, no keys connected)
- [ ] Make dashboard responsive (mobile-friendly)

**Backend Tasks**:
- [ ] Create background job: poll rate limits (Celery)
- [ ] Schedule polling every 5 minutes for all users
- [ ] Store snapshots in `rate_limit_snapshots` table
- [ ] Cache rate limits in Redis (60-second TTL)
- [ ] Optimize `/multi/rate-limits` endpoint (check cache first)

**Testing**:
- [ ] Test dashboard loads quickly (<500ms)
- [ ] Test real-time updates work
- [ ] Test error handling (API key invalid)

**Deliverables**:
- ✅ Functional dashboard showing live rate limits
- ✅ Background polling working
- ✅ Responsive design

#### Week 6: Cost Analytics

**Backend Tasks**:
- [ ] Create `usage_events` table migration
- [ ] Implement usage event logging
- [ ] Create endpoint: `GET /analytics/usage`
- [ ] Create endpoint: `GET /multi/cost-analysis`
- [ ] Implement cost calculation logic
- [ ] Create aggregation queries (by provider, by day, etc.)
- [ ] Add pagination to usage history
- [ ] Implement date range filtering

**Frontend Tasks**:
- [ ] Create Analytics page
- [ ] Display total cost (last 7/30 days)
- [ ] Create cost breakdown by provider (pie chart)
- [ ] Create usage trend chart (line chart, last 7 days)
- [ ] Create cost comparison table
- [ ] Add date range picker
- [ ] Display projected monthly cost
- [ ] Add export button (placeholder)

**Testing**:
- [ ] Test cost calculations are accurate
- [ ] Test charts render correctly
- [ ] Test date filtering works

**Deliverables**:
- ✅ Analytics page showing cost breakdown
- ✅ Charts visualizing usage trends
- ✅ Accurate cost calculations

---

### Week 7-8: Alerts & Notifications

**Sprint Goal**: Implement alert system with email notifications

#### Week 7: Alert Configuration

**Backend Tasks**:
- [ ] Create `alert_configs` table migration
- [ ] Create `alerts` table migration
- [ ] Create endpoint: `POST /alerts/configs` (create alert)
- [ ] Create endpoint: `GET /alerts/configs` (list configs)
- [ ] Create endpoint: `PATCH /alerts/configs/{id}` (update)
- [ ] Create endpoint: `DELETE /alerts/configs/{id}` (delete)
- [ ] Implement `AlertService` class
  - [ ] `check_thresholds()` method
  - [ ] `create_alert()` method
  - [ ] `send_alert()` method

**Frontend Tasks**:
- [ ] Create Alerts page
- [ ] Create alert configuration form
- [ ] Display list of alert configs
- [ ] Implement toggle to enable/disable alerts
- [ ] Add threshold slider (0-100%)
- [ ] Add channel selection (email, Slack, Discord)
- [ ] Implement edit alert config
- [ ] Implement delete alert config

**Testing**:
- [ ] Test creating alert configs
- [ ] Test updating thresholds

**Deliverables**:
- ✅ Users can configure alerts
- ✅ Alert configs saved to database

#### Week 8: Email Notifications

**Backend Tasks**:
- [ ] Set up SendGrid account
- [ ] Create email templates (HTML + text)
- [ ] Implement email sending service
- [ ] Create background job: check alerts (every 5 min)
- [ ] Implement threshold checking logic
- [ ] Send email when threshold exceeded
- [ ] Log alerts to `alerts` table
- [ ] Implement rate limiting on alerts (max 1 per hour per user)

**Frontend Tasks**:
- [ ] Create Alerts History page
- [ ] Display recent alerts
- [ ] Show alert severity (info, warning, critical)
- [ ] Implement "acknowledge" button
- [ ] Add filter by severity
- [ ] Add unacknowledged count badge
- [ ] Create notification bell icon in nav bar

**Testing**:
- [ ] Test email sending works
- [ ] Test alerts triggered at correct thresholds
- [ ] End-to-end: configure alert, exceed threshold, receive email

**Deliverables**:
- ✅ Email alerts working
- ✅ Users receive notifications when approaching limits
- ✅ Alert history visible in dashboard

---

### Week 9-10: Polish & Testing

**Sprint Goal**: Bug fixes, UX improvements, comprehensive testing

#### Week 9: UX Polish

**Frontend Tasks**:
- [ ] Improve loading states (skeleton screens)
- [ ] Add success/error toast notifications
- [ ] Improve error messages (user-friendly)
- [ ] Add empty states (no API keys, no data)
- [ ] Implement keyboard shortcuts
- [ ] Add tooltips for complex features
- [ ] Improve mobile responsiveness
- [ ] Add dark mode support (optional)
- [ ] Optimize bundle size (code splitting)
- [ ] Add favicon and meta tags

**Backend Tasks**:
- [ ] Add request/response logging
- [ ] Implement global error handler
- [ ] Add health check endpoint (`/health`)
- [ ] Add readiness check endpoint (`/readiness`)
- [ ] Optimize slow queries
- [ ] Add database indexes
- [ ] Implement API rate limiting (per user tier)
- [ ] Add CORS configuration

**Documentation**:
- [ ] Write API documentation (OpenAPI/Swagger)
- [ ] Create README.md
- [ ] Write setup instructions
- [ ] Document environment variables

**Deliverables**:
- ✅ Polished UX with smooth interactions
- ✅ Comprehensive error handling
- ✅ Well-documented API

#### Week 10: Testing & QA

**Testing Tasks**:
- [ ] Write unit tests for all services (80%+ coverage)
- [ ] Write integration tests for API endpoints
- [ ] Write end-to-end tests (Playwright)
  - [ ] User registration flow
  - [ ] Login flow
  - [ ] Add API key flow
  - [ ] View dashboard
  - [ ] Configure alerts
- [ ] Test edge cases:
  - [ ] Invalid API keys
  - [ ] Rate limit exceeded
  - [ ] Network errors
  - [ ] Database connection failures
- [ ] Load testing (100 concurrent users)
- [ ] Security testing:
  - [ ] SQL injection attempts
  - [ ] XSS attempts
  - [ ] CSRF protection
  - [ ] JWT expiration
- [ ] Cross-browser testing (Chrome, Firefox, Safari)

**Bug Fixing**:
- [ ] Fix all critical bugs
- [ ] Fix high-priority bugs
- [ ] Document known issues

**Deliverables**:
- ✅ Test coverage >80%
- ✅ No critical bugs remaining
- ✅ Passing end-to-end tests

---

### Week 11-12: Deployment & Launch

**Sprint Goal**: Deploy to production and soft launch

#### Week 11: Production Setup

**Infrastructure Tasks**:
- [ ] Set up production database (AWS RDS PostgreSQL)
- [ ] Set up production Redis (AWS ElastiCache)
- [ ] Set up application server (Railway / Render / AWS EC2)
- [ ] Configure environment variables
- [ ] Set up SSL certificate (Let's Encrypt)
- [ ] Configure custom domain (quotah.com)
- [ ] Set up CDN (Cloudflare)
- [ ] Configure backups (daily snapshots)
- [ ] Set up monitoring (Sentry for errors)
- [ ] Set up analytics (Posthog)
- [ ] Configure log aggregation (CloudWatch / Loki)

**Deployment**:
- [ ] Create production build (frontend)
- [ ] Deploy backend to production
- [ ] Deploy frontend to production (Vercel / Netlify)
- [ ] Run database migrations
- [ ] Verify health checks pass
- [ ] Load test production environment
- [ ] Set up staging environment (for future updates)

**Security**:
- [ ] Enable HTTPS only
- [ ] Configure security headers (HSTS, CSP, etc.)
- [ ] Set up firewall rules
- [ ] Rotate encryption keys
- [ ] Enable database encryption at rest

**Deliverables**:
- ✅ Production environment ready
- ✅ SSL configured
- ✅ Monitoring active

#### Week 12: Launch

**Pre-Launch Tasks**:
- [ ] Create landing page
- [ ] Write launch announcement
- [ ] Prepare demo video
- [ ] Create help documentation
- [ ] Set up customer support email
- [ ] Create Twitter/X account
- [ ] Create Discord server
- [ ] Final QA testing in production
- [ ] Invite beta users (20-50 people)

**Launch Day**:
- [ ] Submit to Product Hunt
- [ ] Post on r/ClaudeAI
- [ ] Post on r/OpenAI
- [ ] Post on r/LocalLLaMA
- [ ] Tweet launch announcement
- [ ] Email beta users
- [ ] Monitor for issues

**Post-Launch**:
- [ ] Respond to user feedback
- [ ] Fix any critical bugs immediately
- [ ] Monitor server performance
- [ ] Track key metrics:
  - [ ] Sign-ups
  - [ ] Active users
  - [ ] API keys added
  - [ ] Alerts sent
- [ ] Iterate on feedback

**Deliverables**:
- ✅ MVP launched publicly
- ✅ 100+ users signed up
- ✅ No major outages

---

## Success Metrics

### Week 6 Checkpoint

- [ ] Backend API functional
- [ ] Dashboard displays rate limits
- [ ] 2+ engineers can develop concurrently
- [ ] CI/CD pipeline working

### Week 10 Checkpoint

- [ ] All core features implemented
- [ ] Test coverage >80%
- [ ] No critical bugs
- [ ] Ready for staging deployment

### Week 12 Checkpoint

- [ ] Production deployment successful
- [ ] 100+ users signed up
- [ ] 50+ users added API keys
- [ ] 10+ users set up alerts
- [ ] < 1% error rate
- [ ] p95 API response time < 500ms

---

## Risk Management

### Potential Risks

1. **API Changes**: AI providers change their rate limit APIs
   - **Mitigation**: Build flexible adapters, monitor provider updates

2. **Scope Creep**: Adding too many features before MVP
   - **Mitigation**: Strict feature freeze after Week 8

3. **Technical Debt**: Rushing leads to poor code quality
   - **Mitigation**: Enforce code reviews, maintain test coverage

4. **Deployment Issues**: Production environment problems
   - **Mitigation**: Use staging environment, deploy early (Week 11)

5. **Low User Adoption**: Users don't find value
   - **Mitigation**: Beta test with real users, gather feedback early

---

## Team Capacity

**Assumptions**:
- 1 full-time developer (you)
- 40 hours/week available
- ~480 hours total over 12 weeks

**Time Allocation**:
- Backend development: 200 hours (42%)
- Frontend development: 150 hours (31%)
- Testing & QA: 60 hours (12.5%)
- DevOps & deployment: 40 hours (8%)
- Documentation: 20 hours (4%)
- Buffer (bugs, unforeseen): 10 hours (2%)

---

## Tools & Services

**Development**:
- VS Code / PyCharm
- Git + GitHub
- Docker + Docker Compose
- Postman / Insomnia (API testing)

**Collaboration**:
- GitHub Projects (task tracking)
- Notion (documentation)
- Slack / Discord (communication)

**Testing**:
- pytest (Python)
- Vitest (TypeScript)
- Playwright (E2E)
- Postman (API)

**Monitoring**:
- Sentry (error tracking)
- Posthog (product analytics)
- Uptime Robot (uptime monitoring)
- AWS CloudWatch (logs)

**Communication**:
- SendGrid (email)
- Slack (webhooks)
- Discord (webhooks)

---

## Post-MVP Roadmap (Weeks 13+)

### Phase 2: Expansion (Weeks 13-20)

- [ ] Add Gemini adapter
- [ ] Add Cohere adapter
- [ ] Build browser extension (Chrome)
- [ ] Implement smart routing recommendations
- [ ] Add Slack integration
- [ ] Add Discord integration
- [ ] Improve analytics (custom date ranges, exports)
- [ ] Add team management features

### Phase 3: Enterprise (Weeks 21-30)

- [ ] Build team dashboard
- [ ] Implement cost allocation
- [ ] Add budget controls
- [ ] SOC2 compliance preparation
- [ ] Self-hosted option
- [ ] Advanced security features
- [ ] White-label options

---

## Conclusion

This 12-week plan is aggressive but achievable. The key is to:
1. **Focus ruthlessly** on MVP scope
2. **Ship early, iterate fast**
3. **Test thoroughly** before launch
4. **Listen to users** after launch

**Remember**: Done is better than perfect. Launch the MVP, gather feedback, and iterate based on real user needs.

---

**Document Status**: ✅ Complete
**Sprint Start**: Week of January 1, 2026
**Sprint End**: Week of March 23, 2026
**Review Frequency**: Weekly standup + biweekly sprint review
