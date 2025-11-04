# 🔐 Microservice Permission Management - Complete Guide

## Detailed Explanation of RBAC & ABAC Implementation

This guide explains **exactly** how permissions work in each microservice, how roles control API access, and how data-level permissions filter what users can see.

---

## 📋 Table of Contents

1. [Permission Architecture Overview](#permission-architecture-overview)
2. [Database Tables in Detail](#database-tables-in-detail)
3. [API-Level Permissions (RBAC)](#api-level-permissions-rbac)
4. [Data-Level Permissions (ABAC)](#data-level-permissions-abac)
5. [Complete Implementation Flow](#complete-implementation-flow)
6. [Real-World Examples](#real-world-examples)
7. [Code Implementation](#code-implementation)

---

## 1. Permission Architecture Overview

### Three-Layer Permission Model

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: JWT Authentication (Central Auth)             │
│ ✓ User authenticated?                                   │
│ ✓ Has access to this microservice?                     │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: API-Level Permission (RBAC)                    │
│ ✓ Does user's ROLE have permission for this API?       │
│ Example: Can role "viewer" call POST /projects?        │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Data-Level Permission (ABAC)                   │
│ ✓ Can user access THIS specific resource?              │
│ ✓ What fields can user see?                            │
│ Example: Can user view project_id=123?                 │
└─────────────────────────────────────────────────────────┘
```

### Key Principle

**Each microservice is autonomous in managing its own permissions**, but references the central `user_id` from the auth service.

---

## 2. Database Tables in Detail

### 2.1 Roles Table

**Purpose**: Define service-specific roles (like job titles)

```sql
CREATE TABLE roles (
    role_id SERIAL PRIMARY KEY,
    role_name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    service_name VARCHAR(50) DEFAULT 'M',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example data
INSERT INTO roles (role_name, description) VALUES
    ('admin', 'Full access to all features and data'),
    ('manager', 'Can manage projects, leads, and team members'),
    ('team_lead', 'Can manage own team projects and leads'),
    ('sales_rep', 'Can view and create leads, limited project access'),
    ('viewer', 'Read-only access to assigned resources'),
    ('analyst', 'Can view data and generate reports');
```

**Real-world mapping**:
- `admin` = System Administrator
- `manager` = Sales Manager / Project Manager
- `team_lead` = Team Leader
- `sales_rep` = Sales Representative
- `viewer` = External Stakeholder / Auditor
- `analyst` = Data Analyst / Reporter

---

### 2.2 User Roles Table

**Purpose**: Assign roles to users (who has which role)

```sql
CREATE TABLE user_roles (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NOT NULL, -- From Central Auth Service
    role_id INT NOT NULL REFERENCES roles(role_id) ON DELETE CASCADE,
    scope JSONB, -- Optional: for multi-tenant/project-specific roles
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, role_id, scope),
    INDEX idx_user_id (user_id),
    INDEX idx_role_id (role_id)
);

-- Example data
INSERT INTO user_roles (user_id, role_id, scope) VALUES
    -- User 1 is admin (global)
    ('550e8400-e29b-41d4-a716-446655440000', 1, NULL),
    
    -- User 2 is manager (global)
    ('550e8400-e29b-41d4-a716-446655440001', 2, NULL),
    
    -- User 3 is team_lead for specific project
    ('550e8400-e29b-41d4-a716-446655440002', 3, '{"project_id": "proj-123"}'),
    
    -- User 4 is sales_rep (global)
    ('550e8400-e29b-41d4-a716-446655440003', 4, NULL),
    
    -- User 5 has multiple roles
    ('550e8400-e29b-41d4-a716-446655440004', 4, NULL), -- sales_rep globally
    ('550e8400-e29b-41d4-a716-446655440004', 3, '{"project_id": "proj-456"}'); -- team_lead for proj-456
```

**Key Points**:
- ✅ One user can have **multiple roles**
- ✅ Roles can be **global** (scope = NULL) or **scoped** (project-specific)
- ✅ `user_id` comes from Central Auth, but role assignment is in microservice

---

### 2.3 Permissions Table

**Purpose**: Define what actions exist in the system (granular actions)

```sql
CREATE TABLE permissions (
    permission_id SERIAL PRIMARY KEY,
    permission_name VARCHAR(100) UNIQUE NOT NULL,
    resource_type VARCHAR(50) NOT NULL, -- project, lead, report, user, team
    action VARCHAR(20) NOT NULL, -- view, create, edit, delete, export, share
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_resource_action (resource_type, action)
);

-- Example data
INSERT INTO permissions (permission_name, resource_type, action, description) VALUES
    -- Project permissions
    ('view_projects', 'project', 'view', 'View project list and details'),
    ('create_projects', 'project', 'create', 'Create new projects'),
    ('edit_projects', 'project', 'edit', 'Edit project details'),
    ('delete_projects', 'project', 'delete', 'Delete projects'),
    ('export_projects', 'project', 'export', 'Export project data'),
    ('share_projects', 'project', 'share', 'Share projects with others'),
    
    -- Lead permissions
    ('view_leads', 'lead', 'view', 'View lead list and details'),
    ('create_leads', 'lead', 'create', 'Create new leads'),
    ('edit_leads', 'lead', 'edit', 'Edit lead information'),
    ('delete_leads', 'lead', 'delete', 'Delete leads'),
    ('assign_leads', 'lead', 'assign', 'Assign leads to team members'),
    ('export_leads', 'lead', 'export', 'Export lead data'),
    
    -- Report permissions
    ('view_reports', 'report', 'view', 'View reports'),
    ('create_reports', 'report', 'create', 'Create custom reports'),
    ('export_reports', 'report', 'export', 'Export reports'),
    
    -- Team/User management permissions
    ('view_users', 'user', 'view', 'View user list'),
    ('manage_users', 'user', 'manage', 'Add/remove/edit users'),
    ('view_teams', 'team', 'view', 'View team structure'),
    ('manage_teams', 'team', 'manage', 'Manage team assignments');
```

**Key Points**:
- ✅ Permissions are **granular** - specific actions on specific resources
- ✅ Format: `{action}_{resource_type}` (e.g., `view_projects`, `create_leads`)
- ✅ Maps to **API endpoints** and **business operations**

---

### 2.4 Role Permissions Table

**Purpose**: Map which permissions each role has (RBAC mapping)

```sql
CREATE TABLE role_permissions (
    id BIGSERIAL PRIMARY KEY,
    role_id INT NOT NULL REFERENCES roles(role_id) ON DELETE CASCADE,
    permission_id INT NOT NULL REFERENCES permissions(permission_id) ON DELETE CASCADE,
    conditions JSONB, -- Optional: ABAC conditions
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(role_id, permission_id),
    INDEX idx_role_permission (role_id, permission_id)
);

-- Example: Admin has ALL permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 1, permission_id FROM permissions;

-- Example: Manager permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 2, permission_id FROM permissions 
WHERE permission_name IN (
    'view_projects', 'create_projects', 'edit_projects', 'export_projects',
    'view_leads', 'create_leads', 'edit_leads', 'delete_leads', 'assign_leads',
    'view_reports', 'create_reports', 'export_reports',
    'view_users', 'view_teams', 'manage_teams'
);

-- Example: Sales Rep permissions (limited)
INSERT INTO role_permissions (role_id, permission_id)
SELECT 4, permission_id FROM permissions 
WHERE permission_name IN (
    'view_projects', -- Can view projects
    'view_leads', 'create_leads', 'edit_leads', -- Can manage leads
    'view_reports' -- Can view reports
);

-- Example: Viewer permissions (read-only)
INSERT INTO role_permissions (role_id, permission_id)
SELECT 5, permission_id FROM permissions 
WHERE permission_name IN (
    'view_projects',
    'view_leads',
    'view_reports'
);

-- Example: Team Lead with conditions (ABAC)
INSERT INTO role_permissions (role_id, permission_id, conditions)
SELECT 3, permission_id, '{"own_team_only": true}'::jsonb
FROM permissions 
WHERE permission_name IN (
    'view_projects', 'edit_projects',
    'view_leads', 'edit_leads', 'assign_leads'
);
```

**Key Points**:
- ✅ This is the **RBAC heart** - defines what each role can do
- ✅ `conditions` allows adding **ABAC rules** (e.g., "only own team data")
- ✅ Easy to modify permissions by changing this table

---

### 2.5 Resource Permissions Table

**Purpose**: Grant specific users access to specific resources (ABAC)

```sql
CREATE TABLE resource_permissions (
    id BIGSERIAL PRIMARY KEY,
    resource_type VARCHAR(50) NOT NULL, -- project, lead, report
    resource_id VARCHAR(255) NOT NULL, -- Specific resource ID
    user_id UUID NOT NULL, -- From Central Auth
    permission_level VARCHAR(20) NOT NULL, -- owner, editor, viewer
    granted_by UUID, -- Who granted this permission
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP, -- Optional: temporary access
    UNIQUE(resource_type, resource_id, user_id),
    INDEX idx_resource_user (resource_type, resource_id, user_id),
    INDEX idx_user_resources (user_id, resource_type)
);

-- Example data
INSERT INTO resource_permissions (resource_type, resource_id, user_id, permission_level, granted_by) VALUES
    -- User A owns Project 1
    ('project', 'proj-123', '550e8400-e29b-41d4-a716-446655440000', 'owner', '550e8400-e29b-41d4-a716-446655440000'),
    
    -- User B is editor on Project 1
    ('project', 'proj-123', '550e8400-e29b-41d4-a716-446655440001', 'editor', '550e8400-e29b-41d4-a716-446655440000'),
    
    -- User C is viewer on Project 1
    ('project', 'proj-123', '550e8400-e29b-41d4-a716-446655440002', 'viewer', '550e8400-e29b-41d4-a716-446655440000'),
    
    -- User D owns Lead 456
    ('lead', 'lead-456', '550e8400-e29b-41d4-a716-446655440003', 'owner', '550e8400-e29b-41d4-a716-446655440003'),
    
    -- User E has temporary viewer access to Report 789 (expires in 30 days)
    ('report', 'report-789', '550e8400-e29b-41d4-a716-446655440004', 'viewer', '550e8400-e29b-41d4-a716-446655440000', NOW() + INTERVAL '30 days');
```

**Permission Levels Hierarchy**:

```
owner (highest)
  ↓ Can do everything including delete and share
editor
  ↓ Can view and edit, cannot delete or share
viewer (lowest)
  ↓ Can only view
```

**Key Points**:
- ✅ This is **ABAC** - specific users get specific access to specific resources
- ✅ Overrides role-based permissions for fine-grained control
- ✅ Supports **temporary access** with `expires_at`
- ✅ Tracks **who granted** access for accountability

---

## 3. API-Level Permissions (RBAC)

### How It Works

When a user calls an API endpoint, the system checks:
1. Does the user have a role?
2. Does that role have the required permission?

### Example Scenario

**User**: John (Sales Rep)  
**Role**: `sales_rep`  
**API Call**: `POST /api/v1/m/projects` (Create Project)

```
Request Flow:
1. API receives request with JWT
2. Extract user_id from JWT
3. Query: What roles does this user have?
   → Result: sales_rep
4. Query: Does sales_rep have "create_projects" permission?
   → Check role_permissions table
   → Result: NO (sales_rep only has view_projects)
5. Response: 403 Forbidden - "Missing permission: create_projects"
```

### Implementation in Middleware

```javascript
// middleware/permissionMiddleware.js

const requirePermission = (permissionName) => {
    return async (req, res, next) => {
        const userId = req.user.user_id; // From JWT
        
        // Step 1: Check cache first
        const cacheKey = `user:${userId}:permissions:M`;
        let userPermissions = await redis.get(cacheKey);
        
        if (!userPermissions) {
            // Step 2: Cache miss - query database
            const result = await db.query(`
                SELECT DISTINCT p.permission_name
                FROM user_roles ur
                JOIN role_permissions rp ON ur.role_id = rp.role_id
                JOIN permissions p ON rp.permission_id = p.permission_id
                WHERE ur.user_id = $1
            `, [userId]);
            
            userPermissions = result.rows.map(r => r.permission_name);
            
            // Step 3: Cache for 10 minutes
            await redis.setex(cacheKey, 600, JSON.stringify(userPermissions));
        } else {
            userPermissions = JSON.parse(userPermissions);
        }
        
        // Step 4: Check if user has required permission
        if (!userPermissions.includes(permissionName)) {
            return res.status(403).json({
                error: 'Permission denied',
                required_permission: permissionName,
                message: 'Your role does not have access to this operation'
            });
        }
        
        // Permission granted!
        next();
    };
};
```

### Applying to Routes

```javascript
// routes/projectRoutes.js

const router = express.Router();
const { requirePermission } = require('../middleware/permissionMiddleware');

// GET /projects - Requires "view_projects" permission
router.get('/', 
    authMiddleware, // Verify JWT first
    requirePermission('view_projects'), // Then check permission
    projectController.listProjects
);

// POST /projects - Requires "create_projects" permission
router.post('/', 
    authMiddleware,
    requirePermission('create_projects'),
    projectController.createProject
);

// PUT /projects/:id - Requires "edit_projects" permission
router.put('/:id', 
    authMiddleware,
    requirePermission('edit_projects'),
    projectController.updateProject
);

// DELETE /projects/:id - Requires "delete_projects" permission
router.delete('/:id', 
    authMiddleware,
    requirePermission('delete_projects'),
    projectController.deleteProject
);
```

### Permission Mapping to APIs

| HTTP Method | Endpoint | Required Permission | Description |
|-------------|----------|-------------------|-------------|
| GET | `/projects` | `view_projects` | List all accessible projects |
| GET | `/projects/:id` | `view_projects` | Get project details |
| POST | `/projects` | `create_projects` | Create new project |
| PUT | `/projects/:id` | `edit_projects` | Update project |
| DELETE | `/projects/:id` | `delete_projects` | Delete project |
| POST | `/projects/:id/share` | `share_projects` | Share project with others |
| GET | `/projects/:id/export` | `export_projects` | Export project data |

---

## 4. Data-Level Permissions (ABAC)

### How It Works

Even if a user has API-level permission (e.g., `view_projects`), they should only see **projects they have access to**.

### Example Scenario

**User**: Sarah (Viewer)  
**Role**: `viewer` (has `view_projects` permission ✓)  
**API Call**: `GET /api/v1/m/projects/123`

```
Request Flow:
1. API receives request
2. Check API permission: Does viewer have "view_projects"?
   → YES ✓
3. Check Resource permission: Does Sarah have access to project 123?
   → Query resource_permissions table
   → Result: YES, permission_level = "viewer"
4. Apply field-level filtering based on permission level
5. Return project data with sensitive fields hidden
```

### Two Approaches for Data-Level Permissions

#### Approach A: List Endpoints (Filter in Query)

For listing endpoints (GET /projects), filter at the SQL level:

```javascript
// controllers/projectController.js

async listProjects(req, res) {
    const userId = req.user.user_id;
    
    // Get projects where user has resource permission
    const projects = await db.query(`
        SELECT 
            p.*,
            rp.permission_level as user_permission_level
        FROM projects p
        INNER JOIN resource_permissions rp 
            ON p.project_id = rp.resource_id 
            AND rp.resource_type = 'project'
        WHERE rp.user_id = $1
            AND (rp.expires_at IS NULL OR rp.expires_at > NOW())
        ORDER BY p.created_at DESC
    `, [userId]);
    
    // Apply field-level filtering
    const filteredProjects = projects.rows.map(project => {
        return applyFieldLevelFiltering(project, project.user_permission_level);
    });
    
    res.json({ projects: filteredProjects });
}
```

**Result**: User only sees projects they have access to.

#### Approach B: Detail Endpoints (Check Specific Resource)

For detail endpoints (GET /projects/:id), check access to specific resource:

```javascript
// controllers/projectController.js

async getProject(req, res) {
    const userId = req.user.user_id;
    const projectId = req.params.id;
    
    // Check if user has access to this specific project
    const accessCheck = await db.query(`
        SELECT permission_level
        FROM resource_permissions
        WHERE resource_type = 'project'
            AND resource_id = $1
            AND user_id = $2
            AND (expires_at IS NULL OR expires_at > NOW())
    `, [projectId, userId]);
    
    if (accessCheck.rows.length === 0) {
        return res.status(403).json({
            error: 'Access denied',
            message: 'You do not have permission to view this project'
        });
    }
    
    const permissionLevel = accessCheck.rows[0].permission_level;
    
    // Get project data
    const project = await db.query(
        'SELECT * FROM projects WHERE project_id = $1',
        [projectId]
    );
    
    if (project.rows.length === 0) {
        return res.status(404).json({ error: 'Project not found' });
    }
    
    // Apply field-level filtering
    const filteredProject = applyFieldLevelFiltering(
        project.rows[0], 
        permissionLevel
    );
    
    res.json({ project: filteredProject });
}
```

### Field-Level Filtering Based on Permission Level

```javascript
// utils/fieldFiltering.js

function applyFieldLevelFiltering(data, permissionLevel) {
    // Define which fields each permission level can see
    const fieldAccess = {
        owner: [
            // Owners see everything
            'all_fields'
        ],
        editor: [
            // Editors see most fields except financial/internal
            'project_id', 'name', 'description', 'status', 'team_members',
            'created_at', 'updated_at', 'created_by', 'milestones', 'tasks'
            // Hidden: budget, internal_notes, profit_margin
        ],
        viewer: [
            // Viewers see basic information only
            'project_id', 'name', 'description', 'status', 
            'created_at', 'created_by'
            // Hidden: budget, internal_notes, profit_margin, team_members
        ]
    };
    
    if (permissionLevel === 'owner') {
        return data; // Return everything
    }
    
    const allowedFields = fieldAccess[permissionLevel] || fieldAccess.viewer;
    
    // Filter the object to only include allowed fields
    const filteredData = {};
    allowedFields.forEach(field => {
        if (data.hasOwnProperty(field)) {
            filteredData[field] = data[field];
        }
    });
    
    // Add permission level to response for transparency
    filteredData.user_permission_level = permissionLevel;
    
    return filteredData;
}
```

### Example Response Based on Permission Level

**Same Project, Different Users**:

**User A (Owner)**:
```json
{
    "project_id": "proj-123",
    "name": "Q4 Sales Campaign",
    "description": "Major sales push for Q4",
    "status": "active",
    "budget": 50000,
    "spent": 32000,
    "profit_margin": 0.35,
    "internal_notes": "Focus on enterprise clients",
    "team_members": ["user-1", "user-2", "user-3"],
    "created_by": "user-1",
    "created_at": "2024-10-01",
    "user_permission_level": "owner"
}
```

**User B (Editor)**:
```json
{
    "project_id": "proj-123",
    "name": "Q4 Sales Campaign",
    "description": "Major sales push for Q4",
    "status": "active",
    "team_members": ["user-1", "user-2", "user-3"],
    "created_by": "user-1",
    "created_at": "2024-10-01",
    "user_permission_level": "editor"
}
```

**User C (Viewer)**:
```json
{
    "project_id": "proj-123",
    "name": "Q4 Sales Campaign",
    "description": "Major sales push for Q4",
    "status": "active",
    "created_by": "user-1",
    "created_at": "2024-10-01",
    "user_permission_level": "viewer"
}
```

---

## 5. Complete Implementation Flow

### Full Request Flow Example

**User**: Alice (Team Lead)  
**Request**: `GET /api/v1/m/projects/proj-456`

```
Step 1: API Gateway (Kong)
├─ Validate JWT signature with public key
├─ Check expiration
└─ Forward to M Microservice ✓

Step 2: M Microservice - Authentication Middleware
├─ Verify JWT signature again (defense in depth)
├─ Extract user_id: '550e8400-e29b-41d4-a716-446655440002'
├─ Check service access: Is 'M' in JWT services array?
└─ Pass user info to next middleware ✓

Step 3: Permission Middleware - API Level (RBAC)
├─ Check Redis: user:550e8400...:permissions:M
├─ Cache HIT: ["view_projects", "edit_projects", "view_leads", ...]
├─ Required permission: "view_projects"
└─ User has permission ✓

Step 4: Controller - Resource Level (ABAC)
├─ Check Redis: user:550e8400...:resource:project:proj-456
├─ Cache MISS
├─ Query Database:
│   SELECT permission_level FROM resource_permissions
│   WHERE user_id = '550e8400...' 
│     AND resource_type = 'project'
│     AND resource_id = 'proj-456'
├─ Result: permission_level = 'editor'
├─ Cache result (TTL: 5 minutes)
└─ User has access ✓

Step 5: Data Retrieval
├─ Query Database:
│   SELECT * FROM projects WHERE project_id = 'proj-456'
└─ Get project data

Step 6: Field-Level Filtering
├─ Permission level: editor
├─ Filter fields based on editor access rules
├─ Hide: budget, internal_notes, profit_margin
└─ Return filtered data

Step 7: Response
└─ 200 OK with filtered project data
```

---

## 6. Real-World Examples

### Example 1: Creating a Project

**User**: Manager Mike  
**Request**: `POST /api/v1/m/projects`  
**Body**: `{ "name": "New Campaign", "description": "..." }`

```
Flow:
1. JWT validated ✓
2. Check permission: manager has "create_projects" ✓
3. Create project in database
4. Automatically grant Mike "owner" permission on this project:
   INSERT INTO resource_permissions 
   (resource_type, resource_id, user_id, permission_level, granted_by)
   VALUES ('project', 'new-proj-id', 'mike-user-id', 'owner', 'mike-user-id')
5. Return project data
```

**Result**: Mike created the project and is automatically the owner.

---

### Example 2: Sharing a Project

**User**: Manager Mike (owner of proj-123)  
**Request**: `POST /api/v1/m/projects/proj-123/share`  
**Body**: `{ "user_id": "sarah-user-id", "permission_level": "viewer" }`

```
Flow:
1. JWT validated ✓
2. Check API permission: manager has "share_projects" ✓
3. Check resource permission: Is Mike owner of proj-123?
   → Query resource_permissions
   → Result: YES, Mike is owner ✓
4. Grant Sarah viewer access:
   INSERT INTO resource_permissions 
   (resource_type, resource_id, user_id, permission_level, granted_by)
   VALUES ('project', 'proj-123', 'sarah-user-id', 'viewer', 'mike-user-id')
5. Clear Sarah's cache (she now has new permissions)
6. Return success
```

**Result**: Sarah can now view proj-123, but with limited fields.

---

### Example 3: Viewing Lead List

**User**: Sales Rep John  
**Request**: `GET /api/v1/m/leads`

```
Flow:
1. JWT validated ✓
2. Check permission: sales_rep has "view_leads" ✓
3. Query leads with resource permission filter:

   SELECT l.*, rp.permission_level
   FROM leads l
   INNER JOIN resource_permissions rp 
       ON l.lead_id = rp.resource_id 
       AND rp.resource_type = 'lead'
   WHERE rp.user_id = 'john-user-id'
       AND (rp.expires_at IS NULL OR rp.expires_at > NOW())

4. Result: Only leads John has access to
5. Apply field-level filtering for each lead
6. Return filtered list
```

**Result**: John only sees HIS leads, not all leads in the system.

---

### Example 4: Multi-Role User

**User**: Tom (has 2 roles)
- Role 1: `sales_rep` (global)
- Role 2: `team_lead` for project "proj-789"

**Request 1**: `POST /api/v1/m/projects` (Create Project)

```
Flow:
1. Check permissions from ALL roles:
   - sales_rep: [view_projects, view_leads, create_leads]
   - team_lead: [view_projects, edit_projects, assign_leads]
2. Combined permissions: [view_projects, view_leads, create_leads, edit_projects, assign_leads]
3. Required: "create_projects"
4. Result: NOT FOUND ✗
5. Response: 403 Forbidden
```

**Result**: Tom cannot create projects (neither role has this permission).

**Request 2**: `PUT /api/v1/m/leads/lead-456/assign` (Assign Lead)

```
Flow:
1. Check permissions from ALL roles:
   - Combined: [..., assign_leads] ✓
2. Check if lead-456 belongs to project proj-789:
   → Query: SELECT project_id FROM leads WHERE lead_id = 'lead-456'
   → Result: project_id = 'proj-789' ✓
3. Check Tom's team_lead scope:
   → Scope: {"project_id": "proj-789"} ✓
4. Tom can assign this lead ✓
5. Process assignment
6. Return success
```

**Result**: Tom can assign leads, but only for his project (proj-789).

---

### Example 5: Temporary Access

**Scenario**: External auditor needs 30-day access to a report

**Admin Action**: `POST /api/v1/m/reports/report-123/grant-access`  
**Body**: 
```json
{
    "user_id": "auditor-user-id",
    "permission_level": "viewer",
    "expires_in_days": 30
}
```

```
Database Insert:
INSERT INTO resource_permissions 
(resource_type, resource_id, user_id, permission_level, granted_by, expires_at)
VALUES (
    'report', 
    'report-123', 
    'auditor-user-id', 
    'viewer', 
    'admin-user-id',
    NOW() + INTERVAL '30 days'
)
```

**After 30 days**:
```
Auditor Request: GET /api/v1/m/reports/report-123

Query Check:
SELECT * FROM resource_permissions
WHERE resource_id = 'report-123'
    AND user_id = 'auditor-user-id'
    AND (expires_at IS NULL OR expires_at > NOW())

Result: No rows (expired)
Response: 403 Forbidden
```

**Result**: Access automatically revoked after 30 days.

---

## 7. Code Implementation

### Complete Permission Service

```javascript
// services/permissionService.js

const redis = require('../config/redis');
const db = require('../config/database');

class PermissionService {
    /**
     * Get all permissions for a user (combines all roles)
     */
    async getUserPermissions(userId) {
        const cacheKey = `user:${userId}:permissions:M`;
        
        // Check cache
        const cached = await redis.get(cacheKey);
        if (cached) {
            return JSON.parse(cached);
        }
        
        // Query database
        const result = await db.query(`
            SELECT DISTINCT 
                p.permission_name,
                p.resource_type,
                p.action,
                p.description,
                rp.conditions
            FROM user_roles ur
            JOIN role_permissions rp ON ur.role_id = rp.role_id
            JOIN permissions p ON rp.permission_id = p.permission_id
            WHERE ur.user_id = $1
        `, [userId]);
        
        const permissions = result.rows;
        
        // Cache for 10 minutes
        await redis.setex(cacheKey, 600, JSON.stringify(permissions));
        
        return permissions;
    }
    
    /**
     * Check if user has specific permission
     */
    async hasPermission(userId, permissionName) {
        const permissions = await this.getUserPermissions(userId);
        return permissions.some(p => p.permission_name === permissionName);
    }
    
    /**
     * Check if user has access to specific resource
     */
    async hasResourceAccess(userId, resourceType, resourceId, requiredLevel = 'viewer') {
        const cacheKey = `user:${userId}:resource:${resourceType}:${resourceId}`;
        
        // Check cache
        const cached = await redis.get(cacheKey);
        if (cached) {
            const level = cached;
            return this.hasRequiredLevel(level, requiredLevel);
        }
        
        // Query database
        const result = await db.query(`
            SELECT permission_level
            FROM resource_permissions
            WHERE user_id = $1 
                AND resource_type = $2 
                AND resource_id = $3
                AND (expires_at IS NULL OR expires_at > NOW())
        `, [userId, resourceType, resourceId]);
        
        if (result.rows.length === 0) {
            return false;
        }
        
        const level = result.rows[0].permission_level;
        
        // Cache for 5 minutes
        await redis.setex(cacheKey, 300, level);
        
        return this.hasRequiredLevel(level, requiredLevel);
    }
    
    /**
     * Get user's resource permission level
     */
    async getResourcePermissionLevel(userId, resourceType, resourceId) {
        const result = await db.query(`
            SELECT permission_level
            FROM resource_permissions
            WHERE user_id = $1 
                AND resource_type = $2 
                AND resource_id = $3
                AND (expires_at IS NULL OR expires_at > NOW())
        `, [userId, resourceType, resourceId]);
        
        return result.rows.length > 0 ? result.rows[0].permission_level : null;
    }
    
    /**
     * Check if user's permission level meets requirement
     */
    hasRequiredLevel(userLevel, requiredLevel) {
        const hierarchy = {
            'viewer': 1,
            'editor': 2,
            'owner': 3
        };
        
        return hierarchy[userLevel] >= hierarchy[requiredLevel];
    }
    
    /**
     * Get all resources user has access to (for list endpoints)
     */
    async getAccessibleResources(userId, resourceType) {
        const result = await db.query(`
            SELECT resource_id, permission_level
            FROM resource_permissions
            WHERE user_id = $1 
                AND resource_type = $2
                AND (expires_at IS NULL OR expires_at > NOW())
        `, [userId, resourceType]);
        
        return result.rows;
    }
    
    /**
     * Grant resource permission
     */
    async grantResourcePermission(resourceType, resourceId, userId, permissionLevel, grantedBy) {
        await db.query(`
            INSERT INTO resource_permissions 
                (resource_type, resource_id, user_id, permission_level, granted_by)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (resource_type, resource_id, user_id)
            DO UPDATE SET 
                permission_level = $4,
                granted_by = $5,
                created_at = NOW()
        `, [resourceType, resourceId, userId, permissionLevel, grantedBy]);
        
        // Invalidate cache
        const cacheKey = `user:${userId}:resource:${resourceType}:${resourceId}`;
        await redis.del(cacheKey);
    }
    
    /**
     * Revoke resource permission
     */
    async revokeResourcePermission(resourceType, resourceId, userId) {
        await db.query(`
            DELETE FROM resource_permissions
            WHERE resource_type = $1 
                AND resource_id = $2 
                AND user_id = $3
        `, [resourceType, resourceId, userId]);
        
        // Invalidate cache
        const cacheKey = `user:${userId}:resource:${resourceType}:${resourceId}`;
        await redis.del(cacheKey);
    }
    
    /**
     * Invalidate all permission caches for a user
     */
    async invalidateUserCache(userId) {
        const pattern = `user:${userId}:*`;
        const keys = await redis.keys(pattern);
        
        if (keys.length > 0) {
            await redis.del(...keys);
        }
    }
}

module.exports = new PermissionService();
```

### Complete Middleware Implementation

```javascript
// middleware/permissionMiddleware.js

const permissionService = require('../services/permissionService');

/**
 * Check if user has API-level permission (RBAC)
 */
const requirePermission = (permissionName) => {
    return async (req, res, next) => {
        try {
            const userId = req.user.user_id;
            
            const hasPermission = await permissionService.hasPermission(userId, permissionName);
            
            if (!hasPermission) {
                return res.status(403).json({
                    error: 'Permission denied',
                    required_permission: permissionName,
                    message: `You need '${permissionName}' permission to perform this action`
                });
            }
            
            next();
        } catch (error) {
            console.error('Permission check error:', error);
            res.status(500).json({ error: 'Permission check failed' });
        }
    };
};

/**
 * Check if user has resource-level permission (ABAC)
 */
const requireResourcePermission = (resourceType, resourceIdParam, requiredLevel = 'viewer') => {
    return async (req, res, next) => {
        try {
            const userId = req.user.user_id;
            const resourceId = req.params[resourceIdParam];
            
            const hasAccess = await permissionService.hasResourceAccess(
                userId,
                resourceType,
                resourceId,
                requiredLevel
            );
            
            if (!hasAccess) {
                return res.status(403).json({
                    error: 'Access denied',
                    resource_type: resourceType,
                    resource_id: resourceId,
                    required_level: requiredLevel,
                    message: `You do not have ${requiredLevel} access to this ${resourceType}`
                });
            }
            
            // Attach permission level to request for field filtering
            const permissionLevel = await permissionService.getResourcePermissionLevel(
                userId,
                resourceType,
                resourceId
            );
            req.resourcePermissionLevel = permissionLevel;
            
            next();
        } catch (error) {
            console.error('Resource permission check error:', error);
            res.status(500).json({ error: 'Permission check failed' });
        }
    };
};

module.exports = {
    requirePermission,
    requireResourcePermission
};
```

### Complete Controller with Permissions

```javascript
// controllers/projectController.js

const permissionService = require('../services/permissionService');
const { applyFieldLevelFiltering } = require('../utils/fieldFiltering');
const db = require('../config/database');

class ProjectController {
    /**
     * List projects - filtered by user's resource permissions
     */
    async listProjects(req, res) {
        try {
            const userId = req.user.user_id;
            
            // Get all projects user has access to
            const projects = await db.query(`
                SELECT 
                    p.*,
                    rp.permission_level as user_permission_level
                FROM projects p
                INNER JOIN resource_permissions rp 
                    ON p.project_id = rp.resource_id 
                    AND rp.resource_type = 'project'
                WHERE rp.user_id = $1
                    AND (rp.expires_at IS NULL OR rp.expires_at > NOW())
                ORDER BY p.created_at DESC
            `, [userId]);
            
            // Apply field-level filtering to each project
            const filteredProjects = projects.rows.map(project => {
                return applyFieldLevelFiltering(project, project.user_permission_level);
            });
            
            res.json({ 
                count: filteredProjects.length,
                projects: filteredProjects 
            });
            
        } catch (error) {
            console.error('List projects error:', error);
            res.status(500).json({ error: 'Failed to retrieve projects' });
        }
    }
    
    /**
     * Create project - automatically grant creator owner permission
     */
    async createProject(req, res) {
        try {
            const userId = req.user.user_id;
            const { name, description, budget, team_members } = req.body;
            
            // Start transaction
            const client = await db.pool.connect();
            
            try {
                await client.query('BEGIN');
                
                // Create project
                const projectResult = await client.query(`
                    INSERT INTO projects (name, description, budget, created_by)
                    VALUES ($1, $2, $3, $4)
                    RETURNING *
                `, [name, description, budget, userId]);
                
                const project = projectResult.rows[0];
                
                // Grant creator owner permission
                await client.query(`
                    INSERT INTO resource_permissions 
                        (resource_type, resource_id, user_id, permission_level, granted_by)
                    VALUES ('project', $1, $2, 'owner', $2)
                `, [project.project_id, userId]);
                
                // If team members specified, grant them editor access
                if (team_members && team_members.length > 0) {
                    for (const memberId of team_members) {
                        await client.query(`
                            INSERT INTO resource_permissions 
                                (resource_type, resource_id, user_id, permission_level, granted_by)
                            VALUES ('project', $1, $2, 'editor', $3)
                        `, [project.project_id, memberId, userId]);
                    }
                }
                
                await client.query('COMMIT');
                
                res.status(201).json({ 
                    message: 'Project created successfully',
                    project: project 
                });
                
            } catch (error) {
                await client.query('ROLLBACK');
                throw error;
            } finally {
                client.release();
            }
            
        } catch (error) {
            console.error('Create project error:', error);
            res.status(500).json({ error: 'Failed to create project' });
        }
    }
    
    /**
     * Get project details - with field-level filtering
     */
    async getProject(req, res) {
        try {
            const { projectId } = req.params;
            const permissionLevel = req.resourcePermissionLevel; // From middleware
            
            // Get project
            const result = await db.query(
                'SELECT * FROM projects WHERE project_id = $1',
                [projectId]
            );
            
            if (result.rows.length === 0) {
                return res.status(404).json({ error: 'Project not found' });
            }
            
            // Apply field-level filtering
            const project = applyFieldLevelFiltering(result.rows[0], permissionLevel);
            
            res.json({ project });
            
        } catch (error) {
            console.error('Get project error:', error);
            res.status(500).json({ error: 'Failed to retrieve project' });
        }
    }
    
    /**
     * Update project
     */
    async updateProject(req, res) {
        try {
            const { projectId } = req.params;
            const { name, description, status } = req.body;
            
            const result = await db.query(`
                UPDATE projects
                SET 
                    name = COALESCE($1, name),
                    description = COALESCE($2, description),
                    status = COALESCE($3, status),
                    updated_at = NOW()
                WHERE project_id = $4
                RETURNING *
            `, [name, description, status, projectId]);
            
            res.json({ 
                message: 'Project updated successfully',
                project: result.rows[0] 
            });
            
        } catch (error) {
            console.error('Update project error:', error);
            res.status(500).json({ error: 'Failed to update project' });
        }
    }
    
    /**
     * Delete project - only owners can delete
     */
    async deleteProject(req, res) {
        try {
            const { projectId } = req.params;
            
            await db.query('DELETE FROM projects WHERE project_id = $1', [projectId]);
            
            res.json({ message: 'Project deleted successfully' });
            
        } catch (error) {
            console.error('Delete project error:', error);
            res.status(500).json({ error: 'Failed to delete project' });
        }
    }
    
    /**
     * Share project - grant another user access
     */
    async shareProject(req, res) {
        try {
            const userId = req.user.user_id;
            const { projectId } = req.params;
            const { share_with_user_id, permission_level } = req.body;
            
            // Validate permission level
            if (!['owner', 'editor', 'viewer'].includes(permission_level)) {
                return res.status(400).json({ 
                    error: 'Invalid permission level. Must be: owner, editor, or viewer' 
                });
            }
            
            // Grant permission
            await permissionService.grantResourcePermission(
                'project',
                projectId,
                share_with_user_id,
                permission_level,
                userId
            );
            
            res.json({ 
                message: 'Project shared successfully',
                shared_with: share_with_user_id,
                permission_level: permission_level
            });
            
        } catch (error) {
            console.error('Share project error:', error);
            res.status(500).json({ error: 'Failed to share project' });
        }
    }
}

module.exports = new ProjectController();
```

### Complete Routes with Both Permission Levels

```javascript
// routes/projectRoutes.js

const express = require('express');
const router = express.Router();
const authMiddleware = require('../middleware/authMiddleware');
const { requirePermission, requireResourcePermission } = require('../middleware/permissionMiddleware');
const projectController = require('../controllers/projectController');

// All routes require authentication
router.use(authMiddleware);

/**
 * LIST: Get all accessible projects
 * Permission: API-level only (view_projects)
 * Filtering: Done in controller via SQL
 */
router.get('/', 
    requirePermission('view_projects'),
    projectController.listProjects
);

/**
 * CREATE: Create new project
 * Permission: API-level only (create_projects)
 * Auto-grants: Creator gets owner permission
 */
router.post('/', 
    requirePermission('create_projects'),
    projectController.createProject
);

/**
 * GET: Get specific project details
 * Permission: API-level + Resource-level
 * - API: Must have view_projects
 * - Resource: Must have viewer+ access to THIS project
 */
router.get('/:projectId', 
    requirePermission('view_projects'),
    requireResourcePermission('project', 'projectId', 'viewer'),
    projectController.getProject
);

/**
 * UPDATE: Update project
 * Permission: API-level + Resource-level
 * - API: Must have edit_projects
 * - Resource: Must have editor+ access to THIS project
 */
router.put('/:projectId', 
    requirePermission('edit_projects'),
    requireResourcePermission('project', 'projectId', 'editor'),
    projectController.updateProject
);

/**
 * DELETE: Delete project
 * Permission: API-level + Resource-level
 * - API: Must have delete_projects
 * - Resource: Must have owner access to THIS project
 */
router.delete('/:projectId', 
    requirePermission('delete_projects'),
    requireResourcePermission('project', 'projectId', 'owner'),
    projectController.deleteProject
);

/**
 * SHARE: Grant another user access to project
 * Permission: API-level + Resource-level
 * - API: Must have share_projects
 * - Resource: Must have owner access to THIS project
 */
router.post('/:projectId/share', 
    requirePermission('share_projects'),
    requireResourcePermission('project', 'projectId', 'owner'),
    projectController.shareProject
);

module.exports = router;
```

---

## Summary

### Key Takeaways

1. **Three Permission Layers**:
   - Layer 1: JWT (Central Auth) - Service access
   - Layer 2: RBAC - API-level permissions via roles
   - Layer 3: ABAC - Resource-level permissions with field filtering

2. **Roles Define API Access**:
   - Roles have permissions (role_permissions table)
   - Users have roles (user_roles table)
   - Permission middleware checks if user's roles have required permission

3. **Data-Level Permissions**:
   - Resource_permissions table controls access to specific resources
   - Permission levels: owner > editor > viewer
   - Field-level filtering based on permission level

4. **Caching Strategy**:
   - API permissions cached for 10 minutes
   - Resource permissions cached for 5 minutes
   - Invalidated on permission changes via Kafka events

5. **Flexibility**:
   - Users can have multiple roles (even scoped to specific projects)
   - Temporary access supported with expiration
   - Easy to add new permissions by adding to permissions table

This design gives you **maximum flexibility** while maintaining **strong security** and **optimal performance**! 🎉
