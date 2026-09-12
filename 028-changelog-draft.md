## 028 — Unlimited re-takes and free renewal after expiry
Shipped: YYYY-MM-DD  <!-- fill in when acceptance 6 and 7 pass -->

**What changed**
- `RETAKES_ALLOWED` in `backend/app/constants/assessment.py` is
  `None` (unlimited), typed `int | None`, with the comment rewritten:
  6.01.2 leaves the count to the sponsor; `None` is unlimited; an
  integer is re-takes after the first sitting per enrollment; every
  attempt is retained whatever the value; 011's policy text renders
  whichever it is.
- `enrollments.retakes_remaining` returns `None` when unlimited;
  `assessment.start_for_enrollment` skips the sittings check on `None`
  (the other refusals — completed, expired, voided, unanswered review
  questions, open attempt — are unchanged and re-proven). The failed
  result, the assessment info (enrollment and preview), and the
  enrollment card carry `retakes_allowed` and `retakes_remaining` as
  nullable, with `retakes_unlimited: bool` beside them.
- `policies.retake_policy_text()` branches: unlimited renders "A
  participant may re-take the qualified assessment as many times as
  needed."; an integer renders the 010 sentence. The how-it-works page
  (`services/instructions.py`) branches the same way and gains one
  sentence on renewal at no charge.
- Free renewal: `enrollments.renewal_refusals` (derived from `paid`
  payment rows and enrollment statuses, never stored), `renewable`
  (per expired card: the most recent enrollment on the course, and
  eligible), and `renew` (the one constructor with
  `source="renewal"`, no Stripe call, no payment row).
  `POST /api/v1/courses/{code}/renew` in `routers/courses.py`, behind
  the site gate and the participant role, one 422 line per failed
  condition (no paid purchase; still active; already completed;
  voided; no enrollment at all), 404 for an unknown course, 422 for an
  unpublished one; answers the new enrollment as the `/my/courses`
  card (`my.enrollment_summary`, now public).
- `enrollments.source` CHECK gains `renewal`: migration
  `c2d8e5f1a028_enrollment_source_renewal.py`, written by hand; no
  other schema change; no FK from the renewal to the payment or the
  expired enrollment.
- 018 narrowed: `payments.start_checkout` refuses a participant with a
  prior `paid` row for the course ("you have already purchased this
  course; renew it from the course page instead of paying again"); the
  active-enrollment refusal drops its "it can be purchased again after
  it expires" clause. Pending-session reuse is unchanged.
- Frontend: `MyEnrollmentSummary.renewable` drives a shared
  `RenewEnrollment` component (`frontend/src/components/RenewEnrollment/`)
  — "Start a new enrollment (no charge)" — on the course page's
  Registration section (re-renders enrolled in place), the `/my/courses`
  card and the enrollment page's next action (both navigate to the new
  enrollment). A shared `retakeLabel` in `pages/MyLesson/nextStep.js`
  drops the "(N left)" count when `retakes_remaining` is null; the
  failed result shows score, threshold, and "Re-take the assessment"
  with no count under the unlimited policy; the intro sentence says the
  assessment may be re-taken as many times as needed. `RetakesExhausted`
  and `isExhausted` are untouched and render only at
  `retakes_remaining === 0`. The admin enrollments table gains a Source
  column.
- Tests: backend 465 → 481 (`tests/test_renewal.py` new: the
  eligibility matrix, pinning to current packages, retention of the
  expired enrollment's answers and attempts, admin sources, only the
  latest expired card renewable, the gate; `set_retakes_allowed` in
  `conftest.py` patches every module that binds the constant for the
  finite tests). Frontend 73 → 82 (`CoursePage.test.jsx` and
  `MyCourses.test.jsx` new; Assessment and nextStep tests extended).
  Router walk green; `INTENTIONALLY_PUBLIC` untouched.

