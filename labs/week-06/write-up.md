# Week 6 evidence record

Your name: Spencer John
Date: 2026-11-26
Atrium version or date of local copy: commit 51f3dc9 (2026-11-25), data reset before testing

Keep this file in `my-work/week-06-write-up.md`. Record what you observed,
including an unexpected result. Do not paste a password or full session cookie.

## 1. Evidence audit

| Item | Scanner claim, observed evidence, background or missing evidence? | Where I can find it |
|---|---|---|
| Static alert and rule | Scanner claim: "SQL built by joining pieces of text", rule `sql-built-by-joining-text`, ERROR, CWE-89 | `npm run scan` output; `scan-rules/sql.yaml` |
| Static code excerpt | Observed evidence: the rule matched lines 8–11; I read lines 7–24 myself | `src/handlers/directory.js:8-13, 19` |
| Dynamic alert and risk label | Scanner claim: Cross Site Scripting (Reflected), High risk, Medium confidence | Week 5 queue Z2 (recorded ZAP run) |
| Dynamic request and response | Observed evidence: `GET /directory?q=Bob<i>x</i>"` → 200, raw `<i>` in the result summary | §3 below |
| Signed-in coverage check | Observed evidence: `GET /directory?q=Alice` → 200 page (not 302 to `/login`) for Alice; 302 without a session | §2 below |
| Ordinary comparison | Observed evidence: `q=Alice` → 200, 1 row, summary `<strong>Alice</strong>` | §2 and §3 |
| Rule's risk label / CWE | Background information: describes the class of weakness, not this app | `scan-rules/sql.yaml` metadata |

What is the most important missing artefact for each alert?

- Static: whether `q` actually reaches the SQL text from a request (source), and a request showing the query changed. Both are now supplied below.
- Dynamic: the response excerpt showing *where* the value lands and whether it is parsed as markup, plus the template line that produces it.

## 2. Static finding

**Title:** Directory search text is joined into the SQL instruction (SQL injection)

**Affected route, file and lines:** `GET /directory`; `src/routes/directory.js:8`; `src/handlers/directory.js:8-13, 19-21`

**Account and conditions tested:** alice.nolan (staff), local Atrium on port 9090, freshly reset data.

**Tool output used:** `npm run scan`: `sql-built-by-joining-text` at `src/handlers/directory.js:8-11`.

**Code trace:** `src/routes/directory.js:8` runs `requireAuth` (the only control, which checks *who* is signed in) and then `showDirectory`. `directory.js:19` reads `q` from `req.query.q` (source) and accepts any string. Line 21 passes it to `searchDirectory`. Lines 10–11 join `q` between `'%` and `%'` in the SQL text. Line 13 `db.prepare(sql).all()` (sink) runs it with no bound values. Nothing on this path validates, escapes or parameterises `q`.

**Ordinary request and result:** `GET /directory?q=Alice` → 200 text/html, "1 colleague shown", row: Alice Nolan.

**Test request and result:** `GET /directory?q=Alice'` → 500 text/plain, body begins `near "'%'": syntax error … SqliteError … at searchDirectory (directory.js:13:13)`. Then `GET /directory?q=zzqx' OR display_name LIKE '` → 200, "3 colleagues shown": Alice Nolan, Bob Keane, Morgan Doyle. Plain `zzqx` → 0 rows. The error alone only shows the input broke the syntax; the different-rows result shows the input changed the condition the database evaluated.

**My conclusion:** Established: a signed-in staff member can alter the WHERE clause of the directory query through `q`. Not tested: selecting other columns/tables (UNION), changing data, or behaviour for other roles. The 500 response also discloses the SQL error and a server stack trace (`src/middleware/errors.js:13`).

**Impact within this scope:** Any staff account can change which `users` rows the query returns. Reading data the directory does not normally show (e.g. `password_hash`) is possible in principle on this query shape but **not demonstrated**.

