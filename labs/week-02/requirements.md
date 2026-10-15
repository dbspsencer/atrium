# Week 2 security requirements

Your name: 20099070
Date: 01/10/2026

Fill this in as you work, rather than at the end. Where you are unsure, write
that you are unsure and say why. A sentence you can support is worth more than a
confident one you cannot.

---

## 1. What this application is

Three or four sentences, in your own words, describing what Atrium does and who
uses it.

Atrium is a small website for staff inside a company. Staff sign in with a
username and password. They can search the staff directory, read shared
documents, and change their own email, department and bio on "My profile".
There are two types of user: normal members (Alice and Bob) and one admin
(Morgan), who can also see a page listing every staff account.

**Where the assistant's explanation did not match the application.** Anything you
checked and found different, however small. Write "nothing found" if that is the
honest answer.
Answer **The README says the roles are "staff" and "administrator", but the app
shows them as "member" and "admin". Each document says who it is "kept by", but
any signed-in person can open every document. The admin page says IT Services
handles account changes, but there is no page in Atrium to change an account,
role or password.**

## 2. What is worth protecting

Four assets. For each one, say what it is and what it would cost if it were seen,
changed or unavailable. Write the cost so that somebody outside the team could
understand it.

| Asset | What it costs if this goes wrong |
| ----- | -------------------------------- |
| Staff contact details (name, department, email) on `/directory` | Someone with the full list could send fake emails to every staff member by name to trick them. If the page is down, staff cannot find who to contact. |
| Passwords, used at `/login` | If a password is seen, someone can sign in as that person. If it is changed by someone else, the real person is locked out. |
| The list of staff accounts and who is admin, on `/admin/users` | An attacker would know every username and which one is the admin, so they know exactly which account to attack. |
| Each person's own profile, on `/profile` | If someone changes Bob's email to their own, colleagues would send Bob's work emails to the attacker without knowing. |

## 3. The requirements

Four sentences, in your own words. Each one should say what is not allowed and to
whom.

1. Whatever anyone types into the search box on `/directory`, the results must
   only show the name, department, email and role of matching colleagues, and
   never usernames or passwords.
2. Only a signed-in admin can get the list of staff accounts, from the
   `/admin/users` page or from `/api/admin/users` that the page uses; a normal
   member or someone not signed in must get nothing.
3. A signed-in person can change only their own email, department and bio on
   `/profile`; they must not be able to change another person's details or
   their own role.
4. No page, API or error message in Atrium may ever show anyone a password or
   the stored password hash, not even the admin.

**Which of these did you write yourself, and which started as a draft from your
assistant?** Say plainly. Both are fine.

All four started as drafts from my AI assistant. I gave it my four assets, then
checked each draft against the pages in Atrium and rewrote them in simpler
words.

## 4. One I rejected or rewrote

- The original sentence: "Users must only be able to modify data they are
  authorised to modify, and all changes must be validated appropriately."
- My version: A signed-in person can change only their own email, department
  and bio on `/profile`; they must not be able to change another person's
  details or their own role.
- Which test it failed, and why: (specific / somebody could check it / about this application)
  It failed all three. It is not specific because it does not say what data or
  who is "authorised". Somebody could not check it without asking me what
  "authorised" means. It is not about Atrium because it names no page; it could
  be about any app. My version names the page and the fields.

## 5. How somebody would check one of these

Pick one requirement. Write the steps for a person who has never seen Atrium and
cannot ask you anything.

- The requirement: Requirement 2. Only a signed-in admin can get the list of
  staff accounts, from `/admin/users` or `/api/admin/users`.
- Sign in as: `alice.nolan` with password `SpringRiver44` (a normal member).
  Then `morgan.doyle` with password `QuietHarbour08` (the admin) to compare.
- Steps:
  1. Run `npm start` in the atrium folder and open `http://localhost:9090`.
  2. Sign in as `alice.nolan`.
  3. Go to `http://localhost:9090/admin/users` and write down what you see.
  4. Go to `http://localhost:9090/api/admin/users` and write down what you see.
  5. Click "Sign out", then repeat steps 3 and 4 without signing in.
  6. Sign in as `morgan.doyle` and repeat steps 3 and 4 to see what the list
     looks like when it is allowed.
- What result would mean the requirement is met: Alice sees "Administrators
  only." at both addresses, with no names or usernames. When not signed in,
  you are sent to the sign-in page or get an error, with no names. Only Morgan
  sees the list.
- What result would mean it is not met: Alice, or someone not signed in, sees
  any name, username or role from the staff list at either address.

---

## Optional, if you had time

Your four requirements in order, most important first, with one sentence each on
why it is in that position.

1. Requirement 4 (passwords never shown): a stolen password lets someone become
   that person, which breaks every other rule.
2. Requirement 3 (change only your own profile): changing someone's email
   quietly sends their work emails to an attacker.
3. Requirement 2 (admin account list): it tells an attacker which account to
   target, but does not let them in by itself.
4. Requirement 1 (directory search): signed-in staff can already see emails and
   departments, so the extra harm is smaller unless usernames or passwords leak.