**Task 0 answers**
1. `enrollments_service.enroll` accepts `source="admin"` (default; the
   admin enrollments router) and `"purchase"` (the Stripe webhook).
   Constrained by CHECK `ck_enrollments_source` in
   `backend/app/models/enrollment.py` and migration `b3e9c41a7f52`;
   `ENROLLMENT_SOURCES` mirrors it. `renewal` needed a hand-written
   migration (above).
2. `retakes_remaining` was consumed by: `assessment.start_for_enrollment`
   (`== 0` refusal), `assessment.result` (failed payload),
   `routers/my.py` `_unavailable_reasons` (`== 0`), `_summary_fields`
   (`retakes > 0` gating `assessment_available`, and the field),
   `get_assessment` (`retakes > 0` gating `available`); schemas
   `MyAssessmentInfo` and `MyEnrollmentSummary` typed it `int`, and
   `AssessmentInfo.retakes_allowed: int`; `services/instructions.py`
   interpolated `RETAKES_ALLOWED` into the how-it-works markdown;
   `policies.retake_policy_text`. Frontend: `Assessment.jsx` (failed
   branch and intro), `RetakesExhausted.jsx`, `MyCourse.jsx`,
   `MyCourses.jsx`, `nextStep.js`, `exhausted.js`. The `> 0` gates became
   `!= 0` so `None` passes; everything else handles null.
3. Tests asserting the exhausted message or an integer:
   `test_completion.py::test_start_refused_when_retakes_exhausted`
   (looped `1 + RETAKES_ALLOWED`, asserted the number and the word
   `RETAKES_ALLOWED` in the message) and
   `::test_failed_enrollment_result_carries_no_feedback` (equality to
   the constant); `test_policies.py::test_retake_text_carries_the_enforced_numbers`
   and `::test_how_it_works_numbers_match_the_constants`
   (`f"{RETAKES_ALLOWED} times"`); `test_assessment.py::test_failed_result_payload_has_no_feedback`
   (equality, still true with `None`). Frontend: `Assessment.test.jsx`
   (finite fixtures) and `nextStep.test.js` ("(3 left)"). The first
   four became finite-policy tests under `set_retakes_allowed`; the
   frontend ones keep their finite fixtures and gained unlimited cases.
4. 018 on an expired participant calling checkout: refused only on an
   *active* enrollment, so it minted a second Stripe session and a second
   `payments` row, and the webhook created a second `purchase`
   enrollment (`test_expired_enrollment_allows_a_fresh_purchase` proved
   exactly this). Confirmed before narrowing.
5. Policy text lives in `policy_versions` only; no seed. The test factory
   `publish_test_policies` in `tests/conftest.py` publishes "Test {kind}
   policy." for each kind. The admin path is `POST /api/v1/admin/policies`
   (`policies.admin_router` → `policies_service.publish`, append-only,
   effective-dated), reached from the admin sponsor page. Nothing in
   code reads the body, so the factory does not need a policy that
   mentions renewal.

**Standards touched**
- 6.01.2 — "The number of re-takes … is at the sponsor's discretion"
  (page 13); unlimited chosen; sub-ii-b-1 on page 14 (no feedback on a
  failed assessment) untouched and re-asserted by the same tests.
- 9.02.2(3) — expiration "no longer than one year from the date of
  purchase or enrollment" (page 24); a renewal is a new enrollment with
  its own year; nothing is extended.
- 8.01 items 8 and 9 — registration and refund policies (page 20); the
  registration policy must state the renewal rule (operator, below);
  the refund policy is unchanged.
- 8.01.1 — policies "formalized, published, and made available" (read
  on page 21, not page 20 as the spec said); the rule lives in the
  published policy, linked, not restated.

**Decisions**
- **Reversal of 018's "re-purchase allowed after expiry":** checkout is
  for a first purchase of a course only; a participant with a `paid`
  row renews instead. The 018 test was rewritten as
  `test_renewal_after_expiry_checkout_refused`, and the 018
  active-enrollment refusal no longer promises a later re-purchase.
