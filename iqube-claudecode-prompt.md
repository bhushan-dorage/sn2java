# iQube — Case Management Platform
## Claude Code Build Prompt (Microservices · Event-Driven · Full-Stack)

---

## CONTEXT & REFERENCE UI

You are building **iQube**, a production-grade, microservices-based, event-driven **Case Management Platform**.

A reference HTML mockup is attached (`iqube-hybrid-6-13.html`). Study it carefully before generating any code:

- **Brand**: HSBC-inspired. Primary red `#DB0011`, surface `#f5f5f5`, card `#ffffff`.
- **Typography**: IBM Plex Mono (monospace accents) + DM Sans (body).
- **Views in the mockup**: Dashboard, All Cases (AG Grid), All Tasks (AG Grid), My Tasks, Case Detail.
- **Key UI features**: Dark/Light theme toggle, sidebar navigation with collapsible state, AG Grid Enterprise (server-side row model), breadcrumbs, filters, column visibility, preferences drawer, pagination, priority/status badges.

> The Angular frontend **must faithfully reproduce** the mockup's layout, colour tokens, typography, component hierarchy, and interaction patterns.

---

## ARCHITECTURE OVERVIEW

```
┌─────────────────────────────────────────────────────────┐
│                  Angular SPA (iQube UI)                  │
│  (AG Grid · Material · HSBC design tokens from mockup)  │
└───────────────────────┬─────────────────────────────────┘
                        │ REST / WebSocket (STOMP)
          ┌─────────────▼──────────────┐
          │      API Gateway           │  Spring Cloud Gateway
          │  (routing · JWT auth)      │
          └──┬──────┬──────┬──────┬───┘
             │      │      │      │
   ┌─────────▼─┐ ┌──▼───┐ ┌▼──────────┐ ┌▼───────────┐
   │  Case      │ │ Task │ │ Workflow  │ │Notification│
   │  Service  │ │Svc   │ │  Service  │ │  Service   │
   │(Spring Boot│ │(SB)  │ │(SB+Activiti│ │  (SB)      │
   └─────┬─────┘ └──┬───┘ └─────┬─────┘ └─────┬──────┘
         │          │           │              │
         └──────────┴───────────┴──────────────┘
                        │ Apache Kafka (Event Bus)
         ┌──────────────┴────────────────────────┐
         │  Topics: case-events · task-events    │
         │  workflow-events · audit-events       │
         └───────────────────────────────────────┘
         │         PostgreSQL (per-service DB)   │
         │         Redis   (cache · sessions)    │
         └───────────────────────────────────────┘
```

---

## TECH STACK

| Layer | Technology |
|---|---|
| Backend services | Java 17, Spring Boot 3.x, Spring Security (JWT) |
| Workflow engine | Activiti 8 (embedded per Workflow Service) |
| Messaging | Apache Kafka (Spring Kafka) |
| Database | PostgreSQL 15 (one schema per service) |
| Cache | Redis (Spring Data Redis + Lettuce) |
| Frontend | Angular 17, AG Grid Enterprise (community licence stub), Angular Material |
| API Gateway | Spring Cloud Gateway |
| Containerisation | Docker + Docker Compose |
| Build | Maven (backend), Angular CLI (frontend) |

---

## PROJECT STRUCTURE TO GENERATE

```
iqube/
├── docker-compose.yml              # Postgres, Kafka, Zookeeper, Redis, all services
├── README.md
│
├── iqube-gateway/                  # Spring Cloud Gateway
│   ├── pom.xml
│   └── src/…
│
├── iqube-case-service/             # Case CRUD + domain events
│   ├── pom.xml
│   └── src/…
│
├── iqube-task-service/             # Task CRUD + assignments
│   ├── pom.xml
│   └── src/…
│
├── iqube-workflow-service/         # Activiti 8 process engine
│   ├── pom.xml
│   └── src/…                      # BPMN files under resources/processes/
│
├── iqube-notification-service/     # Kafka consumer → WebSocket push
│   ├── pom.xml
│   └── src/…
│
└── iqube-ui/                       # Angular 17 SPA
    ├── package.json
    └── src/…
```

