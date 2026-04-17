# Zavia — Architecture Decision Records
> Last updated: 16 April 2026

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

## ADR-013: Soft delete + dependency checks in use case layer (Updated)
**Decision:** All deletes are soft (`is_active=False`). Before soft deleting, use cases check for active dependencies and raise `ValidationError` if found.

**Dependency checks — Academic domain (existing):**
- Student delete → blocked if ACTIVE enrollment exists
- Program delete → blocked if ACTIVE enrollments exist
- AcademicTerm/Intake delete → blocked if ACTIVE enrollments exist
- Department delete → blocked if active Programs or Courses exist
- Course delete → blocked if active CurriculumEntries exist

**Dependency checks — Fees domain (added):**
- FeeStructure (template) delete → blocked if any active intake-bound clone exists (`source_template` back-reference active)
- FeeStructure (intake-bound clone) delete → blocked if any `FeeLedger` references it
- FeeHead delete → blocked if any `LedgerEntry` references it (snapshot exists — the charge has been raised against a student)
- FeeSchedule delete → blocked if any `FeeLedger` references it (schedule has been materialised into instalment rows)
- FeeLedger delete → blocked if any `LedgerTransaction` exists against it (financial movements have been recorded — ledger is part of audit trail)

**Reason:** Hard deletes are irreversible. Deleting a record with active dependencies would orphan related data and break reporting. For financial records specifically, deletion after any transaction has been posted is an accounting integrity violation — the audit trail must remain intact.

**Note on FeeHead:** `FeeHead` uses `on_delete=SET_NULL` on the
`LedgerEntry.fee_head` FK — meaning the FK is nulled if the head is hard deleted at the DB level. However soft delete is enforced in the use case before this is ever reached. The `SET_NULL` is a safety net only.

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

## ADR-019: `enrollment_number` renamed to `student_id` on Student model

**Original decision:** `enrollment_number` stays as required CharField,
generated at Student creation.

**Update (5 April 2026):** Field renamed to `student_id` on both backend
and frontend to eliminate confusion with the `Enrollment` model introduced
later. The field retains the same behaviour — generated at Student creation,
permanent person identifier, never changes.

**Reasoning for rename:**
- `enrollment_number` implied a connection to an Enrollment record
- Once the Enrollment feature was built, the naming caused genuine ambiguity
- `student_id` is unambiguous — it identifies the person, not the transaction

**Implemented:** Backend model, serializer, service function renamed.
Frontend types and display labels updated.

**Status:** Implemented

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
**Reason:** Clients' immediate need is replacing their manual admission register — knowing who is enrolled in which program and intake. Course-level outcome tracking (pass/fail per course, retakes, cross-intake course borrowing) is a transcript feature not yet required by the client.
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

---

## ADR-028: List-page pattern for heavyweight entity pages
**Decision:** Heavyweight entity pages (Students, Enrollments, future Teachers, future Fees) adopt a shared list-page pattern:

1. **List-as-default.** The page opens directly to the entity table. Users do not have to search, click, or interact to see data.
2. **Primary actions are always visible.** "New X", "Import", and other primary CTAs render as buttons in the page header — never hidden behind a keyboard shortcut or palette.
3. **Filter toolbar is always visible.** Search input + entity-specific filter selects live in the toolbar above the table, inside the `p-6` content wrapper. Filters are never hidden behind a command palette.
4. **Command palette is an accelerator, not a primary surface.** The palette provides fast navigation and action execution for power users, but every action it contains is also reachable through visible UI.
5. **Right rail is permitted but optional.** Heavyweight pages may add a `w-[300px]` right rail at `xl:` breakpoint for at-a-glance context (breakdowns, activity, trends). The rail must not hold actions that are not also reachable elsewhere.
6. **Bulk action bar is inline and conditional.** When selection > 0, a dark inline bar appears above the table with bulk actions. It is never persistent chrome.

**Reason:** Admin workflows on entity pages are dominated by browse-filter-act patterns, not known-item lookup. Hiding the list behind a search surface forces the majority of users to take an extra step for every visit. It also breaks convention with every other back-office tool (SIS, CRM, Google Admin) where clicking an entity name opens a list.

