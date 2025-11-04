# Quick Reference Guide - Architecture Diagrams

## 📊 Diagram Overview

### 1. System Architecture Diagram
**File**: `system_architecture.mermaid`

```
Shows:
├── Client Layer (Web, Mobile, Third-party apps)
├── API Gateway (Kong) - Entry point
├── Central Authentication Service
│   ├── Auth API
│   ├── JWT Service (RS256 signing)
│   ├── OTP Service (SMS/Email)
│   ├── User Management
│   └── Admin Service
├── Four Microservices (LO, DH, M, Dashboard)
│   ├── Each with Auth Middleware
│   ├── Permission Service
│   └── Business Logic Controllers
├── Databases (PostgreSQL + Redis per service)
├── Message Queue (Kafka)
├── External Services (Twilio, SendGrid)
├── Key Management (Vault, JWKS)
└── Monitoring (Prometheus, Grafana, ELK)

Key Features:
• Color-coded components
• Request flow paths
• Data connections
• Cache layers
• Event streams
```

### 2. Complete ER Diagram
**File**: `complete_er_diagram.mermaid`

```
Central Auth Tables:
├── USERS (user_id PK, identifier, status)
├── USER_SERVICE_ACCESS (user → services mapping)
├── OTP_LOGS (OTP verification history)
├── REFRESH_TOKENS (long-lived tokens)
└── AUTH_AUDIT_LOGS (security events)

Microservice Tables (per service):
├── ROLES (service-specific roles)
├── USER_ROLES (user → role assignments)
├── PERMISSIONS (granular permissions)
├── ROLE_PERMISSIONS (role → permission mapping)
├── RESOURCE_PERMISSIONS (ABAC - resource access)
├── PROJECTS (business data)
└── LEADS (business data)

Relationships:
• One user → Multiple services
• One user → Multiple roles
• One role → Multiple permissions
• One resource → Multiple user permissions
```

### 3. Auth Service ER Diagram
**File**: `er_diagram_auth_service.mermaid`

```
Focused on Central Authentication Database:

USERS table:
├── user_id (UUID) - Primary key
├── identifier (email/phone) - UNIQUE
├── identifier_type (phone/email)
├── status (active/inactive/suspended)
└── timestamps (created_at, updated_at, last_login_at)

USER_SERVICE_ACCESS table:
├── Links users to services (LO, DH, M, Dashboard)
├── global_role (super_admin, service_admin, user)
├── assigned_by (admin who granted access)
└── Status tracking

OTP_LOGS table:
├── OTP verification tracking
├── Attempts counter (max 3)
├── Expiration time (5 minutes)
└── Verification status

REFRESH_TOKENS table:
├── Token hash (SHA-256)
├── Service-specific tokens
├── Expiration (7 days)
└── Revocation flag

AUTH_AUDIT_LOGS table:
├── All authentication events
├── IP address tracking
├── User agent logging
└── JSONB metadata
```

### 4. Microservice ER Diagram
**File**: `er_diagram_microservice.mermaid`

```
Focused on M Service Permission Structure:

ROLES:
├── admin (full access)
├── manager (manage projects/leads)
├── viewer (read-only)
└── editor (edit but not delete)

USER_ROLES:
├── Links central auth user_id to service roles
├── Scope for multi-tenancy (project-level)
└── Timestamps for audit

PERMISSIONS:
├── permission_name (view_projects, create_lead)
├── resource_type (project, lead, report)
├── action (view, create, edit, delete)
└── Description

ROLE_PERMISSIONS:
├── Many-to-many role ↔ permission mapping
├── ABAC conditions (JSONB)
└── Fine-grained control

RESOURCE_PERMISSIONS:
├── User → specific resource access
├── Permission levels (owner, editor, viewer)
├── Granted_by for accountability
└── Optional expiration

PROJECTS & LEADS:
├── Business data tables
└── Links to resource_permissions for ABAC
```

### 5. Authentication Flow Diagram
**File**: `authentication_flow_diagram.mermaid`

```
Complete sequence showing:

Phase 1: OTP Request
User → Web App → Gateway → Auth Service
├── Check user exists
├── Check service access
├── Generate & hash OTP
├── Store in Redis (5 min TTL)
├── Send via SMS/Email
└── Return success

Phase 2: OTP Verification
User enters OTP → Auth Service
├── Verify OTP from Redis
├── Get user details
├── Get all accessible services
├── Generate JWT (RS256 private key)
├── Generate refresh token
├── Store refresh token hash
└── Return tokens to client

Phase 3: API Request
Client → Gateway → Microservice
├── Gateway validates JWT (public key)
├── Check service in JWT services array
├── Forward to microservice
├── Microservice re-verifies JWT
├── Check permissions (RBAC + ABAC)
├── Filter data by access level
└── Return response

Phase 4: Admin User Creation
Admin → Auth Service
├── Create user in central DB
├── Assign services
├── Publish Kafka event
└── Microservices consume event

Phase 5: Permission Change
Admin updates role → Microservice
├── Update database
├── Invalidate Redis cache
├── Publish Kafka event
└── Other instances clear cache
```

### 6. JWT Token Flow Diagram
**File**: `jwt_token_flow.mermaid`

