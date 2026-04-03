# Design Language — Zavia

A reference for every visual and interaction pattern used across the application. Follow these guidelines precisely when building new UI. Deviating from them — even for convenience — creates visual inconsistency that accumulates into a broken-looking product.

---

## 1. Design Philosophy

The UI is deliberately neutral, dense, and text-forward. It is built for people who use it every day, not for first-time visitors. Key principles:

- **Quiet by default** — no decorative elements, no gradient fills, no bold colour except accent and destructive
- **Information density over whitespace** — rows are compact; padding is measured, not generous
- **Progressive disclosure** — secondary actions (edit, delete) appear on hover, not always visible
- **Responsive and functional** — tables collapse to 2 columns on small screens; detail moves under the primary label
- **Consistent structure** — every list page, table, and empty state follows the same structural template

---

## 2. Colour Tokens

All colours are defined as CSS variables in `src/index.css` and mapped into Tailwind via `@theme inline`. Never use raw hex or Tailwind palette colours (e.g. `gray-500`) — always use these semantic tokens.

### Surface hierarchy

| Token | Light | Dark | Usage |
|---|---|---|---|
| `background` | `hsl(0 0% 99%)` | `hsl(0 0% 6%)` | Page background, input backgrounds |
| `surface` | `hsl(0 0% 97%)` | `hsl(0 0% 9%)` | Hover states on rows and buttons |
| `surface-raised` | `hsl(0 0% 100%)` | `hsl(0 0% 11%)` | Cards, table containers, dropdown menus |

### Border

| Token | Light | Dark | Usage |
|---|---|---|---|
| `border` | `hsl(0 0% 89%)` | `hsl(0 0% 16%)` | Stronger borders (rarely used directly) |
| `border-subtle` | `hsl(0 0% 93%)` | `hsl(0 0% 13%)` | All table borders, card outlines, input borders, dividers |

`border-border-subtle` is the default border for **everything**. `border-border` is used sparingly for emphasis.

### Text

| Token | Usage |
|---|---|
| `text-foreground` | Primary content — names, values, labels |
| `text-text-secondary` | Supporting content — codes, secondary values, metadata |
| `text-text-tertiary` | Placeholder-level — empty states, column headers, labels, icons at rest |

### Semantic

| Token | Usage |
|---|---|
| `accent` / `accent-foreground` | Primary CTA buttons only |
| `destructive` / `destructive-foreground` | Delete buttons, error messages, destructive hover states |
| `success` | Status indicators (e.g. open/active intakes) |
| `warning` | Caution states |

---

## 3. Typography

No custom font is loaded. The system font stack is used throughout.

| Usage | Classes |
|---|---|
| Page title | `text-base font-semibold text-foreground` |
| Section heading | `text-sm font-medium text-text-secondary` |
| Table column header | `text-[11px] font-medium uppercase tracking-wider text-text-tertiary` |
| Section sub-header (dept label inside table groups) | `text-xs font-medium uppercase tracking-wider text-text-tertiary` |
| Primary row content | `text-sm text-foreground` |
| Secondary row content | `text-xs text-text-secondary` |
| Tertiary / placeholder | `text-xs text-text-tertiary` |
| Monospace (codes, enrolment numbers) | `font-mono text-xs text-text-secondary` |
| Tabular numbers (counts, enrollment) | `tabular-nums` added to the number span |

---

## 4. Spacing & Layout

### Page structure

Every full-page view follows this shell:

```tsx
<div className="flex flex-col h-full">

  {/* Header — always border-b */}
  <div className="flex items-center justify-between px-6 py-4 border-b border-border-subtle">
    <h1 className="text-base font-semibold text-foreground">Page Title</h1>
    {canManage && <Button size="sm">New Thing</Button>}
  </div>

  {/* Optional error banner — border-b, destructive text */}
  {error && (
    <div className="px-6 py-3 border-b border-border-subtle text-sm text-destructive">
      {error}
    </div>
  )}

  {/* Content — flex-1 so it fills remaining height, scrolls if needed */}
  <div className="flex-1 overflow-y-auto p-6">
    <div className="flex flex-col gap-4">
      {/* Toolbar, then table/list */}
    </div>
  </div>

</div>
```

