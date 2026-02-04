---
tags: [synapto, architecture, backend, ai, frontend, mobile]
created: 2026-01-05 13:04
updated: 2026-01-05 14:45
status: living-document
---

# Synapto System Architecture

**Complete system architecture overview for the multi-tenant AI platform**

---

## 🎯 Project Overview

Synapto is a **multi-tenant AI-powered business automation platform** that automates email handling and customer management for two Danish cleaning businesses:
- **Rendetalje** - Premium residential cleaning (luxury tone, 349 DKK/hr)
- **FB Rengøring** - Commercial cleaning (efficiency tone, 335 DKK/hr)

### Platform Components

1. **Backend** - FastAPI + Python (AI services, email processing, business logic)
2. **Web Dashboard** - Next.js + React (admin management interface)
3. **Mobile App** - Flutter (field worker offline-first app)
4. **Infrastructure** - Docker, Nginx, PostgreSQL, Prometheus

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENTS                               │
├─────────────────┬───────────────────┬───────────────────────┤
│   Web Dashboard │   Mobile App      │   Email Providers     │
│   (Next.js)     │   (Flutter)       │   (Gmail/Outlook)     │
└────────┬────────┴────────┬──────────┴───────────┬───────────┘
         │                 │                      │
         ▼                 ▼                      ▼
┌────────────────────────────────────────────────────────────┐
│                    NGINX (Reverse Proxy)                    │
│             SSL Termination + Load Balancing                │
└────────────────────────┬───────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────┐
│                  FastAPI Backend                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           API Layer (13 routers)                     │  │
│  │  /emails /inquiries /customers /bookings /chat      │  │
│  └──────────┬───────────────────────────────────────────┘  │
│             │                                               │
│  ┌──────────▼──────────────────────────────────────────┐  │
│  │         Service Layer                               │  │
│  │  • AIService (Groq orchestration)                   │  │
│  │  • IntelligentProcessor (email pipeline)            │  │
│  │  • BusinessLogicEngine (tenant rules)               │  │
│  │  • UnifiedEmailService (Gmail/Outlook)              │  │
│  │  • UnifiedCalendarService (Google/Outlook)          │  │
│  └──────────┬───────────────────────────────────────────┘  │
│             │                                               │
│  ┌──────────▼──────────────────────────────────────────┐  │
│  │      AI Components (ai_components/)                 │  │
│  │  • EmailClassifier                                  │  │
│  │  • ResponseGenerator                                │  │
│  │  • ToolOrchestrator (autonomous)                    │  │
│  └──────────┬───────────────────────────────────────────┘  │
│             │                                               │
│  ┌──────────▼──────────────────────────────────────────┐  │
│  │        Data Layer (SQLAlchemy)                      │  │
│  │  Tenant, Customer, Inquiry, Booking, Memory         │  │
│  └──────────┬───────────────────────────────────────────┘  │
└─────────────┼───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│                 PostgreSQL Database                          │
│           Multi-tenant with tenant_id isolation              │
└─────────────────────────────────────────────────────────────┘

External Services:
• Groq API (LLM)  • Gmail/Outlook APIs  • Calendar APIs  • ChromaDB
```

---

### Multi-Tenant Foundation
- **CRITICAL**: Every database entity scoped by `tenant_id`
- Tenants: Rendetalje (premium) & FB Rengøring (commercial)
- Prevents data leakage between tenants

### Core Components

#### Backend (FastAPI)
- **Location**: `backend/app/`
- **Pattern**: RESTful API with routers
- **Database**: SQLAlchemy ORM + Alembic migrations
- **AI Integration**: Groq API via custom services

#### AI Services
- **AIService**: Main facade delegating to specialized components
- **EmailClassifier**: Email classification using LLM
- **ResponseGenerator**: Tone-aware response generation
- **ToolOrchestrator**: Autonomous agent with tool calling (max 3 iterations)
- **GroqClient**: Groq API wrapper with circuit breaker

#### Data Flow
\\\
Email In → UnifiedEmailService → IntelligentProcessor
  → EmailClassifier (Groq/Llama 3)
  → ResponseGenerator (with ToneSliders)
  → [Gatekeeper threshold check]
  → ResponseSender (Gmail/Outlook)
\\\

### Key Patterns

#### Tool Orchestrator Pattern
- **Email Automation**: Multi-step autonomous agent, up to 3 iterations, read+write access
- **Chatbot Mode**: Internal staff assistant (NEW - Jan 5, 2026)
  - Read-only tools: Calendar lookup, Customer lookup, Inquiry search, Price calculation
  - Up to 3 autonomous iterations with tool calling
  - No invoice/booking creation (staff input required)
  - Location: `backend/app/services/chatbot_service.py`
- Max 3 iterations to prevent loops
- Conversation history tracking
- Graceful fallback on max iterations

#### Circuit Breaker Pattern
- Resilience for external API calls (Groq, Gmail, etc.)
- 5 failures → 60s cooldown
- Automatic fallback responses

#### Context Window Management
- Sliding window for recent messages (default: 20)
- AI summarization for older messages
- 45-80% token reduction

## 🗄️ Database Schema

### Key Models
- **Tenant**: Multi-tenant root (slug, pricing_config, business_rules)
- **Inquiry**: Email classification result with AI confidence
- **Customer**: Tenant-scoped, composite email index
- **Booking**: Calendar events with pricing
- **AgentMemory**: LLM conversation history

### Important Indexes
Always filter by tenant_id:
\\\python
# ❌ WRONG
db.query(Customer).all()

# ✅ CORRECT
db.query(Customer).filter(Customer.tenant_id == tenant_id).all()
\\\

## 📱 Flutter App (Offline-First)

### Architecture
- **State Management**: Riverpod
- **Local Storage**: Hive + flutter_secure_storage
- **Offline Queue**: MutationQueueService for sync
- **Feature-Based**: `lib/features/{feature}/`

## 🔗 Related Notes
- [[API Documentation]]
- [[Database Schema]]
- [[Deployment Guide]]
- [[AI Integration Patterns]]

## 📝 Notes
- See `backend/app/services/` for service implementations
- Multi-tenant is NON-NEGOTIABLE
- Always use async/await for I/O operations