---

## DOMAIN MODEL

### Case

```java
@Entity
public class Case {
    UUID id;
    String caseNumber;          // IQ-2025-XXXXX
    String title;
    CaseType type;              // CUSTOMER_COMPLAINT, KYC_REVIEW, FRAUD_INVESTIGATION,
                                // ACCOUNT_CLOSURE, REGULATORY_BREACH, TRADE_EXCEPTION
    CaseStatus status;          // NEW, IN_PROGRESS, PENDING_APPROVAL, ON_HOLD,
                                // ESCALATED, RESOLVED, CLOSED
    Priority priority;          // CRITICAL, HIGH, MEDIUM, LOW
    String category;
    String assignedTo;
    String team;
    LocalDateTime createdAt;
    LocalDateTime updatedAt;
    LocalDateTime dueDate;
    LocalDateTime slaDeadline;
    String processInstanceId;   // Activiti process link
    Map<String,Object> metadata;
    List<CaseAttachment> attachments;
    List<CaseNote> notes;
    List<AuditEntry> auditLog;
}
```

### Task

```java
@Entity
public class Task {
    UUID id;
    String taskNumber;          // TSK-XXXXX
    UUID caseId;
    String title;
    String description;
    TaskType type;              // REVIEW, APPROVAL, DATA_ENTRY, INVESTIGATION,
                                // COMPLIANCE_CHECK, NOTIFICATION, ESCALATION
    TaskStatus status;          // PENDING, IN_PROGRESS, COMPLETED, CANCELLED, BLOCKED
    Priority priority;
    String assignee;
    String activityId;          // Activiti user-task ID
    LocalDateTime createdAt;
    LocalDateTime dueDate;
    Map<String,Object> variables;
}
```

---

## SERVICE SPECIFICATIONS

### 1. `iqube-case-service` (Port 8081)

**Responsibilities**: Full CRUD for cases, SLA calculation, audit logging, Redis caching for case summaries.

**Key APIs**:
```
POST   /api/v1/cases                        Create case → publishes CaseCreatedEvent
GET    /api/v1/cases?page=&size=&sort=&filter=  Paginated list (supports AG Grid server-side)
GET    /api/v1/cases/{id}                   Case detail (cached in Redis TTL 5 min)
PUT    /api/v1/cases/{id}                   Update → publishes CaseUpdatedEvent
PATCH  /api/v1/cases/{id}/status            Status transition → publishes CaseStatusChangedEvent
DELETE /api/v1/cases/{id}                   Soft-delete
GET    /api/v1/cases/{id}/audit             Full audit trail
POST   /api/v1/cases/{id}/notes             Add note
GET    /api/v1/cases/stats                  Dashboard counts by status/priority/type
```

**AG Grid Server-Side Row Model support**: Accept `IFilterModel`, `ISortModel`, `startRow`, `endRow` from Angular AG Grid; translate to Spring `Pageable` + JPA `Specification`.

**Kafka topics produced**:
- `case-events` → `CaseCreatedEvent`, `CaseUpdatedEvent`, `CaseStatusChangedEvent`, `CaseClosedEvent`

**Redis caching**:
- Cache `GET /cases/{id}` with key `case:{id}`, TTL 5 min
- Evict on any write
- Cache dashboard stats with key `case:stats`, TTL 1 min

---

### 2. `iqube-task-service` (Port 8082)

**Responsibilities**: Task lifecycle management, assignment, completion, linked to Activiti user tasks.

**Key APIs**:
```
POST   /api/v1/tasks
GET    /api/v1/tasks?caseId=&assignee=&status=&page=&size=
GET    /api/v1/tasks/{id}
PUT    /api/v1/tasks/{id}
PATCH  /api/v1/tasks/{id}/status
PATCH  /api/v1/tasks/{id}/assign
GET    /api/v1/tasks/my?assignee=          My Tasks view
GET    /api/v1/tasks/stats
```