**Critical:** The filter toolbar lives **inside** the `p-6` content wrapper, not in a separate bordered row between the header and content. There is no `border-b` between the header and the table — there is one `border-b` on the page header and then white space before the table.

### Content padding

| Area | Padding |
|---|---|
| Page header | `px-6 py-4` |
| Content wrapper | `p-6` |
| Table row | `px-4 py-2.5` |
| Table header row | `px-4 py-2` |
| Table section sub-header | `px-4 py-3` |
| Card / detail info block | `px-4` (rows use `py-2.5` internally) |

---

## 5. Buttons

Defined in `src/components/ui/button-variants.ts`. Always use these — do not create ad-hoc button styles.

### Variants

| Variant | Class | When to use |
|---|---|---|
| `default` | `bg-accent text-accent-foreground hover:bg-accent/85` | Primary CTA — one per page header |
| `destructive` | `bg-destructive text-destructive-foreground` | Destructive confirmation dialogs only |
| `outline` | `border border-border-subtle bg-background hover:bg-surface` | Secondary actions alongside a primary CTA (e.g. Edit next to Delete in a detail page header) |
| `secondary` | `bg-surface hover:bg-surface-raised` | Alternative low-emphasis actions |
| `ghost` | `bg-transparent hover:bg-surface` | Low-emphasis, icon buttons in non-table contexts |

### Sizes

| Size | Height | Use |
|---|---|---|
| `sm` | `h-8` | Page header actions ("New Program", "Add intake") |
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

## 6. Form Controls

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

## 7. Tables & List Views

All list views use a CSS-grid-based layout (not `<table>`), except `StudentGrid` which uses TanStack Table for server-side pagination.

### Container

```tsx
<div className="rounded-lg border border-border-subtle bg-surface-raised overflow-hidden">
```

### Column headers row

```tsx
<div className={cn("hidden md:grid gap-4 px-4 py-2 border-b border-border-subtle", colClass)}>
  <span className="text-[11px] font-medium uppercase tracking-wider text-text-tertiary">Name</span>
  ...
</div>
```

Column headers are **hidden on mobile** (`hidden md:grid`). They are never shown on small screens — the data speaks for itself.

### Data rows

```tsx
<div className={cn("grid gap-4 items-center px-4 py-2.5 border-b border-border-subtle last:border-0 group", colClass)}>
  ...
</div>
```

For clickable rows (navigate on click), add:
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

Secondary data (codes, counts, dates) that is hidden as a column on mobile is collapsed into a **sub-line** below the primary label in the first cell:

```tsx
<div className="min-w-0">
  <span className="text-sm text-foreground">{item.name}</span>
  {/* Sub-line — mobile only */}
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

Hidden desktop columns use `hidden md:block` (or `hidden md:flex`).

### Separator between sub-line items

Use a filled circle, **not** a text character like `·` or `—`:

```tsx
<span className="w-1 h-1 rounded-full bg-text-tertiary shrink-0" />
```

This renders as a solid 4px dot, visually clear at small sizes.

### Hover-reveal actions

Actions (edit, delete) are hidden at rest and revealed on row hover:

```tsx
{/* On the row: className includes "group" */}
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

On mobile (touch), hover does not fire — actions column is hidden entirely (`hidden md:flex`). Navigation happens via row click.

When an action inside a clickable row must not trigger navigation, wrap it with:
```tsx
onClick={(e) => e.stopPropagation()}
```

### Skeleton loading

Skeleton rows mirror the exact grid of real rows. Two-line cells get two skeleton bars:

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

Render 6 skeleton rows by default for list views.

### Grouped tables (departments, courses)

When data is grouped (e.g. courses by department), each group is a self-contained `bg-surface-raised` card:

