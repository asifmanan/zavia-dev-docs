# Design Language — Zavia

A reference for every visual and interaction pattern used across the application. Follow these guidelines precisely when building new UI. Deviating from them — even for convenience — creates visual inconsistency that accumulates into a broken-looking product.

---

## 1. Design Philosophy

The UI is deliberately neutral, dense, and text-forward. It is built for people who use it every day, not for first-time visitors. Key principles:

- **Quiet by default** — no decorative elements, no gradient fills, no bold colour except accent and destructive
- **Information density over whitespace** — rows are compact; padding is measured, not generous
- **Typography-driven layout** — detail pages use flat, editorial layouts. Visual hierarchy is created through type scale and spacing, not borders or backgrounds
- **Progressive disclosure** — secondary actions (edit, delete) appear on hover, not always visible
- **Responsive and functional** — tables collapse to 2 columns on small screens; detail pages use CSS grid for responsive reordering
- **Consistent structure** — every list page, table, detail page, and empty state follows the same structural template

---

## 2. Colour Tokens

All colours are defined as CSS variables in `src/index.css` and mapped into Tailwind via `@theme inline`. Never use raw hex or Tailwind palette colours (e.g. `gray-500`) — always use these semantic tokens.

### Surface hierarchy

| Token | Light | Dark | Usage |
|---|---|---|---|
| `background` | `hsl(0 0% 99%)` | `hsl(0 0% 6%)` | Page background, input backgrounds |
| `surface` | `hsl(0 0% 97%)` | `hsl(0 0% 9%)` | Hover states on rows and buttons |
| `surface-raised` | `hsl(0 0% 100%)` | `hsl(0 0% 11%)` | Action panels, table containers, dropdown menus |

### Border

| Token | Light | Dark | Usage |
|---|---|---|---|
| `border` | `hsl(0 0% 89%)` | `hsl(0 0% 16%)` | Stronger borders (rarely used directly) |
| `border-subtle` | `hsl(0 0% 93%)` | `hsl(0 0% 13%)` | Table borders, input borders, dividers, list containers |

`border-border-subtle` is the default border for tables and dividers. It is **not** used on action panels or interactive surfaces — those use `shadow-sm` instead (see Section 3).

### Text

| Token | Usage |
|---|---|
| `text-foreground` | Primary content — names, values, labels |
| `text-text-secondary` | Supporting content — codes, secondary values, metadata |
| `text-text-tertiary` | Placeholder-level — empty states, column headers, section labels, icons at rest |

### Semantic

| Token | Usage |
|---|---|
| `accent` / `accent-foreground` | Primary CTA buttons only |
| `destructive` / `destructive-foreground` | Delete buttons, error messages, destructive hover states |
| `success` | Status indicators (e.g. open/active intakes) |
| `warning` | Caution states |

---

## 3. Surface Elevation Model

There are three distinct surface treatments. Apply them based on purpose, not aesthetics.

### Flat (no surface)
Static data display — fields, values, read-only content. No background, no border, no shadow.
Sections are separated by whitespace and `border-b border-border-subtle` dividers only.

```tsx
// Correct — static field display
<div className="py-1.5 grid grid-cols-[160px_1fr] gap-4">
  <span className="text-[11px] uppercase tracking-wider text-text-tertiary">Gender</span>
  <span className="text-sm text-foreground">Male</span>
</div>
```

### Raised with border (list containers, tables)
Used for list views and table containers. These are data containers, not interactive surfaces.

```tsx
<div className="rounded-lg border border-border-subtle bg-surface-raised overflow-hidden">
```

### Raised with shadow (action panels, interactive surfaces)
Used for panels that contain actions or entry forms — anything the user interacts with. **No border. Shadow only.**

```tsx
<div className="rounded-xl bg-surface-raised shadow-sm p-5">
```

> **Rule:** `bg-surface-raised` + `border` → list/table containers only.
> `bg-surface-raised` + `shadow-sm` → action panels and interactive surfaces.
> Never mix border and shadow on the same element.

