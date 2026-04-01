# Zavia (zavia.app) – Project Knowledge Base
> Last updated: March 2026

---

## 1. Project Overview

**Zavia** is a multi-tenant SaaS platform for educational institutions, designed to manage academic operations including:

- Student management
- Academics management (Departments, Programs, Curriculum, Courses)
- Enrollment
- Fee tracking
- Institutional administration
- Record keeping and retrieval

The platform is built with clean architecture principles, emphasizing:

- Multi-tenancy
- Clear domain boundaries
- Scalability
- Maintainability
- Separation of concerns

> Initial development is focused on an MVP for a single institution, but the architecture is designed for SaaS scaling from day one.

---

## 2. Core Architecture

### Backend

**Stack**
- Django + Django REST Framework
- PostgreSQL
- SimpleJWT

**Design Principles**
- Domain-driven app separation
- Use-case layer for all business logic
- UUID primary keys on all domain models
- Organization-scoped data (multi-tenancy)
- Soft delete (`is_active = False`) on key models
- Role-based access control

**Request Flow**

```
View (API Layer)
     ↓
Serializer
     ↓
Use Case / Service Layer
     ↓
ORM
     ↓
Database
```

Views are thin — no business logic. All logic lives in use cases. Serializers handle validation and representation only.

---

### Frontend

**Stack**
- Vite + React + TypeScript
- Tailwind CSS v4
- Shadcn/ui (Radix UI primitives)
- Axios
- React Hook Form + Zod (form handling and validation)
- TanStack Table v8 (data tables)
- React Router v7
- Lucide React (icons)
- Vaul (drawer/sheet primitives)

**Feature-based structure**

```
src/
  features/
    students/
      api/
      components/
      hooks/
      pages/
      types.ts
    departments/    (same structure)
    courses/        (same structure)
    programs/       (same structure)
    enrollment/     (same structure)
    org/            (same structure)
    membership/     (same structure)
    user/           (same structure)
  pages/            (top-level route pages, e.g. login)
  components/       (shared UI components)
  components/ui/    (shadcn/ui primitives)
  api/              (Axios instance + interceptors)
  auth/             (auth context, RequireAuth guard)
  providers/        (React context providers)
  app/              (router + app shell)
  layouts/          (persistent shell, sidebar, nav)
  lib/              (utils, cn helper)
  assets/
```

**Design goals**
- Clear domain separation per feature
- Reusable, composable UI components
- API abstraction via feature-level api modules
- Linear.app style — minimal, high-information-density, no decorative elements
- Workflow-based and nested operations — not a CRUD app

---

## 3. Multi-Tenancy Model

Every tenant is an **Organization**. All domain data is scoped to it.

**Frontend routing**
```
/o/{orgSlug}/dashboard
/o/{orgSlug}/students
/o/{orgSlug}/departments
/o/{orgSlug}/courses
/o/{orgSlug}/programs
/o/{orgSlug}/programs/:id
/o/{orgSlug}/programs/:id/curriculum
/o/{orgSlug}/programs/:id/terms
/o/{orgSlug}/enrollments
/o/{orgSlug}/admin/users
/o/{orgSlug}/settings
```

**Backend routing**
```
/api/v1/org/{slug}/students/
/api/v1/org/{slug}/departments/
/api/v1/org/{slug}/courses/
/api/v1/org/{slug}/programs/
/api/v1/org/{slug}/enrollments/
```

**Tenant isolation**

Every domain model has:
```python
organization = ForeignKey(Organization, on_delete=models.CASCADE)
```

All querysets are scoped via `OrgScopedMixin` (`orgs/mixins.py`), which resolves the org from the URL slug and injects it into `get_queryset()` and `perform_create()`.

---

## 4. Domain Apps

### `accounts/`
Handles authentication and user identity.

**Responsibilities:** User registration, JWT auth (login/logout), user search

**Key models/components:** `User` model, JWT token endpoints, user search API

---

### `orgs/`
Represents tenant institutions.

