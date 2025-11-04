# Architecture Diagrams - Complete Index

## 📑 All Diagrams at a Glance

| # | Diagram Name | Type | Purpose | Key Components | Use Case |
|---|-------------|------|---------|----------------|----------|
| 1 | **system_architecture.mermaid** | System Architecture | Complete system overview showing all services, databases, and connections | • API Gateway (Kong)<br>• Auth Service<br>• 4 Microservices (LO, DH, M, Dashboard)<br>• PostgreSQL & Redis per service<br>• Kafka Message Queue<br>• External Services (Twilio, SendGrid)<br>• Monitoring (Prometheus, Grafana, ELK)<br>• Key Management (Vault, JWKS) | Understanding overall system design, component interactions, and deployment architecture |
| 2 | **complete_er_diagram.mermaid** | Entity-Relationship | Full database schema across all services with relationships | **Central Auth:**<br>• users<br>• user_service_access<br>• otp_logs<br>• refresh_tokens<br>• auth_audit_logs<br><br>**Microservices:**<br>• roles<br>• user_roles<br>• permissions<br>• role_permissions<br>• resource_permissions<br>• projects<br>• leads | Database design, understanding table relationships, query planning |
| 3 | **er_diagram_auth_service.mermaid** | Entity-Relationship | Central authentication database schema in detail | • users (PK: user_id UUID)<br>• user_service_access (service mapping)<br>• otp_logs (OTP tracking)<br>• refresh_tokens (token management)<br>• auth_audit_logs (security events) | Setting up central auth database, understanding user identity model |
| 4 | **er_diagram_microservice.mermaid** | Entity-Relationship | Microservice permission database schema (M service example) | • roles (service-specific roles)<br>• user_roles (user assignments)<br>• permissions (granular permissions)<br>• role_permissions (RBAC mapping)<br>• resource_permissions (ABAC)<br>• projects & leads (business data) | Implementing microservice permissions, RBAC/ABAC design |
| 5 | **authentication_flow_diagram.mermaid** | Sequence Diagram | End-to-end authentication and authorization flow | **Phase 1:** OTP Request<br>**Phase 2:** OTP Verification & JWT Generation<br>**Phase 3:** API Request Authorization<br>**Phase 4:** Admin User Creation<br>**Phase 5:** Permission Change & Cache Invalidation | Implementing login flow, understanding request lifecycle, debugging authentication issues |
| 6 | **jwt_token_flow.mermaid** | Flow Diagram | JWT token lifecycle from generation to verification | **Key Generation:**<br>• RSA key pair<br>• Private key storage (Vault)<br>• Public key distribution (JWKS)<br><br>**Token Flow:**<br>• Signing (RS256 private key)<br>• Distribution<br>• Verification (public key)<br>• Caching<br>• Refresh<br>• Invalidation | Implementing JWT security, understanding asymmetric encryption, token management |
| 7 | **permission_check_flow.mermaid** | Decision Tree | Multi-level authorization decision flow | **Level 1:** JWT Authentication<br>• Token validation<br>• Service access check<br><br>**Level 2:** API Permissions (RBAC)<br>• Role-based checks<br>• Permission caching<br><br>**Level 3:** Resource Permissions (ABAC)<br>• Resource-level access<br>• Field-level filtering<br>• Condition evaluation | Implementing authorization logic, understanding permission hierarchy, debugging access issues |

---

## 🎯 Diagram Selection Guide

### By Role

| Role | Primary Diagrams | Secondary Diagrams |
|------|------------------|-------------------|
| **Backend Developer** | • authentication_flow_diagram<br>• jwt_token_flow<br>• permission_check_flow | • system_architecture<br>• er_diagram_auth_service<br>• er_diagram_microservice |
| **Database Administrator** | • complete_er_diagram<br>• er_diagram_auth_service<br>• er_diagram_microservice | • system_architecture |
| **System Architect** | • system_architecture<br>• jwt_token_flow | • complete_er_diagram<br>• authentication_flow_diagram |
| **Security Engineer** | • jwt_token_flow<br>• permission_check_flow<br>• authentication_flow_diagram | • system_architecture |
| **DevOps Engineer** | • system_architecture | • jwt_token_flow<br>• authentication_flow_diagram |
| **Frontend Developer** | • authentication_flow_diagram<br>• jwt_token_flow | • permission_check_flow |
| **Product Manager** | • system_architecture<br>• authentication_flow_diagram | • permission_check_flow |

