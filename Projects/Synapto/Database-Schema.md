---
tags: [synapto, database, schema, sqlalchemy]
created: 2026-01-05 13:05
---

# Database Schema

## Multi-Tenant Pattern
Every model MUST include:
\\\python
tenant_id = Column(String(36), ForeignKey("tenants.id"), nullable=False, index=True)
\\\

## Key Models

### Tenant
- slug: rendetalje | fb-rengoring
- pricing_config: JSON
- business_rules: JSON

### Inquiry
- Email classification result
- AI confidence score (0-1)
- response_draft
- status

### Customer
- Tenant-scoped
- Composite index: (tenant_id, email)

### Booking
- Calendar events
- Pricing and hours

### AgentMemory
- LLM conversation history
- category: conversation | preference | fact

## Migrations
\\\ash
# Create
alembic revision --autogenerate -m "description"

# Apply
alembic upgrade head
\\\

See [[Architecture]] for patterns.