The palette is still valuable — it accelerates the known-item workflow for power users and provides a unified jump-to-anywhere affordance — but making it the only path inverts the cost/benefit: the 80% are taxed to marginally please the 20% who already know the shortcut.

**Keyboard shortcut hierarchy:**
- **Primary: `/`** — single keystroke, no modifier, does not conflict with browser chrome shortcuts. Only triggers when focus is not already in an input/textarea/select.
- **Secondary: `⌘K` / `Ctrl+K`** — wired as a familiar accelerator for users coming from Linear, Slack, GitHub, etc.

`⌘K` / `Ctrl+K` is deliberately *secondary* because Firefox-family browsers (including Zen Browser and LibreWolf) bind `Ctrl+K` to the browser search bar. In those browsers the shortcut may be intercepted by browser chrome before the page handler runs. `/` is reliable across all browsers and provides a safe primary affordance.

**Applies to:** Students (Phase 1), Enrollments (Phase 1), Fees (Phase 1), Teachers (Phase 3), and any future entity with list-level admin operations.

**Does not apply to:** Narrow entity pages that are purely list+drawer (Departments, Courses) — those remain on the existing lightweight shell.

**Alternatives considered:**
- *Command-palette-first, empty centered search on landing* — rejected. Breaks convention for declared-intent navigation and taxes the common case. Appropriate only for zero-context entry points (global search, launchers).
- *Metrics strip above the table on every list page* — rejected as default. Metrics live in the optional right rail when present; otherwise summary counts appear as inline text next to the page title.

**Implementation scope (Phase 1 — Students):**
- Header: title + inline count summary + visible "New Student" + "Import" + palette trigger button
- Toolbar: search input + intake / program / status selects + "Clear filters" + result counter
- Table: checkbox column, avatar initials, student name, student ID, program, intake, status
- Bulk action bar: dark inline bar, appears when selection > 0
- Command palette: `/` primary, `⌘K` secondary, Jump-to + Actions groups only (no Filter-by group — filters live in toolbar)
- **No right rail in Phase 1.** Main content takes full width.

**Phase 2 additions (deferred):**
- Right rail sections: status breakdown (bars), intake distribution (bars), "new this month" sparkline, recent activity feed
- Backend dependencies: activity feed requires an audit log model; sparkline requires a time-series aggregation endpoint

**Status:** Approved — Phase 1 implementation pending

---

## ADR-029: fees/ as a separate Django app
**Decision:** All fee-related models (FeeStructure, FeeHead, FeeSchedule, ScheduleInstalment, FeeLedger, LedgerEntry, LedgerInstalment, LedgerTransaction) live in a dedicated fees/ app.
**Reason:** Fees are a financially distinct domain-independent lifecycle, separate reporting, future payment gateway integration. Coupling to enrollments/ or students/ would create a bloated app with mixed concerns. A separate app allows the fees domain to grow (invoicing, gateway, receipts) without touching academic models.
Alternatives considered: Extending enrollments/ app with fee models.
**Status:** Implemented

---

## ADR-030: FeeStructure — template and intake-bound clone design
**Decision:** FeeStructure has a direct `intake` FK (null = reusable template, set = intake-bound clone). The `FeeStructureIntake` junction model has been removed. A template is cloned to a target intake via `clone_structure_for_intake`, which creates a new FeeStructure with `intake=<target>` and `source_template=<original>`, copying all FeeHead and FeeSchedule/ScheduleInstalment records as independent copies.

**Reason:** The original junction model design allowed one template to be assigned to multiple intakes simultaneously, but this created a dangerous shared-mutation problem: editing a FeeHead on a shared template would affect all assigned intakes including those with existing student ledgers. The revised model makes the relationship explicit — cloning creates a fully independent copy per intake. The `source_template` self-FK preserves lineage for audit and UI (e.g. "Cloned from Health Tech Fees 2026") without creating shared mutation risk. Template names are enforced unique per org (partial index, `intake IS NULL`); intake-bound clones carry no name constraint.