**Repair and how I would check it:** Use placeholders: `WHERE display_name LIKE ? OR department LIKE ?` and pass `"%" + q + "%"` as bound values. Re-run the same three requests (expect 1 row / 0 rows / 0 rows, no 500), `npm test`, and `npm run check`.

**One other static match:** `src/handlers/resources.js:1-5` matches the same rule because it joins strings, but every piece is a literal and the id is bound (`.get(req.params.id)`, line 18). The rule only sees the joining shape; it cannot tell whether a joined piece came from outside, so a match does not settle whether the two cases are alike.

## 3. Dynamic finding

**Title:** Search text is rendered as HTML in the directory result summary (reflected XSS)

**Affected URL and output location:** `GET /directory?q=…`, the "Showing colleagues matching …" summary (`<p class="result-summary">`), not the search input.

**Account and conditions tested:** alice.nolan, signed in, port 9090, reset data. Harmless markup only.

**ZAP alert, risk and confidence:** Cross Site Scripting (Reflected), High / Medium, from the recorded authenticated ZAP run given in the Week 5 queue (Z2). I did not re-run ZAP for this record. *If you re-run it, add your ZAP version and the scan time here.*

**Ordinary request and result:** `GET /directory?q=Alice` → 200; response contains `value="Alice"` and `<strong>Alice</strong>`; browser shows "Showing colleagues matching **Alice**."

**Alert request repeated and result:** Harmless equivalent of the alert input: decoded `Bob<i>x</i>"`. → 200. Response: input `value="Bob&lt;i&gt;x&lt;/i&gt;&#34;"` (escaped) but summary `<strong>Bob<i>x</i>"</strong>` (raw). The browser renders "x" in italics, so the text was parsed as a new element. 0 rows returned.

**Code that explains the result:** `src/handlers/directory.js:19,23` passes the raw `q` to the view. `views/directory/index.ejs:13` uses `<%= q %>` (escaped). `views/directory/index.ejs:20` uses `<%- q %>` (unescaped output). Line 20 is where submitted text enters the page as markup.

**My conclusion:** Established: user-supplied markup in `q` is inserted unescaped into the summary for a signed-in user. Not tested: script execution, behaviour in other browsers, and whether a victim following a crafted link would be affected (the link would need them to be signed in).

**Impact within this scope:** If script runs, it would run in the victim's Atrium session. Because `connect.sid` has no HttpOnly flag (`src/config/session.js:12`), such script could read the session cookie. This is conditional and not demonstrated.

**Repair and how I would check it:** Change line 20 to `<%= q %>`. Repeat the marker request: the summary should show the literal text `Bob<i>x</i>"`. Add a CSP (helmet) and HttpOnly cookie as defence in depth.

## 4. Cross-check and review

What did runtime evidence add to the static finding? It moved it from "shape matches a rule" to "input changes the rows returned" (0 → 3) for a real staff session, and showed the stack-trace disclosure in the 500 response.

What did code evidence add to the dynamic finding? It located the exact output point (line 20 `<%-`, not line 13). That explains why ZAP's attribute alert (Z3) is a different and weaker matter, and points to a one-character repair.

Who or what reviewed my drafts? An AI assistant (Claude).

One specific criticism: the first draft of the dynamic finding said the issue "allows session hijacking". That claim has no artefact, because script execution and cookie theft were never observed.

What I changed, or why I did not change it: rewrote the impact as conditional ("if script runs … could read the cookie") and listed script execution as untested.

## 5. CA1 set-up

| Check | What I recorded |
|---|---|
| Folder name (`procrastimate-` and my student number) | procrastimate-20099070 *(to do: needs procrastimate.zip from Moodle)* |
| Port from `.env` (1 and the last four digits) | 19070 *(check against the CA1 brief's rule)* |
| `npm run doctor` result | *not run yet* |
| Signed in as `maya` at my port | *not done yet* |
| Anything that failed, with the exact message | — |

One question I still have about the CA1 brief: *(add once you've read the brief)*
