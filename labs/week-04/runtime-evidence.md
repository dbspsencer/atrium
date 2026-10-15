# Week 4 runtime evidence

Your name: 20099070
Date: 15/10/2026
Atrium address and port: http://localhost:9090
ZAP run: OWASP ZAP, run locally on the same machine as Atrium, lab seed data
(`npm run reset` before starting)

Fill this in as you work. If a step failed, say what you tried. Do not paste
a full session cookie value.

## 1. Coverage before sign-in

- Account state: signed out (new ZAP session, no Atrium account signed in).
- Routes reached by the spider: `GET /` (redirects to `/login`), `GET /login`,
  and the static files the login page links to (`/css/app.css`). The spider got
  no further, because every other page sits behind `requireAuth`.
- My direct `GET /directory` request: status 302 Found, `Location: /login`. The
  final page after the redirect is the sign-in form, not a directory page, so it
  does not count as a tested directory page.
- What this run could not test: the dashboard, the staff directory and its
  search, the shared documents, the profile and the admin pages. None of them
  were reachable signed out, so a signed-out scan says nothing about them.

## 2. Coverage after sign-in

- Seeded account used: `alice.nolan` (a member; password in README.md).
- Evidence that ZAP received the dashboard as this account: `GET /dashboard`
  returned 200, and the page showed "Alice Nolan" with the Operations
  department, which only appears while she is signed in.
- Evidence that ZAP received `/directory?q=Alice` as this account:
  `GET /directory?q=Alice` returned 200 with a directory page listing Alice
  Nolan, not a 302 redirect to `/login`.
- Routes reached as this user: `/dashboard`, `/directory`,
  `/directory?q=...` (the search), `/resources`, `/resources/:id`, `/profile`
  (GET). Sign-in/sign-out routes were kept out of the scan.
- Routes or states newly reached compared with section 1: everything behind
  sign-in — the dashboard, the directory and its search, documents and the
  profile. The only change from section 1 was holding a valid session; the
  application was the same.
- Active scan target and how I kept login and logout out of it: I scanned
  `GET:directory(q)` (the search with its `q` parameter), with Context set to my
  Atrium context and User set to Alice. I added `http://localhost:9090/logout.*`
  to Exclude from Context, and kept `GET /login`, `POST /login` and
  `POST /logout` out of the active scan so the scanner would not end or lose its
  own session.

## 3. Scanner alert

- Alert name: Cross Site Scripting (Reflected) — ZAP rule 40012.
- Risk and confidence as ZAP reported them: Risk High, Confidence Medium.
- Method and affected URL: `GET http://localhost:9090/directory?q=...`, parameter
  `q`.
- The input ZAP sent: a short HTML/script payload in the `q` parameter (ZAP uses
  a probe such as a `<script>` tag or similar markup).
- Response status and relevant evidence: 200 OK. The evidence ZAP highlighted
  was the `q` value appearing in the response body unchanged, inside the
  "Showing colleagues matching ..." summary line, with its angle brackets intact
  rather than turned into `&lt;`/`&gt;`.
- Where I saw the request and response: ZAP History, the request flagged by the
  active scan against `GET:directory(q)`.

## 4. Manual comparison

**Ordinary search**

- Account, method and full URL: `alice.nolan`,
  `GET http://localhost:9090/directory?q=Alice`.
- Status and any redirect: 200 OK, no redirect.
- Relevant response text or visible result: one row, Alice Nolan, and the
  summary line "Showing colleagues matching **Alice**". The word Alice is plain
  text.

**Alert request repeated by me**

- Account, method and full URL: `alice.nolan`,
  `GET http://localhost:9090/directory?q=<b>hello</b>`.
- Status and any redirect: 200 OK, no redirect.
- Relevant response text or visible result: the summary line read "Showing
  colleagues matching **hello**" with the word **hello** rendered in bold. The
  `<b>` tags were treated as markup and applied to the page, not shown as the
  literal text `<b>hello</b>`.
- Did the alert's observation repeat? Yes. Markup placed in `q` came back as
  markup in the page, which is the reflection the alert describes. (Checking the
  template afterwards, the summary line outputs `q` with EJS's unescaped `<%- q %>`,
  while the search box itself uses the escaped `<%= q %>`, which is why the
  reflection shows in the summary line.)

## 5. Conclusion

**What this evidence proves.** On this version of Atrium, run locally and signed
in as `alice.nolan`, the directory search writes the `q` value into the results
page without escaping it, so HTML markup in `q` is rendered as part of the page.
I saw `<b>hello</b>` come back as bold text in the summary line, where an
ordinary search for `Alice` stayed plain text.

**What this evidence does not prove.** It does not prove that an attacker's
script actually runs in another colleague's browser, that any cookie or data has
been taken, or that a deployed Atrium with different templates or headers behaves
the same way. Showing a real script executing, and whether the session cookie
can be read from the page, would each need a separate test.

## 6. Short finding

The directory search at `GET /directory?q=` (observed signed in as
`alice.nolan`, on localhost with the lab data) returns the typed `q` value into
the results page without escaping it: a search for `<b>hello</b>` came back as
bold text rather than literal characters, where a search for `Alice` did not.
This is reflected output that is rendered as HTML, the basis for a reflected
cross-site scripting weakness; the possible impact, if confirmed, would be a
crafted link causing a colleague's browser to run attacker-supplied script in
the Atrium page, but that impact is not yet demonstrated here. Evidence: the two
requests above, one ordinary and one with markup, both returning 200, with the
markup rendered. A high-level repair is to escape the value on output (use EJS's
escaping `<%= q %>` in the summary line instead of `<%- q %>`, so the value is
always shown as text), which a later week returns to.

A colleague could repeat this with the account `alice.nolan`, the two URLs
`/directory?q=Alice` and `/directory?q=<b>hello</b>`, and the same lab data, and
see the same difference.

---

## Optional, if you had time

Which signed-in page did you open that was absent from your spider's route
list? What does that tell you about this scan's coverage?

Opening `/profile` and `/resources` by hand produced requests that were present
once signed in, but `/admin/users` and `/api/admin/users` did not appear for
Alice at all, because she is a member and the navigation never links her to
them. That shows the spider's route list is shaped by what the signed-in account
can see: scanning as a member never reaches the admin-only routes, so a clean
member scan says nothing about them. They would need a scan signed in as the
administrator (`morgan.doyle`).
