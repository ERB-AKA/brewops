# Plan: Dashboard date-range filter (ticket 005)

Implements `tickets/005-date-filter.md`. Lets people pick a start/end date and
have the dashboard's stat tiles, per-drink bars, and timeline follow that
range.

**Scope decision:** only the dashboard summary (`GET /api/stats`) is
filtered. The per-machine health cards (`GET /api/machines/{id}`, used for
the "Machines" panel) stay lifetime totals — they answer "is this machine
healthy overall", not "what happened last week". Do not touch
`get_machine_health` or its route.

**Granularity decision:** whole days, inclusive on both ends. Dates are
plain `YYYY-MM-DD` strings, no time-of-day. This matches how timestamps are
stored (`'YYYY-MM-DD HH:MM:SS'`, naive local time, per the docstring at the
top of `src/brewops/db/schema.py`), so filtering is a string comparison on
`DATE(timestamp)` — no timezone math needed.

**Default when no filter is set:** all time (current behavior, unchanged).

---

## 1. `src/brewops/db/queries.py` — `get_stats`

Current signature (line 68):

```python
def get_stats(conn: sqlite3.Connection) -> dict[str, Any]:
```

Change to:

```python
def get_stats(
    conn: sqlite3.Connection,
    start: str | None = None,
    end: str | None = None,
) -> dict[str, Any]:
```

`start` and `end` are `YYYY-MM-DD` strings or `None`. `None` means
unbounded on that side (matches ticket: "everyone" wants ranges, not just
two-sided ones — a user picking only a start date should get "since then").

Build the filter once at the top of the function:

```python
where = []
params: list[str] = []
if start is not None:
    where.append("DATE(timestamp) >= ?")
    params.append(start)
if end is not None:
    where.append("DATE(timestamp) <= ?")
    params.append(end)
where_clause = f"WHERE {' AND '.join(where)}" if where else ""
```

Apply it to all three sub-queries:

1. **Total** (line 70): add `where_clause` after `FROM brew_events`, pass
   `params`.
2. **Per-drink** (lines 71-82): the filter must go **inside the `ON`
   clause of the `LEFT JOIN`**, not as a `WHERE`. If you put it in `WHERE`
   instead, drink types with zero brews in the selected range get dropped
   from the result entirely (a plain `WHERE` on a left-joined column turns
   the outer join back into an inner join). Rewrite the join condition as:

   ```sql
   LEFT JOIN brew_events be
     ON be.drink_type = dt.name
     AND DATE(be.timestamp) >= ?   -- only if start is not None
     AND DATE(be.timestamp) <= ?   -- only if end is not None
   ```

   Build this conditionally the same way as `where_clause` above, but as
   an `ON`-clause fragment, and pass the same `params` list before the
   query executes.
3. **Per-day** (lines 83-93): add `where_clause` after `FROM brew_events`,
   pass `params`. `GROUP BY DATE(timestamp)` and `ORDER BY day` stay as is.

Return value shape is unchanged: `{"total_brews": ..., "per_drink": ...,
"per_day": ...}`.

### Edge cases to handle in this function

- **`start` after `end`**: don't validate here — reject earlier, in the API
  layer (see §2), so `queries.py` can assume a sane range.
- **No brews in range**: `total_brews` is `0`, `per_day` is `[]`, and
  `per_drink` is a full list of all drink types each with `count: 0` (this
  is exactly why the filter must live in the `ON` clause, not `WHERE` —
  see above). This is a valid, expected response, not an error.
- **`start == end`** (single day): works unchanged, both comparisons
  collapse to that one day.
- **Range with no matching rows on some days inside it**: `per_day` simply
  omits those days (it's a `GROUP BY`, so days with zero brews never had a
  row to group). The frontend must not assume `per_day` has one entry per
  calendar day in the range — see §4.

## 2. `src/brewops/api/main.py` — `stats` route

Current (line 73-75):

```python
@app.get("/api/stats")
def stats(conn: sqlite3.Connection = Depends(get_db)):
    return queries.get_stats(conn)
```

Add two optional query parameters and validate them before calling the
query function:

```python
@app.get("/api/stats")
def stats(
    start: str | None = None,
    end: str | None = None,
    conn: sqlite3.Connection = Depends(get_db),
):
    start_date = parse_date(start) if start is not None else None
    end_date = parse_date(end) if end is not None else None
    if start_date is not None and end_date is not None and start_date > end_date:
        raise HTTPException(400, "start date must not be after end date")
    return queries.get_stats(conn, start=start, end=end)
```

Add a small helper near `parse_timestamp` (line 41), don't reuse
`parse_timestamp` itself — it expects datetime strings with a time
component, not bare dates:

