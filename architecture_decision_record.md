# Zavia — Architecture Decision Records
> Last updated: April 2026

---

## ADR-001: `courses/` as a separate app
**Decision:** `Course` model lives in its own `courses/` Django app, not inside `programs/`.
**Reason:** Courses are reusable across multiple programs (M2M relationship). They have an independent lifecycle — a course can exist without belonging to any program.
**Alternatives considered:** Putting Course inside `programs/` app.
**Status:** Implemented

---

## ADR-002: `departments/` as a separate app
**Decision:** `Department` model lives in its own `departments/` Django app.
**Reason:** Departments will eventually own staff assignments, budgets, and department-level reporting. Keeping it separate from the start avoids painful migrations later.
**Alternatives considered:** Folding Department into `programs/` app.
**Status:** Implemented

---

## ADR-003: `curriculum/` as a separate app
**Decision:** Curriculum structure (CurriculumVersion, CurriculumLevel, CurriculumEntry) lives in its own `curriculum/` app, separate from `programs/` and `courses/`.
**Reason:** Curriculum is the academic STRUCTURE — which courses belong to a program, in what order, at what named level. This is a distinct domain concern from defining programs or courses.
**Alternatives considered:** Putting ProgramCourse M2M inside `programs/` app.
**Status:** Implemented
**See also:** ADR-026 for the `CurriculumLevel` model added to this app.

---

## ADR-004: `CurriculumVersion` layer added
**Decision:** Programs have versioned curricula via `CurriculumVersion` model. Each version contains `CurriculumEntry` records.
**Reason:** Institutions revise curricula over time. Without versioning, changing a curriculum affects all existing students retroactively. Students enrolled under an older curriculum should remain on it.
**Impact:** `AcademicTerm` references `CurriculumVersion` (auto-assigned from program default). Enrollment inherits version via term.
**Status:** Implemented

---

## ADR-005: `AcademicTerm` named neutrally
**Decision:** The model is named `AcademicTerm`, not `Semester`, `Intake`, or `Batch`.
**Reason:** Different institutions use different terminology — Intake (paramedical), Semester (university), Term (school), Batch (vocational). `name` is a free-text field so each institution fills it as they prefer. `term_type` is optional metadata for display.
**Status:** Implemented

---

## ADR-006: `academic_terms/` as a separate app
**Decision:** `AcademicTerm` lives in its own `academic_terms/` app, not inside `programs/`.
**Reason:** AcademicTerm references `CurriculumVersion` (from `curriculum/`), and `programs/` already references `curriculum/`. Keeping AcademicTerm in `programs/` would create a circular import. Additionally, AcademicTerm will grow its own domain — term status, enrollment caps, term-level reporting.
**Alternatives considered:** Keeping AcademicTerm inside `programs/`.
**Status:** Implemented

---

## ADR-007: `code` is optional on all models
**Decision:** `code` field is `null=True, blank=True` on Department, Course, Program.
**Reason:** Not all institutions use codes. Forcing a code causes onboarding friction. Uniqueness is enforced via partial index when code is provided.
**Constraint pattern:**
```python
UniqueConstraint(
    fields=['organization', 'code'],
    condition=Q(code__isnull=False, is_active=True),
    name='unique_X_code_per_org',
)
```
**Status:** Implemented

---

## ADR-008: `allow_multiple_enrollments` as an org setting
**Decision:** Whether a student can have multiple active enrollments is controlled by `OrganizationSettings.allow_multiple_enrollments` (default: False).
**Reason:** Some vocational institutions offer short parallel programs. A DB constraint cannot be org-aware, so enforcement lives in the use case layer.
**Extension:** `program_type=SHORT` bypasses this check entirely — short programs are always enrollable alongside other programs.
**Status:** Implemented

---

## ADR-009: `terms_enabled` as an org setting
**Decision:** Whether academic_term is required on enrollment is controlled by `OrganizationSettings.terms_enabled` (default: True).
**Reason:** Some institutions use rolling admissions without formal intake terms. Making this a DB-level constraint would break such clients.
**Status:** Implemented

---

## ADR-010: Curriculum built at MVP, not deferred
**Decision:** `curriculum/` app with `CurriculumVersion` and `CurriculumEntry` was built at MVP stage.
**Reason:** Enrollment data references curriculum version via AcademicTerm. If deferred, retrofitting this relationship onto live enrollment data would be a high-risk migration.
**Status:** Implemented

---

