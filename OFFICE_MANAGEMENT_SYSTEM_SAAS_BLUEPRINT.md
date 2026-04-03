# Office Management System (Creative Agency) — Scalable SaaS Blueprint

## 1) Product Architecture

### 1.1 High-Level Architecture (Web + Mobile Responsive)

- **Frontend (BFF-ready):** Next.js (App Router) + React + TypeScript + TailwindCSS + TanStack Query.
- **Backend API:** Node.js + Express (TypeScript) with modular service boundaries.
- **Database:** PostgreSQL (primary OLTP).
- **Auth:** JWT (access + refresh) with role claims (`SUPER_ADMIN`, `MANAGER`, `EMPLOYEE`).
- **Real-time:** WebSocket (Socket.IO) for live activity feed, task updates, notifications.
- **File storage:** S3-compatible object storage for attachments.
- **Background jobs:** BullMQ + Redis for reminders, digest summaries, SLA alerts.
- **Analytics pipeline:** Materialized views + scheduled ETL into summary tables.

### 1.2 Core Service Modules

1. **Identity & Access Module**
   - login, refresh token, password reset, RBAC policy checks.
2. **Organization Module**
   - departments (Design, Digital, Film, Studio), team mappings.
3. **Project Module**
   - project lifecycle, priorities, ownership, deadlines.
4. **Task Module**
   - tasks/subtasks, dependencies, statuses, comments, blockers.
5. **Activity Module**
   - immutable activity log for task/project updates.
6. **Leave Module**
   - leave requests, approval chain, team availability calendar.
7. **Analytics Module**
   - productivity, completion %, delay trends, workload distribution.
8. **Notification Module**
   - in-app + optional Slack/Email hooks.

### 1.3 Multi-Tenant SaaS Scalability

- Add `tenant_id` to all business tables.
- Composite indexes with `tenant_id` first for query isolation.
- Row-level security strategy (optional) in PostgreSQL.
- Feature flags per tenant (e.g., leave approval hierarchy).
- Partition heavy event tables by month (activity, notifications).

### 1.4 Suggested Deployment

- **Frontend:** Vercel.
- **API:** Dockerized Node/Express on ECS/Fargate or Kubernetes.
- **DB:** Managed PostgreSQL (RDS/Cloud SQL) with read replica for analytics reads.
- **Cache/Queue:** Redis.
- **Observability:** OpenTelemetry + Grafana + centralized logs.

---

## 2) Database Schema (PostgreSQL)

### 2.1 Entity Relationship Overview

- `tenants` 1—N `users`
- `roles` 1—N `users`
- `departments` 1—N `users`, 1—N `projects`
- `projects` 1—N `tasks`
- `tasks` 1—N `task_comments`
- `tasks` N—N `users` via `task_assignees`
- `tasks` N—N `tasks` via `task_dependencies`
- `users` 1—N `leave_requests`
- `projects/tasks/users` 1—N `remarks`
- all key actions -> `activity_events`

### 2.2 Schema (DDL-style)