```python
def parse_date(value: str) -> str:
    """Validate a YYYY-MM-DD query param; return it unchanged if valid."""
    try:
        datetime.strptime(value.strip(), "%Y-%m-%d")
    except ValueError:
        raise HTTPException(400, f"unparsable date {value!r}, expected YYYY-MM-DD")
    return value.strip()
```

`start_date`/`end_date` above are the parsed-but-discarded validation
result — pass the original `start`/`end` strings (not datetime objects)
into `queries.get_stats`, since the SQL comparison is string-to-string
against `DATE(timestamp)`, which SQLite renders as `YYYY-MM-DD`.

### Edge cases to handle in this route

- **Malformed date** (e.g. `start=not-a-date`, `start=2026-13-40`): `parse_date`
  raises `HTTPException(400, ...)` — verify the message names the bad
  value.
- **`start` after `end`**: explicit `400` with the message above. Do not
  let this reach `queries.get_stats` silently — it would just produce an
  empty result set, which hides the mistake from the user.
- **Only `start` given, no `end`**: valid, means "from start to now".
- **Only `end` given, no `start`**: valid, means "everything up to end".
- **Neither given**: valid, unchanged current behavior (all-time).
- **Future dates**: allowed. Unlike `parse_timestamp` (used for logging
  brews), do not reject future dates here — someone filtering "this
  month" near month start might reasonably include a future `end`, and it
  just yields fewer/no rows near the end of the range, which is not an
  error.

## 3. `src/brewops/frontend/index.html` — filter control

Add a control inside the `#dashboard` section (`src/brewops/frontend/index.html`,
around line 17-18, just inside `<section id="dashboard">` before
`.stat-tiles`):

```html
<div class="panel filter-panel">
  <label for="filter-start">From</label>
  <input type="date" id="filter-start">
  <label for="filter-end">To</label>
  <input type="date" id="filter-end">
  <button type="button" id="filter-apply">Apply</button>
  <button type="button" id="filter-clear">Clear</button>
  <p id="filter-message" class="message" role="status"></p>
</div>
```

Use native `<input type="date">` (not `datetime-local`, which is what the
brew/maintenance forms use) — this filter is day-granularity by design
(see decision above). Don't reuse the `brew-timestamp`/`maintenance-timestamp`
inputs or IDs; this is a separate control.

No default value on the inputs — empty means "unbounded" on that side,
matching the API's `None` semantics.

## 4. `src/brewops/frontend/app.js` — wire up the filter

### State and fetch

`loadDashboard` (line 80-92) currently does:

```javascript
const stats = await fetchJSON("/api/stats");
```

Change to build the URL from the two date inputs, only including params
that have a value:

```javascript
async function loadDashboard() {
  const start = document.getElementById("filter-start").value;
  const end = document.getElementById("filter-end").value;
  const params = new URLSearchParams();
  if (start) params.set("start", start);
  if (end) params.set("end", end);
  const query = params.toString();
  const stats = await fetchJSON(`/api/stats${query ? "?" + query : ""}`);
  ...
```

The rest of `loadDashboard` (stat tiles, `renderDrinkBars`, `renderTimeline`,
machine cards) is unchanged — it already just consumes whatever `stats`
comes back.

### Event wiring

Add near `setupForms()` (around line 112), a new small setup function
(call it from the bottom of the file next to the existing
`loadDashboard().catch(...)` / `setupForms().catch(...)` calls at
lines 159-163):

```javascript
function setupFilter() {
  document.getElementById("filter-apply").addEventListener("click", () => {
    const start = document.getElementById("filter-start").value;
    const end = document.getElementById("filter-end").value;
    const message = document.getElementById("filter-message");
    message.textContent = "";
    message.className = "message";
    if (start && end && start > end) {
      message.textContent = "From date must not be after To date.";
      message.classList.add("error");
      return;
    }
    loadDashboard().catch((error) => {
      message.textContent = error.message;
      message.classList.add("error");
    });
  });

  document.getElementById("filter-clear").addEventListener("click", () => {
    document.getElementById("filter-start").value = "";
    document.getElementById("filter-end").value = "";
    document.getElementById("filter-message").textContent = "";
    loadDashboard().catch((error) => console.error("Dashboard failed to load:", error));
  });
}
```

Call `setupFilter();` at the bottom of the file, same place as the
existing `setupForms().catch(...)` call.

Note: string comparison `start > end` works correctly here because both
are `YYYY-MM-DD` (lexicographic order matches chronological order for
that format) — this is a client-side pre-check for a faster error message;
the server-side check in `main.py` (§2) is the real guard and must stay.

Also: after a brew/maintenance form submits successfully, `submitForm`
(line 152) already calls `loadDashboard()` to refresh — that call now
picks up whatever filter is currently set in the two date inputs, which is
correct (no change needed there), but worth being aware of when testing.

