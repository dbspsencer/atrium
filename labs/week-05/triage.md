# Week 5 cross-tool triage

Your name: Spencer John
Date: 2026-11-20
Atrium address and port: http://localhost:9090
Atrium version or date of local copy: commit 51f3dc9 (2026-11-25), data reset before testing

Fill this in as you work. If a check could not be completed, record what stopped you and what evidence you used instead. Do not paste a full session cookie value.

## 1. Scan coverage and sources

- Static evidence: my `npm run scan` output: 3 findings, all `sql-built-by-joining-text` (ERROR): `src/db/reset.js:12-13`, `src/handlers/directory.js:8-11`, `src/handlers/resources.js:1-5`.
- Runtime evidence: my ZAP run, Week 4 record, or direct browser checks: direct signed-in HTTP checks (same requests as DevTools → Network), one per queue entry. I did not run a fresh ZAP active scan for this record, so the Z rows are checked against the recorded alerts rather than against my own alert list.
- Signed-in account used for protected routes: alice.nolan (staff, id 17).
- Evidence that a protected route returned its page rather than `/login`: `GET /directory?q=Alice` as Alice → 200 text/html, page shows "1 colleague shown across Atrium" and row Alice Nolan. The same request without a session → 302 Location `/login`.
- Active scan target and how login and logout stayed outside it, if I used ZAP: not used for this record. If I re-run: target `GET:directory(q)`, Context excludes `http://localhost:9090/logout.*`, POST /login flagged as Form-based Auth Login Request, logged-in indicator polled on /dashboard.
- An entry my scan did not reproduce, and how I checked it instead: all Z1–Z5 were checked directly (requests below) because I had no ZAP alert list of my own.

## 2. The queue

