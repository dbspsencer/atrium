# Week 8 remediation record

Your name: Spencer John
Date: 2026-11-26
Atrium version or copy used: commit 51f3dc9 (2026-11-25), local copy; data reset

Fill this in as you work. Keep exact inputs and observed results. If you have not checked something, say so.

## 1. Original finding and before evidence

- Page and route: Directory, `GET /directory?q=`
- Account used: alice.nolan
- Earlier finding or source I started from: Week 6 static finding (SQL text built from `q`, `src/handlers/directory.js:8-11`)
- Ordinary search text, URL, status and results-table rows: `Finance`, `/directory?q=Finance`, 200, rows: Bob Keane
- No-match search text, URL, status and results-table rows: `zzqx`, `/directory?q=zzqx`, 200, no rows ("No colleagues found")
- Comparison search text, URL, status and results-table rows: `zzqx' OR display_name LIKE '`, `/directory?q=zzqx%27%20OR%20display_name%20LIKE%20%27`, 200, rows: Alice Nolan, Bob Keane, Morgan Doyle. (Also `Alice'` → 500 `near "'%'": syntax error`.)
- What the comparison shows, and what it does not show: a term that matches nobody returned every colleague, so the typed text changed the WHERE condition. It does not show that other tables or columns can be read, or that data can be changed.

## 2. Root cause

- Source of the search text, with file and line: `req.query.q`, `src/handlers/directory.js:19`
- Path through the handler: `src/routes/directory.js:8` (`requireAuth` → `showDirectory`) → line 21 `searchDirectory(db, q)` → lines 8–11 build the SQL string by joining `q`
- Database operation, with file and line: `db.prepare(sql).all()`, `src/handlers/directory.js:13`
- Control missing on that path: bound parameters. `q` should be passed to the database as a value, separately from the SQL text.
- Difference between the visible result and the root cause: the visible result is extra rows or a 500 error. The cause is that the search value is placed inside the SQL text, so the database parses it as part of the instruction.

For `Keane` the handler builds:
`… WHERE display_name LIKE '%Keane%' OR department LIKE '%Keane%'`
For the comparison input it builds:
`… WHERE display_name LIKE '%zzqx' OR display_name LIKE '%' OR department LIKE '%zzqx' OR display_name LIKE '%'`
The typed text leaves the quotes after `zzqx`: its `'` closes the string, and `OR display_name LIKE '` becomes SQL.

Comparison with other handlers: `auth.js:15` (`WHERE username = ?` with `.get(username)`) and `profile.js:1,13` (`UPDATE … = ?` with `.run(...)`) pass every value as a bound parameter through the same better-sqlite3 library. The directory is the only one that splices the value into the text.

## 3. Security property and useful behaviour

- One sentence stating the property the repair must enforce: search text must always reach the database as a bound data value and must never change the structure of the SQL instruction, whatever characters it contains.
- Ordinary behaviour that must remain, including name and department search: a partial, case-insensitive name search (`Alice` → Alice Nolan) and a department search (`Finance` → Bob Keane) still work, and an empty search still lists all colleagues.

## 4. Candidate fixes

| Candidate | Where the control acts | Effect on the property and ordinary search | Decision |
|---|---|---|---|
| A. Placeholders: `LIKE ? OR … LIKE ?`, bind `"%" + q + "%"` twice | At the database boundary (prepare/bind) | Enforces the property for every input and every route into the function. Name, department and empty search unchanged. | **Chosen** |
| B. Escape/strip `'` in `q` before joining | In the handler, before the string is built | Blocklist; depends on knowing every special character and SQLite's quoting rules. The SQL is still built from text. It breaks legitimate names such as O'Brien. | Rejected |
| C. Catch the database error and show "no results" | After the query fails | Hides the 500 but `zzqx' OR display_name LIKE '` still returns all rows. Does not enforce the property. | Rejected |
| D. Client-side input pattern / hide the field | In the browser | A direct request to `/directory?q=…` bypasses it entirely. | Rejected |

- Assistant used, if any: Claude (AI assistant), asked for repair options without editing files.
- One suggestion I changed or rejected, and why: the assistant also suggested adding `ESCAPE '\'` and escaping `%`/`_` so they are matched literally. I left this out of this repair because it changes search behaviour rather than the injection boundary. It is noted for Week 9.
- Reason for my choice: A acts at the boundary where the problem occurs. The database receives the instruction and the value separately, so no character in `q` can change the instruction. It also matches the pattern already used in `auth.js` and `profile.js`.

