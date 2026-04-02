# Zavia – Design Language
> Last updated: April 2026

---

## The Principle

The interface should disappear. What remains is the work.

Every border, background, shadow, and colour that doesn't carry meaning is noise. Remove it. The user came to manage their institution — not to admire the UI. Design that calls attention to itself has failed.

This is not minimalism for aesthetic reasons. It is restraint in service of focus.

---

## Colour

The UI is almost entirely monochromatic. Colour appears only when it means something.

**The palette in practice:**
- 90% of the UI is `foreground`, `text-secondary`, `text-tertiary` on `background`
- `surface` is used sparingly — table header rows, tag backgrounds, input fills
- `accent` (blue) appears on: the active sidebar item, primary CTA buttons, and nothing else
- `success` (green), `warning` (amber), `destructive` (red) appear on: status badges, error states, and nothing else

**Never use colour to:**
- Distinguish sections or group content
- Make something look "important"
- Add visual interest to an otherwise bare layout

If you find yourself reaching for a colour that isn't signalling a status or action — stop. Use typography instead.

All colours are CSS tokens from `src/index.css`. Never hardcode hex values.

---

## Typography

Hierarchy is built entirely from font weight, size, and text colour. No decorative elements needed.

| Role | Classes | Where |
|---|---|---|
| Page heading | `text-base font-semibold text-foreground` | Page titles, entity names |
| Section label | `text-xs font-medium uppercase tracking-wider text-text-tertiary` | Column headers, section dividers |
| Body | `text-sm text-foreground` | Row content, form values |
| Supporting | `text-sm text-text-secondary` | Subtitles, descriptions |
| Metadata | `text-xs text-text-tertiary` | Timestamps, counts |
| Code / ID | `font-mono text-xs text-text-secondary` | Codes, UUIDs, technical values |

**Rules:**
- `font-semibold` only for headings and active nav states
- `uppercase tracking-wider` only for section labels and column headers — never body text
- Never below `text-xs` for readable content
- Let colour carry the hierarchy — a `text-text-tertiary` label next to a `text-foreground` value communicates structure without any border or background

---

## Surfaces and Containers

This is where most interfaces go wrong. The default instinct is to wrap things in cards. Resist it.

**Content floats on the page.** Sections are separated by spacing and typography — not by borders, backgrounds, or card chrome.

**When to use a border:**
- Around a list/table as a whole — one outer border, rows separated by subtle inner borders
- Around an input field at rest
- Around a tag or code badge

**When not to use a border:**
- Around a form section
- Around a page content area
- Around a group of related fields
- Around anything that is "just content"

**`surface` background is used for:**
- Table header rows
- Inline code/tag badges
- Hovered sidebar items

**Never use:**
- Shadows on content (drawer shadow is the one exception)
- Rounded cards with backgrounds for page sections
- Coloured section backgrounds

---

## Layout

### Pages
The page header is lean. A title on the left, one action on the right. No hero sections, no descriptive paragraphs, no icon decorations.

```
px-6 py-4   — page header and content padding
```

### Forms
Forms feel like documents. Fields sit directly on the page — no wrapping card, no panel background. The page surface is the form surface.

```
max-w-lg        — constrains single-column form width
gap-1.5         — label to input
gap-5           — field to field
gap-8           — section to section
```

Section labels divide the form:
```
text-xs font-medium uppercase tracking-wider text-text-tertiary mb-4
```

No border, no background — just the label and spacing.

### Tables / Lists
CSS grid, not `<table>`. Grid columns are defined per feature.

```
grid-cols-[{definitions}] gap-4 items-center px-4 py-2.5
border-b border-border-subtle last:border-0
```

Header row:
```
bg-surface border-b border-border-subtle
text-xs font-medium uppercase tracking-wider text-text-tertiary
px-4 py-2
```

---

## Interaction Patterns

### Hover reveals actions
Rows are clean at rest. Edit, delete, and other row actions are invisible until hover. This keeps the list uncluttered when scanning and surfaces actions exactly when needed.