**Model:**
```python
class FeeStructure(models.Model):
    id               # UUID, PK
    organization     # FK → Organization
    intake           # FK → Intake, null=True, blank=True, on_delete=SET_NULL
                     # null = reusable template
                     # set  = intake-bound clone
    source_template  # FK → self, null=True, blank=True, on_delete=SET_NULL
                     # set on clones; null on manually created templates
    name             # CharField
    is_active        # BooleanField, default=True
    created_at       # auto
    updated_at       # auto

    class Meta:
        constraints = [
            UniqueConstraint(
                fields=['organization', 'intake'],
                condition=Q(intake__isnull=False, is_active=True),
                name='unique_active_structure_per_intake'
            ),
            UniqueConstraint(
                fields=['organization', 'name'],
                condition=Q(intake__isnull=True, is_active=True),
                name='unique_active_template_name_per_org'
            ),
        ]
```
**Alternatives considered:** Original junction model (FeeStructureIntake M2M) — rejected because shared template mutation affects all assigned intakes including those with live ledgers, making the lock logic impossible to enforce cleanly. FK from FeeStructure to Program with nullable Intake override — rejected because it creates a stale program-level fallback that is always overridden in practice.
**Status:** Implemented

---

## ADR-031: One active fee structure per intake — constraint on `FeeStructure` directly
**Decision:** The `FeeStructureIntake` junction model has been removed. The one-active-structure-per-intake guarantee is now enforced by a partial unique index directly on `FeeStructure`:

```python
UniqueConstraint(
    fields=['organization', 'intake'],
    condition=Q(intake__isnull=False, is_active=True),
    name='unique_active_structure_per_intake'
)
```

**Reason:** The junction model was introduced to allow M2M (one structure → many intakes), but shared-template mutation on structures with multiple active intake assignments proved dangerous — editing a FeeHead would retroactively affect all assigned intakes including those with live ledgers. Moving to per-intake clones (see ADR-030) eliminates the M2M requirement. The uniqueness guarantee becomes a simple DB constraint on the cloned row itself. Replacing a structure on an intake: `clone_structure_for_intake` soft-deletes the existing active clone (blocked if any `FeeLedger` references it) and creates a new independent clone atomically.

**Status:** Implemented

---

## ADR-032: FeeHead is editable until first LedgerEntry references it — then locked
**Decision:** FeeHead amount and name are freely editable until a LedgerEntry exists that references it. Once referenced, the head is locked for editing. New heads can always be added to a structure. A head can only be deleted if no LedgerEntry references it.
**Reason:** Locking the entire FeeStructure on first ledger generation (as initially considered) was too coarse — it prevented legitimate additions like Semester 2 fees added after Semester 1 ledgers were already generated. Locking at the FeeHead level is the correct granularity. LedgerEntry snapshots (head_name, charged_amount) protect historical records regardless of head state.
**New head propagation:** When a new FeeHead is added to a structure that already has active ledgers, the use case automatically creates a new LedgerEntry on every active ledger for that structure. Runs in a single transaction.atomic().
**Model:**
```
class FeeHead(models.Model):
    id              UUID, PK
    organization    FK → Organization
    structure       FK → FeeStructure
    name            CharField
    amount          DecimalField(max_digits=10, decimal_places=2)
    period_label    CharField, null=True, blank=True  # "Semester 1" — display only
    order           PositiveIntegerField
    is_active       BooleanField, default=True
    created_at      auto
    updated_at      auto
```
**Phase 2 note:** period_label will gain a nullable FK to IntakePeriod when that model is introduced. Existing heads with period_label set and intake_period=null remain valid as unassigned charges.
**Status:** Implemented

---

## ADR-033: `FeeLedger` generation strategy — auto on enrollment, manual bulk for existing enrollments (Updated)
**Decision:** Ledger generation follows two paths:
- **Auto:** When a student is enrolled and a fee structure is already assigned to the intake, `generate_fee_ledger` is called inside `create_enrollment` within the same `transaction.atomic()`.
- **Manual bulk:** Admin triggers "Generate Ledgers" on a fee structure or intake after the fact. Applies to all active enrollments for that intake that do not yet have a ledger. This is an onboarding/migration tool — not part of the normal workflow once the system is live.

