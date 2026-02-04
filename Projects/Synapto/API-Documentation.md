---
tags: [synapto, api, fastapi, backend]
created: 2026-01-05 13:05
---

# Synapto API Documentation

## Core Endpoints

### Inquiries
- POST /api/v1/inquiries/{tenant_id} - Process email
- GET /api/v1/inquiries/{tenant_id} - List inquiries
- GET /api/v1/inquiries/{tenant_id}/{id} - Get inquiry

### Pattern: Always filter by tenant_id
See [[Architecture]] for multi-tenant details.

## OpenAPI
- Swagger: /api/docs
- ReDoc: /api/redoc