### Edge cases to handle in `app.js`

- **Empty range (both inputs blank)**: same as today, all-time stats.
  Verify `Clear` restores this.
- **`start` after `end`**: caught client-side before the fetch (see
  above), so the bad request never reaches the server in the normal UI
  flow. The server-side 400 in §2 is the real guard for direct API calls.
- **Days with no brews inside the range**: `renderTimeline` (line 29-50)
  iterates `perDay` and computes bar width as `width / perDay.length` —
  since `per_day` only contains days that had brews (see §1), a sparse
  range will render fewer, wider bars rather than empty gaps for
  brew-less days. This is existing behavior (already true today for the
  unfiltered timeline) and is out of scope to change, but confirm it still
  looks reasonable for a short, sparse range — if it looks wrong, that's a
  follow-up ticket, not something to silently "fix" by inventing zero-rows
  for missing days.
- **Whole range has zero brews**: `stats.per_day` is `[]`. `renderTimeline`
  already handles this (line 32: `if (perDay.length === 0) return;` — the
  `<svg>` is just left empty). `stats.per_day[stats.per_day.length - 1]`
  (line 83, feeds "brews on last active day" tile) becomes `undefined`,
  and the existing `lastDay ? lastDay.count : 0` guard (line 84) already
  handles that, showing `0`. `renderDrinkBars` still renders all drink
  types with a `0` count each (bar width `0 / max` where `max` is clamped
  to `Math.max(1, ...)` at line 17, so no division-by-zero). No code
  changes needed for this case — just verify it by testing.

## 5. Verification

Manual, since there's no test suite in this repo yet — run
`uv run start`, open `http://localhost:8123`, and check:

1. **Baseline**: load the page with the filter blank. Numbers should
   match what they were before this change (all-time totals).
2. **Normal range**: set `From`/`To` to a range you know has brews (check
   existing data via the timeline hover tooltips first). Click `Apply`.
   Stat tiles, drink bars, and timeline should update to reflect only that
   range. Cross-check the total against `sqlite3` directly, e.g.:
   ```
   sqlite3 <path-to-db> "SELECT COUNT(*) FROM brew_events WHERE DATE(timestamp) BETWEEN '<start>' AND '<end>';"
   ```
   (find the db path via `src/brewops/db/connection.py`).
3. **Single day** (`start == end`): should match
   `WHERE DATE(timestamp) = '<day>'` in sqlite3 directly.
4. **From after To**: type a `From` later than `To`, click `Apply` — should
   show the client-side error message and not change the dashboard. Then
   bypass the UI and hit the API directly to confirm the server also
   rejects it:
   ```
   curl "http://localhost:8123/api/stats?start=2026-09-20&end=2026-09-10"
   ```
   expect a `400` with a message about start/end ordering.
5. **Range with zero brews** (e.g. a range before any data exists, or a
   single quiet day): stat tiles show `0`, drink bars all show `0`,
   timeline is empty (no bars, no error). No console errors.
6. **Only `From` set, `To` blank**: should behave as "from that date to
   now" — confirm via the same `sqlite3` cross-check with an open-ended
   `WHERE DATE(timestamp) >= '<start>'`.
7. **Only `To` set, `From` blank**: same idea, `WHERE DATE(timestamp) <=
   '<end>'`.
8. **Malformed date direct to API**: 
   ```
   curl "http://localhost:8123/api/stats?start=banana"
   ```
   expect `400` naming the bad value.
9. **Per-drink zero-count check**: pick a range where you know one drink
   type (e.g. `hot_water`) was never brewed. Confirm it still appears in
   the `per_drink` response with `"count": 0` rather than being missing
   from the list — this is the specific bug the `LEFT JOIN ... ON`
   placement in §1 is guarding against. Check via:
   ```
   curl "http://localhost:8123/api/stats?start=<range>&end=<range>" | python -m json.tool
   ```
10. **Clear button**: after applying a range, click `Clear` — both date
    inputs empty, dashboard reloads to all-time totals.
11. **Logging a brew while a filter is active**: with a range set, submit
    the "Log a brew" form for a date inside that range, confirm the
    dashboard numbers increment after submit; submit one for a date
    outside the range, confirm the dashboard numbers do *not* change
    (since `loadDashboard` re-fetches with the same filter still applied).

## Out of scope (do not do these)

- Do not change `get_machine_health` or the machine cards — they remain
  lifetime stats (see scope decision at the top).
- Do not add a test suite — this repo doesn't have one; verify manually
  per §5.
- Do not add timezone conversion — timestamps are naive local time
  throughout the codebase already.
- Do not change `parse_timestamp` in `main.py` — it's used for logging
  brews/maintenance and has different rules (rejects future timestamps);
  the new `parse_date` helper is separate and intentionally more
  permissive.