---

## 4. Typography

No custom font is loaded. The system font stack is used throughout.

| Usage | Classes |
|---|---|
| Page / detail title | `text-lg font-semibold text-foreground` |
| Section label (detail page) | `text-[11px] font-medium uppercase tracking-wider text-text-tertiary` |
| Table column header | `text-[11px] font-medium uppercase tracking-wider text-text-tertiary` |
| Section sub-header (dept label inside table groups) | `text-xs font-medium uppercase tracking-wider text-text-tertiary` |
| Primary row content | `text-sm text-foreground` |
| Secondary row content | `text-xs text-text-secondary` |
| Tertiary / placeholder | `text-xs text-text-tertiary` |
| Student ID / metadata below title | `text-sm text-text-tertiary` |
| Monospace (codes, IDs) | `font-mono text-xs text-text-secondary` |
| Tabular numbers (counts) | `tabular-nums` added to the number span |

---

## 5. Spacing & Layout

### List page structure

Every list page view follows this shell:

```tsx
<div className="flex flex-col h-full">

  {/* Header — always border-b */}
  <div className="flex items-center justify-between px-6 py-4 border-b border-border-subtle">
    <h1 className="text-base font-semibold text-foreground">Page Title</h1>
    {canManage && <Button size="sm">New Thing</Button>}
  </div>

  {/* Optional error banner */}
  {error && (
    <div className="px-6 py-3 border-b border-border-subtle text-sm text-destructive">
      {error}
    </div>
  )}

  {/* Content */}
  <div className="flex-1 overflow-y-auto p-6">
    <div className="flex flex-col gap-4">
      {/* Toolbar, then table/list */}
    </div>
  </div>

</div>
```

The filter toolbar lives **inside** the `p-6` content wrapper — never in a separate bordered row.

### Detail page structure

Detail pages use `p-6 sm:p-8` padding and a `mr-auto max-w-4xl` content constraint. The breadcrumb spans full width above the columns.

```tsx
<div className="p-6 sm:p-8">
  <div className="mr-auto max-w-4xl">
    <nav>{/* breadcrumb */}</nav>

    {/* Three-slot grid — see Section 6 */}
    <div className="flex flex-col gap-8 lg:grid lg:grid-cols-[1fr_288px] lg:items-start">
      {/* Slot 1 — Identity */}
      {/* Slot 2 — Action sidebar */}
      {/* Slot 3 — Main content */}
    </div>
  </div>
</div>
```

### Content padding

| Area | Padding |
|---|---|
| Page header | `px-6 py-4` |
| Content wrapper | `p-6` |
| Detail page | `p-6 sm:p-8` |
| Table row | `px-4 py-2.5` |
| Table header row | `px-4 py-2` |
| Table section sub-header | `px-4 py-3` |
| Action panel | `p-5` |
| Inline entry form (raised) | `p-4` |

---

## 6. Detail Page Layout

Detail pages use a three-slot CSS grid layout. This achieves correct mobile ordering (identity → sidebar → content) without duplicating any JSX.

```tsx
{/*
  Mobile  (flex-col): identity → sidebar → content  (DOM order)
  Desktop (lg:grid):  identity (col 1, row 1) | sidebar (col 2, rows 1–2)
                      content  (col 1, row 2) |
*/}
<div className="flex flex-col gap-8 lg:grid lg:grid-cols-[1fr_288px] lg:items-start">

  {/* Slot 1 — Identity: name, ID, primary action (Edit) */}
  <div className="lg:col-start-1 lg:row-start-1">
    <div className="flex items-start justify-between gap-4">
      <div>
        <h1 className="text-lg font-semibold text-foreground">{item.name}</h1>
        <p className="mt-0.5 text-sm text-text-tertiary">#{item.reference_id}</p>
      </div>
      {canManage && <Button variant="outline" size="sm">Edit</Button>}
    </div>
  </div>

  {/* Slot 2 — Action sidebar: raised surface, shadow, sticky on desktop */}
  <div className="lg:col-start-2 lg:row-start-1 lg:row-span-2 lg:sticky lg:top-6">
    <div className="rounded-xl bg-surface-raised shadow-sm p-5">
      {/* Panel content */}
    </div>
  </div>

  {/* Slot 3 — Main content: flat sections, no background */}
  <div className="lg:col-start-1 lg:row-start-2">
    {/* Sections divided by spacing + border-b */}
  </div>

</div>
```