- `retakes_unlimited: bool` was added beside the nullable numbers: the
  failed result already used an absent `retakes_remaining` to mean "a
  preview attempt, no enrollment", so null alone could not tell the
  browser "unlimited" from "preview". The frontend keys the Re-take
  button on the flag and the enrollment case on the key's presence.
- `renewable` is true only on the participant's most recent enrollment
  on the course; an older expired card never offers the button.
- Eligibility lives in `services/enrollments.py` (it owns the
  constructor) and imports the `Payment` model directly; the payments
  service already imports the enrollments service, so the reverse
  import would have been circular.
- A renewal pins the course's *current* published packages. If the
  course was re-reviewed and republished since the expired enrollment,
  the participant reads the current guide; that is correct — the
  expired enrollment keeps its own pin.
- 029 subscriptions: a subscription source will be a second qualifying
  condition beside `has_paid` in `renewal_refusals`; one comment names
  it, nothing is built.
- How-it-works (code, 4.05.3 instructions) gained one sentence on
  renewal at no charge because a "None times" rendering would have been
  a bug and the page must not contradict the policy the operator
  publishes.
- Drafted registration/attendance policy wording (operator publishes as
  a new version):
  > Enrollment. An enrollment in a course begins on the date of purchase
  > or enrollment and expires one year later; the qualified assessment
  > must be completed before the enrollment expires. An enrollment is
  > never extended. A participant who purchased a course and did not
  > complete it before the enrollment expired may start a new one-year
  > enrollment in the same course at no additional charge from the
  > course page. The new enrollment begins with no review questions
  > answered and no assessment attempts, and uses the course's currently
  > published materials.
  >
  > Attendance. There is no attendance requirement; a self-study course
  > is completed by answering every review question and passing the
  > qualified assessment.
  >
  > Re-takes. A participant who does not pass the qualified assessment
  > may re-take it as many times as needed within the enrollment period.
  > No feedback on individual questions is given for an assessment that
  > was not passed.
- Local test database: `tests/conftest.py` builds it once with
  `create_all` and never alters an existing table, so the changed CHECK
  required dropping `supercpe_test`; done as a routine step (test data
  only).

**COMPLIANCE.md**
- Updated: four rows appended — 6.01.2 (re-takes), 9.02.2(3), 8.01
  item 8, 8.01 item 9.

**ROADMAP.md**
- The "028 — exhausted enrollments" improvement note 027 added is struck
  and superseded by this feature's line (append only).

**Known gaps**
- Acceptance 6 (operator publishes the new registration/attendance
  policy version) and 7 (production deploy and re-run of 1 and 2) are
  the operator's; the 8.01 item 8 COMPLIANCE row records the new
  version's effective date once 6 passes.
- Acceptance 1–4 were proven at the API layer by the backend suite
  (five failures then a permitted start; renewal with a fresh year, no
  answers, no attempts; checkout refused with the already-purchased
  message and a first purchase of another course allowed; refunded then
  voided then expired shows no renewal and checkout is allowed). The
  browser walkthrough of the same steps was not performed in the build
  session.
- `assessment.result` for a *preview* attempt under the unlimited policy
  reports `retakes_unlimited: true`; the preview never counted sittings
  anyway.
- The local `supercpe_test` database was dropped and rebuilt; any other
  developer's test database needs the same once (the CHECK is not
  altered by `create_all`).
- Out of scope, reported not built: per-course retake limits; extending
  `expires_at`; renewal of a completed enrollment; subscriptions (029);
  any refund/void change; rewriting 027's exhausted wording. The
  CLAUDE.md "Commands" block still says "<typecheck and lint commands —
  fill in>"; this build used `npm run lint` (oxlint) and
  `python -m pyflakes app tests` (the only linters present) — no
  typechecker exists in either half.
