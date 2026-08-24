# Security notes for go-live

## Database rules — `database.rules.json`

This is a draft ruleset for **Firebase Realtime Database → Rules**. Paste it into
the Firebase console (or `firebase deploy --only database` if you use the CLI)
and run it through the console's **Rules Playground** before publishing — I
could not reach Firebase from this environment to test it against the live
project, so treat it as a reviewed draft, not a verified one.

### What it actually protects

- Closes the default-deny root, so nobody can write new top-level nodes outside
  `config` / `slots` / `candidates`.
- Once a candidate record exists, its personal/exam/resume fields (name,
  email, phone, marks, ICAI number, resume, etc.) become **immutable to public
  writers** — nobody can overwrite another applicant's data.
- A new candidate record must have its `id` field match the database key, so
  App IDs can't be spoofed to collide with or shadow another applicant.
- After creation, the public `status` field can only ever be moved to
  `slot_booked` or `rescheduled` — the two values the booking/reschedule flow
  sets. It can no longer be pushed straight to `shortlisted`, `accepted`, etc.
  by a direct API call.

### What it deliberately does *not* protect — and why

**This app has no real authentication anywhere.** `admin.html`'s login is a
password typed into a JS `if` statement, checked entirely in the browser. From
Realtime Database's point of view, the admin's browser and a random visitor's
browser are indistinguishable — there is no `auth` token to key rules off.

That means rules **cannot** gate admin-only actions without also locking the
admin panel out of itself:

- `config` (firm name, EmailJS keys, offer letter template, portal URL) —
  writable by anyone. Admin settings has to be able to save it.
- `slots/*` — writable by anyone (create/edit/delete). Admin needs to manage
  slots, and the booking flow needs to append/remove IDs in `bookedBy`.
- `candidates/*/status` to `shortlisted` / `waitlisted` / `accepted` /
  `rejected`, plus `adminNotes`, `interviewer`, `score`, `docs` — all still
  writable by anyone. These are exactly the actions the admin panel performs,
  and there's currently no way to say "except the admin" in rules.

So: this ruleset meaningfully reduces the blast radius of someone poking the
open API key (they can no longer forge acceptances, corrupt other people's
applications, or trash arbitrary data), but it does **not** make the admin
panel a real access-controlled system. Anyone who opens devtools on the live
site and calls the Firebase SDK directly can still shortlist/accept/reject
candidates, edit scores, or rewrite firm settings — same as today.

### The actual fix: Firebase Authentication

Closing that gap needs the admin to sign in with real Firebase Auth (e.g.
email/password), and rules that check `auth != null` on the admin-only paths
above. That's a real, if fairly small, follow-up:

1. In the Firebase console: **Authentication → Sign-in method → enable
   Email/Password**, then **Authentication → Users → add your admin user**
   (email + password of your choosing) — I can't do this step, it needs your
   console access.
2. Code change in `admin.html`: replace the `localStorage` password check
   with `firebase.auth().signInWithEmailAndPassword(email, password)`, and
   swap every remaining `".write": true` above for `".write": "auth != null"`.

I didn't make that code change yet since it changes the login flow and
depends on you completing step 1 first — happy to wire it up whenever you're
ready to do the console side.