**Rules:**
- Slot 1 and Slot 3 are always flat — no `bg-surface-raised`, no border, no shadow
- Slot 2 is the only raised element on a detail page
- `lg:row-span-2` on the sidebar lets it span the full height of Slot 1 + Slot 3 on desktop
- Never use `order-` utilities or duplicate JSX to control responsive ordering — use DOM order + grid placement

### Detail page section labels

Each section within Slot 3 opens with a label:

```tsx
<p className="mb-3 text-[11px] font-medium uppercase tracking-wider text-text-tertiary">
  Personal Info
</p>
```

Sections are separated by:
```tsx
<div className="my-8 border-b border-border-subtle" />
```

### Detail row (field/value pair)

```tsx
<div className="grid grid-cols-[160px_1fr] gap-4 py-1.5">
  <span className="text-[11px] text-text-tertiary uppercase tracking-wider font-medium self-start pt-px">
    {label}
  </span>
  <span className="text-sm text-foreground break-words">{value || "—"}</span>
</div>
```

No borders between rows. Spacing only.

### Destructive zone

The Delete action lives at the bottom of Slot 3, after a final divider. It is never in the page header or sidebar.

```tsx
{canManage && (
  <>
    <div className="my-8 border-b border-border-subtle" />
    <div className="flex items-start justify-between gap-4">
      <div>
        <p className="text-sm font-medium text-foreground">Delete this {entity}</p>
        <p className="mt-0.5 text-xs text-text-tertiary">
          Permanently removes this record. This action cannot be undone.
        </p>
      </div>
      <Button
        variant="outline"
        size="sm"
        onClick={() => setShowDelete(true)}
        className="shrink-0 border-border-subtle text-destructive hover:bg-destructive/5 hover:text-destructive"
      >
        <Trash2 className="w-3.5 h-3.5" />
        Delete
      </Button>
    </div>
  </>
)}
```

### Inline entry forms on detail pages

When an entry form appears inline (e.g. adding an external ID), it renders on a raised surface:

```tsx
<div className="mt-4 rounded-xl bg-surface-raised shadow-sm p-4">
  {/* form controls */}
</div>
```

### Destructive confirmation — typed confirm

For irreversible destructive actions on detail pages, the confirmation dialog requires the user to type a word to proceed. Pass `requireConfirmText` to `ConfirmDeleteDialog`:

```tsx
<ConfirmDeleteDialog
  requireConfirmText="delete"
  ...
/>
```

---

## 7. Buttons

Defined in `src/components/ui/button-variants.ts`. Always use these — do not create ad-hoc button styles.

### Variants

| Variant | Class | When to use |
|---|---|---|
| `default` | `bg-accent text-accent-foreground hover:bg-accent/85` | Primary CTA — one per page header |
| `destructive` | `bg-destructive text-destructive-foreground` | Destructive confirmation dialogs only |
| `outline` | `border border-border-subtle bg-background hover:bg-surface` | Secondary actions — Edit, back links, secondary CTAs |
| `secondary` | `bg-surface hover:bg-surface-raised` | Alternative low-emphasis actions |
| `ghost` | `bg-transparent hover:bg-surface` | Low-emphasis, icon buttons in non-table contexts |

### Sizes

| Size | Height | Use |
|---|---|---|
| `sm` | `h-8` | Page header actions, detail page actions |
| `default` | `h-9` | Forms, dialogs |
| `lg` | `h-10` | Rarely used |
| `icon` | `h-9 w-9` | Square icon-only buttons |