**`Organization` model**
```
- id              UUID, PK
- display_name    CharField  ← the institution's human-readable name
- slug            SlugField, unique
- created_by      FK → User
- is_active       BooleanField, default=True
- created_at      auto
- updated_at      auto
```

> ⚠️ The field is `display_name`, not `name`. This distinction matters in use cases and anywhere the org name is referenced (e.g. `organization.display_name`).

**`OrganizationMembership` model**
```
- id            UUID, PK
- organization  FK → Organization
- user          FK → User
- role          CharField, choices=[OWNER, ADMIN, MEMBER, GUEST]
- joined_at     auto
- updated_at    auto
```

Constraints:
- `UniqueConstraint(organization, user)` — one membership per user per org
- `UniqueConstraint(organization, condition=role=OWNER)` — only one owner per org

**Roles and permissions**

| Role   | Access |
|--------|--------|
| OWNER  | Full access; cannot be removed or demoted via normal flow |
| ADMIN  | Manage all domain data; cannot invite/promote other Admins |
| MEMBER | Standard access (department-level scope, planned) |
| GUEST  | Read-only |

Permission classes used across views:
- `IsOrgOwner` — DELETE operations
- `IsOrgAdminOrOwner` — write operations (POST, PATCH, PUT)
- `IsOrgAffiliated` — read operations (GET)

**`OrganizationSettings` model**
```
- id                            UUID, PK
- organization                  OneToOneField → Organization
- terms_enabled                 BooleanField, default=True
- allow_multiple_enrollments    BooleanField, default=False
- created_at                    auto
- updated_at                    auto
```

**Org creation side effects** (all in one `transaction.atomic()` via `create_organization` use case):
1. Organization record saved
2. Owner `OrganizationMembership` created
3. `OrganizationSettings` created
4. Default `Department` created (named after `organization.display_name`, `is_default=True`)

**Use cases in `orgs/use_cases.py`:**
- `create_organization(user, data)` — full setup in single transaction
- `update_organization(org, data)` — field updates
- `delete_organization(org)` — soft delete (future: dependency checks)
- `create_membership(organization, requester, data)` — role hierarchy validation + create
- `update_membership(membership, requester, data)` — role hierarchy validation + update
- `delete_membership(membership, requester)` — protection checks + hard delete
- `create_organization_settings(organization)` — kept as safety net for `get_or_create` in settings view

---

### `students/`
Student registry within an organization.

**`Student` model**
```
- id                UUID, PK
- organization      FK → Organization
- enrollment_number CharField (YYMM-SEQ format, e.g. 2506-0001)
- first_name        CharField
- last_name         CharField
- gender            CharField
- date_of_birth     DateField
- is_active         BooleanField, default=True
- created_at        auto
- updated_at        auto
```

**Enrollment number format:** `YYMM` + zero-padded sequence per org per month (e.g. `2506-0001`, `2506-0002`). Generated in use case on create.

**Implemented:** create, retrieve, update, list, soft delete, dependency check before delete

**Business logic in:** `students/use_cases/`

---

### `departments/`
Top-level administrative grouping. Owns both Programs and Courses.

**`Department` model**
```
- id            UUID, PK
- organization  FK → Organization
- name          CharField, required
- code          CharField, optional
- is_default    BooleanField, default=False
- is_active     BooleanField, default=True
- created_at    auto
- updated_at    auto
```

Constraints:
- `UniqueConstraint(organization, code, condition=code__isnull=False)` — partial index, PostgreSQL

**Default department rules (enforced in use case):**
- One default department is auto-created per org on org creation, named after `organization.display_name`
- Default department can be renamed but never deleted
- `delete_department` blocks if: `is_default=True`, active Programs exist, or active Courses exist

Will eventually own staff assignments, budgets, and department-level reporting.

---

### `courses/`
Reusable academic subjects owned by a department. Used across programs via curriculum.

**`Course` model**
```
- id            UUID, PK
- organization  FK → Organization
- department    FK → Department, PROTECT, null=True, blank=True
- name          CharField, required
- code          CharField, optional
- description   TextField, optional
- is_active     BooleanField, default=True
- created_at    auto
- updated_at    auto
```

