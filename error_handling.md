# Error Handling Standards – Zavia
> Standardised approach for error handling across backend and frontend.
> Reference this document when implementing any new feature.
> **Claude Code prompts must explicitly reference this file for any feature
> involving forms or mutations.**

---

## 1. Backend — Use Case Layer

All business logic errors must be raised as DRF `ValidationError`, never as
raw Django exceptions or database errors.

### Rules

- Always import from `rest_framework.exceptions`, not `django.core.exceptions`
- Catch `IntegrityError` in every `create` and `update` use case and re-raise
  as `ValidationError` with a human-readable message
- Never let a raw `IntegrityError`, `DatabaseError`, or `ProgrammingError`
  reach the client
- Field-specific errors: `raise ValidationError({"field_name": "Message."})`
- Non-field errors: `raise ValidationError("Message.")` or
  `raise ValidationError({"non_field_errors": "Message."})`
- When catching `IntegrityError`, inspect the error string to identify which
  field caused it and raise a field-specific error accordingly:

```python
except IntegrityError as e:
    err = str(e)
    if 'name' in err:
        raise ValidationError({"name": "A record with this name already exists."})
    if 'code' in err:
        raise ValidationError({"code": "A record with this code already exists."})
    raise ValidationError("A record with this value already exists.")
```

### Template — create use case

```python
from django.db import IntegrityError, transaction
from rest_framework.exceptions import ValidationError

def create_x(*, organization, data) -> X:
    with transaction.atomic():
        payload = dict(data)
        payload['organization'] = organization
        obj = X(**payload)
        try:
            obj.save()
        except IntegrityError as e:
            err = str(e)
            if 'name' in err:
                raise ValidationError({"name": "A record with this name already exists."})
            if 'code' in err:
                raise ValidationError({"code": "A record with this code already exists."})
            raise ValidationError("A record with this value already exists.")
        return obj
```

### Template — update use case

```python
def update_x(*, obj, data) -> X:
    with transaction.atomic():
        for field, value in data.items():
            setattr(obj, field, value)
        try:
            obj.save()
        except IntegrityError as e:
            err = str(e)
            if 'name' in err:
                raise ValidationError({"name": "A record with this name already exists."})
            if 'code' in err:
                raise ValidationError({"code": "A record with this code already exists."})
            raise ValidationError("A record with this value already exists.")
        return obj
```

---

## 2. Backend — Input Normalisation

Normalise user-supplied strings in the use case before saving, not in the
serializer or view.

### Rules

- `code` fields: always `.strip().upper()` before save
- `name` fields: always `.strip()` before save
- `email` fields: always `.strip().lower()` before save
- Apply normalisation in both `create` and `update` use cases

### Template

```python
def create_x(*, organization, data) -> X:
    with transaction.atomic():
        payload = dict(data)
        if payload.get('code'):
            payload['code'] = payload['code'].strip().upper()
        if payload.get('name'):
            payload['name'] = payload['name'].strip()
        ...
```

---

## 3. Backend — Response Shape Contract

DRF `ValidationError` produces these response shapes. The frontend parser
handles all of them:

| Raised as | Response shape |
|-----------|---------------|
| `ValidationError({"field": "msg"})` | `{ "field": "msg" }` |
| `ValidationError("msg")` | `{ "detail": "msg" }` |
| `ValidationError({"non_field_errors": "msg"})` | `{ "non_field_errors": "msg" }` |

---

## 4. Frontend — extractErrorMessage

Single shared utility at `src/lib/extract-error-message.ts`.
All hooks must use this — never parse `error.response.data` manually.

### Parse order

1. `response.data` is a short string (< 200 chars) → return it directly
2. `response.data` is a long string (≥ 200 chars) → return fallback
   (guards against HTML tracebacks from unexpected 500s)
3. `response.data` is an object → try in order:
   - `data.detail`
   - `data.non_field_errors`
   - first value of any other key on the object
   Each value may be `string` or `string[]` — take `[0]` if array.
4. `error.message` (Axios network error)
5. `fallback` string passed by the caller

### Usage in hooks

```ts
import { extractErrorMessage } from "@/lib/extract-error-message";

try {
  await api.doSomething();
} catch (e) {
  setError(extractErrorMessage(e, "Failed to do something."));
  throw e; // re-throw so inline components can catch locally
}
```

### Rules

- Always pass a meaningful fallback — never pass an empty string
- Always re-throw after setting error so inline row components can catch
  and display locally without the page-level error overriding them
- Never render `error` strings longer than 200 characters — the
  `extractErrorMessage` utility already guards this, but components should
  not bypass it by accessing `error.response.data` directly

---

## 5. Frontend — Field-Level Error State (CRITICAL)

> This pattern must be applied to every form and inline edit row.
> Do not use a single `error: string | null` for forms with multiple fields.
> A generic string cannot identify which field failed, so no field can be
> highlighted red. This was the source of repeated bugs.

### The pattern