```
Token Lifecycle:

1. Key Generation (Setup)
   ├── Generate RSA 4096-bit key pair
   ├── Private key → Vault/KMS
   └── Public key → JWKS endpoint

2. Token Creation (Login)
   ├── Build payload {user_id, services}
   ├── Sign with RS256 private key
   ├── 30-minute expiry
   └── Return to client

3. Token Distribution
   ├── Client stores in localStorage
   └── Include in Authorization header

4. Token Verification (Gateway)
   ├── Fetch public key from JWKS
   ├── Verify signature
   ├── Check expiration
   ├── Check audience
   └── Route request

5. Token Verification (Microservice)
   ├── Get public key (cached locally)
   ├── Verify signature again
   ├── Extract user_id and services
   ├── Check service access
   └── Process request

6. Permission Checks
   ├── Check Redis cache
   ├── Query DB if cache miss
   ├── Cache results (10 min TTL)
   └── Authorize action

7. Resource Checks (ABAC)
   ├── Check resource permission cache
   ├── Query resource_permissions table
   ├── Cache results (5 min TTL)
   └── Apply field-level filtering

8. Token Refresh
   ├── Access token expires
   ├── Send refresh token
   ├── Verify refresh token
   ├── Issue new access token
   └── Continue session

9. Cache Invalidation
   ├── Permission change detected
   ├── Publish Kafka event
   ├── Delete Redis keys
   └── Next request gets fresh data
```

### 7. Permission Check Flow Diagram
**File**: `permission_check_flow.mermaid`

```
Decision Tree for Authorization:

API Request Received
    ↓
[1] Extract JWT Token
    ↓
[2] Token Present? → NO → ❌ 401 Unauthorized
    ↓ YES
[3] Verify JWT Signature → Invalid → ❌ 401 Invalid Token
    ↓ Valid
[4] Service 'M' in services array? → NO → ❌ 403 No Access
    ↓ YES
✅ Level 1: Authentication PASSED

[5] Check API Permission (RBAC)
    ↓
[6] Permission in Cache?
    ├─ YES → Check permission
    └─ NO → Query DB → Cache result
    ↓
[7] Has 'view_projects' permission? → NO → ❌ 403 Permission Denied
    ↓ YES
✅ Level 2: API Permission PASSED

[8] Accessing Specific Resource?
    ├─ NO (List All)
    │   ├─ Get accessible resources
    │   ├─ Filter projects by access
    │   ├─ Apply field-level filtering
    │   └─ ✅ 200 OK (Filtered List)
    │
    └─ YES (Resource ID: 123)
        ↓
    [9] Check Resource Permission (ABAC)
        ↓
    [10] Resource permission in cache?
        ├─ YES → Check access
        └─ NO → Query DB → Cache result
        ↓
    [11] Has access to Project 123? → NO → ❌ 403 Access Denied
        ↓ YES
    [12] Check permission level
        ├─ viewer → Hide sensitive fields
        ├─ editor → Hide some fields
        └─ owner → Full access
        ↓
    ✅ Level 3: Resource Permission PASSED

    [13] ABAC Conditions exist?
        ├─ NO → Fetch data
        └─ YES → Evaluate conditions
            ↓
        [14] Conditions met? → NO → ❌ 403 Conditions Failed
            ↓ YES
        [15] Fetch project data
            ↓
        [16] Apply field-level masking
            ↓
        ✅ 200 OK (Project Data)

    [17] Log access in audit trail
        ↓
    END
```

---

## 🎨 How to Use These Diagrams

### For Developers
- **system_architecture.mermaid** → Understand system components
- **authentication_flow_diagram.mermaid** → Implement login flow
- **jwt_token_flow.mermaid** → Implement JWT handling
- **permission_check_flow.mermaid** → Implement authorization

### For Database Admins
- **complete_er_diagram.mermaid** → Full database structure
- **er_diagram_auth_service.mermaid** → Central auth schema
- **er_diagram_microservice.mermaid** → Service-specific schema

### For Architects
- **system_architecture.mermaid** → Overall design
- All diagrams → Technical implementation details

### For Security Auditors
- **jwt_token_flow.mermaid** → Token security model
- **permission_check_flow.mermaid** → Authorization logic
- **authentication_flow_diagram.mermaid** → Authentication process

---

## 📝 Key Concepts Illustrated

### 1. Centralized Identity + Distributed Authorization
- **Central Auth**: One source of truth for users
- **Microservices**: Each manages its own permissions
- **Benefits**: Scalability, autonomy, performance

### 2. Asymmetric JWT (RS256)
- **Private Key**: Signs tokens (Auth Service only)
- **Public Key**: Verifies tokens (All microservices)
- **Security**: Can't forge tokens without private key

### 3. Multi-Level Authorization
- **Level 1**: Service access (JWT services array)
- **Level 2**: API permissions (RBAC - role-based)
- **Level 3**: Resource permissions (ABAC - attribute-based)

### 4. Caching Strategy
- **Permission cache**: 10 minutes TTL
- **Resource cache**: 5 minutes TTL
- **Invalidation**: Event-driven via Kafka

### 5. Same Email, Multiple Services
- One user record in central auth
- Multiple service mappings
- Independent permissions per service

---

## 🚀 Implementation Order

1. ✅ Review all diagrams
2. ✅ Understand authentication flow
3. ✅ Set up central auth database
4. ✅ Implement JWT generation/verification
5. ✅ Set up first microservice with permissions
6. ✅ Implement caching layer
7. ✅ Add Kafka for events
8. ✅ Deploy monitoring

---

**Tip**: Start with the `system_architecture.mermaid` to get the big picture, then dive into specific diagrams based on what you're implementing.