## ADR-011: Single active enrollment enforced in use case, not DB constraint
**Decision:** The rule "one active STANDARD program enrollment per student per org" is enforced in `enrollments/use_cases/create_enrollment.py`, not as a DB constraint.
**Reason:** DB constraints are binary — they cannot be made org-aware. The use case checks `OrganizationSettings.allow_multiple_enrollments` and `program.program_type` before enforcing.
**Status:** Implemented

---

## ADR-012: `OrganizationSettings` auto-created on org creation
**Decision:** When a new Organization is created, `OrganizationSettings` is auto-created in the same transaction.
**Reason:** Every org needs settings from day one. Null reference errors in enrollment use cases would occur if settings don't exist. `get_or_create` used as a safety net in settings view for older orgs.
**Status:** Implemented

---

## ADR-013: Soft delete + dependency checks in use case layer
**Decision:** All deletes are soft (`is_active=False`). Before soft deleting, use cases check for active dependencies and raise `ValidationError` if found.
**Dependency checks:**
- Student delete → blocked if ACTIVE enrollment exists
- Program delete → blocked if ACTIVE enrollments exist
- AcademicTerm delete → blocked if ACTIVE enrollments exist
- Department delete → blocked if active Programs exist
- Course delete → blocked if active CurriculumEntries exist
**Reason:** Hard deletes are irreversible. Deleting a record with active dependencies would orphan related data and break reporting.
**Status:** Implemented

---

## ADR-014: Partial unique index for optional `code` field
**Decision:** Use PostgreSQL partial index (`condition=Q(code__isnull=False, is_active=True)`) instead of standard unique constraint for optional `code` fields.
**Reason:** Standard `UniqueConstraint` on nullable fields rejects multiple NULL values incorrectly in some DB configurations. Partial index enforces uniqueness only when code is provided and record is active. Soft deleted records with the same code do not block recreation.
**Status:** Implemented

---

## ADR-015: `program_type` — STANDARD vs SHORT
**Decision:** `Program` has a `program_type` field with choices `STANDARD` and `SHORT`.
**Reason:** Short courses (workshops, CPDs, training) should not trigger the single-enrollment exclusivity check. A student in a 3-year Health Technology program should be able to simultaneously enroll in a 4-week First Aid short course.
**Behavior:** `program_type=SHORT` bypasses the active enrollment check in `create_enrollment` use case entirely.
**Status:** Implemented

---

## ADR-016: `program_level` field on Program
**Decision:** `Program` has an optional `program_level` field with choices: UNDERGRADUATE, POSTGRADUATE, DIPLOMA, CERTIFICATE, VOCATIONAL, OTHER.
**Reason:** Institutions classify programs by academic level for reporting and display. Using level descriptors (not degree names like BS/MS) keeps it institution and country neutral.
**Note:** `SHORT` was deliberately excluded from `program_level` — it belongs to `program_type`, not academic level hierarchy.
**Status:** Implemented

---

## ADR-017: `CurriculumVersion` tied to `AcademicTerm`, not `Enrollment`
**Decision:** `AcademicTerm` holds the FK to `CurriculumVersion`, not `Enrollment`.
**Reason:** In real institutions, a cohort (term/intake) follows a specific curriculum — not individual students. All students in "Spring 2026" follow the same curriculum version. Version on Enrollment would require per-student version assignment — unnecessary complexity.
**Auto-assignment:** When an AcademicTerm is created, it auto-assigns the program's default CurriculumVersion if not explicitly provided.
**Status:** Implemented

---

## ADR-018: Student model refactor deferred — no PersonProfile abstraction yet
**Decision:** Personal data (name, gender, DOB, phone, address) remains on the `Student` model. No `PersonProfile` abstract base or FK model introduced at this stage.
**Reason:** Admission management is a Phase 2 concern. Requirements for the Applicant model are unknown. Building the abstraction now based on assumptions is riskier than building it later based on real requirements.
**When to revisit:** When building `admissions/` app.
**What will need changing:**
- Extract PersonProfile from Student
- Create Applicant model
- Data migration required
- Frontend API response shape will change
**Risk level:** Medium — manageable with proper migration planning.
**Status:** Deferred

---

## ADR-019: `enrollment_number` remains required on Student — nullable deferred
**Decision:** `enrollment_number` stays as `CharField(max_length=8, editable=False)` — not nullable.
**Reason:** Currently all student records are admin-created and always receive an enrollment number at creation. Making it nullable is only needed when the applicant flow is built (students can exist without a number until formally admitted).
**When to revisit:** When building `admissions/` app — at that point make `null=True, blank=True` and only generate number at formal admission.
**Current behaviour:** Enrollment number generated immediately on student creation via `generate_next_enrollment_number()` service.
**Status:** Deferred

---

