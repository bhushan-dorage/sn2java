Claude Code Prompt
CONTEXT
=======
I have an iQube V2 case management platform with:

1. A working HTML/JS mockup (case-management.html) — attached. This is the 
   visual reference and source of truth for UI, terminology, data structures, 
   colour scheme, and navigation. All Angular components must match this 
   mockup exactly in look and behaviour.

2. A Spring Boot backend (iQube V2) already generated with:
   - Case/Incident CRUD APIs (REST)
   - Task management APIs
   - Dashboard metrics aggregation APIs
   - My Tasks / assignment APIs
   - PostgreSQL, Kafka, Redis, GKE deployment ready
   - Activiti 8 BPM for workflow

DESIGN SYSTEM
=============
- HSBC brand: primary colour #DB0011, white surfaces, #333333 text
- Fonts: DM Sans (body), IBM Plex Mono (IDs, numbers, code)
- Oat UI (https://oat.ink) for base components via CDN — keep using it
- All styling in component SCSS matching the CSS variables in case-management.html
- No Tailwind, no Bootstrap, no Material — Oat UI + custom SCSS only

TASK
====
Generate a complete, production-ready Angular 17+ project named "iqube-ui" 
that replicates the case-management.html mockup as a fully wired Angular 
application connected to the Spring Boot backend.

PROJECT STRUCTURE
=================
iqube-ui/
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── models/
│   │   │   │   ├── case.model.ts          (Case, CaseStatus, CasePriority, CaseType enums)
│   │   │   │   ├── task.model.ts          (Task, TaskStatus, TaskType, TaskHistory)
│   │   │   │   ├── dashboard.model.ts     (DashboardMetrics, KpiCard, SlaCompliance, TeamWorkload)
│   │   │   │   └── user.model.ts          (User, AssignmentGroup)
│   │   │   ├── services/
│   │   │   │   ├── case.service.ts        (CRUD + filter + pagination)
│   │   │   │   ├── task.service.ts        (CRUD + status toggle + my-tasks)
│   │   │   │   ├── dashboard.service.ts   (KPIs, category breakdown, SLA, team workload)
│   │   │   │   └── auth.service.ts        (current user, token)
│   │   │   ├── interceptors/
│   │   │   │   ├── auth.interceptor.ts    (Bearer token injection)
│   │   │   │   └── error.interceptor.ts   (global error toast)
│   │   │   └── guards/
│   │   │       └── auth.guard.ts
│   │   ├── shared/
│   │   │   ├── components/
│   │   │   │   ├── header/                (HSBC logo, iQube wordmark, search, avatar)
│   │   │   │   ├── sidebar/               (nav links, badges, collapsible toggle)
│   │   │   │   ├── breadcrumb/            (dynamic breadcrumb from router)
│   │   │   │   ├── priority-badge/        (Critical/High/Medium/Low pills)
│   │   │   │   ├── status-badge/          (New/In Progress/Pending/Resolved/Closed)
│   │   │   │   ├── task-type-pill/        (Investigate/Remediate/Communicate etc)
│   │   │   │   └── confirm-modal/         (reusable confirm dialog)
│   │   │   └── pipes/
│   │   │       ├── due-date.pipe.ts       (overdue/today/upcoming class logic)
│   │   │       └── truncate.pipe.ts
│   │   ├── features/
│   │   │   ├── dashboard/
│   │   │   │   ├── dashboard.component.ts/html/scss
│   │   │   │   ├── kpi-card/              (single KPI card with delta)
│   │   │   │   ├── sparkline/             (SVG case volume trend)
│   │   │   │   ├── category-bars/         (horizontal bar chart)
│   │   │   │   ├── sla-rings/             (SVG compliance rings)
│   │   │   │   ├── recent-cases/          (mini table)
│   │   │   │   ├── my-tasks-summary/      (top 5 tasks widget)
│   │   │   │   └── team-workload/         (avatar + dot bars)
│   │   │   ├── case-list/
│   │   │   │   ├── case-list.component.ts/html/scss
│   │   │   │   ├── case-filter-bar/       (chips + add filter)
│   │   │   │   └── new-case-modal/        (modal form)
│   │   │   ├── case-detail/
│   │   │   │   ├── case-detail.component.ts/html/scss
│   │   │   │   ├── tabs/
│   │   │   │   │   ├── details-tab/       (form grid, save/update)
│   │   │   │   │   ├── activity-tab/      (work notes, field changes, post note)
│   │   │   │   │   ├── tasks-tab/         (current tasks + history)
│   │   │   │   │   ├── related-tab/       (linked cases, changes)
│   │   │   │   │   └── approvals-tab/
│   │   │   │   └── detail-panel/          (SLA bars, dates, CI, contact, watchers)
│   │   │   ├── task-list/
│   │   │   │   └── task-list.component.ts/html/scss  (flat table, 4 filters)
│   │   │   └── my-tasks/
│   │   │       ├── my-tasks.component.ts/html/scss
│   │   │       ├── task-kanban/           (Overdue/Today/Upcoming columns)
│   │   │       └── task-history/          (completed timeline)
│   │   ├── app.routes.ts
│   │   ├── app.config.ts
│   │   └── app.component.ts
│   ├── environments/
│   │   ├── environment.ts                 (apiUrl: 'http://localhost:8080/api/v1')
│   │   └── environment.prod.ts            (apiUrl: '/api/v1')
│   └── styles/
│       ├── _variables.scss                (all CSS vars from mockup as SCSS vars)
│       ├── _oat-overrides.scss            (Oat UI customisations)
│       ├── _badges.scss                   (priority, status, task-type pills)
│       └── styles.scss

API CONTRACTS
=============
Generate Angular services that call these REST endpoints.
Infer the request/response shapes from the mockup data structures:

# Cases
GET    /api/v1/cases                    ?page&size&status&priority&type&q
GET    /api/v1/cases/{id}
POST   /api/v1/cases
PUT    /api/v1/cases/{id}
PATCH  /api/v1/cases/{id}/status
GET    /api/v1/cases/{id}/activity
POST   /api/v1/cases/{id}/notes
GET    /api/v1/cases/{id}/related

# Tasks
GET    /api/v1/tasks                    ?caseId&status&type&priority&assignee&page&size
GET    /api/v1/tasks/{id}
POST   /api/v1/tasks
PATCH  /api/v1/tasks/{id}/status
PUT    /api/v1/tasks/{id}
GET    /api/v1/tasks/my-tasks           (current user's tasks grouped: overdue/today/upcoming)
GET    /api/v1/tasks/history            ?assignee&days=30

# Dashboard
GET    /api/v1/dashboard/metrics        (KPI counts + deltas)
GET    /api/v1/dashboard/sla            (response/resolution/csat %)
GET    /api/v1/dashboard/volume         ?days=7 (sparkline data points)
GET    /api/v1/dashboard/by-category    (bar chart data)
GET    /api/v1/dashboard/team-workload  (per-member open/in-progress counts)

SPECIFIC REQUIREMENTS
=====================
1. ROUTING
   - /dashboard              → DashboardComponent (default)
   - /cases                  → CaseListComponent
   - /cases/:id              → CaseDetailComponent
   - /cases/:id/tasks        → redirects to CaseDetailComponent with tasks tab active
   - /tasks                  → TaskListComponent
   - /my-tasks               → MyTasksComponent
   - Lazy load all feature modules

2. STATE MANAGEMENT
   - Use Angular Signals (17+) for local component state
   - Use RxJS BehaviorSubject in services for shared state 
     (sidebar collapsed, current user, active filters)
   - No NgRx needed for this scope

3. SIDEBAR
   - Collapsible via ☰ hamburger in header (matches mockup toggle)
   - Active route highlighted with HSBC red left border
   - Badges (case counts) fetched from dashboard metrics API
   - Persists collapsed state in localStorage

4. DATA TABLE (case-list, task-list)
   - Client-side sort on all columns
   - Server-side pagination (page/size params)
   - Debounced search input (300ms)
   - Filter state synced to URL query params so filters survive refresh

5. CASE DETAIL TABS
   - Tab state in URL fragment (#details, #activity, #tasks, #related, #approvals)
   - Tasks tab: inline status toggle (open → in-progress → done) calls 
     PATCH /api/v1/tasks/{id}/status immediately
   - Activity tab: infinite scroll or load-more for long history
   - Auto-save indicator on details form (dirty state detection)

6. DASHBOARD
   - Refresh every 60 seconds via RxJS interval + switchMap
   - Sparkline and bar chart are pure SVG (no chart library dependency)
   - SLA rings use SVG stroke-dasharray animation on load
   - Recent cases table rows navigate to /cases/:id on click

7. MY TASKS KANBAN
   - Drag-and-drop between columns updates task status via API
   - Use @angular/cdk DragDrop module
   - Overdue column shows count badge in red

8. STYLING
   - Global CSS variables match case-management.html exactly:
     --brand: #DB0011, --text: #333333, --border: #e0e0e0, --surface: #f5f5f5
   - Oat UI loaded via angular.json styles array (CDN link in index.html)
   - Component SCSS uses variables only, no hardcoded colours
   - All hover/focus states match mockup behaviour

9. ERROR HANDLING
   - HTTP errors show a toast notification (bottom-right, auto-dismiss 4s)
   - 401 → redirect to /login
   - 404 on case detail → show "Case not found" empty state
   - Loading skeletons on all list/detail views (match Oat UI skeleton component)

10. ENVIRONMENT & BUILD
    - Angular 17 standalone components (no NgModules)
    - Strict TypeScript mode
    - Generate proxy.conf.json for local dev (proxy /api → localhost:8080)
    - Docker-ready: generate Dockerfile (nginx:alpine, copy dist, nginx.conf)
    - Generate nginx.conf with try_files for SPA routing

DELIVERABLES
============
Generate ALL of the following files with complete, working code 
(no stubs, no TODO comments):

1. All model interfaces (case, task, dashboard, user)
2. All service files with typed HTTP calls
3. All interceptors
4. App routes configuration
5. App config (provideHttpClient, provideRouter)
6. Header component (with HSBC SVG logo inline)
7. Sidebar component (with toggle, active state, badges)
8. Dashboard component + all sub-components
9. Case list component + filter bar + new case modal
10. Case detail component + all 5 tabs + side panel
11. Task list component
12. My Tasks component + kanban + history
13. All shared badge/pill components
14. Global SCSS files
15. proxy.conf.json
16. Dockerfile + nginx.conf
17. angular.json (with styles, assets configured)
18. package.json

REFERENCE
=========
The attached case-management.html is the complete visual and behavioural 
reference. Every screen, component, colour, interaction and data field in 
the Angular app must match it. Where the HTML uses inline JS mock data, 
replace with real API calls using the service layer described above.
How to use this with Claude Code:
Open your terminal in your project root
Run claude to start a session
Paste the prompt above
When it asks for attachments or context, reference case-management.html — either paste its contents or use:
claude --file case-management.html
After generation, run:
cd iqube-ui
npm install
ng serve --proxy-config proxy.conf.json
