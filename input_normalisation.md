# Input Normalisation Standards – Zavia
> Standardised rules for normalising user input before persistence.
> Reference this document when implementing any new feature.

---

## 1. The Problem

User input is inconsistent. Without normalisation:
- `"cs"`, `"CS"`, `"Cs"` are treated as different codes — uniqueness
  constraints silently allow duplicates that are logically the same value
- Leading/trailing whitespace causes lookup mismatches
- Mixed-case names produce inconsistent display

Normalisation must happen server-side in the use case layer — never rely
on the client to send clean data.

---

## 2. Normalisation Rules by Field Type

| Field type | Rule | Example |
|------------|------|---------|
| `code` | `.strip().upper()` | `" cs "` → `"CS"` |
| `name` | `.strip()` | `" Computer Science "` → `"Computer Science"` |
| `email` | `.strip().lower()` | `" A@B.COM "` → `"a@b.com"` |
| `slug` | `.strip().lower()` | handled by Django `SlugField` |
| `phone` | `.strip()` | remove surrounding whitespace only |
| `description` | `.strip()` | remove surrounding whitespace only |

---

## 3. Where to Apply

**Always in the use case layer** — not in serializers, not in views, not
on the model.

Apply in both `create` and `update` use cases for every field that has a
normalisation rule.

### Template

```python
def create_x(*, organization, data) -> X:
    with transaction.atomic():
        payload = dict(data)

        # --- Normalisation ---
        if payload.get('code'):
            payload['code'] = payload['code'].strip().upper()
        if payload.get('name'):
            payload['name'] = payload['name'].strip()
        # ---------------------

        payload['organization'] = organization
        obj = X(**payload)
        try:
            obj.save()
        except IntegrityError:
            raise ValidationError(
                {"code": "A record with this code already exists."}
            )
        return obj


def update_x(*, obj, data) -> X:
    with transaction.atomic():
        for field, value in data.items():
            setattr(obj, field, value)

        # --- Normalisation ---
        if obj.code:
            obj.code = obj.code.strip().upper()
        if obj.name:
            obj.name = obj.name.strip()
        # ---------------------

        try:
            obj.save()
        except IntegrityError:
            raise ValidationError(
                {"code": "A record with this code already exists."}
            )
        return obj
```

---

## 4. Frontend — Display Consistency

Even though normalisation happens server-side, apply these display rules
on the frontend for immediate feedback before the server responds:

| Field type | Display rule |
|------------|-------------|
| `code` | Always render in `font-mono`. No case transformation needed — server returns normalised value |
| `name` | Render as-is |
| `email` | Render lowercase |

For `code` input fields, add `className="uppercase"` as a CSS hint so the
user sees uppercase as they type — but do not transform the value in JS.
The server is the source of truth.

```tsx
<Input
  placeholder="Code"
  className="font-mono uppercase"  {/* visual hint only */}
  value={code}
  onChange={(e) => setCode(e.target.value)}
/>
```

---

## 5. Checklist — Per Feature

### Backend
- [ ] `code` field normalised to `.strip().upper()` in create use case
- [ ] `code` field normalised to `.strip().upper()` in update use case
- [ ] `name` field normalised to `.strip()` in create use case
- [ ] `name` field normalised to `.strip()` in update use case
- [ ] `IntegrityError` caught after save with field-specific message

### Frontend
- [ ] `code` input has `className="... uppercase"` for visual hint
- [ ] `code` displayed in `font-mono`
- [ ] No client-side `.toUpperCase()` transforms — server handles it
