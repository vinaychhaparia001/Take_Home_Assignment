# Submission Notes

## Coverage

```
All files        |   97.45 |    97.75 |   93.33 |    97.2
 src/app.js      |   69.23 |       75 |       0 |   69.23   (10-11, 17-18)
 src/routes      |      100|      100 |     100 |     100
 src/services    |      100|    94.73 |     100 |     100
 src/utils       |      100|      100 |     100 |     100
Test Suites: 2 passed, 2 total
Tests:       59 passed, 59 total
```

`app.js` is the only file under 80%: the two uncovered blocks are the
`app.listen(...)` bootstrap (guarded by `require.main === module`, so it
never runs when the app is `require`'d in tests — this is intentional and
correct) and the global error-handling middleware, which is currently
unreachable dead code (see the last bullet in `BUGS.md`) since nothing in
the app calls `next(err)`. I didn't fabricate an error-throwing test route
just to hit that line, since it would test scaffolding I invented rather
than real behavior.

## Design decisions for `PATCH /tasks/:id/assign`

- **Validation:** `assignee` must be present and a non-empty string after
  trimming. I trim before storing so `"  Vinay  "` and `"Vinay"` land the
  same, matching how `title` validation already treats whitespace-only
  strings as empty.
- **Unknown task:** returns `404 { error: 'Task not found' }`, consistent
  with the existing `PUT`/`DELETE`/`complete` endpoints.
- **Already-assigned tasks:** the assignment explicitly asks "what if the
  task is already assigned?" — I chose to allow reassignment
  (overwriting the previous assignee) rather than rejecting the request.
  Reasoning: nothing in the brief suggests assignment should be
  write-once, and blocking reassignment would mean there's no way to
  reassign a task without a separate "unassign" step first, which seems
  like unnecessary friction for a task manager. If the intent was
  instead "assignment should require an explicit unassign step," that's
  a one-line change (check `task.assignee` and 409 if already set) and
  worth confirming with product before shipping.
- **Task shape:** added `assignee: null` as a default on `create()` so
  every task has a consistent shape from the start, rather than the field
  only appearing after a task has been assigned at least once. This also
  makes it show up cleanly in the existing `GET /tasks` responses.
- I did not add an "unassign" endpoint (e.g. `assignee: null`) since it
  wasn't asked for, but flagging it as an easy follow-up if needed.

## What I'd test next with more time

- Concurrent/rapid `PATCH .../assign` and `PATCH .../complete` calls
  against the same id — the in-memory store isn't guarded against races,
  though with Node's single-threaded event loop and no `await` in these
  handlers it's likely fine in practice.
- Response shape/testing for very large `limit` values in pagination once
  bug #2 is fixed, plus negative/zero `page`/`limit`.
- `getStats()` behavior at exact `dueDate` boundaries (task due exactly
  "now").
- Whatever persistence layer eventually replaces the in-memory store —
  none of the current tests would catch a data-layer regression since
  everything runs against the same process-lifetime array.

## What surprised me in the codebase

- The pagination bug (#2) is in the *default* code path — `GET /tasks`
  with no query params at all skips pagination entirely and returns
  everything, but the moment a client passes `page`/`limit` (even just
  `page=1`, the natural first request), it silently drops the first page
  of real data. That's an easy one to miss in a demo where someone pages
  through with `page=0` or omits `page` altogether.
- `completeTask` (bug #3) touching `priority` at all was unexpected —
  none of the other single-purpose PATCH-style operations mutate
  unrelated fields, so this one stood out as likely accidental.
- The full test suite runs in about a second — the in-memory store is
  convenient for the assignment, but the tests can't say anything about
  how correctness holds up once it's backed by a real database.

## Questions I'd ask before shipping this to production

1. Is `PUT /tasks/:id` intended to be a full replace or a partial patch?
   The validator allows partial bodies today, which is more like `PATCH`
   semantics — worth confirming which one the API contract actually
   promises to clients.
2. Should `assignee` be validated against a known set of users (e.g. an
   existing users table/service), or is any free-text string acceptable
   as it is now?
3. Should reassigning an already-assigned task be allowed outright (what
   I implemented), require a separate unassign step, or notify/log the
   previous assignee?
4. Is there a real datastore planned before this reaches production? The
   in-memory store means every deploy/restart loses all data, which is
   fine for this exercise but presumably not for a real task manager.
5. Should `DELETE`/`PUT` behave differently for a task that's already
   `done` (e.g. block edits to completed tasks), or is any task editable
   regardless of status?
