# 009 — Accounts, roles, sessions, and site mode

## Goal

Replace the shared `X-Admin-Token` with named accounts, so that every record
superCPE keeps from here on — a review under 4.02, a completion under 6.01, a
certificate under 9.01 — names a person rather than "admin". Three roles:
participant, reviewer, admin. Sessions are server-side and revocable. Site
mode (`coming_soon` / `open`) is a logged setting on the sponsor profile, not
an environment variable, so Phase B can flip it without a deploy.

Nothing about courses, credit, questions, the player, or the assessment
changes. This feature changes *who* may reach them and *how that is recorded*.

## In scope

- `accounts` table with email, password hash, role, active flag, and a
  first-login password change.
- `sessions` table; login, logout, logout-everywhere; HttpOnly cookie.
- `require_role(...)` dependency replacing `require_admin` everywhere;
  `ADMIN_TOKEN` removed from config and `.env.example`.
- Admin account management: list, create (with a one-time initial
  password), change role, deactivate/reactivate, revoke sessions. A CLI
  command creates the first admin.
- Reviewer role: sees the course list and preview, and a record-review form
  for the courses; cannot edit content, attach, publish, or manage accounts.
- `course_reviews.recorded_by_account_id`: reviews recorded from this
  feature on name the account that entered them; the 008 literal "admin"
  rows are preserved as history.
- `site_mode` on `sponsor_profile` with a `site_mode_changes` log; public
  routes and pages gated on it; `/login` reachable but unlinked.
- Frontend: login page, session-aware nav, the four admin token forms
  deleted (the ROADMAP improvement note), account management page, site
  mode switch on the sponsor page with the change log beneath it.

## Out of scope

- Self-registration and email verification (016). Accounts are created by
  an admin or the CLI only.
- Password reset by email; no email is sent by anything in Phase A.
- Enrollment; the player and assessment stay a preview behind admin and
  reviewer (010 moves them behind enrollment).
- The coming-soon landing page and waiting list (013). This feature ships a
  one-sentence placeholder so the gate has something to render.
- Linking `subject_matter_experts` to `accounts`. 008 decided there is no
  FK between them, ever; this feature honors that (see Decisions).
- MFA, OAuth, rate limiting beyond a login attempt counter.

## Locators — read these paragraphs before writing code

- **4.02** — content review "by content reviewers other than those who
  developed the programs." 008 built the record; this feature lets the
  reviewer enter it in the first person and records who did.
- **4.02.1** — reviewers "qualified in the subject matter"; the SME record
  stays the qualification, the account is only the login.
- **6.01** — "Self-certification of attendance/completion alone is not
  sufficient." Completion verification (010) needs a participant identity
  that the server, not the participant, vouches for. This feature is that
  identity.
- **9.02** — five-year retention. Accounts are deactivated, never deleted;
  a deleted account would orphan the 9.02.2(1) records 010 attaches to it.
- **9.02.2(1)** — completion records "by individual participant." The
  account is the individual.
- **9.02.2(4)** — reviewer names and credentials retained. Unchanged from
  008 in content; the account that recorded the review is now retained
  beside it.

## Data model

New tables, one migration, CHECKs by hand.

**`accounts`**
- `id`, `email` (citext or lowercased on write; unique), `password_hash`
  (argon2id), `role` CHECK IN ('participant', 'reviewer', 'admin'),
  `display_name`, `is_active` (default true), `must_change_password`
  (default true on admin-created accounts), `failed_logins` (int, reset on
  success), `locked_until` (nullable), `created_at`, `created_by_account_id`
  (nullable self-FK; null for the CLI-created first admin),
  `deactivated_at` (nullable).
- No `deleted` column and no delete path. Deactivation is the only removal.

**`sessions`**
- `id`, `account_id` FK RESTRICT, `token_hash` (sha256 hex of a 32-byte
  random token; the raw token is only ever in the cookie), `created_at`,
  `last_seen_at`, `expires_at`, `revoked_at` (nullable), `user_agent`,
  `ip`. Unique on `token_hash`.
