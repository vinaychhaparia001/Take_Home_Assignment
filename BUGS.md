# Bug Report

Found by reading `src/` and pinning behavior down with tests in `tests/`.
Bug #1 is fixed in this submission; the rest are documented but left as-is,
per the assignment instructions ("pick one bug and fix it").

---

## Bug #1 — `getByStatus` matches on substring, not equality (FIXED)

**Where:** `src/services/taskService.js`, `getByStatus`

```js
// before
const getByStatus = (status) => tasks.filter((t) => t.status.includes(status));
```

**Expected:** `GET /tasks?status=todo` returns only tasks whose status is
exactly `"todo"`.

**Actual:** `String.prototype.includes` does substring matching, not
equality. `?status=do` matches both `"todo"` and `"done"` (both contain
`"do"`). `?status=in_progress` also happens to work today only because it's
an exact string, but any partial value like `?status=progress` would
incorrectly match it too.

**How I found it:** Manually curled the running server with
`?status=do` after seeding a `todo` task and saw it come back, which
shouldn't happen for an exact-match status filter. Wrote
`taskService.test.js` → `getByStatus` → "does not match statuses that
merely contain the filter as a substring" to pin it down, and an
integration test hitting `GET /tasks?status=` through Supertest.

**Fix:** exact equality instead of substring containment:

```js
const getByStatus = (status) => tasks.filter((t) => t.status === status);
```

**Why this matters:** silently returning tasks that don't match the
requested filter is a correctness bug that's easy to miss in manual
testing (most manual testers type the full, valid status value), but
would show up as confusing/incorrect results for any client that passes
a partial or unexpected value.

---

## Bug #2 — Pagination is off by one page (`getPaginated`)

**Where:** `src/services/taskService.js`, `getPaginated`

```js
const getPaginated = (page, limit) => {
  const offset = page * limit;
  return tasks.slice(offset, offset + limit);
};
```

**Expected:** `GET /tasks?page=1&limit=10` returns the *first* 10 tasks
(items 0–9).

**Actual:** `offset = page * limit`, so page `1` computes `offset = 10`
and returns items 10–19 — the *second* page. Page `1` never returns the
first `limit` items at all; they're permanently unreachable through this
endpoint.

**How I found it:** Seeded a few tasks and requested `?page=1&limit=2`;
the first task created never appeared in any page's results. Confirmed by
reading the route (`routes/tasks.js`) which defaults `page` to `1` when
absent — so this is also the default pagination path, not an edge case.

**What a fix would look like:** treat `page` as 1-indexed:

```js
const getPaginated = (page, limit) => {
  const offset = (page - 1) * limit;
  return tasks.slice(offset, offset + limit);
};
```
It would also be worth clamping `page`/`limit` to sane minimums (e.g.
`page < 1` or `limit <= 0`) since `parseInt(...) || 1` in the route only
guards against `NaN`, not against `0` or negative values.

---

## Bug #3 — `completeTask` silently resets `priority` to `medium`

**Where:** `src/services/taskService.js`, `completeTask`

```js
const completeTask = (id) => {
  const task = findById(id);
  if (!task) return null;

  const updated = {
    ...task,
    priority: 'medium',
    status: 'done',
    completedAt: new Date().toISOString(),
  };
  ...
};
```

**Expected:** completing a task changes its `status` to `done` and stamps
`completedAt`. `priority` is unrelated to completion and should be left
alone.

**Actual:** every call to `PATCH /tasks/:id/complete` overwrites
`priority` to `'medium'`, regardless of what it was before. A `high`
priority task silently becomes `medium` the moment it's marked done.

**How I found it:** Created a task with `priority: 'high'`, called
`PATCH /:id/complete`, and the response came back with `priority:
'medium'`. Nothing in the endpoint's contract (README/ASSIGNMENT) suggests
completion should touch priority, so this looks like a copy-paste
artifact rather than intentional behavior.

**What a fix would look like:** drop the `priority` field from the spread
so it's left untouched:

```js
const updated = {
  ...task,
  status: 'done',
  completedAt: new Date().toISOString(),
};
```

---

## Bug #4 — `update` (`PUT /tasks/:id`) allows overwriting server-controlled fields

**Where:** `src/services/taskService.js`, `update`

```js
const update = (id, fields) => {
  const index = tasks.findIndex((t) => t.id === id);
  if (index === -1) return null;

  const updated = { ...tasks[index], ...fields };
  tasks[index] = updated;
  return updated;
};
```

**Expected:** `PUT /tasks/:id` updates user-editable fields (`title`,
`description`, `status`, `priority`, `dueDate`). Server-assigned fields
like `id`, `createdAt`, and `completedAt` shouldn't be client-writable.

**Actual:** `fields` is the raw request body spread directly onto the
existing task with no allow-list, so a request body containing
`{"id": "...", "createdAt": "...", "completedAt": "..."}` silently
overwrites those fields too. `validateUpdateTask` only checks
`title`/`status`/`priority`/`dueDate` — it doesn't reject unknown or
server-owned keys.

**How I found it:** Read `update` while reasoning about what
`validateUpdateTask` does and doesn't check, then confirmed with a
Supertest request that included `id` and `createdAt` in the PUT body and
saw both change in the response.

**What a fix would look like:** destructure only the allowed fields
before merging, e.g.:

```js
const update = (id, fields) => {
  const index = tasks.findIndex((t) => t.id === id);
  if (index === -1) return null;

  const { title, description, status, priority, dueDate } = fields;
  const patch = { title, description, status, priority, dueDate };
  Object.keys(patch).forEach((k) => patch[k] === undefined && delete patch[k]);

  tasks[index] = { ...tasks[index], ...patch };
  return tasks[index];
};
```

---

## Smaller things worth a mention (not full bugs)

- **`PUT` accepts partial bodies.** The README calls `PUT /tasks/:id` a
  "full update", but `validateUpdateTask` treats every field as optional
  — you can `PUT` just `{"priority": "high"}` and it behaves like a
  `PATCH`. Not incorrect, but worth confirming the intended semantics
  before shipping (see "Questions" in `NOTES.md`).
- **No `GET /tasks/:id`.** There's no route to fetch a single task by id
  — only list/filter/paginate. Might be intentional for this slice, but
  it's a gap if clients need to re-fetch a task after a background change.
- **Error-handling middleware in `app.js` looks unreachable.** All
  current route handlers are synchronous with no `try/catch` that calls
  `next(err)`, so the generic 500 handler at the bottom of `app.js`
  never actually fires today. It's not wrong, just currently dead code —
  worth revisiting once any handler does async work (e.g. a real
  database) that can throw.