Constraints:
- `UniqueConstraint(organization, code, condition=code__isnull=False)`

**Ownership model:** A course is owned by one department (the department responsible for delivering it). It can be used (borrowed) by any program via `CurriculumEntry` regardless of which department owns the program. This mirrors real-world academic practice — e.g. "Intro to Computing" owned by CS dept, included in a Civil Engineering program's curriculum.

`department` is nullable at DB level (optional during onboarding) but should always be set. `PROTECT` prevents deleting a department that owns courses — reassign first.

Search supported via `?search=` query param (name + code token matching).
Filter supported via `?department=<id>` query param.

---

### `programs/`
Top-level academic offering of an institution (what institutions call "courses" on their website).

**`Program` model**
```
- id                UUID, PK
- organization      FK → Organization
- department        FK → Department, PROTECT, null=True, blank=True
- name              CharField, required
- code              CharField, optional
- description       TextField, optional
- duration          PositiveIntegerField, optional
- duration_unit     CharField, choices=[YEARS, MONTHS], optional
- program_type      CharField, choices=[STANDARD, SHORT]
- program_level     CharField, choices=[UNDERGRADUATE, POSTGRADUATE, DIPLOMA, CERTIFICATE, VOCATIONAL, OTHER], optional
- is_active         BooleanField, default=True
- created_at        auto
- updated_at        auto
```

Constraints:
- `UniqueConstraint(organization, code, condition=code__isnull=False)`

`department` is nullable at DB level but should always be set. `PROTECT` prevents deleting a department that owns programs.

On program creation, a default `CurriculumVersion` is auto-created via use case.

---

### `curriculum/`
Defines the academic structure of a program — which courses belong to it, in what order and year level. Separate from program/course definitions.

**`CurriculumVersion` model**
```
- id            UUID, PK
- organization  FK → Organization
- program       FK → Program
- name          CharField (e.g. "Default Curriculum", "2024 Revision")
- is_default    BooleanField
- is_active     BooleanField, default=True
- created_at    auto
- updated_at    auto
```

Only one version can be default per program. Default version is auto-created with the program. Non-default versions can be deleted; default cannot.

**`CurriculumEntry` model**
```
- id            UUID, PK
- organization  FK → Organization
- version       FK → CurriculumVersion
- course        FK → Course
- year_level    PositiveIntegerField, optional (1, 2, 3…)
- order         PositiveIntegerField, optional
- is_core       BooleanField, default=True
- is_active     BooleanField, default=True
- created_at    auto
- updated_at    auto
```

Constraints:
- `UniqueConstraint(version, course)`

---

### `academic_terms/`
Represents a cohort/intake/semester within a program. Terminology varies per institution — handled via free-text `name` + optional `term_type`.

**`AcademicTerm` model**
```
- id                    UUID, PK
- organization          FK → Organization
- program               FK → Program
- curriculum_version    FK → CurriculumVersion, optional (auto-assigned default)
- name                  CharField (e.g. "Spring 2026", "Intake 03")
- term_type             CharField, choices=[SEMESTER, INTAKE, TERM, BATCH], optional
- start_date            DateField, optional
- end_date              DateField, optional
- is_active             BooleanField, default=True
- created_at            auto
- updated_at            auto
```

Constraints:
- `UniqueConstraint(program, name, condition=is_active=True)`

If `OrganizationSettings.terms_enabled = False`, academic terms are not required on enrollment.

---

### `enrollments/`
Records a student's enrollment in a program, optionally under a specific academic term.

**`Enrollment` model**
```
- id                UUID, PK
- organization      FK → Organization
- student           FK → Student
- program           FK → Program
- academic_term     FK → AcademicTerm, optional (null if terms_enabled=False)
- status            CharField, choices=[ACTIVE, COMPLETED, WITHDRAWN, SUSPENDED]
- enrolled_at       DateTimeField, auto_now_add
- is_active         BooleanField, default=True
- created_at        auto
- updated_at        auto
```

Constraints:
- `UniqueConstraint(student, program, academic_term)`