## ADR-020: `application_reference_number` remains on Student model
**Decision:** `application_reference_number` stays on the `Student` model despite conceptually belonging to an `Applicant` model.
**Reason:** No `Applicant` model exists yet. Removing it now would lose existing functionality. It will migrate naturally when the admissions app is built.
**When to revisit:** When building `admissions/` app.
**Status:** Deferred

---

## ADR-021: `StudentExternalId` with org-configured `ExternalIdType`
**Decision:** External identifiers (e.g. Health Department ID, Board Exam Number) are stored as `StudentExternalId` records, where the identifier type is configured at org level via `ExternalIdType` model.
**Reason:** Different clients have different external bodies. Free-text labels per student would cause inconsistency. Org-configured types ensure consistency across all students within an org while remaining flexible across clients.
**Models:**
- `ExternalIdType` — org-level configuration (name, description, is_required)
- `StudentExternalId` — per-student assignment (id_type FK, value)
**Uniqueness:** No two students in the same org can have the same value for the same ID type.
**Status:** Planned

---

## ADR-022: `is_required` on ExternalIdType is soft enforcement only
**Decision:** `is_required=True` on `ExternalIdType` is a UI/reporting hint — not a DB or system constraint. Student creation is never blocked for missing external IDs.
**Reason:** External bodies (government departments, boards) are slow and bureaucratic. Blocking admin from creating student records until an external ID is received causes real operational problems. Required means "this should always be present" — not "block without it".
**Behavior:**
- UI flags missing required IDs as warnings
- Admin dashboard shows students with missing required IDs
- No hard enforcement anywhere in the system
**Status:** Planned

---

## ADR-023: External ID enforcement at program level deferred
**Decision:** Linking `ExternalIdType` to specific programs (e.g. Health Dept ID only required for Health Technology students) is deferred.
**Reason:** No client requirement exists yet. Implementation would require a `ProgramExternalIdType` junction model, enrollment validation logic, and frontend configuration UI — a complete feature with no current justification.
**When to revisit:** When a client requires program-specific ID enforcement.
**What will need changing:**
- `ProgramExternalIdType` junction model
- Enrollment use case validation
- Frontend program configuration UI
- Admin dashboard per-program missing ID reports
**Status:** Deferred

---

## ADR-024: Dedicated student search endpoint (deferred)
**Decision:** Defer a dedicated /students/search/ endpoint that returns match context (e.g. which external ID matched and its type name).
**Reason:** No immediate client requirement. Current search works correctly, it just lacks visual match feedback in the grid for external ID hits. Acceptable for MVP.
**When to revisit:** After implementing the frontend for student.
**What will need changing:**
- New /students/search/ view with a richer response shape
- Matched external ID value + type name annotated on results
- Frontend search to hit the new endpoint when query is active
- Grid row to render match context tag conditionally
**Status:** Deferred

Add ADR-025 to architecture_decision_record.md.

Append the following entry at the end of the file:

---

## ADR-025: Course-level enrollment tracking deferred to Phase 2
**Decision:** MVP enrollment model tracks program-level enrollment only (Student → Program → AcademicTerm). Individual course registration and outcome tracking per student is deferred.
**Reason:** AMMIMS immediate need is replacing their manual admission register — knowing who is enrolled in which program and intake. Course-level outcome tracking (pass/fail per course, retakes, cross-intake course borrowing) is a transcript feature not yet required by the client.
**Limitation:** The current model cannot represent scenarios where a student failed a course in one intake and is retaking it alongside courses from a new intake. All students in an enrollment are assumed to follow the full curriculum of their intake.
**Future model when needed:**
```python
class StudentCourseRegistration(models.Model):
    student          FK → Student
    enrollment       FK → Enrollment
    curriculum_entry FK → CurriculumEntry
    academic_term    FK → AcademicTerm
    status           choices=[REGISTERED, COMPLETED,
                              FAILED, WITHDRAWN]
    created_at       auto
    updated_at       auto
```
**What will need changing:**
- New `StudentCourseRegistration` model and app
- Enrollment creation use case updated to auto-register student against all curriculum entries for their term
- Grade/outcome recording on each registration
- Transcript view per student showing all course outcomes
- Re-enrollment flow for failed courses across intakes
**Trigger:** When a client requires course-level pass/fail
tracking or transcript generation.
**Status:** Deferred

Add ADR-026 to architecture_decision_record.md.

Append the following entry at the end of the file:

---