**Kafka topics produced**: `task-events` → `TaskCreatedEvent`, `TaskCompletedEvent`, `TaskAssignedEvent`

**Kafka topics consumed**: `workflow-events` → auto-create tasks when workflow reaches a user-task node.

---

### 3. `iqube-workflow-service` (Port 8083)

**Responsibilities**: Activiti 8 process engine, BPMN definitions per case type, process instance management.

**BPMN processes to include** (create `.bpmn20.xml` files under `src/main/resources/processes/`):

1. **customer-complaint.bpmn20.xml** — `New → Triage → Investigation → Resolution → Closure`
2. **kyc-review.bpmn20.xml** — `Initiate → Document Collection → Compliance Check → Approval → Closure`
3. **fraud-investigation.bpmn20.xml** — `Alert → Initial Assessment → Deep Investigation → Legal Review → Action → Close`

Each BPMN must include:
- User tasks (assignee expression `${assignee}`)
- Service tasks (Java delegate classes)
- Gateway-based conditional routing based on `priority` and `decision` variables
- Timer boundary events for SLA breach escalation
- Message events to receive external Kafka signals

**Key APIs**:
```
POST   /api/v1/workflow/start              Start process for a case type
GET    /api/v1/workflow/{processInstanceId} Process state
POST   /api/v1/workflow/{processInstanceId}/complete  Complete user task
POST   /api/v1/workflow/{processInstanceId}/signal    Send signal
GET    /api/v1/workflow/definitions        List available processes
GET    /api/v1/workflow/{processInstanceId}/diagram   Active task highlight
```

**Kafka topics produced**: `workflow-events` → `ProcessStartedEvent`, `UserTaskCreatedEvent`, `UserTaskCompletedEvent`, `ProcessCompletedEvent`, `SlaBreachEvent`

---

### 4. `iqube-notification-service` (Port 8084)

**Responsibilities**: Consume all domain events from Kafka, persist as notifications, push to connected Angular clients via WebSocket (STOMP).

**Kafka topics consumed**: `case-events`, `task-events`, `workflow-events`

**WebSocket endpoint**: `ws://localhost:8084/ws` — STOMP broker `/topic/notifications/{userId}`

**REST APIs**:
```
GET    /api/v1/notifications?userId=&unread=true
PATCH  /api/v1/notifications/{id}/read
PATCH  /api/v1/notifications/read-all
```

---

### 5. `iqube-gateway` (Port 8080)

**Responsibilities**: Route all traffic, JWT validation, CORS, rate limiting.

**Routes**:
```yaml
/api/cases/**     → iqube-case-service:8081
/api/tasks/**     → iqube-task-service:8082
/api/workflow/**  → iqube-workflow-service:8083
/api/notifications/** → iqube-notification-service:8084
/ws/**            → iqube-notification-service:8084 (WebSocket upgrade)
```

---

## KAFKA EVENTS CONTRACT

All events extend `BaseEvent`:
```java
public abstract class BaseEvent {
    String eventId;       // UUID
    String eventType;
    String aggregateId;
    String aggregateType;
    Instant occurredAt;
    String triggeredBy;
    Map<String,Object> payload;
}
```

Serialisation: JSON via Jackson. Topic key = `aggregateId`.

---

## ANGULAR FRONTEND SPECIFICATIONS

### Project setup
```bash
ng new iqube-ui --routing --style=scss --standalone
npm install ag-grid-community ag-grid-angular @stomp/stompjs sockjs-client
```