### By Implementation Phase

| Phase | Diagrams to Review |
|-------|-------------------|
| **1. Architecture Planning** | • system_architecture<br>• complete_er_diagram |
| **2. Database Setup** | • complete_er_diagram<br>• er_diagram_auth_service<br>• er_diagram_microservice |
| **3. Authentication Development** | • authentication_flow_diagram<br>• jwt_token_flow<br>• er_diagram_auth_service |
| **4. Authorization Development** | • permission_check_flow<br>• er_diagram_microservice<br>• jwt_token_flow |
| **5. Microservice Integration** | • system_architecture<br>• authentication_flow_diagram<br>• permission_check_flow |
| **6. Testing & Debugging** | • authentication_flow_diagram<br>• permission_check_flow<br>• jwt_token_flow |
| **7. Security Audit** | • jwt_token_flow<br>• permission_check_flow<br>• system_architecture |
| **8. Deployment** | • system_architecture |

### By Question Type

| Question | Refer to Diagram |
|----------|-----------------|
| How does the overall system work? | **system_architecture.mermaid** |
| What tables do I need? | **complete_er_diagram.mermaid** or specific ER diagrams |
| How does login work? | **authentication_flow_diagram.mermaid** |
| How are JWT tokens created and verified? | **jwt_token_flow.mermaid** |
| How do permissions work? | **permission_check_flow.mermaid** |
| What's the relationship between users and services? | **er_diagram_auth_service.mermaid** |
| How does RBAC/ABAC work in microservices? | **er_diagram_microservice.mermaid** + **permission_check_flow.mermaid** |
| Where is data cached? | **jwt_token_flow.mermaid** + **system_architecture.mermaid** |
| How are services connected? | **system_architecture.mermaid** |

---

## 📊 Diagram Statistics

| Metric | Count |
|--------|-------|
| Total Diagrams | 7 |
| System Architecture Diagrams | 1 |
| ER Diagrams | 3 |
| Flow Diagrams | 3 |
| Total Components Shown | 50+ |
| Total Tables Documented | 14 |
| Total Relationships | 30+ |

---

## 🔍 Key Features Illustrated Across All Diagrams

### Security Features
- ✅ Asymmetric JWT (RS256) - shown in jwt_token_flow
- ✅ Private key isolation - shown in jwt_token_flow & system_architecture
- ✅ Public key distribution via JWKS - shown in jwt_token_flow & system_architecture
- ✅ Multi-level authorization - shown in permission_check_flow
- ✅ OTP with rate limiting - shown in authentication_flow_diagram
- ✅ Comprehensive audit logging - shown in er_diagram_auth_service
- ✅ Token refresh mechanism - shown in jwt_token_flow
- ✅ Cache invalidation strategy - shown in authentication_flow_diagram

### Scalability Features
- ✅ Distributed architecture - shown in system_architecture
- ✅ Service autonomy - shown in system_architecture & complete_er_diagram
- ✅ Redis caching - shown in all flow diagrams
- ✅ Event-driven sync (Kafka) - shown in system_architecture & authentication_flow
- ✅ Horizontal scaling support - shown in system_architecture
- ✅ Database per service pattern - shown in system_architecture

### Business Logic Features
- ✅ Multi-service user access - shown in er_diagram_auth_service
- ✅ Role-based permissions (RBAC) - shown in er_diagram_microservice & permission_check_flow
- ✅ Attribute-based permissions (ABAC) - shown in er_diagram_microservice & permission_check_flow
- ✅ Field-level data filtering - shown in permission_check_flow
- ✅ Resource-specific permissions - shown in permission_check_flow
- ✅ Multi-tenant support - shown in er_diagram_microservice

---

## 📈 Complexity Matrix