**Empty ledger is always created:** Even if no fee structure is assigned to the intake at enrollment time, a `FeeLedger` record is still created for the student with zero `LedgerEntry` rows. This ensures every enrolled student always has a ledger to post transactions against if needed — without requiring a fee structure to exist first. Enrollment is never blocked by fee configuration state.

**Resolution logic:**
```python
def _resolve_fee_structure(intake, organization):
    """Returns the active intake-bound FeeStructure for this intake, or None."""
    return FeeStructure.objects.filter(
        organization=organization,
        intake=intake,
        is_active=True,
    ).first()

# Inside create_enrollment (same transaction.atomic()):
enrollment = Enrollment(...)
enrollment.save()

ledger = FeeLedger.objects.create(
    organization=organization,
    student=student,
    enrollment=enrollment,
)

structure = _resolve_fee_structure(enrollment.intake, organization)
if structure:
    generate_ledger_entries(ledger=ledger, structure=structure)
    assign_default_schedule(ledger=ledger, structure=structure)
```

**Manual bulk trigger behaviour:**
- Applies to all ACTIVE enrollments for the intake with no existing ledger
- All-or-nothing: generates for all qualifying enrollments, not selectable per student — this is a compliance requirement (see ADR-032 discussion)
- Action is logged: "Ledgers generated for Intake X — N students — by [user] on [date]"
- Idempotent: re-running does not duplicate ledgers — use case checks for existing ledger before creating

**Reason:** Auto-generation is the correct default once the system is live.
The manual bulk trigger handles the bootstrapping problem — institutions onboarding with students already enrolled before any fee structure was configured. Once live, the manual trigger becomes a rarely used safety net.

**Status:** Implemented

---

## ADR-034: LedgerEntry stores snapshot fields — charged amount never mutates
**Decision:** `LedgerEntry` stores `head_name` and `charged_amount` as snapshots at time of ledger generation. These fields never change after creation regardless of subsequent `FeeHead` edits.
**Reason:** A student's fee obligation is established at the point of ledger generation. Retroactive changes to fee head amounts must not affect existing financial records — this is both an accounting principle and a compliance requirement.
**Model:**
```python
class LedgerEntry(models.Model):
    id              UUID, PK
    ledger          FK → FeeLedger
    fee_head        FK → FeeHead, null=True, on_delete=SET_NULL
    head_name       CharField       # snapshot
    charged_amount  DecimalField    # snapshot — never mutated
    period_label    CharField, null=True  # snapshot of FeeHead.period_label
    is_active       BooleanField, default=True
    created_at      auto
    updated_at      auto
```
`fee_head` FK is nullable (`on_delete=SET_NULL`) — if a head is soft deleted, the ledger entry remains intact with its snapshot values. The financial record is never orphaned.
**Status:** Implemented

---

## ADR-035: `LedgerTransaction` unifies all post-generation ledger movements

**Decision:** Payments, concessions, additional charges, and reversals are all represented as `LedgerTransaction` records distinguished by `transaction_type`. Transactions carry a directional effect on the ledger balance — credits reduce it, debits increase it.

**Transaction types and their direction:**

| Type | Direction | Description |
|---|---|---|
| `PAYMENT` | Credit | Money received from student |
| `CONCESSION` | Credit | Scholarship, discount, hardship waiver |
| `ADDITIONAL_CHARGE` | Debit | Late fee penalty, miscellaneous addition |
| `REVERSAL` | Correction | Corrects an incorrectly recorded transaction |

**Reason:** From an accounting standpoint, payments and concessions are both credit transactions against an accounts receivable ledger — they reduce what the student owes. Additional charges (e.g. late fee penalties) are debit transactions — they increase what the student owes. Separating these into different models creates artificial complexity, splits the audit trail, and complicates balance computation. A unified transaction log with explicit direction is how real accounting ledgers work.

Crucially, concessions and additional charges are semantically opposite — conflating them (e.g. recording a late fee as a "concession with negative label") is architecturally incorrect and misleading in reporting.