### Module structure
```
src/app/
├── core/
│   ├── auth/          (JWT interceptor, auth guard, login)
│   ├── services/      (case.service.ts, task.service.ts, workflow.service.ts,
│   │                   notification.service.ts, websocket.service.ts)
│   └── models/        (case.model.ts, task.model.ts, event.model.ts)
├── shared/
│   ├── components/    (badge, priority-chip, status-chip, ag-grid-wrapper,
│   │                   breadcrumb, loading-spinner, confirm-dialog)
│   └── directives/
├── features/
│   ├── dashboard/     (summary cards + charts)
│   ├── cases/         (cases-list, case-detail, case-create, case-edit)
│   ├── tasks/         (tasks-list, my-tasks, task-detail)
│   ├── workflow/      (workflow-viewer component — BPMN diagram with active task)
│   └── notifications/ (notification-panel)
└── layout/
    ├── header/
    ├── sidebar/
    └── main-layout/
```

### Design Implementation (from mockup)

**Reproduce EXACTLY from the mockup**:

1. **CSS Variables** — copy all `:root` and `[data-theme="dark"]` token blocks verbatim into `styles.scss`.

2. **Header** (`layout/header`):
   - Fixed, 52px height, 3px brand-red bottom border
   - iQube logo (brand red, bold), module breadcrumb, global search input, notification bell with dot, avatar circle, theme toggle pill button

3. **Sidebar** (`layout/sidebar`):
   - 200px wide, collapsible (0px collapsed), fixed left
   - Sections: WORKSPACE (Dashboard, My Tasks), CASES (All Cases, New Case, Escalated, Closed), TASKS (All Tasks, Pending Approval), ADMIN
   - Active item: left 3px brand-red border + brand-light background
   - Badges (count chips) in red and grey

4. **Dashboard view**:
   - 4 KPI cards: Open Cases, In Progress, Pending Approval, Resolved Today
   - Case breakdown by Type (bar-style rows with counts)
   - Recent Activity feed

5. **Cases List view** (AG Grid server-side):
   - Columns: Case #, Title, Type, Status, Priority, Category, Assigned To, Created, SLA, Actions
   - Status badges with colour mapping from mockup
   - Priority chips
   - Inline row actions (View, Edit, Escalate)
   - Toolbar: New Case button, filter bar (Type, Status, Priority dropdowns), search, export

6. **Tasks List view** (AG Grid server-side):
   - Columns: Task #, Title, Type, Status, Priority, Case #, Assigned To, Due Date, Actions
   - Same filter/sort/export pattern as Cases

7. **Case Detail view**:
   - Header bar with case number, title, status badge, priority chip, action buttons
   - Tabbed content: Overview | Tasks | Workflow | Notes | Attachments | Audit Log
   - Workflow tab: render BPMN diagram with active step highlighted (call workflow service)
   - Notes tab: timeline-style note list + add note form
   - Audit Log tab: immutable chronological event table

8. **Preferences Drawer** — reproduce the slide-in drawer from the mockup with General, Cases, Tasks tabs.

9. **WebSocket Notifications** — connect on login, update notification dot count in real-time, show toast on new event.

### AG Grid Server-Side Integration

```typescript
// In CasesListComponent
datasource: IServerSideDatasource = {
  getRows: (params: IServerSideGetRowsParams) => {
    const request = params.request; // sortModel, filterModel, startRow, endRow
    this.caseService.getServerSideRows(request).subscribe({
      next: ({rows, lastRow}) => params.success({rowData: rows, rowCount: lastRow}),
      error: () => params.fail()
    });
  }
};
```

---

## SHARED INFRASTRUCTURE

### `docker-compose.yml` (required)

Include services:
- `postgres` — port 5432, volumes for data, init scripts create databases `iqube_cases`, `iqube_tasks`, `iqube_workflow`, `iqube_notifications`
- `redis` — port 6379
- `zookeeper` — port 2181
- `kafka` — port 9092, auto-create topics: `case-events`, `task-events`, `workflow-events`, `audit-events`
- `iqube-gateway` — port 8080
- `iqube-case-service` — port 8081, depends on postgres, kafka, redis
- `iqube-task-service` — port 8082, depends on postgres, kafka
- `iqube-workflow-service` — port 8083, depends on postgres, kafka
- `iqube-notification-service` — port 8084, depends on kafka, postgres
- `iqube-ui` — port 4200, depends on iqube-gateway