- A session is valid iff `revoked_at IS NULL AND expires_at > now()`; derived
  in one function, never stored as a boolean.

**`course_reviews`**
- Add `recorded_by_account_id` FK RESTRICT, nullable. Existing rows keep
  `recorded_by = 'admin'` and a null account. New rows set both: the
  account id, and `recorded_by` = the account's email at the time (a
  snapshot, so the record reads the same after a display-name change).

**`sponsor_profile`**
- Add `site_mode` CHECK IN ('coming_soon', 'open'), default
  `'coming_soon'`; the migration sets the existing row to `coming_soon`.

**`site_mode_changes`**
- `id`, `from_mode`, `to_mode`, `changed_by_account_id` FK RESTRICT,
  `changed_at`, `note` (free text, may be blank). Append-only; no update or
  delete path.

Constants in `app/constants/auth.py`: `SESSION_IDLE_MINUTES` (60),
`SESSION_ABSOLUTE_HOURS` (12), `MAX_FAILED_LOGINS` (5),
`LOCKOUT_MINUTES` (15), `MIN_PASSWORD_LENGTH` (12), `ROLES`,
`SITE_MODES`. None of these are NASBA numbers; they are ours, and the
docstring says so.

## Tasks

1. **Dependency.** Add `argon2-cffi` (justify in the changelog: a single
   maintained library for the one hashing primitive we need; `passlib` is
   unmaintained). No JWT library; sessions are rows.

2. **Models and migration** as above. Backfill nothing except
   `sponsor_profile.site_mode`.

3. **`app/services/auth.py`:**
   - `create_account(db, email, role, initial_password, created_by)` —
     lowercases email, hashes, sets `must_change_password`.
   - `authenticate(db, email, password)` — constant-time on unknown email
     (hash a dummy), increments `failed_logins`, sets `locked_until` at
     the threshold, refuses inactive accounts with the same message as a
     wrong password.
   - `open_session(db, account, user_agent, ip)` → raw token;
     `resolve_session(db, raw_token)` → account or None, bumping
     `last_seen_at` and refusing past idle or absolute expiry;
     `revoke_session`, `revoke_all_sessions(account)`.
   - `change_password(db, account, current, new)` — clears
     `must_change_password`, revokes every *other* session.
   - `set_role`, `deactivate` (revokes all sessions), `reactivate`.
   - Violations raise `AuthRuleViolation` → the same 422 `{"errors": [...]}`
     shape as every prior feature. Authentication failures are 401 with
     one fixed message; authorization failures are 403.

4. **`require_role(*roles)`** in `app/deps.py` (or wherever `require_admin`
   lives): reads the cookie, resolves the session, checks `is_active` and
   role membership, and — except on the change-password route itself —
   refuses with 403 `must_change_password` while that flag is set. Delete
   `require_admin`, `ADMIN_TOKEN`, and the `.env.example` lines.
   Role hierarchy is explicit, not implied: `admin` is listed wherever it
   is allowed. Every existing `/api/v1/admin/*` route takes
   `require_role("admin")`. The player and assessment preview routes take
   `require_role("admin", "reviewer")`.

5. **Reviewer surface.** `GET /api/v1/review/courses` (list: code, title,
   status, current-review standing) and
   `POST /api/v1/review/courses/{code}/reviews` behind
   `require_role("reviewer", "admin")`. The body names an SME id, decision,
   date, notes, `impractical_basis` — the 008 form — and the service sets
   `recorded_by_account_id`. The 008 `reviewer_is_developer` and
   `cpa_participation` findings are untouched and still gate publish; a
   reviewer recording a review does not publish anything.

6. **Site mode.** `app/services/site.py`: `get_site_mode`,
   `set_site_mode(db, to_mode, account, note)` — writes the log row and the
   profile in one transaction; no-op change (same mode) is refused as a
   422 naming the current mode. `GET /api/v1/site` (public, always):
   `{site_mode, sponsor_name}`. `PUT /api/v1/admin/site-mode` and
   `GET /api/v1/admin/site-mode/changes`.
   A `require_site_open_or_session` dependency on every public route
   (`GET /courses`, `GET /courses/{code}`, `GET /sponsor`): passes when
   `site_mode = open`, or when a valid session of any role is present;
   otherwise 404 — not 401 — so the closed site does not advertise what
   is behind it. `/api/v1/health` and `/api/v1/site` and the auth routes
   are never gated.

