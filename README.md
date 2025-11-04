# Multi-Microservice Authentication & Authorization System
## Architecture & Database Design Documentation

This documentation package contains comprehensive diagrams for a distributed authentication and authorization system with **centralized identity management** and **distributed permissions** across multiple microservices (LO, DH, M, Dashboard).

---

## 📋 Documentation Contents

### 1. **system_architecture.mermaid**
   - **Complete System Architecture Diagram**
   - Shows all components, services, databases, and their interactions
   - Includes: API Gateway, Auth Service, 4 Microservices, Message Queue, External Services
   - Visual representation of request flow and data connections

### 2. **complete_er_diagram.mermaid**
   - **Comprehensive Entity-Relationship Diagram**
   - Shows ALL database tables across Central Auth and Microservices
   - Includes relationships between tables
   - Shows primary keys, foreign keys, and key fields

### 3. **er_diagram_auth_service.mermaid**
   - **Central Authentication Service Database Schema**
   - Tables: users, user_service_access, otp_logs, refresh_tokens, auth_audit_logs
   - Detailed field types and constraints

### 4. **er_diagram_microservice.mermaid**
   - **Microservice Permission Database Schema** (M Service Example)
   - Tables: roles, user_roles, permissions, role_permissions, resource_permissions
   - Shows RBAC and ABAC implementation structure

### 5. **authentication_flow_diagram.mermaid**
   - **Complete Authentication Flow Sequence Diagram**
   - End-to-end flow from OTP request to API authorization
   - Shows all phases:
     - Phase 1: OTP Request
     - Phase 2: OTP Verification & JWT Generation
     - Phase 3: API Request with JWT
     - Phase 4: Admin User Creation
     - Phase 5: Permission Change & Cache Invalidation

### 6. **jwt_token_flow.mermaid**
   - **JWT Token Lifecycle Diagram**
   - Shows how tokens are generated, distributed, and verified
   - Includes:
     - Key generation and storage
     - Token signing with private key
     - Token verification with public key
     - Cache invalidation strategy

### 7. **permission_check_flow.mermaid**
   - **Permission Checking Decision Tree**
   - Multi-level authorization flow:
     - Level 1: JWT Authentication
     - Level 2: API-Level Permissions (RBAC)
     - Level 3: Resource-Level Permissions (ABAC)
   - Shows all decision points and error conditions

---

## 🎨 How to View These Diagrams

These diagrams are created in **Mermaid format**, which can be viewed in several ways:

### Option 1: Online Mermaid Live Editor (Easiest)
1. Go to https://mermaid.live/
2. Copy the content of any `.mermaid` file
3. Paste it into the editor
4. View the rendered diagram instantly
5. Export as PNG/SVG if needed

### Option 2: VS Code Extension
1. Install "Markdown Preview Mermaid Support" extension in VS Code
2. Create a markdown file and wrap the mermaid code:
   ```markdown
   ```mermaid
   [paste diagram content here]
   ```
   ```
3. Open markdown preview (Ctrl+Shift+V)

### Option 3: GitHub/GitLab
1. Upload the `.mermaid` files to a GitHub repository
2. Create a README.md and embed the diagrams:
   ```markdown
   ```mermaid
   [paste diagram content here]
   ```
   ```
3. GitHub will render them automatically

