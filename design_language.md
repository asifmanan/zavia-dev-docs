# Zavia – UI Design Language
> Living document. Update when new patterns are established or existing ones are refined.

---

## 1. Philosophy

Zavia's UI is inspired by Linear.app — minimal, high-information-density, and typography-driven. The interface steps back so the content steps forward. Every element earns its place.

**Core principles:**
- **Typography over decoration** — hierarchy is communicated through font weight, size, and colour — not borders, shadows, or backgrounds
- **Colour for meaning only** — colour is never decorative. It signals status, error, success, or warning
- **Density over whitespace** — admin tools are used repeatedly by professionals. Compact, scannable layouts are preferred over airy, marketing-style layouts
- **Workflow-driven** — the UI reflects real operational workflows, not CRUD forms. Every screen should answer "what is the user trying to accomplish?"
- **No decorative elements** — no gradients, no shadows on cards, no illustrations, no icon decorations unless they carry meaning

---

## 2. Colour System

All colours are CSS custom properties defined in `src/index.css`. Never use hardcoded hex values — always use the design token.

### Surfaces
| Token | Usage |
|---|---|
| `background` | Page background — near white (light) / near black (dark) |
| `surface` | Subtle raised surface — table headers, input backgrounds, tags |
| `surface-raised` | Elevated surface — popovers, dropdowns, cards |

### Borders
| Token | Usage |
|---|---|
| `border` | Standard border — dividers, table outer borders |
| `border-subtle` | Subtle border — table row separators, input borders at rest |

### Text
| Token | Usage |
|---|---|
| `foreground` / `text-primary` | Primary content — headings, values, important labels |
| `text-secondary` | Supporting content — descriptions, secondary values |
| `text-tertiary` | De-emphasised content — placeholders, column headers, metadata |

### Semantic
| Token | Usage |
|---|---|
| `accent` | Primary CTA buttons, active tab indicators, links |
| `destructive` | Delete actions, error states, error borders |
| `success` | Positive status — OPEN intake, active state |
| `warning` | Caution status — UPCOMING intake, pending state |

### Status colour mapping
| Status | Colour | Token |
|---|---|---|
| OPEN / Active | Green | `success` |
| UPCOMING / Pending | Amber | `warning` |
| CLOSED / Inactive | Grey | `text-tertiary` |
| Error | Red | `destructive` |

---

## 3. Typography

No custom font — system font stack via Tailwind. Typography hierarchy is achieved through size + weight + colour combinations.

### Scale in use
| Role | Classes | Usage |
|---|---|---|
| Page title | `text-base font-semibold text-foreground` | Page and section headings |
| Section label | `text-xs font-medium uppercase tracking-wider text-text-tertiary` | Column headers, section dividers |
| Body | `text-sm text-foreground` | Primary content in rows, forms |
| Supporting | `text-sm text-text-secondary` | Descriptions, subtitles |
| Metadata | `text-xs text-text-tertiary` | Timestamps, counts, codes |
| Mono | `font-mono text-xs text-text-secondary` | Codes, IDs, technical values |

### Rules
- Never go below `text-xs` for readable content
- `font-semibold` is reserved for headings and active states — not emphasis within body text
- Uppercase tracking (`uppercase tracking-wider`) is reserved for section labels and column headers only

---

## 4. Spacing & Layout

### Page layout
- Full-height shell with persistent sidebar
- Content area: `px-6 py-4` for page headers, `p-6` for content areas
- Maximum form width: `max-w-lg` for single-column forms
- Maximum content width: unconstrained — tables fill available width

### Component spacing
- Form field groups: `gap-1.5` between label and input, `gap-5` between fields
- Table row padding: `px-4 py-2.5`
- Section padding: `px-6 py-4`
- Inline action button padding: `p-1.5` (via `iconButtonClass`)

---

## 5. Component Patterns

### Buttons

Four button types in use — defined in `src/components/ui/button-variants.ts`:

| Variant | Usage |
|---|---|
| `default` (accent) | Primary CTA — one per view maximum |
| `outline` | Secondary action alongside a primary CTA |
| `ghost` | Low-emphasis actions, inline row actions |
| `destructive` | Delete confirmations only |
| `link` | Inline text actions, no chrome |

Two bare button classes for cases where shadcn Button is too heavy:
- `iconButtonClass` — icon-only table row actions
- `navButtonClass` — breadcrumb and back navigation links

**Rules:**
- Never more than one `default` (accent) button visible at a time
- Destructive actions always require a confirmation dialog before executing
- Icon buttons use `iconButtonClass`, never a full `Button` component

---

### Forms

**Drawers** — for create/edit of entities with 3–6 fields that are contextually tied to a parent (e.g. editing an intake from its detail page). Uses Vaul drawer, slides from the right.

**Full page forms** — for create flows that require cross-entity selection (e.g. creating an intake requires selecting a program). Route: `/entity/new`.

**Inline edit rows** — for list entities with 1–2 editable fields (e.g. Departments). Edit state triggered by pencil icon, confirmed by check icon or Enter key.

**Rules:**
- No card wrappers on full page forms — fields sit directly on the page surface
- Drawer width: `max-w-md`
- Always include Cancel + Save in a bottom action bar
- Save button shows `Loader2` spinner when saving
- Field label always above input, never inline placeholder-only
- Helper text (`text-xs text-text-tertiary`) below input when field meaning needs clarification

---

### Tables / Lists

All list views use a CSS grid layout — not `<table>` elements. Grid columns are defined explicitly per feature based on content needs.

**Standard row structure:**
```
grid-cols-[{col-definitions}] gap-4 items-center px-4 py-2.5
border-b border-border-subtle last:border-0
```

**Column header row:**
```
grid-cols-[{same}] gap-4 px-4 py-2 bg-surface border-b border-border-subtle
text-xs font-medium uppercase tracking-wider text-text-tertiary
```

**Rules:**
- Row actions (edit, delete) are hidden by default, visible on `group-hover` via `opacity-0 group-hover:opacity-100`
- Action icons use `iconButtonClass`
- Delete icon on hover: `hover:text-destructive hover:bg-destructive/5`
- Monospaced values (codes, IDs): `font-mono text-xs text-text-secondary`
- Empty values: render `—` (em dash), never blank
- Numeric counts: `tabular-nums`, muted when zero

**Responsive:**
- Hide lower-priority columns below `md` breakpoint using `hidden md:block`
- Always keep: name, status/primary value, actions visible on mobile

---

### Empty States

Two patterns depending on context:

**Full empty state** — when a list has no items at all. Centred, dashed border, icon + heading + subtext + implicit or explicit call to action.

```
rounded-lg border border-dashed border-border px-6 py-12
text-center — PlusCircle icon, heading, subtext
```

**Compact add button** — when items exist and adding more is frequent (e.g. Add Level in curriculum). Dashed border, minimal padding, sits below the list.

```
rounded-lg border border-dashed border-border px-4 py-2
w-full text-sm text-text-tertiary hover:text-foreground hover:border-border
```

**Rules:**
- Empty state only when the list is genuinely empty — not on filter/search with no results (use a "no results" message instead)
- Compact add button only when the add action is frequent and contextual — not for entities created via a separate page

---

### Status Badges

Inline badge showing current status. Coloured background with matching text.

**Pattern:**
```
px-2 py-0.5 rounded-full text-xs font-medium
```

**Interactive status badge** (e.g. IntakeStatusBadge):
- Clicking opens a small absolute-positioned dropdown
- Dropdown uses `useRef` + `useEffect` for outside-click detection
- `type="button"` on all buttons to prevent form submission
- Disabled + spinner during save
- Current selection highlighted with `font-semibold`
- Closes immediately on selection

---

### Drawers

Right-side slide-in panel using Vaul. Used for create/edit forms.

**Structure:**
```
fixed right-0 top-0 bottom-0 z-50
flex w-full max-w-md flex-col bg-background shadow-xl
```

