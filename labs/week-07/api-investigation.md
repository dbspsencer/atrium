# Week 7 API investigation

Your name: Spencer John
Date: 2026-11-26
Atrium address and port: http://localhost:9090
Atrium version or date of local copy: commit 51f3dc9 (2026-11-25), data reset

Fill this in as you work. Give statuses and short response excerpts. Record
whether a request carried a session cookie, but never paste its full value
or a password.

## 1. Original profile request

- Signed-in account: alice.nolan (id 17, role member)
- Page that made the request: `/profile` (script `public/js/profile.js` fetches `/api/profile/` + the hidden `userId` field)
- Method and full API path: `GET http://localhost:9090/api/profile/17`
- Was a session cookie present? (yes/no, value hidden): yes, `connect.sid=s%3A******`
- First response status and content type: 200, `application/json; charset=utf-8`
- Returned ID, username and field names: id 17, username `alice.nolan`; fields `id, username, displayName, email, department, role, bio`
- What did the request ask for, in your own words? It asks the server for the profile record with id 17, using Alice's session. The `id` and `username` fields in the JSON identify which record came back.

## 2. Replayed and changed profile requests

| Request | Account whose session I used | First status | Returned ID and name, or error | Evidence reference |
|---|---|---|---|---|
| Repeat of the original path | Alice | 200 | 17, Alice Nolan (identical body to §1) | R1 |
| Path with one ID changed | Alice | 200 | 18, Bob Keane: `{"id":18,"username":"bob.keane",…,"email":"bob.keane@atrium.example","department":"Finance","role":"member","bio":"Looks after purchase orders…"}` | R2 |

- What I changed: only the final path number, 17 → 18.
- What stayed the same: method, session (Alice's cookie), headers, status 200, field names.
- Whose record the changed ID names, and how I checked: Bob Keane. Confirmed by `username: bob.keane` in the response, the README account list, and `src/db/seed-data.js` (17 Alice, 18 Bob, 99 Morgan).
- What the comparison shows: with Alice's session the server returned another person's full profile record. It did not check that the requested id belongs to the signed-in user.

## 3. Without sign-in

- How I made sure the window had no Atrium session: a new private window / client with no cookies; the request carried no `Cookie` header.
- `GET /api/profile/17`: first status and response body: 401, `{"error":"You need to sign in to use this part of Atrium."}`
- `GET /profile`: first status, redirect if any, and final page: 302 `Location: /login`; ends on the sign-in page.
- What this establishes about authentication: the API and page both refuse requests with no session (`requireAuth`).
- What it does not establish about authorisation: it says nothing about which records a signed-in person may read. §2 shows that question is answered differently (any signed-in user gets any id).

## 4. Administration page and API

| Account | Request path | First status | Relevant response text or JSON fields | Evidence reference |
|---|---|---|---|---|
| Alice | `/admin/users` | 403 | `Administrators only.` | A1 |
| Alice | `/api/admin/users` | 200 | array of 3: `id, displayName, username, department, role` | A2 |
| Morgan | `/admin/users` | 200 | "Staff accounts" page HTML (table body empty until script runs) | A3 |
| Morgan | `/api/admin/users` | 200 | same 3-item array as A2 | A4 |

- Which request did Morgan's page make after loading? `GET /api/admin/users` (from `public/js/admin-users.js`), 200.
- What role decision should the server make for each route? Both should require `role === "admin"`: 200 for Morgan, 403 for Alice.
- Where did the observed decisions agree or differ? They agree for Morgan (both 200). They differ for Alice: the page refuses (403) but the API behind it returns the full account list (200).

## 5. Security conclusions

**Profile request.** Signed in as alice.nolan, `GET /api/profile/18` returned 200 with Bob Keane's record (`username`, `email`, `department`, `role`, `bio`). The rule should be: a staff member may read only their own profile through this API (id must equal the session user's id), unless they are an administrator. Observed: the server checks only that a session exists (`src/routes/api.js:8` `requireAuth`) and returns any id (`src/handlers/api.js:10`). This is broken object-level authorisation (IDOR). Editing another profile was not tested in this lab.

**Administration request.** Signed in as alice.nolan (role member), `GET /api/admin/users` returned 200 with every account's `id, username, displayName, department, role`, while `GET /admin/users` returned 403. The rule should be: only `admin` may list accounts, at both entry points. Observed: the page uses `requireAdmin` (`src/routes/admin.js:8`) but the API uses `requireAuth` only (`src/routes/api.js:9`). Fix: use `requireAdmin` on `/api/admin/users`.

**Directory comparison.** `/directory` already shows every signed-in colleague `display_name`, `department`, `email` and `role`. The profile API adds `id`, `username` and `bio`. The admin API adds `id` and `username` (login names). So the impact is limited: the access-control error is proven, but most returned values are not secret. The new exposure is usernames (useful for password guessing) and bios.

**Evidence still missing or uncertain.** Whether `POST /profile` lets one user change another's profile (`saveProfile` uses `body.userId` from the form, `src/handlers/profile.js:13`). This could be settled with a controlled POST on a reset copy, which is outside this GET-only lab.

**Assistant or self-review.** The assistant suggested that Alice's access to `/api/admin/users` exposes password hashes. I checked `ACCOUNTS_QUERY` in `src/handlers/api.js:4-5` and response A2: no `password_hash` field is selected or returned. The claim was not supported, so I removed it.

---

## Optional, if you had time

- `GET /api/profile/4242` as Alice: status and body: 404, `{"error":"There is no colleague with that id."}`. The different response for missing ids also lets a user enumerate which ids exist.
- Which guard does each API route call in `src/routes/api.js`? `/api/profile/:id` → `requireAuth` (line 8); `/api/admin/users` → `requireAuth` (line 9).
- Which guard does the administration page call in `src/routes/admin.js`? `requireAdmin` (line 8).
- How does that source comparison explain, but not replace, your recorded responses? It explains why Alice gets 403 on the page and 200 on the API, and why any id is returned: `profileJson` never compares `req.params.id` with `req.currentUser.id`. But only the recorded responses show what the running server actually sends.