### Option 4: Command Line Tool
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i system_architecture.mermaid -o system_architecture.png
```

---

## 🏗️ System Architecture Overview

### Core Components

1. **API Gateway (Kong)**
   - Entry point for all requests
   - JWT validation with public key
   - Rate limiting and routing

2. **Central Authentication Service**
   - User management (create, login, OTP)
   - JWT token generation (RS256 signing)
   - Service-level access control
   - JWKS endpoint for public key distribution

3. **Microservices (LO, DH, M, Dashboard)**
   - Independent services with own databases
   - JWT verification with public key
   - Service-specific permission management
   - RBAC (Role-Based Access Control)
   - ABAC (Attribute-Based Access Control)

4. **Message Queue (Kafka)**
   - Event-driven architecture
   - Permission synchronization
   - User lifecycle events

5. **Caching Layer (Redis)**
   - OTP storage
   - Permission caching
   - Session management

---

## 🔐 Security Model

### Authentication
- **Asymmetric JWT (RS256)**: Private key signs, public key verifies
- **OTP-based login**: SMS for phone, Email for email addresses
- **Short-lived access tokens**: 30 minutes expiry
- **Long-lived refresh tokens**: 7 days expiry with revocation support

### Authorization - Multi-Level Approach

#### Level 1: Service-Level Access
- Controlled by `user_service_access` table in Central Auth
- JWT contains `services` array
- User must have explicit service access

#### Level 2: API-Level Permissions (RBAC)
- Role-based permission checks
- Cached in Redis for performance
- Examples: `view_projects`, `create_lead`, `edit_report`

#### Level 3: Resource-Level Permissions (ABAC)
- Specific resource access control
- Permission levels: owner, editor, viewer
- Field-level filtering based on permission level
- Condition-based access (e.g., region, department)

---

## 📊 Database Design Highlights

### Central Auth Database
- **users**: Core identity table (user_id is UUID)
- **user_service_access**: Maps users to services (enables same email across multiple services)
- **otp_logs**: OTP verification audit trail
- **refresh_tokens**: Long-lived token management
- **auth_audit_logs**: Complete authentication event logging

### Microservice Databases (Each service has these tables)
- **roles**: Service-specific roles
- **user_roles**: User-role assignments (references central user_id)
- **permissions**: Granular permission definitions
- **role_permissions**: Many-to-many role-permission mapping
- **resource_permissions**: Resource-level access control (ABAC)

### Key Design Decisions

✅ **Same Email, Multiple Services**: One user record in central auth, multiple service access records

✅ **Distributed Permissions**: Each microservice manages its own permissions for autonomy and scalability

✅ **Caching Strategy**: Redis cache with 5-10 minute TTL, invalidated on permission changes

✅ **Event-Driven Sync**: Kafka events for permission changes and user lifecycle

✅ **Audit Everything**: Comprehensive logging for security and compliance

---

## 🔄 Key Workflows

### Workflow 1: New User Creation (Email-based)
1. Admin creates user in Central Auth with email
2. Admin assigns services (e.g., "M", "Dashboard")
3. User record created in `users` table
4. Service mappings created in `user_service_access` table
5. Kafka event published: `user.service.assigned`
6. Microservices receive event and create default role assignments

### Workflow 2: User Login
1. User enters email/phone and selects service
2. Central Auth checks user exists and has service access
3. OTP generated, hashed, and sent via SMS/Email
4. User enters OTP for verification
5. Auth Service generates JWT with RS256 private key
6. JWT includes user_id and services array
7. Token returned to client

### Workflow 3: API Request Authorization
1. Client sends request with JWT in Authorization header
2. API Gateway validates JWT signature with public key
3. Gateway checks service access in JWT
4. Request forwarded to microservice
5. Microservice re-verifies JWT with public key
6. Microservice checks API-level permission (RBAC) in cache/DB
7. Microservice checks resource-level permission (ABAC) if accessing specific resource
8. Data returned with appropriate field-level filtering

### Workflow 4: Permission Change
1. Admin updates user role/permissions in microservice
2. Microservice updates database
3. Microservice deletes Redis cache keys for user
4. Microservice publishes Kafka event: `permission.changed`
5. Other service instances receive event and clear their caches
6. Next API call fetches fresh permissions from database

---

## 📈 Performance Optimization

### Caching Strategy
- **Permission cache TTL**: 10 minutes
- **Resource permission cache TTL**: 5 minutes
- **Cache key patterns**:
  - `user:{user_id}:permissions:{service}`
  - `user:{user_id}:resource:{resource_type}:{resource_id}`

### Cache Invalidation Triggers
- User role change
- Permission assignment/revocation
- Service access change
- User deletion

### Query Optimization
- Indexes on all foreign keys
- Composite indexes on frequently queried columns
- Resource permission queries filtered in SQL WHERE clause
- Connection pooling for all database connections

---

## 🔑 JWT Token Structure

### Access Token (30 min expiry)
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "identifier": "user@example.com",
  "identifier_type": "email",
  "services": ["M", "Dashboard"],
  "type": "access",
  "iss": "auth.yourapp.com",
  "aud": ["LO", "DH", "M", "Dashboard"],
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "exp": 1699000000,
  "iat": 1698998200,
  "jti": "unique-token-id"
}
```