**Model:**
```python
class LedgerTransaction(models.Model):
    id                  # UUID, PK
    organization        # FK → Organization
    ledger              # FK → FeeLedger
    transaction_type    # CharField, choices=[
                        #     PAYMENT,
                        #     CONCESSION,
                        #     ADDITIONAL_CHARGE,
                        #     REVERSAL,
                        # ]
    amount              # DecimalField — always positive
    transaction_date    # DateField

    # Payment-specific (null for other types)
    payment_method      # CharField, choices=[CASH, BANK_TRANSFER, CHEQUE, OTHER], null=True
    reference_number    # CharField, null=True, blank=True

    # Concession / Additional Charge specific (null for PAYMENT)
    label               # CharField, null=True
                        # e.g. "Merit Scholarship", "Late Fee Penalty"

    # Shared
    note                # TextField, blank=True
    recorded_by         # FK → User
    is_active           # BooleanField, default=True
    created_at          # auto
    updated_at          # auto
```

**Balance computation:**
```python
@property
def balance(self):
    total_charged = sum(
        e.charged_amount for e in self.entries.filter(is_active=True)
    )
    credits = sum(
        t.amount for t in self.transactions.filter(
            transaction_type__in=[PAYMENT, CONCESSION],
            is_active=True
        )
    )
    debits = sum(
        t.amount for t in self.transactions.filter(
            transaction_type=ADDITIONAL_CHARGE,
            is_active=True
        )
    )
    return total_charged + debits - credits
```

**REVERSAL type** is included for future-proofing — correcting an incorrectly recorded transaction without deleting financial records. Deletion of transaction records is an accounting red flag. Reversal semantics (which transaction it corrects, full vs partial) are deferred to Phase 2 when the requirement concretely arises.

**Reporting filters by type:**
- Collections report → filter `PAYMENT`
- Concessions report → filter `CONCESSION`
- Penalties report → filter `ADDITIONAL_CHARGE`
- Full audit trail → all types ordered by `transaction_date`

**Alternatives considered:**
- Separate `Concession` and `Payment` models — rejected: splits audit trail, complicates balance computation, two queries with different semantics.
- `Concession` model with negative amounts for penalties — rejected: semantically incorrect, concessions are credits and cannot represent debits.

**Status:** Implemented

---

## ADR-036: Instalment schedule is a template on `FeeStructure` — materialised per ledger (Updated)

**Decision:** `FeeSchedule` and `ScheduleInstalment` define the instalment
template on the structure. `LedgerInstalment` is the materialised per-student
copy generated when the ledger is created. The two layers are independent
after generation.

**Reason:** A template cannot hold student-specific due dates or amounts —
these depend on enrollment date and net fees after transactions. Materialising
the schedule per ledger allows per-student overrides and recalculation without
touching the template.

**Models:**
```python
class FeeSchedule(models.Model):
    id              # UUID, PK
    organization    # FK → Organization
    structure       # FK → FeeStructure
    name            # CharField — e.g. "3-Instalment Plan", "Full Payment"
    is_default      # BooleanField, default=False
    is_active       # BooleanField, default=True
    created_at      # auto
    updated_at      # auto


class ScheduleInstalment(models.Model):
    id                        # UUID, PK
    schedule                  # FK → FeeSchedule
    label                     # CharField — e.g. "Upon Admission", "Instalment 2"
    percentage                # DecimalField, null=True
    fixed_amount              # DecimalField, null=True
    due_days_from_enrollment  # PositiveIntegerField, null=True
    order                     # PositiveIntegerField
    # Constraints (enforced in use case):
    # - Exactly one of percentage or fixed_amount must be set
    # - Sum of all percentages for a schedule must equal 100


class LedgerInstalment(models.Model):
    id                    # UUID, PK
    ledger                # FK → FeeLedger
    schedule_instalment   # FK → ScheduleInstalment, null=True
                          # null = manually created outside a schedule
    label                 # CharField — snapshot
    due_date              # DateField — computed from enrollment date + due_days
    amount_due            # DecimalField — computed; recalculated on balance change
    is_active             # BooleanField, default=True
    created_at            # auto
    updated_at            # auto
```

**Auto-assignment:** If a `FeeStructure` has a `FeeSchedule` with
`is_default=True`, it is auto-assigned when the ledger is generated.