```tsx
<div className="rounded-lg border border-border-subtle overflow-hidden bg-surface-raised">
  {/* Group header */}
  <div className="px-4 py-3 border-b border-border-subtle">
    <h2 className="text-xs font-medium uppercase tracking-wider text-text-tertiary">
      {groupLabel}
    </h2>
  </div>
  {/* Column headers — desktop only */}
  <div className={cn("hidden md:grid gap-4 px-4 py-2 border-b border-border-subtle", COL)}>
    ...
  </div>
  {/* Rows */}
  ...
</div>
```

Groups stack with `flex flex-col gap-4`.

---

## 8. Empty States

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
- Always use `<button>`, never `<div>` — even when not yet wired to an action
- Always use `PlusCircle` from lucide-react, `w-8 h-8 opacity-40`
- `disabled={!canManage}` — non-managers see it but cannot interact
- `py-14` vertical padding — do not reduce this
- Sub-hint ("Click to…") only shown when `canManage` is true

When a list page has an empty state inside a filter-applied context, the message changes:

```tsx
{hasFilters ? "No items match the selected filters" : "No items yet"}
```

---

## 9. Badges & Status Indicators

### Metadata chip (read-only label)

Used for program level, program type, and similar categorical metadata:

```tsx
const chip =
  "text-[10px] uppercase tracking-wide border border-border-subtle rounded px-1.5 py-0.5 text-text-secondary whitespace-nowrap";
```

### Status badge

Status badges are interactive dropdowns (e.g. intake status). They use a custom `IntakeStatusBadge` component. Do not create raw styled `<span>` elements for statuses — use the shared badge component.

### "Default" badge

Do not use "Default" badges on list items. They add visual noise without actionable information. The `is_default` flag is used only to suppress the delete button on default departments — it is not displayed.

---

## 10. Navigation & Breadcrumbs

Breadcrumbs appear in detail page headers:

```tsx
<nav className="flex items-center gap-1.5 text-xs text-text-tertiary mb-3">
  <Link to={parentPath} className={navButtonClass}>
    Parent Section
  </Link>
  <ChevronRight className="w-3 h-3" />
  <span className="text-foreground truncate max-w-[240px]">{item.name}</span>
</nav>
```

- `navButtonClass` for the parent link
- `ChevronRight w-3 h-3` separator
- Current page shown as plain `text-foreground` span, not a link

---

## 11. Page-level Filters: URL vs Local State

| Filter type | State location | Reason |
|---|---|---|
| Department filter on Programs page | URL search param (`?department=id`) | Shareable, bookmarkable, linked from other pages (e.g. department program count) |
| Level filter on Programs page | Local component state | Not shareable; reset on navigation is acceptable |
| Search input | Local state (debounced for server-side) | Fast UX; no need to persist in URL |
| Program/Status filter on Intakes page | Local state → API params | Server-side filtering; not shareable |

When a filter is URL-based, use `useSearchParams` + `useNavigate` with `{ replace: true }` to avoid polluting browser history.

---

## 12. Server-side Search Safety Nets

Every search input that triggers a backend API call must implement:

1. **Debounce** — 300 ms delay between the last keystroke and the request
2. **AbortController** — cancel the previous in-flight request before firing a new one; silently drop `axios.isCancel(e)` errors
3. **Minimum character threshold** — only send `?search=` when the trimmed value is ≥ 2 characters; send nothing (full list) if the field is empty

See `production_readiness.md` for full implementation reference.

---

## 13. Icons

All icons are from `lucide-react`. Standard sizes:

| Context | Size class |
|---|---|
| Table row action buttons | `w-3.5 h-3.5` |
| Empty state illustration | `w-8 h-8 opacity-40` |
| Page header button icon | `w-3.5 h-3.5` |
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

## 14. Responsive Breakpoints

The application uses two effective breakpoints:

| Breakpoint | Tailwind prefix | Behaviour |
|---|---|---|
| < 768px | (default) | Mobile — 2-column tables, sub-lines, no column headers, no hover actions |
| ≥ 768px | `md:` | Desktop — full table, column headers, hover-reveal actions |

`sm:` (640px) is used sparingly. Default styles target mobile, `md:` targets desktop. This is the standard Tailwind mobile-first approach.

---

*Last updated: April 2026*