7. **Auth routes** under `/api/v1/auth`: `POST /login` (sets the cookie:
   `HttpOnly`, `SameSite=Lax`, `Secure` unless `settings.dev`, path `/`),
   `POST /logout`, `POST /logout-all`, `GET /me` (id, email, role,
   display_name, must_change_password), `POST /change-password`.
   Mutating routes require `Content-Type: application/json`; with
   `SameSite=Lax` and same-origin CORS that is the CSRF posture, and the
   changelog says so.

8. **Admin account routes** under `/api/v1/admin/accounts`: list, create
   (returns the initial password once, in the response, never stored in
   clear or logged), `PUT …/{id}/role`, `POST …/{id}/deactivate`,
   `…/reactivate`, `…/revoke-sessions`. An admin cannot deactivate or
   demote themself (422 naming the rule).

9. **CLI.** `python -m app.cli create-admin --email … ` prompts for a
   password (no `--password` flag; it would land in shell history), refuses
   if any admin exists unless `--force`, sets `must_change_password` false.

10. **Frontend.**
    - `src/auth/`: a session context loaded from `GET /me` on boot;
      `useSession()`; `RequireRole` route wrapper that redirects to
      `/login` and, when `must_change_password`, to `/change-password`.
    - `/login`: email, password, one error line for every failure ("Email
      or password is incorrect"). Not linked from any page.
    - `/change-password`.
    - `AdminNav` gains the account's email and a Sign out; the four token
      forms (AdminPackages, AdminSponsor inline, the shared
      `admin/TokenForm.jsx`) are deleted, and `api/client.js` sends
      `credentials: 'include'` and drops the header.
    - `/admin/accounts`: table (email, role, active, last sign-in, open
      sessions), create form showing the initial password once with a
      copy button and the sentence "This will not be shown again", role
      select, deactivate/reactivate, "Sign out everywhere".
    - `/admin/sponsor`: a Site mode card — current mode, a switch with a
      confirm step that quotes what changes ("The catalog and course pages
      become public"), an optional note, and the change log (who, when,
      from → to, note).
    - `/review`: the reviewer's home — course list with current-review
      standing; `/review/courses/:code` — read-only course facts, a Preview
      link into the existing player and assessment preview, and the
      record-review form. Reviewers see nothing under `/admin`.
    - Public `/` and `/courses*`: when `site_mode = coming_soon` and no
      session, render the placeholder ("superCPE is not yet open." — one
      sentence, sponsor name from `/api/v1/site`, nothing else). With a
      session, the existing pages.
    - `SiteMode` must never render the words "National Registry" or a
      sponsor ID; nothing in this feature reads `may_claim_registry`, and
      the changelog says so.

11. **Attempts.** `X-Preview-Id` stays for the assessment preview; nothing
    in 007 changes. Note in the changelog that 010 replaces it with the
    enrollment.

## Tests (`tests/test_auth.py`, `tests/test_site.py`)

- Login with correct credentials sets a cookie; `GET /me` returns the
  account; logout clears it and the session row is revoked.
- Wrong password, unknown email, and inactive account all return 401 with
  the identical body.
- Five failures lock the account for `LOCKOUT_MINUTES`; a correct password
  during lockout is refused; the counter resets after success.
- Idle expiry and absolute expiry each refuse a previously valid session.
- `must_change_password` blocks every route except change-password and
  `/me`; change-password clears it and revokes other sessions.
- `require_role`: participant → 403 on `/admin/*` and `/review/*`;
  reviewer → 403 on `/admin/*`, 200 on `/review/*`, 200 on the player and
  assessment preview; admin → 200 everywhere.
- A reviewer records a review: `recorded_by_account_id` is the reviewer's
  account, `recorded_by` is their email; the review appears in the 008
  history with standing computed as before; `reviewer_is_developer` still
  blocks publish if the named SME is the developer.
- Admin cannot deactivate or demote self (422). Deactivation revokes all
  sessions; the next request on an old cookie is 401.
- Initial password is in the create response and absent from the account
  row and from any subsequent response.
- Every previously admin-token-protected route refuses without a session
  (401) and with a participant session (403); the test walks the router
  table so a new route cannot be added unguarded.
- Site mode: default `coming_soon`; public `GET /courses` is 404 with no
  session and 200 with a participant session; after `set_site_mode(open)`
  it is 200 with no session; the log row records from/to/who/when; setting
  the same mode is 422.
- `GET /api/v1/site`, `/health`, and `/auth/login` are reachable in both
  modes with no session.
- No response under `/api/v1/site` or `/auth` contains the string
  "National Registry" (walk the payloads, as 003 and 008 did).
- All prior tests pass with their fixtures switched from the token header
  to a logged-in admin client; the count goes up, not down.

## COMPLIANCE.md rows

Add:

| 4.02 | (existing row; append to Where in code) | 009 | `recorded_by_account_id` on `course_reviews`; reviewers enter reviews at `POST /api/v1/review/courses/{code}/reviews` behind `require_role("reviewer", "admin")` | The reviewer's login is not their qualification; the SME record is. Nothing verifies that the person holding the account is the person named in the SME record. |
| 6.01 | "Self-certification of attendance/completion alone is not sufficient." | 009 | `accounts`/`sessions`; `require_role` in `app/deps.py` — the server-vouched participant identity 010's completion record hangs on | Nothing is completed yet; 010 attaches attempts and completions to the account through the enrollment. |
| 9.02 | (existing row; append to Where in code) | 009 | Accounts are deactivated (`deactivated_at`), never deleted; `sessions` and `course_reviews` FK RESTRICT to `accounts` | Retention period still not a constant (011). |
| 9.02.2(4) | (existing row; append) | 009 | Who recorded each review is retained beside the reviewer's name and credentials | — |

Say in the changelog that 4.02.1 and 9.02.2(1) rows are unchanged.

## Acceptance

1. `alembic upgrade head` on the 008 database adds the tables and sets
   `site_mode = coming_soon`; `python -m app.cli create-admin` creates the
   first admin; `ADMIN_TOKEN` no longer appears anywhere in the repo
   (`grep -r ADMIN_TOKEN` is empty, `.env.example` included).
2. With no session, every `/admin/*` route is 401, every public course
   route is 404, `/api/v1/site` reports `coming_soon`, and the frontend
   `/` shows the one-sentence placeholder with no link to `/login`.
3. Sign in as admin at `/login`; every admin page works as it did under
   the token; the token form is gone from all four places.
4. Create a reviewer account; sign in with the initial password; forced to
   `/change-password`; afterwards `/review` lists ASC842-PCX; record a
   review naming an SME; the admin course page's review history shows it
   with the reviewer's email as recorded-by.
5. Create a participant account; sign in; the catalog and course pages
   render; `/admin` and `/review` redirect away.
6. Flip site mode to `open` with a note; sign out; the catalog is public;
   the log shows the change; flip back; it is closed again.
7. Deactivate the reviewer; their open session is dead on the next
   request; the review they recorded is still in the history.
8. `pytest`: all tests pass; the count exceeds 115.

## When done

- Changelog entry per CLAUDE.md, including the `argon2-cffi` justification,
  the CSRF posture, and the "no SME↔account FK" decision restated.
- COMPLIANCE.md rows above.
- Strike the "admin token form is now duplicated in four places" line from
  ROADMAP.md's improvement notes — this feature retires it.
- List anything out of scope you hit, especially: any place that still
  reads `recorded_by` as a literal, and whether 010 should require the
  reviewer to hold a `reviewer` account (today an admin may still enter a
  review on a reviewer's behalf, and the record shows that it was the admin
  who did).