### Refresh Token (7 days expiry)
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "service": "M",
  "type": "refresh",
  "iss": "auth.yourapp.com",
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "exp": 1699604800,
  "iat": 1698998200,
  "jti": "unique-refresh-token-id"
}
```

---

## 🚀 Deployment Architecture

### Production Setup
- **API Gateway**: Kong on Kubernetes, 3+ replicas
- **Auth Service**: 3+ replicas with load balancing
- **Microservices**: Independent deployment, auto-scaling enabled
- **Databases**: PostgreSQL with read replicas
- **Redis**: Redis Cluster for high availability
- **Kafka**: 3-broker cluster with replication
- **Secret Management**: HashiCorp Vault or AWS KMS for private keys

### Monitoring
- **Prometheus**: Metrics collection
- **Grafana**: Dashboards and alerting
- **ELK Stack**: Log aggregation and analysis
- **Key Metrics**:
  - Login success/failure rates
  - Token issuance trends
  - Permission check latency
  - Cache hit ratios
  - API request duration

---

## 🛡️ Security Best Practices Implemented

✅ Asymmetric JWT (RS256) - can't forge tokens without private key
✅ Private key stored in secure vault (Vault/KMS), never in code
✅ Public key distributed via JWKS endpoint
✅ Short-lived access tokens (30 min)
✅ Refresh token rotation and revocation
✅ OTP with rate limiting (max 3 attempts, 5 min expiry)
✅ Comprehensive audit logging
✅ SQL injection prevention (parameterized queries)
✅ HTTPS/TLS for all communication
✅ Redis for caching, not for authentication data
✅ Permission caching with invalidation strategy
✅ Multi-level authorization (service, API, resource)

---

## 📝 API Endpoints Summary

### Authentication Service
- `POST /api/v1/auth/request-otp` - Request OTP for login
- `POST /api/v1/auth/verify-otp` - Verify OTP and get tokens
- `POST /api/v1/auth/refresh-token` - Refresh access token
- `POST /api/v1/auth/logout` - Logout and revoke tokens
- `GET /.well-known/jwks.json` - Get public keys (JWKS)

### Admin Endpoints
- `POST /api/v1/admin/users` - Create new user (email-based)
- `PUT /api/v1/admin/users/:userId/services` - Add/remove service access
- `GET /api/v1/admin/users/:userId` - Get user details

### Microservice Endpoints (Example: M Service)
- `GET /api/v1/m/projects` - List accessible projects
- `POST /api/v1/m/projects` - Create new project
- `GET /api/v1/m/projects/:id` - Get project details
- `PUT /api/v1/m/projects/:id` - Update project
- `DELETE /api/v1/m/projects/:id` - Delete project

---

## 📚 Technology Stack

### Backend
- **Language**: Node.js / JavaScript
- **Framework**: Express.js
- **Authentication**: jsonwebtoken, bcryptjs
- **Database**: PostgreSQL 15+
- **Cache**: Redis 7+
- **Message Queue**: Apache Kafka
- **API Gateway**: Kong

### Infrastructure
- **Container**: Docker
- **Orchestration**: Kubernetes
- **Secret Management**: HashiCorp Vault / AWS KMS
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack

### External Services
- **SMS**: Twilio
- **Email**: SendGrid / AWS SES
- **Cloud**: AWS / GCP / Azure

---

## 🎯 Key Benefits of This Architecture

1. **Scalability**: Each microservice scales independently
2. **Security**: Asymmetric JWT prevents token forgery
3. **Performance**: Redis caching minimizes database load
4. **Flexibility**: Services can evolve permissions independently
5. **Auditability**: Comprehensive logging for compliance
6. **Resilience**: Event-driven architecture with retry logic
7. **Multi-tenancy**: Same email can access multiple services
8. **Fine-grained Control**: RBAC + ABAC for precise authorization

---

## 🔄 Next Steps for Implementation

1. **Phase 1: Core Infrastructure**
   - Set up databases (PostgreSQL, Redis)
   - Deploy Kafka cluster
   - Configure Kong API Gateway

2. **Phase 2: Central Auth Service**
   - Implement JWT generation/verification
   - Build OTP service with Twilio/SendGrid
   - Create user management APIs
   - Set up JWKS endpoint

3. **Phase 3: First Microservice (LO)**
   - Implement JWT verification middleware
   - Create permission tables
   - Build permission service with caching
   - Implement Kafka event consumers

4. **Phase 4: Remaining Microservices**
   - Replicate permission structure
   - Deploy with same patterns
   - Configure event subscriptions

5. **Phase 5: Monitoring & Security**
   - Set up Prometheus metrics
   - Configure Grafana dashboards
   - Implement ELK stack
   - Security audit and penetration testing

6. **Phase 6: Production Hardening**
   - Load testing
   - Key rotation procedures
   - Disaster recovery planning
   - Performance optimization

---

## 📞 Support & Documentation

For questions or clarifications about this architecture:
- Review the sequence diagrams for workflow understanding
- Check the ER diagrams for database relationships
- Refer to the permission flow diagram for authorization logic

---

**Version**: 1.0  
**Last Updated**: November 2025  
**Status**: Design Complete - Ready for Implementation