Use a structured error object, not a flat string:

```ts
const [errors, setErrors] = useState<{
  name?: string;
  code?: string;
  general?: string;
}>({});
```

Add a `parseErrors` helper in the component file to extract field-level
errors from the DRF response:

```ts
function parseErrors(e: unknown) {
  const err = e as { response?: { data?: Record<string, string> } };
  const data = err?.response?.data;
  if (!data || typeof data !== 'object') {
    return { general: "Something went wrong. Please try again." };
  }
  return {
    name: data.name,
    code: data.code,
    general: data.detail ?? data.non_field_errors,
  };
}
```

Adapt the keys to match the fields in the form being built.

On catch: `setErrors(parseErrors(e))`
On name input onChange: `setErrors(prev => ({...prev, name: undefined}))`
On code input onChange: `setErrors(prev => ({...prev, code: undefined}))`
On cancel: `setErrors({})`

### Deriving the display message

For a single consolidated error message (preferred for narrow layouts):

```ts
const errorMessage = errors.name ?? errors.code ?? errors.general;
```

This surfaces the most specific error first (field-level before general).

---

## 6. Frontend — Red Border on Failing Field (CRITICAL)

> Every input that can fail validation must show a red border when its
> field error is set. This is non-negotiable — the error message alone
> is not sufficient UX. The user must see immediately which field failed.

Apply conditional error styling using `cn()`:

```tsx
<Input
  className={cn(
    "h-7 text-sm",
    errors.name && "border-destructive focus-visible:ring-destructive"
  )}
  value={name}
  onChange={(e) => {
    setName(e.target.value);
    setErrors(prev => ({...prev, name: undefined}));
  }}
/>

<Input
  className={cn(
    "h-7 text-sm font-mono uppercase",
    errors.code && "border-destructive focus-visible:ring-destructive"
  )}
  value={code}
  onChange={(e) => {
    setCode(e.target.value);
    setErrors(prev => ({...prev, code: undefined}));
  }}
/>
```

---

## 7. Frontend — Error Message Positioning (CRITICAL)

> Error messages must be visually attached to the form that caused them.
> A common mistake is rendering the error as a sibling element outside
> the form's wrapper div — it then appears as a separate row in the list,
> visually attached to the wrong element.

### Rule: wrap form + error together

The input row and its error message must share a single wrapper div:

```tsx
{/* CORRECT */}
<div className="border-t border-border-subtle">
  <div className="grid grid-cols-[1fr_100px_120px_72px] gap-4
  items-center px-4 py-2.5">
    {/* inputs and action buttons */}
  </div>
  {errorMessage && (
    <p className="px-4 pb-2 text-xs text-destructive">{errorMessage}</p>
  )}
</div>

{/* WRONG — error is a sibling, appears as a separate list row */}
<div className="border-t border-border-subtle">
  <div className="grid ...">
    {/* inputs */}
  </div>
</div>
{errorMessage && (
  <p className="px-4 pb-2 text-xs text-destructive">{errorMessage}</p>
)}
```

The same rule applies to inline edit rows — the error paragraph must be
inside the same wrapper div as the grid row, not a sibling to it.

---

## 8. Frontend — Inline Row Error Display

For inline edit rows (e.g. DepartmentRow, ExternalIdRow), errors are local
to the row — never bubbled to the page level.

```tsx
<div className="border-b border-border-subtle last:border-0">
  <div className="grid grid-cols-[1fr_100px_120px_72px] gap-4
  items-center px-4 py-2.5">
    {/* inputs */}
  </div>
  {errorMessage && (
    <p className="px-4 pb-2 text-xs text-destructive">{errorMessage}</p>
  )}
</div>
```

Clear errors on input `onChange` and on cancel.

---

## 9. Frontend — Page-Level Error Display

For page-level or list-level errors (load failures, delete failures):

```tsx
{error && (
  <div className="px-6 py-3 text-sm text-destructive border-b border-border-subtle">
    {error}
  </div>
)}
```

---

## 10. Checklist — Before Marking a Feature Complete

### Backend
- [ ] `IntegrityError` caught with field-specific detection in create
- [ ] `IntegrityError` caught with field-specific detection in update
- [ ] All string inputs normalised (strip, case) before save
- [ ] No raw Django or DB exceptions can reach the client

### Frontend
- [ ] Structured `errors` object used — not a flat `error: string | null`
- [ ] `parseErrors` helper extracts field-level errors from DRF response
- [ ] Every input has red border styling when its field error is set
- [ ] Error message derived as `errors.fieldA ?? errors.fieldB ?? errors.general`
- [ ] Error message rendered inside the same wrapper div as the form inputs
- [ ] Errors clear on input `onChange` for the relevant field
- [ ] Errors clear on cancel
- [ ] All hooks use `extractErrorMessage` — no manual `response.data` parsing
- [ ] Inline row errors are local state, not page-level state
- [ ] Error strings over 200 chars never rendered (handled by utility)