Business rules (enforced in use case):
- If `allow_multiple_enrollments = False`: student can only have one ACTIVE enrollment per org
- Cannot soft-delete an ACTIVE enrollment — must be withdrawn/completed first
- Cannot delete an academic term that has active enrollments

---

## 5. Domain Relationship Map

```
Organization
  ├── OrganizationSettings (1:1)
  ├── OrganizationMembership (users + roles)
  ├── Department
  │     ├── Program
  │     │     ├── CurriculumVersion
  │     │     │     └── CurriculumEntry ──→ Course (owned by a dept, borrowed here)
  │     │     └── AcademicTerm ──→ CurriculumVersion
  │     └── Course (owned by this department)
  └── Student
        └── Enrollment ──→ Program
                       ──→ AcademicTerm (optional)
```

**Key relationship note:** A `Course` is owned by one `Department` (via FK). It can be included in any `Program`'s curriculum via `CurriculumEntry` regardless of which department owns the program. Ownership ≠ usage.

---

## 6. Backend Design Conventions

**UUID primary keys** — on all domain models. Reasons: distributed safety, avoids ID enumeration, safe for public API exposure.

**Soft delete** — `is_active = False` instead of hard delete. Reasons: auditability, compliance, recoverability. Dependency checks are enforced in the use case layer before soft delete is applied.

**Partial unique indexes** (PostgreSQL) — for optional unique fields like `code`:
```python
UniqueConstraint(fields=['organization', 'code'], condition=Q(code__isnull=False, is_active=True), name='...')
```

**OrgScopedMixin** (`orgs/mixins.py`) — shared mixin used across all domain views to resolve `organization` from URL slug and inject it into queryset filtering and object creation.

**Use-case layer** — all business logic lives in `app/use_cases.py` or `app/use_cases/`. Views call use cases; use cases call the ORM. Serializers only handle validation and representation.

**Views are thin** — no business logic in views. Every `perform_create`, `perform_update`, `perform_destroy` delegates to a use case and assigns `serializer.instance = result` where applicable.

**RBAC on views**
```python
# Read (GET)
[IsAuthenticated(), IsOrgAffiliated()]

# Write (POST, PATCH, PUT)
[IsAuthenticated(), IsOrgAdminOrOwner()]

# Delete
[IsAuthenticated(), IsOrgOwner()]
```

**Error handling** — all validation errors raised as `rest_framework.exceptions.ValidationError`. `IntegrityError` and DB constraint violations are caught in use cases and re-raised with human-readable messages. No raw Django exceptions reach the client.

---

## 7. API Design

**Base pattern:** `/api/v1/org/{slug}/resource/`