```tsx
// Row
<div className="group grid-cols-[...] ...">

// Action — invisible at rest, visible on hover
<button className={cn(iconButtonClass, "opacity-0 group-hover:opacity-100")}>
```

### Inline edit
For simple entities (1–2 fields), edit happens inline in the row. No drawer, no page navigation. Pencil icon on hover → row transforms into an edit state → check/X to confirm or cancel.

### Drawers
For entities with 3–6 fields that are contextually tied to a parent. Right-side slide-in, `max-w-md`. The drawer is a focused editing surface — not a mini page.

### Full page forms
Only when the create flow requires cross-entity selection (e.g. selecting a program when creating an intake). Anything simpler belongs in a drawer.

### Confirmation dialogs
Every destructive action requires explicit confirmation. No exceptions. Single-click delete is never acceptable.

---

## Empty States

Two patterns only:

**Full empty state** — list has no items. Centred, dashed border container, icon + message + implicit action.
```
rounded-lg border border-dashed border-border
px-6 py-12 text-center text-sm text-text-tertiary
```

**Compact add button** — items exist, adding more is a frequent action.
```
rounded-lg border border-dashed border-border
w-full px-4 py-2 text-sm text-text-tertiary
hover:text-foreground hover:border-border transition-colors
```

Use neither when the list has items and adding is infrequent — a header button is sufficient.

---

## Status Badges

Small, tight, coloured pills. The only place in the UI where colour is used for state — each colour maps to a specific meaning, never applied arbitrarily.

```
px-2 py-0.5 rounded-full text-xs font-medium
```

| Status | Colour |
|---|---|
| OPEN / Active | `success` green |
| UPCOMING / Pending | `warning` amber |
| CLOSED / Inactive | `text-tertiary` grey |
| Error | `destructive` red |

Interactive badges (status change inline) open a small dropdown on click. The dropdown is absolutely positioned, `z-50`, outside-click to dismiss.

---

## Buttons

One primary button per view. Everything else is secondary or ghost.

| Variant | When |
|---|---|
| `default` (accent fill) | The one primary action on the page |
| `outline` | Secondary action alongside a primary |
| `ghost` | Low-emphasis inline actions |
| `destructive` | Inside confirmation dialogs only |
| `iconButtonClass` | Icon-only row actions — lighter than Button |
| `navButtonClass` | Breadcrumb links, back navigation |

Destructive buttons never appear directly in the UI — only inside a confirm dialog after the user has initiated a delete action.

---

## Navigation

### Sidebar
Minimal. Icon + label, no section backgrounds, no nested trees deeper than one level. Active state is `font-semibold text-foreground` — not a coloured background.

Structure:
```
Dashboard
─────────────
Academic
  Departments
  Courses
  Programs
─────────────
Operations
  Intakes
  Enrollments
  Fees (future)
─────────────
Admin
  Users
  Settings
```

### Breadcrumbs
Used on detail and form pages. Maximum 3 levels.
```
text-xs text-text-tertiary — links via navButtonClass
ChevronRight w-3 h-3      — separator
text-foreground            — current page (not a link)
```

### Tabs
On detail pages with multiple content sections.
```
Active:   border-b-2 border-foreground text-foreground font-medium
Inactive: border-b-2 border-transparent text-text-secondary hover:text-foreground
```

---

## Z-Index

| Context | Value |
|---|---|
| Inline dropdowns, search results | `z-10` |
| Drawers and overlays | `z-50` |
| Status badge dropdowns | `z-50` |

Dropdowns inside tables must use `z-50`. Parent containers with `overflow-hidden` will clip them — use `overflow-visible` when dropdowns are present.

---

## What We Never Do

- Wrap form sections in cards
- Use colour for decoration
- Add shadows to content areas
- Use more than one accent button per view
- Delete without confirmation
- Use `<table>` elements — CSS grid only
- Hardcode colours
- Show row actions without hover
- Use uppercase for anything other than labels and column headers
- Add decorative icons, illustrations, or gradients