```sql
-- Tenancy
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TYPE role_type AS ENUM ('SUPER_ADMIN', 'MANAGER', 'EMPLOYEE');
CREATE TYPE dept_type AS ENUM ('DESIGN', 'DIGITAL', 'FILM', 'STUDIO');
CREATE TYPE project_priority AS ENUM ('LOW', 'MEDIUM', 'HIGH', 'URGENT');
CREATE TYPE task_status AS ENUM ('TODO', 'IN_PROGRESS', 'DONE', 'BLOCKED');
CREATE TYPE leave_status AS ENUM ('PENDING', 'APPROVED', 'REJECTED', 'CANCELLED');

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  full_name TEXT NOT NULL,
  email TEXT NOT NULL,
  password_hash TEXT NOT NULL,
  role role_type NOT NULL,
  department dept_type,
  manager_id UUID REFERENCES users(id),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, email)
);

CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  description TEXT,
  department dept_type NOT NULL,
  priority project_priority NOT NULL DEFAULT 'MEDIUM',
  status TEXT NOT NULL DEFAULT 'ACTIVE',
  start_date DATE,
  deadline DATE,
  created_by UUID NOT NULL REFERENCES users(id),
  manager_id UUID NOT NULL REFERENCES users(id),
  progress_percent NUMERIC(5,2) NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE project_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),
  role_in_project TEXT NOT NULL DEFAULT 'CONTRIBUTOR',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, project_id, user_id)
);

CREATE TABLE tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  parent_task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  status task_status NOT NULL DEFAULT 'TODO',
  priority project_priority NOT NULL DEFAULT 'MEDIUM',
  deadline TIMESTAMPTZ,
  blocked_reason TEXT,
  progress_percent NUMERIC(5,2) NOT NULL DEFAULT 0,
  created_by UUID NOT NULL REFERENCES users(id),
  updated_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE task_assignees (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),
  assigned_by UUID NOT NULL REFERENCES users(id),
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, task_id, user_id)
);

CREATE TABLE task_dependencies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  depends_on_task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (task_id <> depends_on_task_id),
  UNIQUE (tenant_id, task_id, depends_on_task_id)
);

CREATE TABLE task_comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  author_id UUID NOT NULL REFERENCES users(id),
  body TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE task_attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  file_name TEXT NOT NULL,
  file_url TEXT NOT NULL,
  mime_type TEXT,
  file_size_bytes BIGINT,
  uploaded_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE remarks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  target_type TEXT NOT NULL, -- PROJECT | TASK | USER
  target_id UUID NOT NULL,
  author_id UUID NOT NULL REFERENCES users(id),
  body TEXT NOT NULL,
  visibility TEXT NOT NULL DEFAULT 'PRIVATE_TO_MANAGEMENT',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE leave_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  user_id UUID NOT NULL REFERENCES users(id),
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  reason TEXT,
  status leave_status NOT NULL DEFAULT 'PENDING',
  reviewed_by UUID REFERENCES users(id),
  reviewed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (end_date >= start_date)
);

CREATE TABLE daily_updates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  user_id UUID NOT NULL REFERENCES users(id),
  update_date DATE NOT NULL,
  summary TEXT NOT NULL,
  blockers TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, user_id, update_date)
);

CREATE TABLE activity_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  actor_id UUID REFERENCES users(id),
  entity_type TEXT NOT NULL, -- TASK | PROJECT | LEAVE | COMMENT
  entity_id UUID NOT NULL,
  action TEXT NOT NULL,      -- CREATED | UPDATED | STATUS_CHANGED ...
  metadata JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes for dashboard and analytics
CREATE INDEX idx_tasks_tenant_status_deadline ON tasks (tenant_id, status, deadline);
CREATE INDEX idx_tasks_tenant_project ON tasks (tenant_id, project_id);
CREATE INDEX idx_projects_tenant_department ON projects (tenant_id, department);
CREATE INDEX idx_activity_tenant_created_at ON activity_events (tenant_id, created_at DESC);
CREATE INDEX idx_leaves_tenant_status ON leave_requests (tenant_id, status);
```

---

## 3) API Structure (REST + Realtime)

Base path: `/api/v1`

### 3.1 Auth
- `POST /auth/login`
- `POST /auth/refresh`
- `POST /auth/logout`
- `GET /auth/me`

### 3.2 Users & Teams
- `GET /users` (CEO full, manager dept-filtered)
- `POST /users` (CEO)
- `PATCH /users/:id` (CEO, manager limited)
- `GET /teams/departments`
- `GET /teams/:department/overview`

### 3.3 Projects
- `GET /projects`
- `POST /projects`
- `GET /projects/:id`
- `PATCH /projects/:id`
- `DELETE /projects/:id` (CEO)
- `POST /projects/:id/members`

### 3.4 Tasks
- `GET /tasks` (filters: assignee, project, status, deadline, department)
- `POST /tasks`
- `GET /tasks/:id`
- `PATCH /tasks/:id`
- `POST /tasks/:id/status`
- `POST /tasks/:id/comments`
- `POST /tasks/:id/dependencies`
- `POST /tasks/:id/attachments`