```
# Students
GET    /api/v1/org/{slug}/students/
POST   /api/v1/org/{slug}/students/
GET    /api/v1/org/{slug}/students/{id}/
PATCH  /api/v1/org/{slug}/students/{id}/
DELETE /api/v1/org/{slug}/students/{id}/

# Departments
GET    /api/v1/org/{slug}/departments/
POST   /api/v1/org/{slug}/departments/
GET    /api/v1/org/{slug}/departments/{id}/
PATCH  /api/v1/org/{slug}/departments/{id}/
DELETE /api/v1/org/{slug}/departments/{id}/

# Courses
GET    /api/v1/org/{slug}/courses/
POST   /api/v1/org/{slug}/courses/
GET    /api/v1/org/{slug}/courses/{id}/
PATCH  /api/v1/org/{slug}/courses/{id}/
DELETE /api/v1/org/{slug}/courses/{id}/

# Programs
GET    /api/v1/org/{slug}/programs/
POST   /api/v1/org/{slug}/programs/
GET    /api/v1/org/{slug}/programs/{id}/
PATCH  /api/v1/org/{slug}/programs/{id}/
DELETE /api/v1/org/{slug}/programs/{id}/

# Curriculum Versions (nested under program)
GET    /api/v1/org/{slug}/programs/{program_id}/curriculum/
POST   /api/v1/org/{slug}/programs/{program_id}/curriculum/
GET    /api/v1/org/{slug}/programs/{program_id}/curriculum/{id}/
PATCH  /api/v1/org/{slug}/programs/{program_id}/curriculum/{id}/
DELETE /api/v1/org/{slug}/programs/{program_id}/curriculum/{id}/
POST   /api/v1/org/{slug}/programs/{program_id}/curriculum/{id}/set-default/

# Curriculum Entries (nested under version)
GET    /api/v1/org/{slug}/programs/{program_id}/curriculum/{version_id}/entries/
POST   /api/v1/org/{slug}/programs/{program_id}/curriculum/{version_id}/entries/
PATCH  /api/v1/org/{slug}/programs/{program_id}/curriculum/{version_id}/entries/{id}/
DELETE /api/v1/org/{slug}/programs/{program_id}/curriculum/{version_id}/entries/{id}/

# Academic Terms (nested under program)
GET    /api/v1/org/{slug}/programs/{program_id}/terms/
POST   /api/v1/org/{slug}/programs/{program_id}/terms/
GET    /api/v1/org/{slug}/programs/{program_id}/terms/{id}/
PATCH  /api/v1/org/{slug}/programs/{program_id}/terms/{id}/
DELETE /api/v1/org/{slug}/programs/{program_id}/terms/{id}/

# Enrollments
GET    /api/v1/org/{slug}/enrollments/
POST   /api/v1/org/{slug}/enrollments/
GET    /api/v1/org/{slug}/enrollments/{id}/
PATCH  /api/v1/org/{slug}/enrollments/{id}/
DELETE /api/v1/org/{slug}/enrollments/{id}/

# Membership
GET    /api/v1/org/{slug}/members/
POST   /api/v1/org/{slug}/members/
PATCH  /api/v1/org/{slug}/members/{id}/
DELETE /api/v1/org/{slug}/members/{id}/

# Organization Settings
GET    /api/v1/org/{slug}/settings/
PATCH  /api/v1/org/{slug}/settings/
```

---

## 8. Architecture Decision Log

| # | Decision | Reason |
|---|----------|--------|
| 1 | `courses/` is a separate app | Courses are reusable across programs (M2M); independent lifecycle |
| 2 | `departments/` is a separate app | Will own staff, budgets, reporting — avoids painful migrations |
| 3 | `curriculum/` is a separate app | Academic structure is distinct from program/course definition |
| 4 | `CurriculumVersion` layer added | Allows curriculum revisions without disrupting existing enrollments |
| 5 | `AcademicTerm` named neutrally | Clients use Intake/Semester/Term/Batch — free-text `name` handles all |
| 6 | `code` is optional on all models | Not all institutions use codes — forcing causes onboarding friction |
| 7 | `allow_multiple_enrollments` is a setting | Some vocational institutions allow parallel program enrollments |
| 8 | `terms_enabled` is a setting | Some institutions use rolling admissions without formal terms |
| 9 | Curriculum built at MVP, not deferred | Enrollment data references it — migrating on live data is costly |
| 10 | Single active enrollment enforced in use case | DB constraints are binary; use case allows org-aware conditional enforcement |
| 11 | `OrganizationSettings` auto-created on org creation | Every org needs settings from day one — prevents null reference errors |
| 12 | Soft delete + dependency checks | Delete is safe only when dependencies are verified in use case layer |
| 13 | Partial unique index for optional `code` | Standard unique constraint rejects `(org, null)` duplicates incorrectly in PostgreSQL |
| 14 | `Course` has `department` FK (owning dept) | Departments own courses in real institutions — e.g. CS dept owns "Intro to Computing" even if CE program uses it. Ownership is separate from usage (via CurriculumEntry) |
| 15 | `Course.department` is PROTECT not SET_NULL | Deleting a dept that owns courses must be blocked — reassign courses first. Prevents orphaned courses |
| 16 | `Program.department` changed to PROTECT | Same reasoning — a program without a department is an invalid state once default dept guarantee exists |
| 17 | Default department auto-created on org creation | Guarantees every org has at least one department from day one. Eliminates null-handling burden on Course and Program FKs across the entire codebase |
| 18 | Default department cannot be deleted | Prevents orphaning all courses/programs at a small institution that only ever has one department. Can be renamed freely |
| 19 | All orgs/ business logic moved to use cases | Views were carrying membership hierarchy checks and org setup logic directly. Moved to `orgs/use_cases.py` for consistency with all other apps and to ensure atomic org creation |
| 20 | `Organization.display_name` not `name` | The org model field is `display_name`. Any use case or code referencing the org's human-readable name must use `organization.display_name` |

