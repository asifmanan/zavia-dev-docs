# Development Workflow – Zavia
> Agreed working rules to avoid rework and wasted time.

---

## Feature Implementation Checklist

Before writing any frontend code, complete the backend contract review below.
Only once all checks pass do we move to frontend implementation.

### 1. Backend Contract Review

**Mutating views**
- [ ] `perform_create` — does it assign `serializer.instance` to the created object?
- [ ] `perform_update` — does it assign `serializer.instance = updated_object` so DRF returns fresh data?
- [ ] `perform_destroy` — no return value needed (204 No Content)

**Error handling in use case layer**
- [ ] All `IntegrityError` / DB constraint violations are caught and re-raised as `ValidationError` with a human-readable message
- [ ] No raw Django or database errors can reach the client

**Serializer response shape**
- [ ] Response includes exactly what the frontend needs — no missing fields, no unnecessary nested data
- [ ] Read-only denormalised fields (e.g. `id_type_name`) are present where the frontend needs them
- [ ] No sensitive or redundant data included

**Queryset**
- [ ] `.distinct()` applied wherever joins on related tables are used (to prevent duplicate rows)
- [ ] `prefetch_related` / `select_related` added where nested data is serialized

---

## Frontend Implementation Order

Only start frontend work after backend contract review is complete.

1. `types.ts` — define all types from the API response shape
2. `api/` — API module with all endpoints
3. `hooks/` — data fetching and mutation hooks
4. Components — UI built against the confirmed contract
5. Page — mount components, wire up hooks

---

## General Rules

- **Finalise model and API design before writing code** — changes after implementation are expensive
- **Backend first, frontend second** — never assume the backend is correct; verify the contract
- **One concern per commit** — backend fixes and frontend features are separate commits
- **Patterns are documented** — when a non-obvious problem is solved, add it to `ui_patterns.md` so it's never solved twice