### 3.5 Dashboard / Analytics
- `GET /dashboard/summary` (role-aware payload)
- `GET /dashboard/departments/:dept`
- `GET /analytics/productivity`
- `GET /analytics/workload`
- `GET /analytics/delays`

### 3.6 Leave
- `POST /leaves`
- `GET /leaves`
- `POST /leaves/:id/approve`
- `POST /leaves/:id/reject`

### 3.7 Activity Feed
- `GET /activity` (team/project/user filters)

### 3.8 WebSocket Events
- `task.updated`
- `task.status_changed`
- `comment.created`
- `leave.requested`
- `leave.reviewed`
- `project.updated`
- `notification.created`

### 3.9 RBAC Middleware (Express pseudo)

```ts
export const requireRole = (...allowed: Role[]) => {
  return (req, res, next) => {
    if (!req.user) return res.status(401).json({ message: 'Unauthorized' });
    if (!allowed.includes(req.user.role)) {
      return res.status(403).json({ message: 'Forbidden' });
    }
    next();
  };
};

export const requireDepartmentScope = (req, res, next) => {
  const user = req.user;
  if (user.role === 'SUPER_ADMIN') return next();

  const dept = req.query.department || req.body.department;
  if (user.role === 'MANAGER' && dept && dept !== user.department) {
    return res.status(403).json({ message: 'Out of department scope' });
  }
  next();
};
```

---

## 4) Wireframe Descriptions

### 4.1 App Shell

- **Left Sidebar (fixed):**
  - Logo + workspace switcher
  - Dashboard, Projects, Tasks, Teams, Analytics, Leaves
  - Quick create button (+)
- **Top Bar (sticky):**
  - Global search
  - Filters chips (dept, priority, status, overdue)
  - Notifications bell
  - Profile/avatar menu

### 4.2 CEO Dashboard Wireframe

- Row 1: KPI cards (Productivity %, Open Tasks, Delayed Tasks, Team Utilization)
- Row 2: **4 Department Buckets** (Design, Digital, Film, Studio) as equal-width cards
- Row 3: Delay heatmap + workload bar chart
- Row 4: Live activity stream + pending leave approvals

### 4.3 Manager Dashboard Wireframe

- KPI cards scoped to one department
- Department task board (To Do / In Progress / Done / Blocked)
- Team member utilization table
- Leave request queue

### 4.4 Employee Dashboard Wireframe

- “My Day” card: today tasks + deadlines
- Personal Kanban and blockers panel
- Daily update composer
- Leave balance and request shortcut

---

## 5) UI Component Breakdown (Next.js + Tailwind)

### 5.1 Core Layout Components
- `AppShell`
- `SidebarNav`
- `Topbar`
- `CommandSearch`
- `NotificationPanel`

### 5.2 Dashboard Components
- `KpiStatCard`
- `DepartmentBucketCard`
- `EmployeeTaskList`
- `StatusPill` (green/yellow/red)
- `ProgressRing`
- `DeadlineBadge`
- `BlockerTag`
- `WorkloadChart`
- `DelayTrendChart`
- `ActivityFeedList`

### 5.3 Task & Project Components
- `ProjectCreateModal`
- `TaskDrawer`
- `TaskKanbanBoard` (drag and drop)
- `DependencyGraphPopover`
- `CommentThread`
- `AttachmentUploader`

### 5.4 Leave & Team Components
- `LeaveRequestForm`
- `LeaveApprovalTable`
- `TeamAvailabilityCalendar`

### 5.5 UX States
- Skeleton loaders on all cards
- Empty states with CTA (e.g., “Create first project”)
- Inline validation + optimistic updates
- Keyboard shortcuts (e.g., `C` create, `/` search)

---

## 6) Sample Dashboard Layout (Detailed)

### 6.1 Visual Language
- **Design style:** Notion + Linear minimalism
- **Spacing:** 8px grid system
- **Surface:** light neutral cards with subtle borders/shadows
- **Color semantics:**
  - Done = green
  - In Progress = yellow
  - Blocked/Delayed = red

