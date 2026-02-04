# Synapto Technology Stack

**Complete overview of all technologies, frameworks, and services used in the Synapto platform**

> **Last Updated**: January 5, 2026 | **Performance Optimized**: -550 MB RAM

---

## 🚀 Performance Optimizations (Jan 2026)

### Critical Improvements Deployed

| Optimization | Impact | Status |
|--------------|--------|--------|
| Singleton EmbeddingService | -100 MB RAM | ✅ ACTIVE |
| Lazy Loading for Knowledge Base | -150 MB RAM | ✅ ACTIVE |
| ResponseCacheService Singleton | -50 MB RAM | ✅ ACTIVE |
| Development Environment Tuning | -400+ MB RAM | ✅ ACTIVE |
| **Total Memory Reduction** | **-550+ MB** | **✅ PRODUCTION** |

### Implementation Details

**1. Singleton EmbeddingService**
- Single FastEmbed instance across all services
- Prevents 100 MB duplicate model loading
- Lazy initialization on first access
- Location: `backend/app/services/embedding_service.py`

**2. Lazy Loading for Knowledge Base**
- Knowledge base loads only when RAG accessed
- Property-based deferred initialization
- Location: `backend/app/services/knowledge_base_service.py`

**3. ResponseCacheService Optimization**
- Uses shared singleton embeddings
- Cosine similarity with embed_query()
- Consistent embedding model across services

**4. Development Environment**
- Uvicorn selective reload (app/ only, exclude tests/)
- Logging: WARNING level instead of INFO
- Database pool size: 5 (was 20)
- Disabled non-essential caching

---

## Backend Stack

### Core Framework
- **FastAPI 0.109** - Modern async web framework
- **Python 3.11+** - Primary language
- **Uvicorn** - ASGI server
- **Pydantic 2.0** - Data validation & settings

### Database
- **SQLite** (Development) - File-based database
- **PostgreSQL** (Production) - Production RDBMS
- **SQLAlchemy 2.0** - ORM with async support
- **Alembic** - Database migration management

### AI & LLM
- **Groq API** - Primary LLM provider
  - llama-3.3-70b-versatile (main model)
  - mixtral-8x7b-32768 (fallback)
- **LangChain** - LLM orchestration framework
- **ChromaDB** - Vector database for RAG
- **FastEmbed** - Local embedding generation
- **Pybreaker** - Circuit breaker pattern for resilience

### Email & Calendar
- **Gmail API** - Google email integration
- **Microsoft Graph API** - Outlook email/calendar
- **Google Calendar API** - Calendar management

### Security & Authentication
- **JWT (python-jose)** - Token authentication
- **Passlib + bcrypt** - Password hashing
- **cryptography** - Data encryption
- **python-multipart** - File upload handling

### Infrastructure
- **Docker** - Containerization
- **Nginx** - Reverse proxy & load balancer
- **Kubernetes** - Container orchestration (optional)
- **Prometheus + Grafana** - Monitoring & metrics

---

## Frontend Web Stack

### Core Framework
- **Next.js 16.1.1** - React framework with App Router
- **React 19.2.3** - UI library
- **TypeScript 5** - Type-safe JavaScript

### State Management & Data
- **TanStack React Query 5.90** - Server state management
- **Axios 1.13** - HTTP client
- **React Hook Form 7.69** - Form handling
- **Zod 4.3** - Schema validation

### UI & Styling
- **Tailwind CSS 4** - Utility-first CSS
- **Lucide React 0.562** - Icon library
- **Recharts 3.6** - Chart library
- **react-big-calendar 1.19** - Calendar component
- **date-fns 4.1** - Date utilities

### Build & Development
- **Webpack 5** (via Next.js)
- **PostCSS** - CSS processing
- **ESLint** - Code linting
- **Docker** - Multi-stage build

---

## Mobile App Stack

### Core Framework
- **Flutter 3.27.1** - Cross-platform framework
- **Dart 3.6.0** - Programming language

### State Management
- **Riverpod 2.5.1** - State management
- **riverpod_annotation 2.3.5** - Code generation
- **StateNotifier** - State handling pattern

### Networking
- **Dio 5.4.0** - HTTP client
- **Retrofit 4.0.3** - Type-safe REST client
- **retrofit_generator** - Code generation