| ID | Status | Evidence or missing check | Overlap with another ID? | Investigate now, later, or close? |
|---|---|---|---|---|
| S1 | **supported** | `directory.js:19` takes `q` from `req.query.q` (outside Atrium). Lines 10–11 join it between quote marks into the SQL text; line 13 `db.prepare(sql).all()` passes **no** bound values. `GET /directory?q=Alice'` → 500 `near "'%'": syntax error`; `q=zzqx' OR display_name LIKE '` → 200, all 3 colleagues (an ordinary `zzqx` returns 0). | Same path as Z1 | Now (#1) |
| S2 | **ruled out** | `resources.js:1-5` joins five **fixed** strings only; the id reaches the database as a bound `?` value (`.get(req.params.id)`, line 18). No external text enters the SQL text. The rule matched the shape, not a source. | None | Close |
| S3 | **ruled out** | `reset.js:12-13` joins two fixed strings; values come from `seed-data.js` and are bound via `addUser.run(...)` (line 32). Not reachable from any request. | None | Close |
| Z1 | **supported** | Ordinary `GET /directory?q=Alice` → 200, 1 row. `GET /directory?q=Alice'` (one apostrophe) → 500 text/plain with SQLite error and stack trace. Different-rows test above confirms the query structure changes, not only that it errors. | Runtime side of S1 (one weakness) | Now (#1, with S1) |
| Z2 | **supported** | `views/directory/index.ejs:20` prints `<%- q %>` (unescaped) in the result summary. Harmless marker `q=Bob<i>x</i>"` → summary contains raw `<strong>Bob<i>x</i>"</strong>`, so the browser treats it as markup. Script execution not tested. | Same request as Z3 but a **different output point** (line 20, not 13) | Now (#2) |
| Z3 | **ruled out** | `index.ejs:13` prints `value="<%= q %>"` (escaped). Same marker request → `value="Bob&lt;i&gt;x&lt;/i&gt;&#34;"`; the `"` is encoded, so the value cannot leave the attribute. The alert only says the value is reflected in an attribute. | Same URL as Z2, different place in the response | Close (reopen if line 13 changes to `<%-`) |
| Z4 | **supported** | `GET /dashboard` (Alice) → 200, response headers contain no `Content-Security-Policy`. `helmet` is in `package.json` but never used in `src/app.js`. Missing header is a defence-in-depth gap, not a vulnerability by itself. | Would reduce the impact of Z2 | Later |
| Z5 | **supported** | `POST /login` → 302, `Set-Cookie: connect.sid=s%3A******; Path=/` with no `HttpOnly`, no `SameSite`, no `Secure`. Source agrees: `src/config/session.js:12-14`. | Raises the impact of Z2 (script could read the cookie) | Now (#3, linked to Z2) |

## 3. First three matters to investigate

1. **First:**
   - Queue ID or IDs: S1 + Z1 (one underlying path: `q` → SQL text in `searchDirectory`)
   - Account and place: any signed-in staff member (alice.nolan); `GET /directory?q=`, `src/handlers/directory.js:8-13`
   - Why first: code and runtime evidence both confirm search text changes the database instruction (all rows returned for a non-matching term). Any staff account can reach it. Possible consequence, untested: reading other columns or tables (e.g. `password_hash`) through the same query.
   - Next check: as Alice, test whether a UNION-style search can return a column not shown in the directory, on a reset copy only; then repair with bound parameters.
2. **Second:**
   - Queue ID or IDs: Z2
   - Account and place: alice.nolan; `GET /directory?q=` result summary, `views/directory/index.ejs:20`
   - Why second: confirmed that markup in `q` is rendered. Because the payload sits in a GET URL, a link sent to a colleague would render it in their signed-in session. Script execution and CSP behaviour still unconfirmed.
   - Next check: with a harmless marker, confirm in the browser Elements panel that a new element is created; change line 20 to `<%= q %>` and re-test.
3. **Third:**
   - Queue ID or IDs: Z5 (ranked here because of Z2)
   - Account and place: any account at sign-in; `POST /login`, `src/config/session.js:12-14`
   - Why third: a short check changes the severity of Z2. If the cookie is readable by page script, a reflected-markup issue can become session theft.
   - Next check: in a signed-in tab run `document.cookie` in the Console and see whether `connect.sid` is listed (do not record the value).

## 4. One exclusion or combination

- Entry ID: S2
- Close, exclude from active work, or combine with another entry? Close.
- Evidence for this decision: `src/handlers/resources.js:1-5` — every joined piece is a string literal; the only external value (`req.params.id`) is passed as a bound parameter at line 18 `db.prepare(RESOURCE_QUERY).get(req.params.id)`. A colleague can confirm by opening the file and by requesting `/resources/1'` (expect 404, not 500).
- What change would make me reopen it: any variable, request field or template literal (`${...}`) being added to the `RESOURCE_QUERY` string, or the `?` being replaced with concatenated text.

## 5. One static and runtime comparison

- Semgrep entry and ZAP entry: S1 and Z1
- What the code observation shows: the exact source (`req.query.q`, line 19), sink (`db.prepare(sql).all()`, line 13) and the missing control (no bound parameter; quote marks inside the fixed text).
- What the request and response show: that the running app, for a signed-in staff account, actually lets the input change the query: one apostrophe → 500 syntax error; `zzqx' OR display_name LIKE '` → all 3 rows instead of 0.
- Do they concern the same underlying matter? Why? Yes. Same parameter, same handler, and the SQLite error's stack trace names `searchDirectory (directory.js:13)`.
- One claim they do not yet prove: that data outside the three columns shown (for example password hashes) can be extracted, or that data can be changed through this query.

## 6. One finding for a colleague

**Finding:** *Directory search text is read as part of the SQL instruction.* As any signed-in staff member (tested as alice.nolan), the search field `q` on `GET /directory` is joined directly into the SQL statement in `searchDirectory` (`src/handlers/directory.js:8-11`) and executed without bound values (line 13). A single apostrophe after a name returns HTTP 500 with a SQLite syntax error and stack trace, and the search `zzqx' OR display_name LIKE '` returns all three colleagues where an ordinary non-matching search returns none. This shows the input changes which rows the query selects. If further testing confirms it, the same path could expose other columns of the `users` table to any staff account. Repair: pass the search pattern as bound parameters (`LIKE ?`) and stop sending error stacks to the browser (`src/middleware/errors.js:13`).

**Evidence references:** S1 (`npm run scan`, directory.js:8-11); Z1 (recorded ZAP alert); direct requests in §2 rows S1/Z1: `q=Alice` 200/1 row, `q=Alice'` 500, `q=zzqx` 200/0 rows, `q=zzqx' OR display_name LIKE '` 200/3 rows.

---

## Optional, if you had time

What does `src/config/session.js` show about the session that the Z5 response cannot? The code shows the session signing secret is a hard-coded constant (`"atrium-session-secret"`, line 5) and that `sameSite: false` and `secure: false` are set on purpose, not by a default. Someone with the source could forge signed cookies. The response cannot show this. What does the response confirm that source inspection alone cannot? That the running server really sends `connect.sid` with only `Path=/` (no HttpOnly/SameSite/Secure), so no proxy or other middleware is adding those flags at runtime.
