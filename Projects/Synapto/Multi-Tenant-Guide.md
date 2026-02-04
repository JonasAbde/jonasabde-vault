# Multi-Tenant Architecture Guide

**Complete guide to Synapto's multi-tenant isolation system for Rendetalje and FB Rengøring**

---

## Overview

Synapto serves **two separate cleaning businesses** from a single codebase:
- **Rendetalje** - Premium residential cleaning (luxury tone, 349 DKK/hr)
- **FB Rengøring** - Commercial cleaning (efficiency tone, 335 DKK/hr)

**CRITICAL**: Every piece of data MUST be isolated by `tenant_id` to prevent data leakage between companies.

---

## The Tenant Model

### Database Schema

```python
# app/models/tenant.py
class Tenant(Base):
    __tablename__ = "tenants"
    
    id = Column(String(36), primary_key=True)
    name = Column(String(100), nullable=False)
    slug = Column(String(50), unique=True)  # "rendetalje" or "fb-rengoring"
    
    # Business Configuration
    pricing_config = Column(JSON)  # Hourly rates, service types
    business_rules = Column(JSON)  # Operating hours, service areas
    tone_settings = Column(JSON)   # AI response tone/style
    
    # Email Configuration
    email_domains = Column(JSON)   # Incoming email domains
    gmail_credentials = Column(Text)
    outlook_credentials = Column(Text)
    
    # Branding
    logo_url = Column(String(500))
    primary_color = Column(String(7))
    
    created_at = Column(DateTime)
    updated_at = Column(DateTime)
```

### Tenant Configuration Examples

**Rendetalje** (Premium):
```json
{
  "pricing_config": {
    "hourly_rate": 349,
    "service_types": ["private", "moving", "periodic"]
  },
  "business_rules": {
    "operating_hours": "08:00-18:00",
    "service_area": "Greater Copenhagen"
  },
  "tone_settings": {
    "formality": "high",
    "language_style": "luxury",
    "signature": "Venlig hilsen\nRendetalje Team"
  }
}
```

**FB Rengøring** (Commercial):
```json
{
  "pricing_config": {
    "hourly_rate": 335,
    "service_types": ["commercial", "office", "deep_clean"]
  },
  "business_rules": {
    "operating_hours": "06:00-20:00",
    "service_area": "Nationwide Denmark"
  },
  "tone_settings": {
    "formality": "medium",
    "language_style": "efficient",
    "signature": "Med venlig hilsen\nFB Rengøring"
  }
}
```

---

## Data Isolation Pattern

### Rule #1: Every Model Has tenant_id

**Template for New Models:**
```python
from sqlalchemy import Column, String, ForeignKey, Index

class YourModel(Base):
    __tablename__ = "your_table"
    
    id = Column(String(36), primary_key=True)
    tenant_id = Column(
        String(36), 
        ForeignKey("tenants.id"), 
        nullable=False,
        index=True  # CRITICAL for query performance
    )
    
    # Your other fields here
    name = Column(String(100))
    
    # Composite indexes for common queries
    __table_args__ = (
        Index('ix_your_table_tenant_name', 'tenant_id', 'name'),
    )
```

### Rule #2: Filter Everything by tenant_id

**Always filter queries:**
```python
# ❌ WRONG - Data leakage risk
customers = session.query(Customer).all()

# ✅ CORRECT - Tenant isolation
customers = session.query(Customer)\
    .filter(Customer.tenant_id == tenant_id)\
    .all()

# ✅ ALSO CORRECT - Using SQLAlchemy relationship
tenant = session.query(Tenant).filter_by(slug="rendetalje").first()
customers = tenant.customers  # Automatic filtering via relationship
```

### Rule #3: Use BusinessLogicEngine

**Never hardcode tenant-specific logic:**
```python
from app.services.business_logic import BusinessLogicEngine

# ❌ WRONG
hourly_rate = 349  # Hardcoded for Rendetalje

# ✅ CORRECT
tenant = get_tenant_by_slug("rendetalje")
business_logic = BusinessLogicEngine(tenant=tenant)
hourly_rate = business_logic.get_hourly_rate()  # Returns 349 for Rendetalje
```

---

## API Layer Isolation

### Dependency Injection

```python
# app/api/dependencies.py
from fastapi import Header, HTTPException, Depends
from sqlalchemy.orm import Session

async def get_tenant_id(
    x_tenant_id: str = Header(..., alias="X-Tenant-ID")
) -> str:
    """Extract tenant ID from request header."""
    return x_tenant_id

async def get_tenant(
    tenant_id: str = Depends(get_tenant_id),
    db: Session = Depends(get_db)
) -> Tenant:
    """Get full tenant object."""
    tenant = db.query(Tenant).filter(Tenant.id == tenant_id).first()
    if not tenant:
        raise HTTPException(404, "Tenant not found")
    return tenant
```

### Protected Endpoints

```python
# app/api/routers/customers.py
from fastapi import APIRouter, Depends
from app.api.dependencies import get_tenant_id

router = APIRouter()

@router.get("/customers")
async def list_customers(
    tenant_id: str = Depends(get_tenant_id),
    db: Session = Depends(get_db)
):
    # Automatic tenant filtering via dependency
    customers = db.query(Customer)\
        .filter(Customer.tenant_id == tenant_id)\
        .all()
    return customers
```