## 5. Change made

- File and function changed: `src/handlers/directory.js`, `searchDirectory`
- Changed lines or a short before and after code extract:

```js
// before (lines 8-13)
const sql =
  "SELECT id, display_name, email, department, role FROM users " +
  "WHERE display_name LIKE '%" + q + "%' " +
  "OR department LIKE '%" + q + "%'";
return db.prepare(sql).all();

// after (lines 8-13)
const sql =
  "SELECT id, display_name, email, department, role FROM users " +
  "WHERE display_name LIKE ? OR department LIKE ?";
const pattern = "%" + q + "%";
return db.prepare(sql).all(pattern, pattern);
```

- How the change keeps search text separate from the database instruction: the SQL text is now fixed and contains only `?` placeholders. `q` is only joined with `%` to make a LIKE pattern, which is a data value. better-sqlite3 binds that value after the statement is parsed, so the database treats it purely as text to compare.
- What I left outside this repair: the unescaped result summary (`views/directory/index.ejs:20`, `<%- q %>`); `%` and `_` acting as LIKE wildcards; the stack trace in 500 responses (`src/middleware/errors.js:13`).

## 6. First checks after the change

(Server restarted, signed in again as alice.nolan.)

- Same ordinary search: `Finance` → 200, Bob Keane; also `Alice` → 200, Alice Nolan
- Same no-match search: `zzqx` → 200, no rows
- Same comparison search: `zzqx' OR display_name LIKE '` → 200, **no rows** (was all 3). `Alice'` → 200, no rows (was 500). `' OR '1'='1` → 200, no rows.
- Empty search result: `/directory?q=` → 200, all 3 colleagues (unchanged)
- `npm test` result and any failing test: 22 tests, 22 pass, 0 fail
- `npm run scan` result, if available: still reports `sql-built-by-joining-text` at `src/handlers/directory.js:8-10`. This is a false positive. The joined pieces on lines 9–10 are both fixed string literals with `?` placeholders, and `q` only appears in the bound `pattern` on lines 12–13. The rule matches the joining shape, not the source of the text.
- Also `npm run check`: Keane GREEN; `Morgan Doyle'` GREEN (was RED); `' OR '1'='1` GREEN (was RED); summary `<u>W9</u>` still RED (the separate output issue, left out on purpose).

## 7. Handoff to Week 9

- What this evidence supports: the tested inputs that previously changed the query now return no rows, and ordinary name, department and empty searches behave as before.
- Cases or related paths still to check: `%` and `_` (both currently return all 3 colleagues as wildcards, which is not injection but may not be wanted); very long input; non-ASCII names; array input `q[]=` (handler already falls back to ""); other places that might build SQL from request data.
- A focused automated test I would add: in `tests/functional/directory.test.js`, sign in as Alice and assert `GET /directory?q=' OR '1'='1` returns 200 with no colleague rows, and `GET /directory?q=Alice'` returns 200 (not 500).
- Any known regression or remaining risk: none seen in tests. Remaining risks: reflected markup in the summary (`index.ejs:20`), stack traces on errors, missing CSP, and the non-HttpOnly session cookie.

## 8. Document description output

- Resource changed for the temporary check: id 2, "Supplier register". `src/db/seed-data.js` description set to `Before <b>bold</b> after`, then `npm run reset` and restart.
- List and detail page before the repair: `/resources` → `<span class="resource-description">Before <b>bold</b> after`; `/resources/2` → `<p>Before <b>bold</b> after</p>`. In both, "bold" was rendered in bold and the tags were not visible.
- Rule for printing a description as page content: a description is data and must be HTML-escaped when placed in the page, so any markup in it is shown as text, never interpreted.
- Two view files and lines changed: `views/resources/index.ejs:14` `<%- item.description %>` → `<%= item.description %>`; `views/resources/show.ejs:11` `<%- resource.description %>` → `<%= resource.description %>`
- List and detail page after the repair: both now contain `Before &lt;b&gt;bold&lt;/b&gt; after`, so the browser shows the literal text `Before <b>bold</b> after`.
- Ordinary description checked after restoring the seed data: seed restored, reset and restarted; `/resources/2` shows "Approved suppliers, contract end dates and the named contact for each." normally.
- What this check does not establish: other output contexts (attributes, scripts, URLs), other fields or views using `<%-` (for example `directory/index.ejs:20`), behaviour once a description-editing form exists, or how stored data that is already HTML-formatted would display.