**Per-student schedule override:** Admin can assign a different `FeeSchedule` to a specific student's ledger. Behaviour:
- Existing `LedgerInstalment` rows are deleted and regenerated from the new schedule
- Blocked if any `LedgerTransaction` of type `PAYMENT` already exists — admin must handle manually in that case
- Concession-only transactions do not block override (no cash received yet)

**Instalment recalculation on balance change:** When a `LedgerTransaction`
of type `CONCESSION` or `ADDITIONAL_CHARGE` is posted, all unpaid `LedgerInstalment` rows are recalculated proportionally against the new net balance using the original percentage ratios from `ScheduleInstalment`.

```python
def recalculate_instalments(ledger):
    """
    Recalculate unpaid instalment amounts against current ledger balance.
    Called after any CONCESSION or ADDITIONAL_CHARGE transaction is posted.
    Paid instalments (where a PAYMENT >= amount_due has been received) are
    never touched.
    """
    net_balance = ledger.balance  # accounts for all transactions
    unpaid = ledger.instalments.filter(is_active=True, is_paid=False).order_by('order')
    total_pct = sum(
        i.schedule_instalment.percentage
        for i in unpaid
        if i.schedule_instalment and i.schedule_instalment.percentage
    )
    for instalment in unpaid:
        if instalment.schedule_instalment and instalment.schedule_instalment.percentage:
            share = instalment.schedule_instalment.percentage / total_pct
            instalment.amount_due = net_balance * share
            instalment.save(update_fields=['amount_due', 'updated_at'])
```

**Direction of recalculation:**
- `CONCESSION` posted → net balance decreases → unpaid instalments reduce
- `ADDITIONAL_CHARGE` posted → net balance increases → unpaid instalments increase to absorb the additional charge (e.g. late fee penalty distributed across remaining instalments)

**Payments are against ledger balance — not individual instalments:**
Payments hit the ledger total. Instalment rows serve as due-date markers and overdue indicators. Per-instalment payment allocation is deferred to Phase 2.

**Status:** Implemented

---

## ADR-037: Late fee is computed on demand — formally recorded as `ADDITIONAL_CHARGE`
**Decision:** Late fees are not stored automatically by the system. They are computed at read time when a `LedgerInstalment` is overdue beyond the configured grace period. If admin chooses to formally raise the late fee against a student, it is recorded as a `LedgerTransaction` of type `ADDITIONAL_CHARGE`. Late fee settings live on `OrganizationSettings`.

**Reason:** 
- Automatically storing late fees as transactions would require a scheduled background job running daily — operationally fragile, harder to audit, and potentially surprising to admins. 
- Computing on demand means the indicator is always accurate and reflects current settings without any stored state. 
- The formal recording step is an explicit, deliberate admin action — consistent with how manual fee administration actually works in institutions.

**Settings additions to `OrganizationSettings`:**
```python
late_fee_enabled        # BooleanField, default=False
late_fee_type           # CharField, choices=[PERCENTAGE, FIXED], null=True
late_fee_value          # DecimalField, null=True
late_fee_grace_days     # PositiveIntegerField, default=0
```

**On-demand computation:**
```python
def compute_late_fee(instalment, settings):
    if not settings.late_fee_enabled:
        return Decimal(0)
    grace_deadline = instalment.due_date + timedelta(
        days=settings.late_fee_grace_days
    )
    if date.today() <= grace_deadline:
        return Decimal(0)
    if settings.late_fee_type == PERCENTAGE:
        return instalment.amount_due * (settings.late_fee_value / 100)
    return settings.late_fee_value
```

**UI behaviour:**
- Overdue instalments are flagged visually in the ledger view with the computed late fee amount shown as an indicator.
- Admin sees a "Raise Late Fee" action on the overdue instalment row.
- Confirming posts a `LedgerTransaction` of type `ADDITIONAL_CHARGE` with `label="Late Fee"` and `amount=computed_late_fee`.
- Once posted, the transaction appears in the ledger audit trail and increases the student's outstanding balance via the debit path in balance computation (see ADR-035).
- Admin can also manually post an `ADDITIONAL_CHARGE` of any amount and label for non-standard penalty scenarios (e.g. library damage, exam re-sit fee).