### 6.2 CEO Dashboard Composition

1. **Global Header Strip**
   - title + date range selector + saved views
2. **KPI Cluster (4 cards)**
   - Team productivity (`completed_tasks / planned_tasks`)
   - Workload balance (std deviation of active tasks per employee)
   - Delay ratio (`overdue/open`)
   - Leave impact index (people on leave / total)
3. **Department Buckets Grid (2x2)**
   - Each bucket contains:
     - department label + manager avatar
     - progress bar (overall completion)
     - employee chips (availability indicators)
     - top 5 active tasks with status and due date
     - blockers count + quick expand link
4. **Operational Insights Row**
   - left: delayed tasks panel grouped by department
   - right: workload distribution stacked bar by employee
5. **Live Activity + Approvals Row**
   - activity feed with real-time updates
   - pending leave approvals with one-click actions

### 6.3 Department Bucket Card (Detailed UX)

- Header: Department icon, name, completion %
- Body columns:
  - **People:** name, role, active tasks count, utilization badge
  - **Tasks:** task title, assignee, status pill, deadline
- Footer:
  - blockers summary (`2 critical blockers`)
  - button: “Open Department Dashboard”
- Interaction:
  - click card = deep-dive department page
  - hover task = quick actions (comment, reassign, mark blocked)

### 6.4 Mobile Responsive Behavior

- Sidebar becomes bottom tab bar.
- KPI cards become horizontal swipe carousel.
- Department buckets stack vertically.
- Floating “+” quick action for create task/project/update.
- Sticky compact filter chip row at top.

---

## 7) Key Implementation Snippets

### 7.1 Role-Aware Dashboard Query (Backend pseudo)

```ts
async function getDashboardSummary(user: AuthUser) {
  if (user.role === 'SUPER_ADMIN') {
    return dashboardRepo.getGlobalSummary(user.tenantId);
  }
  if (user.role === 'MANAGER') {
    return dashboardRepo.getDepartmentSummary(user.tenantId, user.department);
  }
  return dashboardRepo.getPersonalSummary(user.tenantId, user.id);
}
```

### 7.2 Department Bucket Component (React)

```tsx
export function DepartmentBucketCard({ bucket }: { bucket: DepartmentBucket }) {
  return (
    <div className="rounded-2xl border bg-white p-4 shadow-sm">
      <div className="mb-3 flex items-center justify-between">
        <h3 className="font-semibold">{bucket.department}</h3>
        <span className="text-sm text-slate-500">{bucket.progress}%</span>
      </div>
      <div className="space-y-2">
        {bucket.tasks.slice(0, 5).map((task) => (
          <div key={task.id} className="flex items-center justify-between rounded-lg bg-slate-50 px-3 py-2">
            <span className="text-sm">{task.title}</span>
            <span className={`text-xs ${statusClass(task.status)}`}>{task.status}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 7.3 Daily Standup Auto-Summary (AI-ready flow)

1. Collect each employee `daily_updates` at 6 PM local time.
2. Group by department and project.
3. Prompt LLM for concise summary:
   - completed today
   - in-progress
   - blockers requiring manager action
4. Post summary to activity feed + optional Slack channel.

---

## Bonus Features Roadmap

### Phase 1 (MVP)
- Auth + RBAC
- Projects, tasks, comments
- CEO/Manager/Employee dashboards
- Leave request + approvals

### Phase 2
- Drag-and-drop Kanban
- Advanced analytics + delay prediction rules
- Notification center + email digests

### Phase 3
- AI assistant (activity summarization, risk hints)
- Capacity planning suggestions
- Cross-tenant benchmark dashboards

---

## Recommended Delivery Plan (8–10 weeks)

- **Week 1–2:** Foundation, schema, auth, RBAC, org setup
- **Week 3–4:** Projects/tasks/dependencies/comments
- **Week 5–6:** Dashboard v1 (role-aware + department buckets)
- **Week 7:** Leave + activity feed real-time
- **Week 8:** Analytics + performance optimization
- **Week 9–10:** Hardening, QA, mobile polish, launch checklist