### Bare button classes (not shadcn `<Button>`)

Use these when `<Button>` is too heavy:

```ts
// Icon-only action in table rows, inline controls
export const iconButtonClass =
  "p-1.5 rounded text-text-tertiary hover:text-foreground hover:bg-surface transition-colors cursor-pointer";

// Text navigation — breadcrumbs, back links
export const navButtonClass =
  "text-text-tertiary hover:text-foreground transition-colors cursor-pointer";
```

---

## 8. Form Controls

### Input

```tsx
<Input className="border-border-subtle bg-background" />
```

Always override the default `border-input` with `border-border-subtle`. Always ensure `bg-background` so the input doesn't inherit surface colour from a `bg-surface-raised` container.

Default height is `h-10`. Inline table inputs use `h-7` with `text-sm`.

### Select (dropdown)

```tsx
<Select>
  <SelectTrigger className="w-40 h-9 text-sm border-border-subtle bg-background focus:ring-0 focus:ring-offset-0">
    <SelectValue placeholder="All items" />
  </SelectTrigger>
  <SelectContent className="bg-surface-raised border-border-subtle shadow-sm">
    <SelectItem value="all">All items</SelectItem>
    ...
  </SelectContent>
</Select>
```

Key overrides:
- `border-border-subtle` replaces the default `border-input`
- `bg-background` prevents surface colour inheritance
- `focus:ring-0 focus:ring-offset-0` removes the focus ring on the trigger (looks heavy on a toolbar)
- `bg-surface-raised` on content — same as card background
- `shadow-sm` replaces `shadow-md` — lighter elevation

Standard widths: `w-40` for status/level filters, `w-44` for department/program filters.

### Filter toolbar pattern

Every list page toolbar follows the same structure:

```tsx
<div className="flex flex-wrap items-center gap-3">
  <Input className="max-w-xs border-border-subtle bg-background" placeholder="Search…" />
  <Select>…</Select>
  <Select>…</Select>
  {hasFilters && (
    <button
      type="button"
      onClick={clearFilters}
      className="text-xs text-text-tertiary hover:text-foreground transition-colors cursor-pointer"
    >
      Clear filters
    </button>
  )}
</div>
```

- `flex-wrap` allows filters to wrap on narrow screens
- `gap-3` between all filter elements
- Clear filters button only appears when at least one filter is active

---

## 9. Tables & List Views

All list views use a CSS-grid-based layout (not `<table>`), except `StudentGrid` which uses TanStack Table for server-side pagination.

### Container

```tsx
<div className="rounded-lg border border-border-subtle bg-surface-raised overflow-hidden">
```

> Table containers use `border`, not `shadow-sm` — they are data containers, not interactive surfaces.

### Column headers row

```tsx
<div className={cn("hidden md:grid gap-4 px-4 py-2 border-b border-border-subtle", colClass)}>
  <span className="text-[11px] font-medium uppercase tracking-wider text-text-tertiary">Name</span>
  ...
</div>
```

Column headers are **hidden on mobile** (`hidden md:grid`). They are never shown on small screens.

### Data rows

```tsx
<div className={cn("grid gap-4 items-center px-4 py-2.5 border-b border-border-subtle last:border-0 group", colClass)}>
  ...
</div>
```

For clickable rows:
```tsx
className={cn("... cursor-pointer hover:bg-surface transition-colors", colClass)}
onClick={() => navigate(`/o/${orgSlug}/resource/${id}`)}
```

### Responsive grid pattern

All tables collapse to a 2-column layout on small screens:

```ts
// sm (default): Primary content (1fr) | Actions (72px)
// md+:          Full column layout
const COL = "grid-cols-[1fr_72px] md:grid-cols-[1fr_100px_120px_72px]";
```

Secondary data hidden on mobile is collapsed into a **sub-line** below the primary label:

```tsx
<div className="min-w-0">
  <span className="text-sm text-foreground">{item.name}</span>
  <p className="md:hidden flex items-center gap-1.5 mt-0.5">
    {item.code && (
      <>
        <span className="text-xs text-text-secondary font-mono">{item.code}</span>
        <span className="w-1 h-1 rounded-full bg-text-tertiary shrink-0" />
      </>
    )}
    <span className="text-xs text-text-tertiary">{item.count} programs</span>
  </p>
</div>
```

### Separator between sub-line items

Use a filled circle, **not** a text character like `·` or `—`:

```tsx
<span className="w-1 h-1 rounded-full bg-text-tertiary shrink-0" />
```

### Hover-reveal actions

```tsx
{/* Row must have className including "group" */}
<div className="opacity-0 group-hover:opacity-100 transition-opacity flex items-center gap-1">
  <button className={iconButtonClass} onClick={() => onEdit(item)} title="Edit">
    <Pencil className="w-3.5 h-3.5" />
  </button>
  <button
    className={cn(iconButtonClass, "hover:text-destructive hover:bg-destructive/5")}
    onClick={() => onDelete(item)}
    title="Delete"
  >
    <Trash2 className="w-3.5 h-3.5" />
  </button>
</div>
```

On mobile, actions column is hidden entirely (`hidden md:flex`). Navigation happens via row click.

When an action inside a clickable row must not trigger navigation:
```tsx
onClick={(e) => e.stopPropagation()}
```

### Skeleton loading

Skeleton rows mirror the exact grid of real rows:

```tsx
<div className={cn("grid gap-4 items-center px-4 py-3 border-b border-border-subtle", COL)}>
  <div className="space-y-1.5">
    <div className="h-3.5 w-3/5 rounded bg-border-subtle animate-pulse" />
    <div className="h-2.5 w-2/5 rounded bg-border-subtle animate-pulse md:hidden" />
  </div>
  <div className="hidden md:block h-3.5 w-4/5 rounded bg-border-subtle animate-pulse" />
  <div className="h-3.5 w-8 rounded bg-border-subtle animate-pulse ml-auto" />
  {canManage && <div className="hidden md:block" />}
</div>
```

Render 6 skeleton rows by default.

### Grouped tables

Each group is a self-contained `bg-surface-raised` card with border:

```tsx
<div className="rounded-lg border border-border-subtle overflow-hidden bg-surface-raised">
  <div className="px-4 py-3 border-b border-border-subtle">
    <h2 className="text-xs font-medium uppercase tracking-wider text-text-tertiary">
      {groupLabel}
    </h2>
  </div>
  ...
</div>
```

Groups stack with `flex flex-col gap-4`.

---

## 10. Empty States

All empty states follow the same pattern — a dashed-border card with a centered `<button>`:

```tsx
<div className="rounded-lg border border-border-subtle border-dashed bg-surface-raised">
  <button
    type="button"
    disabled={!canManage}
    onClick={handleCreate}
    className={cn(
      "w-full flex flex-col items-center justify-center gap-2 py-14 text-center transition-colors rounded-lg",
      canManage
        ? "text-text-tertiary hover:text-text-secondary hover:bg-surface cursor-pointer"
        : "text-text-tertiary cursor-default"
    )}
  >
    <PlusCircle className="w-8 h-8 opacity-40" />
    <span className="text-sm font-medium">No items yet</span>
    {canManage && (
      <span className="text-xs text-text-tertiary">
        Click to create the first item
      </span>
    )}
  </button>
</div>
```

Rules:
- Always use `<button>`, never `<div>`
- Always use `PlusCircle` from lucide-react, `w-8 h-8 opacity-40`
- `disabled={!canManage}` — non-managers see it but cannot interact
- `py-14` vertical padding — do not reduce this
- Sub-hint ("Click to…") only shown when `canManage` is true

When filters are active: `"No items match the selected filters"` instead of `"No items yet"`.

---

## 11. Badges & Status Indicators

### Metadata chip (read-only label)

```tsx
const chip =
  "text-[10px] uppercase tracking-wide border border-border-subtle rounded px-1.5 py-0.5 text-text-secondary whitespace-nowrap";
```