## ADR-026: CurriculumLevel as a first-class model
**Decision:** Curriculum structure is organised via a dedicated `CurriculumLevel` model rather than storing a plain integer `level` field on `CurriculumEntry`.
**Reason:** An integer field (e.g. `level=2`) is implicit — it carries no name, no order guarantee, and no metadata. Different institutions use different terminology for academic levels (Year 1, Semester 1, Foundation Year, Clinical Year). 
A first-class model allows free-text naming per level, explicit ordering independent of name, and a natural place to attach future metadata (level description, credit requirements, etc).
**Model:**
```python
class CurriculumLevel(models.Model):
    id            UUID, PK
    organization  FK → Organization
    version       FK → CurriculumVersion
    name          CharField  # free text — no enum
    order         PositiveIntegerField
    is_active     BooleanField, default=True
    created_at    auto
    updated_at    auto

    class Meta:
        ordering = ['order', 'created_at']
        constraints = [
            UniqueConstraint(
                fields=['version', 'name'],
                condition=Q(is_active=True),
                name='unique_level_name_per_version'
            )
        ]
```
**CurriculumEntry.curriculum_level** is a nullable FK to `CurriculumLevel` with `on_delete=SET_NULL`. Null means the entry is unassigned — not yet placed in a level. This is intentional: nullable at DB level for safety, but the UI always places courses into a level (no path to create an unassigned entry through normal workflow).
**Naming:** Fully flexible free text. No enum, no pattern enforcement. The UI provides placeholder guidance ("e.g. Level 1, Semester 1, Foundation Year") without constraining the admin's choice.
**No default level auto-created:** Programs start with zero levels. The admin builds the structure organically in the curriculum builder — creating levels first, then adding courses into them. This matches how academics actually plan curriculum rather than forcing upfront structural decisions.
**Ordering:** Levels have an explicit `order` field managed via a dedicated reorder endpoint (`POST .../levels/reorder/`). Order is independent of name — "Foundation Year" can be first without its name encoding that. On creation, `order` is set to `max(order) + 1` among active levels for that version (not a count), so soft-delete gaps do not corrupt the position of newly created levels.
**Queryset ordering note:** `Meta.ordering = ['order', 'created_at']` is defined on the model, but Django's `.annotate()` (used to compute `entry_count`) strips `Meta.ordering`. The list view therefore applies an explicit `.order_by('order', 'created_at')` on the queryset.
**Supersedes:** The integer `level` field on `CurriculumEntry` (removed in migration `curriculum/0003`).
**Alternatives considered:**
- Integer level field — rejected: implicit, no naming flexibility
- Enum level types (YEAR_1, SEMESTER_1 etc.) — rejected: too rigid for multi-institution SaaS
- Storing level_count on Program — rejected: doesn't support named levels or per-version structure
**Status:** Implemented

---

## ADR-027: Enrollment progression model — phased approach

**Decision:** Student academic progression is modelled in three phases:

- **Phase 1 (MVP):** `Enrollment` only — records the association between a
  student, program, and intake. Status lifecycle: ACTIVE → SUSPENDED →
  COMPLETED / WITHDRAWN.

- **Phase 2:** `IntakePeriod` introduced — a scheduled segment of an intake
  mapped to a CurriculumLevel, with a date range and status (UPCOMING, ACTIVE,
  COMPLETED). This is the time axis that answers "where is this intake right
  now in the curriculum?"

- **Phase 2:** `EnrollmentCourse` introduced — records a student's association
  with a specific course within a specific intake period. References Enrollment
  + IntakePeriod + CurriculumEntry. Status: ENROLLED, PASSED, FAILED,
  INCOMPLETE, WITHDRAWN. Grade and AttendanceRecord attach here.

**Reasoning:**
- A student progresses through individual courses, not years — year-level
  progression is derived from course outcomes, not stored directly.
- `IntakePeriod` bridges the academic calendar (time) and curriculum structure
  (content). Without it, the system cannot answer which semester an intake is
  currently on, or which courses a student is actively attending.
- `CurriculumEntry` already carries course + level + order context — 
  EnrollmentCourse references it directly rather than Course, avoiding
  denormalisation.
- Deferring IntakePeriod and EnrollmentCourse to Phase 2 keeps MVP simple
  while ensuring the architecture extends cleanly without breaking existing
  enrollment records.

**Answers this model provides (Phase 2):**
- Which semester is Intake 03 currently on?
- What courses is Student X taking this semester?
- Has Student X completed Year 1?
- Which students are currently in Semester 3?

**Deferred to Phase 3+:** Grade, AttendanceRecord (attach to EnrollmentCourse)

**Status:** Partially implemented — Enrollment (MVP) complete.
IntakePeriod and EnrollmentCourse deferred to Phase 2.