- Title in header: `px-6 py-4 border-b border-border-subtle text-base font-semibold`
- Form content: `flex flex-col gap-5 p-6 flex-1 overflow-y-auto`
- Action bar: `mt-auto flex items-center justify-end gap-2 px-6 py-4 border-t border-border-subtle`
- Overlay: `fixed inset-0 z-50 bg-black/40`
- `onPointerDown={(e) => e.stopPropagation()}` on date inputs to prevent drawer drag interference

---

### Confirm Delete Dialog

Always used before destructive actions. Never delete on single click.

- Title: "Delete {entity}"
- Description: names the specific record being deleted
- Confirm button: `destructive` variant
- Shows error inline if delete fails (e.g. dependency check)
- Busy state disables both buttons during deletion

---

### Breadcrumbs

Used on detail and form pages. Sits above the page title.

```
flex items-center gap-1.5 text-xs text-text-tertiary mb-3
```

- Links use `navButtonClass`
- Separator: `<ChevronRight className="w-3 h-3" />`
- Current page: `text-foreground` (no link)
- Maximum 3 levels deep

---

### Tabs

Used on detail pages with multiple content sections.

```
border-b-2 transition-colors capitalize px-4 py-2 text-sm
Active: border-foreground text-foreground font-medium
Inactive: border-transparent text-text-secondary hover:text-foreground
```

---

### Section Labels

Used to divide content areas and as table column headers.

```
text-xs font-medium uppercase tracking-wider text-text-tertiary
```

---

## 6. Page Structures

### List page
```
Header row: title (left) + primary action button (right)
Filter row: dropdowns for filtering (when applicable)
Table: column headers + rows
Empty state: when no items
```

### Detail page
```
Breadcrumb
Header: title + code badge (if applicable) + subtitle + actions (edit, delete)
Tab bar (if multiple sections)
Tab content
```

### Full page form
```
Breadcrumb
Page title
Form sections with section labels
Fields with labels and optional helper text
Bottom action bar: Cancel + Save
```

---

## 7. Icons

Lucide React — used throughout. Icon size conventions:

| Context | Size |
|---|---|
| Inline row action icons | `w-3.5 h-3.5` |
| Button icons | `w-4 h-4` (handled by buttonVariants) |
| Empty state icons | `w-8 h-8` or `w-6 h-6` |
| Breadcrumb separator | `w-3 h-3` |
| Loading spinner | `w-3.5 h-3.5 animate-spin` (inline), `w-4 h-4 animate-spin` (button) |

---

## 8. Z-Index Scale

| Layer | Value | Usage |
|---|---|---|
| Dropdown menus | `z-10` | Inline dropdowns, search results |
| Sticky elements | `z-20` | Sticky headers |
| Drawers / overlays | `z-50` | Vaul drawers, modal overlays |
| Status badge dropdown | `z-50` | Must clear table overflow |

**Rule:** Always use `z-50` for absolutely-positioned menus that must clear table rows. Tables with `overflow-hidden` on parent containers will clip dropdowns — use `overflow-visible` on the table container when dropdowns are present.

---

## 9. Dark Mode

All colour tokens have dark mode equivalents defined in `src/index.css` under `.dark`. Dark mode is toggled by adding the `dark` class to `<html>`.

**Rules:**
- Never hardcode light-mode colours — always use tokens
- Test all new components in both light and dark mode before marking complete
- Status colours (`success`, `warning`, `destructive`) are adjusted for dark mode contrast

---

## 10. What We Avoid

- ❌ Card wrappers on full page forms
- ❌ Shadows on list rows or content cards
- ❌ Decorative illustrations or icons
- ❌ Gradient backgrounds
- ❌ More than one accent/primary button visible at once
- ❌ Inline delete without confirmation
- ❌ Colour for decoration — only for meaning
- ❌ `<table>` elements — use CSS grid
- ❌ Hardcoded hex colours
- ❌ Font sizes below `text-xs` for readable content
- ❌ Uppercase for body text — only for labels and column headers