**Why `ADDITIONAL_CHARGE` and not a dedicated `LATE_FEE` type:**
`ADDITIONAL_CHARGE` is intentionally generic — late fees, exam re-sit charges, equipment damage fees all share the same debit semantics. A dedicated `LATE_FEE` type would add specificity with no meaningful behavioral difference. The `label` field carries the human-readable distinction for reporting purposes.

**Alternatives considered:**
- Scheduled job auto-posting late fee transactions daily — rejected: fragile, surprising to admins, hard to audit.
- Storing late fee as a `CONCESSION` with negative label — rejected: concessions are credits; a late fee is a debit. Semantically incorrect and architecturally inconsistent with ADR-034.

**Status:** Implemented

---

## ADR-038: `tax/` as a separate Django app for tax rate management

**Decision:** Tax rate configuration lives in a dedicated `tax/` app,
not in `fees/` or `orgs/`.

**Model:**
```python
class TaxRate(models.Model):
    id              UUID, PK
    organization    FK → Organization, on_delete=PROTECT
    name            CharField, max_length=100
    rate            DecimalField, max_digits=5, decimal_places=2
    is_active       BooleanField, default=True
    created_at      auto
    updated_at      auto

    class Meta:
        ordering = ['name']
        constraints = [
            UniqueConstraint(
                fields=['organization', 'name'],
                condition=Q(is_active=True),
                name='unique_active_tax_rate_name_per_org'
            )
        ]
```

**Reason:** Tax is a cross-domain financial concern — not a fees
concern specifically. If future modules (library, hostel, payroll)
require taxable charges, they would need to import TaxRate from
fees/ which would be architecturally wrong. A dedicated tax/ app
owns the concept cleanly and allows tax reporting, exemption
certificates, and return summaries to be added later without
touching fees/ or orgs/.

**Alternatives considered:**
- `fees/` — rejected: tax is not a fees-specific concept
- `orgs/` — rejected: orgs/ handles identity and membership,
  not financial configuration

**Status:** Implemented

---

## ADR-039: Tax fields on `FeeHead` and `LedgerEntry` — tax exclusive, stored snapshots

**Decision:** Tax is modelled as tax exclusive — `charged_amount`
on `LedgerEntry` is the pre-tax net amount. `tax_amount` and
`gross_amount` are computed at ledger generation time and stored
as immutable snapshots alongside `charged_amount`.

**Fields added to `FeeHead`:**
```python
tax_rate    FK → TaxRate, on_delete=SET_NULL,
            null=True, blank=True
            # null = not taxable
```

**Fields added to `LedgerEntry`:**
```python
is_taxable          BooleanField, default=False
tax_rate_snapshot   DecimalField, max_digits=5,
                    decimal_places=2, null=True
                    # snapshot of TaxRate.rate at generation time
tax_amount          DecimalField, max_digits=10,
                    decimal_places=2, default=Decimal('0.00')
                    # charged_amount * tax_rate_snapshot / 100
                    # stored snapshot — never recomputed
gross_amount        DecimalField, max_digits=10,
                    decimal_places=2, default=Decimal('0.00')
                    # charged_amount + tax_amount
                    # stored snapshot — never recomputed
```

**Why tax exclusive:**
Tax exclusive is the international standard (IFRS, GAAP, VAT/GST
regulations globally). Revenue is recognised at the net amount;
tax is a liability collected on behalf of the government. Every
serious accounting system (Xero, QuickBooks, SAP, Stripe) uses
this approach. Tax inclusive hides the tax inside charged_amount
making it impossible to report net revenue or tax liability
accurately without backing out tax from every record.

**Why gross_amount is stored, not computed:**
Both `charged_amount` and `gross_amount` are financial record
snapshots. Storing `gross_amount` as a field rather than a
property ensures the figure is immutable and independently
auditable — consistent with how `charged_amount` and `head_name`
are already treated. A computed property could silently drift
if either component field were ever amended via a correction.

**Balance computation uses gross_amount:**
`FeeLedger.balance` sums `gross_amount` (not `charged_amount`)
across active entries — this is the total student obligation
including tax. `charged_amount` is available for net revenue
reporting. `tax_amount` is available for tax liability reporting.