---

## 9. Implementation Status

### Backend

| App | Models | Use Cases | Views/APIs | Status |
|-----|--------|-----------|------------|--------|
| `accounts/` | ✅ | ✅ | ✅ | Complete |
| `orgs/` | ✅ | ✅ | ✅ | Complete — fully refactored to use case pattern |
| `students/` | ✅ | ✅ | ✅ | Complete |
| `departments/` | ✅ | ✅ | ✅ | Complete — `is_default` added, dependency checks updated |
| `courses/` | ✅ | ✅ | ✅ | Complete — `department` FK added |
| `programs/` | ✅ | ✅ | ✅ | Complete — `department` on_delete tightened to PROTECT |
| `curriculum/` | ✅ | ✅ | ✅ | Complete |
| `academic_terms/` | ✅ | ✅ | ✅ | Complete |
| `enrollments/` | ✅ | ✅ | ✅ | Complete |

### Frontend

| Feature | Status |
|---------|--------|
| Auth (login/logout/JWT refresh) | ✅ Complete |
| Org context + routing | ✅ Complete |
| Membership management | ✅ Complete |
| Student CRUD + table | ✅ Complete |
| Departments | ⬜ To Do |
| Courses | ⬜ To Do |
| Programs | ⬜ To Do |
| Curriculum builder | ⬜ To Do |
| Academic Terms | ⬜ To Do |
| Enrollments | ⬜ To Do |

---

## 10. Planned Modules (Roadmap)

### Phase 1 — MVP (Current)
Core academic back-office for institution administrators.

- ✅ Student registry
- ✅ Academic structure backend (Departments → Programs → Curriculum → Courses → Enrollments)
- ⬜ Frontend for academic structure
- ⬜ Fee tracking / Payments

### Phase 2 — Academic Operations
- Academic grading
- Attendance management
- Invoicing
- Reporting

### Phase 3 — Institution Management
- Teacher management
- Class scheduling
- Examinations
- Notifications
- Student / parent portal

### Phase 4 — SaaS Platform
- Billing + subscription plans
- Usage monitoring
- Feature flags
- White labeling
- Marketplace integrations

---

## 11. Observability (Planned)

| Tool | Purpose |
|------|---------|
| Sentry | Error tracking |
| PostHog | Product analytics |
| Prometheus + Grafana | Infrastructure metrics |
| `ActivityLog` model | Audit trail |

**`ActivityLog` model (planned)**
```
- organization  FK → Organization
- user          FK → User
- action        CharField
- object_type   CharField
- object_id     UUIDField
- timestamp     DateTimeField, auto_now_add
```

---

## 12. Dev Workflow

**Git branching**
```
main        → tested, stable
dev         → integration
feature/*   → active development
```

Rules:
- `main` = tested version (deployed on release)
- `dev` = integration branch
- All work happens on `feature/*` branches off `dev`

**Tooling split**
- Claude chat — architecture decisions, design, planning, code review
- Claude Code CLI — large scaffolding, migrations, multi-file changes
- Claude Code VSCode extension — targeted single-file edits

**Prompts** stored as numbered markdown files in `_prompts/` at project root.

---

## 13. Long-Term Vision

Zavia aims to evolve into a full SaaS education management platform, potentially including:

- Headless academic data API
- No-code admin builder
- Multi-institution dashboards
- Analytics platform
- AI-assisted operations

Comparable platforms: PowerSchool, Blackbaud, OpenSIS, Fedena — but with modern architecture and SaaS-first design.