### Local Storage
- **Hive 2.2.3** - NoSQL local database
- **hive_flutter 1.1.0** - Flutter integration
- **SharedPreferences 2.2.2** - Simple key-value storage
- **flutter_secure_storage 4.2.1** - Encrypted storage

### Offline & Connectivity
- **connectivity_plus 6.0.3** - Network status monitoring
- **Custom MutationQueueService** - Offline-first mutations

### UI Components
- **flutter_form_builder 9.1.1** - Form widgets
- **table_calendar 3.0.9** - Calendar widget
- **fl_chart 0.66.2** - Chart library

### Utilities
- **encrypt 5.0.3** - Encryption utilities
- **intl** - Internationalization
- **path_provider** - File system access

### Monitoring
- **sentry_flutter 7.14.0** - Crash reporting & monitoring

---

## DevOps & Infrastructure

### Version Control
- **Git** - Source control
- **GitHub** - Repository hosting
- **GitHub Actions** - CI/CD pipelines

### Containerization
- **Docker** - Container runtime
- **Docker Compose** - Multi-container orchestration
- **Multi-stage builds** - Optimized images

### Deployment
- **Nginx** - Web server & reverse proxy
- **Systemd** - Service management (Linux)
- **PostgreSQL** - Production database
- **Let's Encrypt** - SSL certificates

### Monitoring & Logging
- **Prometheus** - Metrics collection
- **Grafana** - Metrics visualization
- **Sentry** - Error tracking (Backend + Mobile)
- **Custom LoggingConfig** - Structured logging

### Environment Management
- **python-dotenv** - Environment variables
- **pydantic-settings** - Configuration management
- **.env files** - Environment-specific config

---

## Testing Stack

### Backend Testing
- **pytest** - Test framework
- **pytest-asyncio** - Async test support
- **pytest-cov** - Coverage reporting
- **httpx** - Async HTTP client for testing
- **faker** - Test data generation

### Frontend Testing
- **Jest** - JavaScript testing
- **React Testing Library** - Component testing
- **Playwright** - E2E testing

### Mobile Testing
- **flutter_test** - Widget testing
- **integration_test** - E2E testing
- **mockito** - Mocking framework

---

## Third-Party Services

### AI Services
- **Groq Cloud** - LLM API (primary)
- **Custom Tool Orchestrator** - Autonomous agent system

### Email Providers
- **Gmail** (via Google Workspace API)
- **Outlook** (via Microsoft Graph API)

### Calendar Providers
- **Google Calendar**
- **Outlook Calendar**

### Infrastructure Services
- **GitHub** - Code hosting & CI/CD
- **Docker Hub** - Container registry (optional)
- **Sentry.io** - Error monitoring

---

## Development Tools

### Code Quality
- **Flake8** - Python linting
- **Black** - Python formatting
- **isort** - Import sorting
- **ESLint** - JavaScript/TypeScript linting
- **Prettier** - Code formatting

### IDE & Extensions
- **VS Code** - Primary IDE
- **GitHub Copilot** - AI code assistance
- **Custom Agents** - Project-specific AI helpers

### Package Management
- **pip** - Python packages
- **npm** - Node.js packages  
- **pub** - Dart/Flutter packages

---

## Architecture Patterns

### Backend Patterns
- **Multi-tenant Architecture** - tenant_id isolation
- **Repository Pattern** - Data access abstraction
- **Service Layer Pattern** - Business logic separation
- **Circuit Breaker** - Resilience for external APIs
- **Unified Service Pattern** - Provider abstraction (Email/Calendar)

### Frontend Patterns
- **App Router** (Next.js) - File-based routing
- **Server Components** - React Server Components
- **Client Components** - Interactive UI components
- **API Routes** - Backend integration layer

### Mobile Patterns
- **Feature-based Architecture** - Modular code organization
- **Offline-first** - Local storage with sync
- **Mutation Queue** - Deferred API operations
- **Provider Pattern** - Dependency injection

---

## Related Documentation
- [[Architecture]] - System architecture overview
- [[Multi-Tenant-Guide]] - Multi-tenant implementation
- [[Deployment-Guide]] - Production deployment
- [[Testing-Strategy]] - Testing approach

#tech-stack #documentation #infrastructure