| Diagram | Technical Complexity | Business Complexity | Implementation Priority |
|---------|---------------------|--------------------|-----------------------|
| system_architecture | ⭐⭐⭐⭐⭐ | ⭐⭐ | 🔴 High |
| complete_er_diagram | ⭐⭐⭐⭐ | ⭐⭐⭐ | 🔴 High |
| er_diagram_auth_service | ⭐⭐⭐ | ⭐⭐ | 🔴 High |
| er_diagram_microservice | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 🟡 Medium |
| authentication_flow_diagram | ⭐⭐⭐⭐ | ⭐⭐⭐ | 🔴 High |
| jwt_token_flow | ⭐⭐⭐⭐⭐ | ⭐⭐ | 🔴 High |
| permission_check_flow | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 🟡 Medium |

**Legend:**
- ⭐ = Complexity level (more stars = more complex)
- 🔴 High Priority = Implement first
- 🟡 Medium Priority = Implement after high priority items

---

## 💡 Pro Tips for Using These Diagrams

### For Learning
1. Start with **system_architecture** for big picture
2. Move to **authentication_flow_diagram** to understand user journey
3. Deep dive into **jwt_token_flow** for security understanding
4. Study **permission_check_flow** for authorization logic
5. Reference ER diagrams when implementing databases

### For Implementation
1. Use **system_architecture** to plan infrastructure
2. Follow **er_diagrams** exactly for database schema
3. Implement flows step-by-step from sequence diagrams
4. Test against **permission_check_flow** decision tree

### For Debugging
1. Trace request through **authentication_flow_diagram**
2. Verify JWT handling against **jwt_token_flow**
3. Check authorization logic against **permission_check_flow**
4. Validate data model against **er_diagrams**

### For Documentation
1. Include **system_architecture** in technical docs
2. Reference **er_diagrams** in database documentation
3. Use **flow_diagrams** in API documentation
4. Share with team for onboarding

---

## 🔗 Diagram Interconnections

```
system_architecture.mermaid
    ├─ Shows WHERE components are
    └─ References
        ├─ complete_er_diagram.mermaid (database structure)
        ├─ authentication_flow_diagram.mermaid (how auth works)
        └─ jwt_token_flow.mermaid (how tokens flow)

authentication_flow_diagram.mermaid
    ├─ Shows HOW authentication happens
    └─ References
        ├─ er_diagram_auth_service.mermaid (tables used)
        ├─ jwt_token_flow.mermaid (token generation)
        └─ permission_check_flow.mermaid (authorization after auth)

jwt_token_flow.mermaid
    ├─ Shows HOW tokens work
    └─ References
        ├─ system_architecture.mermaid (key storage locations)
        └─ permission_check_flow.mermaid (how tokens are verified)

permission_check_flow.mermaid
    ├─ Shows HOW authorization decisions are made
    └─ References
        ├─ er_diagram_microservice.mermaid (permission tables)
        └─ jwt_token_flow.mermaid (token validation step)

complete_er_diagram.mermaid
    ├─ Shows ALL database tables
    └─ Combines
        ├─ er_diagram_auth_service.mermaid
        └─ er_diagram_microservice.mermaid
```

---

## 📝 Change Log

| Version | Date | Changes | Diagrams Updated |
|---------|------|---------|------------------|
| 1.0 | Nov 2025 | Initial release | All 7 diagrams created |

---

## 🎓 Learning Path

### Beginner Level (Understand the System)
1. Read system_architecture.mermaid
2. Review authentication_flow_diagram.mermaid
3. Understand jwt_token_flow.mermaid basics

### Intermediate Level (Implement Core Features)
1. Study er_diagram_auth_service.mermaid
2. Implement authentication flow
3. Study er_diagram_microservice.mermaid
4. Implement permission system

### Advanced Level (Master the System)
1. Deep dive into permission_check_flow.mermaid
2. Optimize caching strategies
3. Implement event-driven architecture
4. Handle edge cases and security hardening

---

**Last Updated**: November 2025  
**Total Pages**: 7 diagram files + 3 documentation files  
**Format**: Mermaid (.mermaid) + Markdown (.md)
