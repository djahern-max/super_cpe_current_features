## 029 — Annual subscription
Shipped: YYYY-MM-DD  <!-- fill in when acceptance 7, 8, and 9 pass -->

**What changed**
- Constants: `backend/app/constants/subscription.py` —
  `SUBSCRIPTION_PRICE_CENTS = 14900` (ours), `SUBSCRIPTION_PERIOD_DAYS =
  365` (ours; Stripe's yearly interval is the schedule), and the Stripe
  status lists the two new CHECKs mirror.
- Config: `STRIPE_SUBSCRIPTION_PRICE_ID` joined 012's all-or-nothing
  `STRIPE_VARS` (`backend/app/config.py`), so 018's
  `payments_not_configured` open-gate finding covers it; the "swap all
  three in one edit" wording became "all four" in config, readiness,
  preflight, `.env.example`, and OPERATIONS.md. `preflight`
  (`backend/app/cli.py`) retrieves the Price once through the boundary
  and, on an open site, refuses a deploy whose Price amount or currency
  differs from the constant (naming both numbers) or whose Price cannot
  be read; while coming-soon it prints a note. Not configured is silent
  (the open gate owns that).
- Data model, migration `d4b8e2a7c029`: `accounts.stripe_customer_id`
  (nullable, unique, set once); `subscriptions` (status CHECK, period,
  `cancel_at_period_end`, `canceled_at`, `credit_applied_cents`,
  `livemode`, plus `checkout_url` for live-session reuse);
  `subscription_invoices` (status CHECK, amount, currency, period,
  `livemode`, plus `stripe_payment_intent_id` so a refund can find the
  row); `payments.credited_to_subscription_id`; `enrollments.source`
  CHECK gains `subscription`. All hand-written; docstrings say financial
  record, never deleted, outlives `RETENTION_YEARS`.
- Boundary: `services/stripe_gateway.py` gained `create_customer`,
  `create_credit_coupon`, `create_subscription_checkout_session`
  (`mode="subscription"`, `payment_method_collection="always"`,
  `discounts=[{"coupon": …}]`, metadata on both the session and
  `subscription_data`), `retrieve_subscription`, `create_portal_session`,
  and `retrieve_price`. No second client.
- `services/subscriptions.py`: `current` (status `active` and
  `current_period_end` ahead, both as Stripe last reported; grace
  decided once, as none — `past_due` is not current), `enroll_subscriber`
  (010's constructor, `source="subscription"`, no Stripe call),
  `subscription_enrollable` (the card's "Enroll again (included)"),
  `credit_payments`/`credit_cents` (paid, uncredited, capped),
  `start_subscribe` (customer ensured and committed first, coupon only
  when credit > 0, row written `incomplete` before the URL returns,
  live incomplete session reused), `portal_url`, and the webhook
  handlers. Credit is consumed only by the completed-session handler,
  from `total_details.amount_discount`, with the payment FKs in the
  same transaction.
- Webhook: `payments.handle_event` stays the one dispatch point.
  `checkout.session.completed` with `mode == "subscription"` links the
  Stripe subscription id, re-stamps `livemode`, records the discount and
  the FKs, and copies status and period from one `retrieve_subscription`
  (the session object carries neither); `customer.subscription.updated`
  / `.deleted` copy status, period, `cancel_at_period_end`,
  `canceled_at` (found by Stripe id, else by the row id in the metadata,
  so an update that outruns the completion still lands); `invoice.paid`
  upserts the invoice row and syncs the period from the lines;
  `invoice.payment_failed` upserts the row as `open` and nothing else;
  `charge.refunded` on a subscription invoice marks it `refunded` and
  stops (018's `_handle_refunded` tries payments first, then invoices,
  then logs unknown). Orphans log loudly and answer 200; each new type
  is idempotent through the existing event table, recorded and committed
  with the handler's changes in one transaction.
- 028 join: `renewal_refusals` in `services/enrollments.py` — "or any
  prior `subscription`-sourced enrollment for the course" beside
  `has_paid`. 018 join: `start_checkout` refuses a current subscriber
  ("your subscription covers this course; enroll directly"). The renew
  route refuses a current subscriber the same way (the subscriber's
  enroll takes precedence), and the card never offers both.
- Routes: `POST /api/v1/courses/{code}/enroll` (201 the card; 422 per
  condition; 404 unknown course); `GET/POST /api/v1/subscribe`,
  `GET /api/v1/subscribe/me`, `POST /api/v1/subscribe/portal`,
  `GET /api/v1/subscribe/{session_id}/status` (owner-only) in
  `routers/subscribe.py`, all behind the site gate; `GET
  /api/v1/admin/subscriptions` in `routers/admin_subscriptions.py`.
  `MeOut` gained `subscription_current` (derived per read; false for
  non-participants); `MyEnrollmentSummary` gained
  `subscription_enrollable`; `AdminPaymentOut` gained
  `credited_to_subscription_id`.
- Frontend: `/subscribe` (offer from the payload, credit line only when
  > 0, both policy links, Subscribe → Stripe, sign-in links for
  visitors), `/subscribe/success` (polls, refreshes the session, links
  the catalog, 018's ~30s honest-delay state), `/account` Subscription
  section (none / current, renews on / cancels on / past due with
  "Update payment method" / lapsed; credit consumed; Manage subscription
  → Customer Portal), header Subscribe link for a participant without a
  current subscription (only while `site_mode` is open; null in
  coming-soon and under `/admin` as before), footer Subscribe link at
  open, course page two-choice section / included-enroll button /
  "Enroll again (included)" via the shared `SubscriptionEnroll`
  component (also on the `/my/courses` card, checked before 028's
  renewal), `/admin/subscriptions` (table, invoices, both flags, Void
  beside each active enrollment of a refunded subscription, Stripe link
  honoring `livemode`), `/admin/payments` credited marker, AdminNav
  link.
- Docs: OPERATIONS.md "Subscriptions (029)" (Product/Price, scopes,
  four new event types on both endpoints, portal configuration, dunning
  emails, Stripe Tax note, CLI walkthrough with its log table, the
  subscription refund runbook) and the Payments (018) section's counts
  and event list; four COMPLIANCE.md rows appended; `.env.example`.
- Tests: backend 481 → 523 (`tests/test_subscriptions.py`, 38, plus 4
  preflight price tests; 018's boot all-or-nothing test and 009's
  exact `/me` payload test extended); frontend 82 → 104 (header,
  course page, footer extended; Subscribe, Account, SubscribeSuccess
  new). Router walk green; `INTENTIONALLY_PUBLIC` untouched; every new
  route 404s anonymously in coming_soon; no new response carries a
  course fact or "National Registry".

**Task 0 answers**
1. `create_checkout_session` did not accept a mode — `mode="payment"`
   was hard-coded with inline `price_data`. Rather than a mode flag on
   a function whose line items, customer handling, and return shape
   differ, a separate `create_subscription_checkout_session` was added
   to the same boundary. `stripe==12.4.0` (API version
   `2025-07-30.basil`) supports Checkout `subscription` mode,
   `payment_method_collection="always"`, and `discounts=[{"coupon":
   …}]`, checked in its `Session.CreateParams`. Basil also moved
   `current_period_*` to the subscription item, `invoice.subscription`
   under `parent.subscription_details`, `invoice.payment_intent` under
   `invoice.payments`, and removed `charge.invoice`; the handlers read
   both shapes and copy whichever Stripe sends.
2. 018 created Stripe guest customers per Checkout (`customer_email`,
   no Customer object); no account carried a customer id. This feature
   adds `accounts.stripe_customer_id`, creates the Customer on the
   first subscription checkout, and never backfills guests.
3. `handle_event` dispatched on `checkout.session.completed`,
   `checkout.session.expired`, and `charge.refunded`; every other type
   was logged by name at INFO and answered 200 without a record. New:
   `checkout.session.completed` (subscription mode),
   `customer.subscription.updated`, `customer.subscription.deleted`,
   `invoice.paid`, `invoice.payment_failed`, and `charge.refunded` on a
   subscription invoice. Each is idempotent by event id through
   `stripe_webhook_events` (the replay check runs before dispatch; the
   new handlers record the event and commit in the same transaction).
   `customer.subscription.created` stays unhandled by design (below).
4. `renewal_refusals` in `backend/app/services/enrollments.py`, the
   `if not has_paid(db, account, course):` line; it became `if not
   (has_paid(...) or any(e.source == "subscription" for e in rows))`.
5. 026's key had Checkout Sessions Write and Payment Intents, Charges,
   Refunds Read. Missing for Billing, to be added by the operator:
   Customers Write, Subscriptions Read, Coupons Write, Billing Portal
   Write (sessions), Invoices Read, and Prices/Products Read (for
   preflight's `Price.retrieve`). Nothing was widened in code.
6. `Registration` in `frontend/src/pages/CoursePage/CoursePage.jsx`
   (018, revised by 028 for the renewal button); it gained the second
   option, the included-enroll button, and the "Enroll again" branch.

**Standards touched**
- 9.02.2(3) — read on printed page 24: "no longer than one year from
  the date of purchase or enrollment" for individual courses. A
  subscription is not an enrollment; each course enrolled under it
  carries its own year from enrollment through 010's constructor, and
  a subscription ending touches no enrollment. COMPLIANCE row appended.
- 8.01 items 8 and 9 — read on printed page 20; 8.01.1 on page 21
  ("formalized, published, and made available"). The subscription is a
  fee; `/subscribe` discloses price, term, renewal, cancellation, and
  links both policies. COMPLIANCE row appended; the operator's new
  policy versions are pending (below).
- 9.02 — read on printed page 22. `subscriptions`,
  `subscription_invoices`, and the credit FK are financial records
  retained beyond `RETENTION_YEARS`; a lapse deletes nothing. Two
  COMPLIANCE rows appended (retention; amounts as Stripe reported).

**Decisions**
- **Reversal of 018's "the webhook is the sole creator of
  enrollments":** a current subscriber's enroll is a click —
  `POST /courses/{code}/enroll` calls 010's constructor with no Stripe
  call and no payment row. The webhook remains the sole creator of
  *purchased* enrollments; `services/payments.py`'s docstring says so.
- **Reversal of 018's "the refund policy covers course sales":** it
  now also covers subscriptions. The webhook marks, the admin decides
  (018's rule kept); the admin subscriptions view raises two loud flags
  (refunded-with-current-subscription, refunded-with-active-enrollments);
  cancelling in Stripe is the admin's separate act.
- **Accepted sponsor exposure:** completed enrollments and issued
  certificates are immutable 9.02 records a refund cannot unmake. A
  participant who subscribes, completes courses, and is then refunded in
  full keeps that credit. Recorded here, and in the refund runbook.
- Status and period are never taken from the Checkout Session object
  (it carries neither): the completed-session handler retrieves the
  subscription once through the boundary and copies from it; if the
  retrieve fails the row stays `incomplete`, loudly, until
  `customer.subscription.updated` arrives. This keeps the registered
  event list at the four the spec named and keeps "copy, never infer"
  honest. `customer.subscription.created` is therefore unhandled.
- Two columns beyond the spec's list: `subscriptions.checkout_url`
  (the live-session reuse the spec asks for needs the URL, as
  `payments` keeps it) and `subscription_invoices.stripe_payment_intent_id`
  (`charge.refunded` carries a payment intent and, under basil, no
  `invoice`; without it a refund could not find the row).
- The status CHECK lists all eight Stripe subscription statuses, not
  the six the spec expected: `trialing` and `paused` cannot arise from
  this configuration, but a CHECK that refused a status Stripe sent
  would 500 the webhook and make Stripe retry forever. Neither is ever
  current.
- `past_due` is not current: no grace period, decided in `current`'s
  docstring. Stripe's dunning and the participant's "update your
  payment method" state are the whole handling.
- The header and footer Subscribe links render only while `site_mode`
  is open, not merely while the header's face is open: a signed-in
  participant on a coming-soon site sees the header (025) but not an
  offer whose page names a price.
- `subscription_current` rides on `/auth/me` (derived per read) so the
  header needs no second request; the success page calls the session's
  `refresh()` on landing.
- A current subscriber's expired course is re-started through the
  subscriber's enroll; the 028 renew route refuses them by name and the
  card never offers both buttons. A lapsed subscriber gets 028's
  renewal for courses started under the subscription.
- The `/admin/subscriptions` page reuses `AdminPayments.module.css`
  and the existing void endpoint; no new admin action exists.
- `tests/test_payments.py`'s `pay` helper reuses one event id and one
  intent id; the new suite's `pay_course` gives each purchase its own
  so multi-course credit can be computed. The local `supercpe_test`
  database was dropped and rebuilt for the new columns and CHECKs
  (test data only); the migration was applied to the local dev
  database with `alembic upgrade head` and the CHECKs inspected.
- Drafted registration/attendance policy wording (operator publishes as
  a new version, appended to 028's text):
  > Annual subscription. A subscription begins on the date of purchase,
  > runs for one year, and renews automatically at the end of each year
  > until cancelled. While a subscription is current, the subscriber may
  > enroll in any published course at no additional charge. Each course
  > enrolled in under a subscription is a separate enrollment with its
  > own one-year completion window from the date of enrollment; the end
  > of a subscription does not shorten or extend any enrollment already
  > started. If a subscription lapses, every enrollment, completion, and
  > certificate is retained, and the subscriber may subscribe again at
  > any time. Course purchases made before a first subscription are
  > credited, dollar for dollar and once, against the first subscription
  > payment, up to the subscription price.
- Drafted refund and cancellation policy wording (operator publishes as
  a new version):
  > Courses. A course purchase is refundable in full on request, with no
  > questions asked. When a purchase is refunded, access to that
  > enrollment ends; a completed course and its certificate stand.
  >
  > Subscriptions. The current subscription payment is refundable in
  > full on request, with no questions asked; refunds are never
  > pro-rated. A subscriber may cancel at any time from their account;
  > access continues to the end of the paid period and the subscription
  > does not renew. When a subscription payment is refunded, the
  > subscription is cancelled and access to enrollments still in
  > progress under it ends. Courses completed and certificates issued
  > before the refund stand; they are permanent records.
- The 018 restricted-key scopes the operator must add: Customers Write,
  Subscriptions Read, Coupons Write, Billing Portal Write, Invoices
  Read, Prices/Products Read (OPERATIONS.md "Subscriptions (029)").

**COMPLIANCE.md**
- Updated: four rows appended — 9.02.2(3); 8.01 items 8 and 9 with
  8.01.1; 9.02 (retention); 9.02 (018 payment row, amounts as reported).

**Known gaps**
- Acceptance 7 (Stripe test-mode walkthrough with the CLI), 8 (publish
  both policy versions), and 9 (deploy and repeat 1 and 2 on
  production in test mode) are the operator's. The walkthrough log in
  OPERATIONS.md says "Not yet run"; the 8.01 COMPLIANCE row records the
  policy versions' effective dates once 8 passes.
- Acceptance 1–5 were proven at the API layer by the backend suite and
  the page states by the frontend suite; the browser walkthrough of the
  same steps was not performed in the build session.
- Preflight on an open site now needs Stripe reachable (one
  `Price.retrieve`); a transient outage refuses a deploy, as 026's
  bucket-versioning check does. Coming-soon is unaffected.
- The webhook's completed-session handler makes one outbound Stripe
  call. If Stripe is unreachable at that moment the row stays
  `incomplete` until the next subscription event; no retry of our own.
- Stripe Tax on subscriptions is a note in OPERATIONS.md, not built.
- Out of scope, reported not built: monthly or other intervals; team or
  multi-seat plans; gifting; plan switching, trials, promo codes, any
  coupon but the per-account credit; automatic voiding or automatic
  cancellation on refund; any billing email of superCPE's own; a grace
  period for `past_due` (decided: none); Google sign-in (030);
  `customer.subscription.created` handling; automatic tax.
- pyflakes reports two pre-existing unused imports
  (`app/routers/checkout.py`, `app/models/enrollment.py`) untouched by
  this feature; oxlint's remaining warnings are all on untouched files.