**Migration note — existing rows:**
When this migration runs on a system with existing `LedgerEntry`
records, `gross_amount` defaults to `Decimal('0.00')`. This is
safe only on pre-production systems. On any system with live
financial data, a data migration must be run immediately after
to backfill `gross_amount = charged_amount` on all existing
rows (all existing entries are non-taxable, so
`gross_amount = charged_amount` is correct for them).

**Future tax features (deferred):**
- GST/VAT return summary endpoint
- Tax exemption certificates per student
- Tax registration number on invoices (Phase 2, PDF)
- Compound tax (tax on tax — rare, jurisdiction specific)
- Reverse charge VAT (B2B EU — not relevant for education)

**Status:** Implemented

---

## ADR-040: Currency, locale, and timezone settings on `OrganizationSettings`

**Decision:** Currency display, number formatting, timezone, and
date format are stored as org-level settings on
OrganizationSettings. These are display and formatting concerns
only — all monetary values are stored as plain DecimalField in
the database with no currency encoding at the data layer.

**Fields added to `OrganizationSettings`:**
```python
# Currency
currency_code       CharField, max_length=3, default='GBP'
                    # ISO 4217 currency code
currency_symbol     CharField, max_length=5, default='£'
                    # display symbol

# Locale
timezone            CharField, max_length=50,
                    default='Europe/London'
                    # IANA timezone string
                    # Europe/London (not UTC) — correctly
                    # handles BST (GMT+1 in summer)
date_format         CharField, max_length=20,
                    default='DD/MM/YYYY'
                    # frontend display format only
                    # DB always stores ISO 8601

# Number formatting
currency_position   CharField, max_length=6,
                    choices=[PREFIX, SUFFIX],
                    default='PREFIX'
                    # PREFIX: £50,000 / SUFFIX: 50,000£
decimal_separator   CharField, max_length=1, default='.'
thousands_separator CharField, max_length=1, default=','
```

**Why display-only (no MoneyField or currency-encoded storage):**
Zavia is single-currency per org — one institution operates
in one currency. Storing the currency code once on
OrganizationSettings and applying it at display time is the
correct separation of concerns. Coupling monetary values to
currency codes at the data layer (MoneyField pattern)
complicates arithmetic, aggregation, and ORM queries with
no benefit for single-currency orgs. Multi-currency support
(accepting payments in foreign currencies) is a future
concern not required by any current client.

**Why Europe/London not UTC:**
UTC does not observe Daylight Saving Time. Europe/London
correctly handles GMT (winter) and BST/GMT+1 (summer).
Using UTC as the default for a UK-targeted product would
cause due date and overdue calculations to be off by one
hour during British Summer Time. Always use IANA timezone
names — never raw UTC offsets.

**Default rationale — GBP / Europe/London / DD/MM/YYYY:**
Zavia targets the UK market as its primary market. GBP,
Europe/London, and DD/MM/YYYY are the correct defaults for
a UK-first product. These defaults are safety fallbacks only
— the onboarding flow detects the admin's browser locale
(via Intl.DateTimeFormat and navigator.language) and
pre-populates settings before the org is created. In
practice the DB default is never used if onboarding
is completed correctly.

**Onboarding flow (frontend, to be implemented):**
```javascript
// Detect from browser at org creation time
const timezone = Intl.DateTimeFormat()
    .resolvedOptions().timeZone
const locale = navigator.language
// Pre-populate settings form — admin confirms or overrides
// On save → stored on OrganizationSettings
```

**Timezone usage in backend:**
- DateField values (due_date, transaction_date) store
  date only — timezone does not affect stored values
- DateTimeField values (created_at, updated_at) stored
  as UTC in DB — standard Django behaviour
- Backend uses timezone setting when computing relative
  date comparisons (e.g. overdue checks in
  compute_late_fee) via django.utils.timezone
- Frontend uses timezone setting to display datetime
  values in the institution's local time

**Date format usage:**
- Backend always stores and returns ISO 8601 (YYYY-MM-DD)
- Frontend uses date_format setting for display only
- No backend date formatting — purely a frontend hint

**Status:** Implemented