---

## CROSS-CUTTING CONCERNS

### Security
- JWT authentication in Gateway (validate Bearer token on all `/api/**` routes)
- Spring Security in each service (extract user from JWT claims)
- CORS configured in Gateway for `http://localhost:4200`

### Observability
- Each service: Spring Boot Actuator (`/actuator/health`, `/actuator/metrics`)
- MDC logging with `correlationId`, `caseId`, `userId` on every log line
- Structured JSON logging (Logback + logstash-logback-encoder)

### Error Handling
- `@ControllerAdvice` global exception handler in each service
- `ProblemDetail` (RFC 7807) responses with `type`, `title`, `status`, `detail`, `instance`

### Validation
- Jakarta Bean Validation on all request DTOs
- Custom validators: case type + workflow compatibility check

### Database
- Flyway migrations for each service (`db/migration/V*.sql`)
- Auditing via Spring Data JPA `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`

### Redis Caching (Case Service)
```java
@Cacheable(value = "cases", key = "#id", unless = "#result == null")
public CaseDetailDto getCaseById(UUID id) { ... }

@CacheEvict(value = "cases", key = "#id")
public CaseDetailDto updateCase(UUID id, UpdateCaseRequest req) { ... }
```

### Kafka Configuration
```yaml
# Per service application.yml pattern
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: ${spring.application.name}
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

---

## SAMPLE DATA & SEED SCRIPTS

Generate `data.sql` for Case Service with 50 realistic cases across all 6 types, statuses, and priorities — matching the column data visible in the mockup (case numbers IQ-2025-00001 through IQ-2025-00050).

---

## DELIVERABLES CHECKLIST

Claude Code must produce **all** of the following:

- [ ] `docker-compose.yml` — fully wired, runnable with `docker compose up`
- [ ] `iqube-gateway/` — Spring Cloud Gateway with JWT filter and routing
- [ ] `iqube-case-service/` — full CRUD, Kafka producer, Redis cache, AG Grid pagination endpoint, Flyway migrations
- [ ] `iqube-task-service/` — full CRUD, Kafka producer + consumer
- [ ] `iqube-workflow-service/` — Activiti 8 engine, 3 BPMN process definitions, REST API
- [ ] `iqube-notification-service/` — Kafka consumer, WebSocket STOMP push, REST API
- [ ] `iqube-ui/` — Angular 17 SPA matching mockup pixel-for-pixel with all 5 views implemented and wired to backend APIs
- [ ] `README.md` — setup, run, architecture diagram (ASCII), environment variables table

---

## EXECUTION INSTRUCTIONS FOR CLAUDE CODE

1. **Read the attached HTML mockup first** (`iqube-hybrid-6-13.html`). Extract every CSS variable, component, and layout pattern before writing a single line of code.
2. Scaffold the Maven parent POM with `<modules>` for all 4 backend services + gateway.
3. Build services in order: gateway → case-service → task-service → workflow-service → notification-service.
4. Build the Angular UI last, wiring each feature module to the corresponding backend service.
5. After each service, verify `docker compose up <service>` compiles and starts healthy.
6. Ensure `docker compose up` from the root brings the entire platform online.
7. Do **not** use placeholder `// TODO` stubs — implement every method fully.
8. AG Grid: use Community edition APIs only (no Enterprise licence required at runtime); column definitions must match the mockup column headers exactly.
9. BPMN files must be valid Activiti 8 XML parseable by the engine — test with a unit test using `ProcessEngineConfiguration`.
10. All Kafka event classes must be in a shared `iqube-events` Maven module imported by all services.

---

*Attached reference file: `iqube-hybrid-6-13.html` — treat this as the design source of truth.*