### Status badge

Status badges are interactive dropdowns (e.g. intake status). Use the shared `IntakeStatusBadge` component. Do not create raw styled `<span>` elements for statuses.

### "Default" badge

Do not display `is_default` badges on list items. The flag is used only to suppress the delete button — it is not shown to users.

---

## 12. Navigation & Breadcrumbs

Breadcrumbs appear on all detail and edit pages. Intermediate segments are clickable links; only the current (last) segment is a plain span.

```tsx
<nav className="flex items-center gap-1.5 text-xs text-text-tertiary mb-6">
  <button onClick={() => navigate(listPath)} className={navButtonClass}>
    Section
  </button>
  <ChevronRight className="w-3 h-3" />
  {/* Intermediate segment — clickable if not the current page */}
  <button onClick={() => navigate(detailPath)} className={navButtonClass}>
    {item.name}
  </button>
  <ChevronRight className="w-3 h-3" />
  {/* Current page — plain span */}
  <span className="text-foreground">Edit</span>
</nav>
```

- `navButtonClass` for all clickable segments
- `ChevronRight w-3 h-3` separator
- Current page is always a plain `text-foreground` span, never a link
- Name segments truncate at `max-w-[200px]`

---

## 13. Page-level Filters: URL vs Local State

| Filter type | State location | Reason |
|---|---|---|
| Department filter on Programs page | URL search param (`?department=id`) | Shareable, bookmarkable, linked from other pages |
| Level filter on Programs page | Local component state | Not shareable; reset on navigation is acceptable |
| Search input | Local state (debounced for server-side) | Fast UX; no need to persist in URL |
| Program/Status filter on Intakes page | Local state → API params | Server-side filtering; not shareable |

When a filter is URL-based, use `useSearchParams` + `useNavigate` with `{ replace: true }` to avoid polluting browser history.

---

## 14. Server-side Search Safety Nets

Every search input that triggers a backend API call must implement:

1. **Debounce** — 300 ms delay between the last keystroke and the request
2. **AbortController** — cancel the previous in-flight request before firing a new one; silently drop `axios.isCancel(e)` errors
3. **Minimum character threshold** — only send `?search=` when the trimmed value is ≥ 2 characters; send nothing (full list) if the field is empty

See `production_readiness.md` for full implementation reference.

---

## 15. Icons

All icons are from `lucide-react`. Standard sizes:

| Context | Size class |
|---|---|
| Table row action buttons | `w-3.5 h-3.5` |
| Empty state illustration | `w-8 h-8 opacity-40` |
| Page header / detail page button icon | `w-3.5 h-3.5` |
| Breadcrumb separator | `w-3 h-3` |
| Navigation / UI chrome | `w-4 h-4` |

Common icon → use mapping:

| Icon | Use |
|---|---|
| `PlusCircle` | Create CTA, empty state |
| `Pencil` | Edit action |
| `Trash2` | Delete action |
| `ChevronRight` | Breadcrumb separator |
| `ChevronLeft` / `ChevronRight` | Pagination |
| `Check` | Confirm inline edit |
| `X` | Cancel inline edit |
| `Loader2` | Saving spinner (`animate-spin`) |

---

## 16. Responsive Breakpoints

| Breakpoint | Tailwind prefix | Behaviour |
|---|---|---|
| < 768px | (default) | Mobile — 2-column tables, sub-lines, no column headers, no hover actions |
| ≥ 768px | `md:` | Full table columns, column headers, hover-reveal actions |
| ≥ 1024px | `lg:` | Detail page two-column grid layout |

`sm:` (640px) is used sparingly for padding adjustments (`p-6 sm:p-8`). Default styles target mobile; `md:` targets desktop tables; `lg:` targets detail page layout switches.

Never use CSS `order-` utilities or duplicate JSX to achieve responsive reordering on detail pages. Use DOM order + CSS grid placement (see Section 6).

---

*Last updated: April 2026*