---

## Service Layer Isolation

### UnifiedEmailService

Automatically routes emails to correct tenant based on domain:

```python
# app/services/email_router.py
class EmailRouter:
    def determine_tenant(self, email_address: str) -> str:
        """
        Routes email to tenant based on domain.
        
        rendetalje@gmail.com -> Rendetalje tenant
        fb@fbrengoering.dk -> FB Rengøring tenant
        """
        domain = email_address.split("@")[1]
        
        # Query tenant by email domain
        tenant = db.query(Tenant)\
            .filter(Tenant.email_domains.contains([domain]))\
            .first()
        
        return tenant.id if tenant else None
```

### IntelligentProcessor

Processes emails with tenant-specific rules:

```python
# app/services/intelligent_processor.py
class IntelligentProcessor:
    async def process_email(self, email_id: str, tenant_id: str):
        # Get tenant-specific configuration
        tenant = db.query(Tenant).get(tenant_id)
        business_logic = BusinessLogicEngine(tenant=tenant)
        
        # Classify email (tenant-specific prompts)
        classifier = EmailClassifier(tenant=tenant)
        classification = await classifier.classify(email)
        
        # Generate response (tenant-specific tone)
        generator = ResponseGenerator(tenant=tenant)
        response = await generator.generate(
            classification,
            tone=tenant.tone_settings
        )
        
        return response
```

---

## Frontend Integration

### API Client Setup

**Web Frontend (Next.js):**
```typescript
// lib/api.ts
class ApiClient {
  private tenantId: string;
  
  constructor() {
    this.tenantId = localStorage.getItem('tenantId');
  }
  
  async get(endpoint: string) {
    return axios.get(endpoint, {
      headers: {
        'X-Tenant-ID': this.tenantId,
        'X-API-Key': this.apiKey
      }
    });
  }
}
```

**Mobile App (Flutter):**
```dart
// lib/core/api/api_client.dart
class ApiClient {
  final String tenantId;
  final Dio dio;
  
  ApiClient(this.tenantId) {
    dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) {
        options.headers['X-Tenant-ID'] = tenantId;
        return handler.next(options);
      },
    ));
  }
}
```

---

## Testing Multi-Tenant Isolation

### Test Template

```python
# tests/test_tenant_isolation.py
import pytest

@pytest.fixture
def tenant_rendetalje(db_session):
    return Tenant(id="tenant1", slug="rendetalje", name="Rendetalje")

@pytest.fixture
def tenant_fb(db_session):
    return Tenant(id="tenant2", slug="fb-rengoring", name="FB Rengøring")

def test_customer_isolation(db_session, tenant_rendetalje, tenant_fb):
    # Create customers for each tenant
    customer1 = Customer(
        id="c1",
        tenant_id=tenant_rendetalje.id,
        name="Customer 1"
    )
    customer2 = Customer(
        id="c2",
        tenant_id=tenant_fb.id,
        name="Customer 2"
    )
    db_session.add_all([customer1, customer2])
    db_session.commit()
    
    # Query with tenant filtering
    rendetalje_customers = db_session.query(Customer)\
        .filter(Customer.tenant_id == tenant_rendetalje.id)\
        .all()
    
    # Verify isolation
    assert len(rendetalje_customers) == 1
    assert rendetalje_customers[0].id == "c1"
```

---

## Common Pitfalls

### ❌ Pitfall #1: Forgetting tenant_id Filter

```python
# DANGER: Returns customers from BOTH tenants
all_customers = session.query(Customer).all()

# FIX: Always filter by tenant
customers = session.query(Customer)\
    .filter(Customer.tenant_id == current_tenant_id)\
    .all()
```

### ❌ Pitfall #2: Hardcoded Business Rules

```python
# DANGER: Only works for Rendetalje
price = hours * 349

# FIX: Use BusinessLogicEngine
business_logic = BusinessLogicEngine(tenant=tenant)
price = business_logic.calculate_price(hours)
```

### ❌ Pitfall #3: Missing Index

```python
# SLOW: Full table scan without index
class Customer(Base):
    tenant_id = Column(String(36))  # No index!

# FAST: Indexed for quick filtering
class Customer(Base):
    tenant_id = Column(String(36), index=True)
```

---

## Migration Checklist

When adding a new feature:

- [ ] Add `tenant_id` column to new models
- [ ] Create indexes on `tenant_id`
- [ ] Filter all queries by `tenant_id`
- [ ] Use `BusinessLogicEngine` for tenant-specific logic
- [ ] Update API dependencies to inject `tenant_id`
- [ ] Add tenant isolation tests
- [ ] Update Alembic migration
- [ ] Test with both Rendetalje and FB Rengøring data

---

## Related Documentation
- [[Architecture]] - System architecture
- [[Database-Schema]] - Complete schema reference
- [[Backend/Business-Logic]] - BusinessLogicEngine details
- [[Testing-Strategy]] - Testing approach

#multi-tenant #security #architecture #critical
