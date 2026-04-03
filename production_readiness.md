# Production Readiness

A living document capturing hardening decisions, safety nets, and recommended improvements as the application grows toward production. Each section covers a specific concern area with what has been implemented and what is recommended but not yet done.

---

## Search & Query Endpoint Protection

### Context

Any feature that fires API requests in response to user input (search boxes, filter dropdowns) is a potential source of backend abuse and poor UX if left unguarded. A single unprotected search input can generate hundreds of redundant requests per minute, return stale data, or allow unbounded queries.

The following safety nets were designed and implemented during the intake search feature (April 2026) and should be applied to every server-side search across the application.

---

### Safety Nets — Implemented

#### 1. Debouncing

**What it is:** Wait until the user stops typing for N milliseconds before firing the request. A held key fires one request when released, not one per keypress.

**Implementation:** A `useEffect` with a `setTimeout` of **300 ms** resets on every `searchQuery` change. The resolved value (`debouncedSearch`) is what `loadIntakes` depends on.

```ts
useEffect(() => {
  const t = setTimeout(() => setDebouncedSearch(searchQuery), 300);
  return () => clearTimeout(t);
}, [searchQuery]);
```

**Applies to:** `IntakesPage` (intake name search). Apply the same pattern to any future server-side search input.

---

#### 2. Request Cancellation

**What it is:** Cancel the previous in-flight request before firing a new one. Prevents stale responses from overwriting fresh ones (race condition), and avoids wasted bandwidth from superseded requests.

**Implementation:** A `useRef<AbortController>` stores the current controller. At the top of every fetch, the previous controller is aborted and a new one is created. Axios receives the `signal`. Cancelled errors are silently dropped in the `catch` block.

```ts
const abortRef = useRef<AbortController | null>(null);

// Inside loadIntakes:
abortRef.current?.abort();
const controller = new AbortController();
abortRef.current = controller;

// Pass to Axios:
const data = await intakesApi.listFlat(orgSlug, params, controller.signal);

// In catch:
if (axios.isCancel(e)) return;
```

**API layer:** The `signal` is threaded through the API function into the Axios config:

```ts
async listFlat(orgSlug, params?, signal?) {
  const res = await http.get(flatBase(orgSlug), { params, signal });
  return res.data;
}
```

**Applies to:** `IntakesPage`. Apply to any hook or fetch function that can be superseded by a new call.

---

#### 3. Minimum Character Threshold

**What it is:** Do not send a search query until the user has typed at least 2 characters. A single character triggers extremely broad queries (potentially matching most records) and is rarely a meaningful search intent.

**Frontend implementation:** The `search` param is only appended to the request if `trimmed.length >= 2`. If the field is empty, no search param is sent and the full list is returned normally.

```ts
const trimmed = debouncedSearch.trim();
if (trimmed.length >= 2) params.search = trimmed;
```

**Backend implementation (defence in depth):** The backend mirrors the same rule and also silently truncates inputs longer than 200 characters, regardless of the client:

```python
search = self.request.query_params.get('search', '').strip()[:200]
if len(search) >= 2:
    qs = qs.filter(name__icontains=search)
```

The backend check is not redundant — it protects against non-browser clients (scripts, curl, other API consumers) that skip the frontend threshold entirely.

**Applies to:** `IntakesPage` (frontend + backend). Apply the same `[:200]` / `len >= 2` guard to every search param in every view.

---

### Recommendations — Not Yet Implemented

The following practices are industry-standard for production API endpoints. They have not been implemented yet because they affect the whole application or require infrastructure decisions. They are recorded here for future sprints.

---

#### 4. Backend Rate Limiting / Throttling

**What it is:** Limit the number of requests a user (or anonymous IP) can make to a given endpoint per time window. Prevents abuse, scraping, and accidental hammering from runaway clients.

**How to implement (Django DRF):**

Add to `settings.py`:
```python
REST_FRAMEWORK = {
    ...
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'user': '120/min',
    },
}
```

Or apply per-view for finer control:
```python
from rest_framework.throttling import UserRateThrottle

class SearchThrottle(UserRateThrottle):
    rate = '60/min'

class IntakeFlatListCreateView(IntakeFlatOrgScopedMixin, ListCreateAPIView):
    throttle_classes = [SearchThrottle]
```

**Priority:** High. Should be added before any public-facing or unauthenticated endpoints are exposed.

---

#### 5. Pagination and Result Size Cap

**What it is:** Return at most N rows per response. Without a limit, a search that matches 10,000 records returns all 10,000 in a single JSON payload — a cheap way to slow down both the server and the client.

**How to implement (Django DRF):**
```python
from rest_framework.pagination import PageNumberPagination

class StandardPagination(PageNumberPagination):
    page_size = 50
    page_size_query_param = 'page_size'
    max_page_size = 200
```

Apply globally in `settings.py` or per-view via `pagination_class`.

**Note:** Adding pagination to existing endpoints is a breaking API change. Plan this alongside frontend updates to handle paginated responses.

**Priority:** Medium. Critical before any list endpoint is expected to hold more than a few hundred records.

---

#### 6. Database Index for Text Search (Postgres)

**What it is:** `icontains` translates to a `LIKE '%term%'` query, which performs a full table scan. For small datasets this is fine. At scale (thousands of records), it becomes slow.

**How to implement:** Use Postgres trigram indexes via `django.contrib.postgres`:

```python
# In the migration:
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.operations import TrigramExtension

class Migration(migrations.Migration):
    operations = [
        TrigramExtension(),
        migrations.AddIndex(
            model_name='intake',
            index=GinIndex(
                fields=['name'],
                name='intake_name_trgm_idx',
                opclasses=['gin_trgm_ops'],
            ),
        ),
    ]
```

This makes `icontains` index-backed, reducing search query time from O(n) to O(log n).

**Priority:** Low until record counts grow. Revisit when any searchable model exceeds ~5,000 rows.

---

#### 7. Frontend Search Result Caching

**What it is:** Memoize recent `(search, filters)` → results combinations in a client-side `Map` ref. A cache hit returns instantly without a network round-trip. Particularly useful when users backspace and retype the same query.

**Sketch:**
```ts
const cacheRef = useRef(new Map<string, Intake[]>());

// In loadIntakes:
const cacheKey = JSON.stringify({ search: trimmed, program: programFilter, status: statusFilter });
if (cacheRef.current.has(cacheKey)) {
  setIntakes(cacheRef.current.get(cacheKey)!);
  setLoadStatus("success");
  return;
}
// ... fetch, then:
cacheRef.current.set(cacheKey, data);
```

Clear the cache after any mutation (create, update, delete).

**Priority:** Low. Worth implementing if users report search feeling slow or if API latency is consistently above 200 ms.

---

### Summary Table

| # | Practice | Scope | Status |
|---|---|---|---|
| 1 | Debouncing (300 ms) | Frontend | ✅ Implemented |
| 2 | Request cancellation (AbortController) | Frontend + API layer | ✅ Implemented |
| 3 | Minimum 2-character threshold | Frontend + Backend | ✅ Implemented |
| 4 | Rate limiting / throttling | Backend (DRF) | ⬜ Recommended |
| 5 | Pagination and result size cap | Backend + Frontend | ⬜ Recommended |
| 6 | Trigram index for text search | Backend (Postgres) | ⬜ Recommended |
| 7 | Frontend result caching | Frontend | ⬜ Recommended |

---

*Last updated: April 2026 — initial entry covering intake search hardening.*
