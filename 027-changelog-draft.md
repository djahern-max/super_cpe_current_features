## 027 — Participant flow: every screen names the next step
Shipped: YYYY-MM-DD  (fill in after acceptance 7 passes on production)

**What changed**
- Task 0 (recon), answered before code was written:
  1. `ATO` is text-first, so the walkthrough's clip was a supplemental
     clip inside the reader. They do render today
     (`frontend/src/components/Reader/Reader.jsx`, a `<video controls>`
     at the clip's `after_section`, no seek handler). W3 applied to the
     reader clip: nothing happened on `ended`. W4 on a reader clip is
     not a lock — native controls seek freely and both the local media
     route (`FileResponse`, Range-aware) and Spaces presigned URLs honor
     Range requests. On the video-only player the participant sees no
     seek control at all: no native controls, only Play/Pause and Mute
     beside a thin bar.
  2. Backward seeking on the video-only player already worked: `seekTo`
     clamps to `[0, furthest]` and `handleSeeked` undoes only a forward
     seek past the furthest point. It was reachable only by clicking the
     bar or the arrow keys. **Not a fix — a control added** ("Rewind
     15 s", `REWIND_SECONDS` in `Player.jsx`).
  3. The reader receives one `sections[]` array in manifest order with
     `locked` and `markdown: null` per locked section, plus `questions[]`
     (`after_section`, `answered`) and `media[]` (`after_section`). The
     stepper was built on that payload unchanged; `reader.build` and
     `schemas/reader.py` are untouched.
  4. `assessment_available`, `assessment_unavailable_reasons`, per-lesson
     `done`/`review_answered`/`review_total`, `retakes_remaining`,
     `failed_attempts`, and `open_attempt_id` are on the enrollment
     detail (`GET /api/v1/my/enrollments/{id}`) only; the reader payload
     has none of them. `MyLesson` already fetched the detail to choose
     the medium; it now keeps it and refetches it after every graded
     answer.
  5. A failed enrollment attempt's result carries `retakes_allowed` and
     `retakes_remaining` (integers) beside `score_pct`, `passing_pct`,
     `correct_count`, `question_count` (`result` in
     `backend/app/services/assessment.py`). Preview attempts carry no
     `retakes_remaining`. The assessment info (`MyAssessmentInfo`) also
     carries both.
  6. `/policies` renders `retake_policy_text()` under "Assessment and
     re-takes" with no anchor. It now has `id="retakes"`; the failed
     result and the course page link `/policies#retakes` and do not
     restate it.
- Reader as a section stepper (`frontend/src/components/Reader/Reader.jsx`,
  `Reader.module.css`, new `stepper.js`): one section on screen; a
  contents column (sidebar from 60rem, a "Contents" toggle below) listing
  every section as read / current / unread / locked, with glossary and
  appendix under "Reference"; a locked entry is a disabled title marked
  Locked. Front matter first, always ("Start here"); body sections show
  "Section N of M" and a thin bar; reference sections say "Reference"
  and offer "Back to the guide". Questions placed after the current
  section render inline beneath it, all of them, in package order;
  Continue is disabled with the hint "Answer the review question above
  to continue" until every one is answered, then opens the next section
  — calling the page's `onContinue` (a refetch) first, so it shows what
  the server unlocked. Position is `?section=<key>` in the URL (reload
  and back/forward work; nothing written to the server); with no
  section in the URL the reader lands on the section after the last
  passed gate (`resumeKey`), or front matter when no gate has been
  passed. Search hits and glossary entries with a `section_key` in this
  lesson set the URL position. Left/right arrow keys step; no new
  dependency.
- Completion call to action: after the last body section, once every
  question in the lesson is answered, a card names the next step from
  `deriveNextStep` (`frontend/src/pages/MyLesson/nextStep.js`): the next
  lesson not yet `done` (in position order), else "Take the qualified
  assessment" / "Re-take … (N left)" when `assessment_available`, else
  "Resume the assessment" for an open attempt, else the course page.
  `MyCourse` shows the next action as a primary button under the
  deadline ("Continue reading (lesson 2 of 3)", "Continue watching …",
  "Take the qualified assessment", "Re-take the qualified assessment (N
  left)", "Resume the assessment", "View your certificate"); `/my/courses`
  cards carry the same labels ("Continue reading" by `lessons_kind`).
- Video direction and controls: reader clips keep `controls` and no seek
  handler (test pins a settled seek standing) and show "Continue
  reading" on `ended`, which scrolls to the open question after the
  section or opens the next section. The video-only player
  (`frontend/src/components/Player/Player.jsx`) gains "Rewind 15 s" and
  an end panel: the remaining review questions when the enrollment says
  any are unanswered (asked again in place — a reload mid-question
  resumes past the review point and never re-asks it), else the derived
  next step, else "End of this lesson." in the preview; "Watch again"
  beside it. Forward-seek lock untouched (test pins the refusal).
- Failed-result and exhausted wording
  (`frontend/src/components/Assessment/Assessment.jsx`, new
  `frontend/src/components/RetakesExhausted/`): with sittings left —
  score, threshold, "You have N re-takes left on this enrollment", a
  "Re-take the assessment" button, "Back to the study guide" (or "Back
  to the lessons" on a video course); with none — score, threshold, "You
  have used all N re-takes on this enrollment", "Read the re-take
  policy" → `/policies#retakes`, "Contact us about re-enrolling:
  <address>", and "The study guide stays open: you can keep reading
  it". The course page renders the same notice once, in the next-action
  slot, and drops its "Qualified assessment" section in that state;
  "The assessment is not available yet" is now only shown for unanswered
  review questions (the "No re-takes left" reason is filtered out of the
  list). `MyCourse` reads `retakes_allowed` from the assessment info
  only when exhausted (the detail carries the sittings left, not the
  allowance).
- Video-wording grep (section 5), participant surfaces, as found:
  `pages/MyCourses/progressLabel.js` (kind-aware, kept),
  `pages/MyCourse/MyCourse.jsx:176` (a comment), `pages/MyLesson/
  MyLesson.jsx` (medium dispatch), `pages/Catalog/Catalog.jsx` ("N
  minutes of video", shown only for video lessons with ≥1 minute, kept),
  `components/Assessment/retryAdvice.js` (kind-aware, kept),
  `components/Reader/Reader.jsx` (the clip caption "Watch, skip, or
  replay it as you like", about a clip, kept), `components/Player/
  Player.jsx` ("Re-watch this section", the video player, kept). One
  miss found and fixed: the `/how-it-works` text
  (`backend/app/services/instructions.py`) said "consider re-watching
  the lessons before trying again" for every course; it now says
  "re-reading the guide, or re-watching the lessons".
- Chrome: `MyCourses` keeps only its heading (email, Account link, and
  Sign out were the header's); `ReviewHeader` is removed from
  `ReviewHome.jsx` and `ReviewCourse.jsx` along with its CSS — after
  dropping its email and Sign out only a second wordmark remained.
  `/login` shows "New here? Create account" below the form under
  `siteFace() === OPEN`. New `frontend/src/components/SiteFooter/` under
  exactly the header's rule (null while loading or coming-soon, under
  `/admin`, and on `/change-password`): links `/policies`,
  `/how-it-works`, and the sponsor's contact address (mailto). Mounted
  in `App.jsx` after `<Routes>`.
- Backend (two small changes, both reported): `SponsorProfilePublic`
  gains `contact_email` and the public `GET /api/v1/sponsor` serves it
  (`backend/app/schemas/sponsor.py`, `backend/app/routers/sponsor.py`;
  new test `test_public_endpoint_carries_contact_email` pins the field
  and that nothing else joined it) — the address was in no
  participant-facing payload and the spec asks for it on two surfaces;
  and the instructions wording above. Suite 464 → 465 (the one new
  test). No model change, no migration.
- Frontend tests 30 → 73: `Reader.test.jsx` (D2 kept and extended
  through the stepper: the verdict survives leaving and returning; one
  section at a time; locked entry title-only and its body absent from
  the DOM; Continue disabled/enabled; progress counts body sections;
  URL round-trip and fallback; resume landing; no `is_correct`,
  `correct_choice_key`, or feedback text in the DOM before grading;
  completion card conditions and next-lesson variant; clip `controls`,
  no seek lock, `ended` affordance), `stepper.test.js`,
  `nextStep.test.js`, `Player.test.jsx` (backward seek and rewind,
  forward refused, end panel variants), `Assessment.test.jsx` (both
  failed variants and the preview, nothing per question),
  `SiteFooter.test.jsx` (footer present at open with and without the
  sponsor read, absent in coming-soon / under `/admin` /
  `/change-password`, no course fact or Registry string; `/login`
  Create account at open and not in coming-soon signed out; one Sign
  out and one email on `/my/courses`, `/review`, `/review/courses/ATO`).
- COMPLIANCE.md: four rows appended (4.05.3 item 4, 5.01.2.1, 6.01.2
  re-takes, 8.01.1). ROADMAP.md: the 028 improvement note.

**Standards touched**
- 4.05.3 items 4 and 5 — read in `docs/2026-Statement-on-Standards-for-
  CPE-Programs.pdf` on printed pages 7–8. The front matter that answers
  item 4 renders first in the stepper and is never hidden; item 5's
  review questions render inline with feedback as before.
- 5.01.2.1 — page 9. Placement is the package's `after_section`; the
  stepper moves, batches, and defers nothing, and adds no gate of its
  own.
- 5.01.2.2 — page 10. Feedback stays on screen through the stepper and
  through leaving and returning to a section.
- 6.01.2 — pages 13–14. "The number of re-takes … is at the sponsor's
  discretion": the failed result now says what the count means and
  links the published policy; sub-ii-b-1, "may not provide feedback":
  still nothing per question on a failed attempt, asserted in both
  variants.
- 8.01.1 — page 20. The footer adds a second path to `/policies`; the
  policies themselves are unchanged.

**Decisions**
- The reader's "read" state for an ungated section is browser state: the
  payload can say which gates are passed and what is locked, not
  whether a section between gates was read. Sections before the resume
  point are marked read from the payload; sections this session moved
  on from are marked read locally; nothing is written to the server. A
  server-side reading position would be a new participant record and
  needs a retention decision — not built, as the spec anticipated.
- "Continue reading (Section 4 of 14)" on the course page and
  `/my/courses` is rendered as "(lesson 2 of 3)": the enrollment payload
  carries lessons and their question counts, not section counts, and the
  spec says to add no field.
- The contact address is served from the already-gated public `/sponsor`
  payload rather than `/site` (which is public in coming_soon) — the
  footer and the exhausted notice render only at open or with a
  session, so the gate matches the surfaces.
- `ReviewHeader` removed entirely rather than trimmed (see above).
- `isExhausted` lives in `pages/MyCourse/exhausted.js` — oxlint's
  react-refresh rule flags non-component exports from component files
  (the 026 `stripeDashboard.js` precedent).
- On the course page the exhausted allowance `N` is read from the
  assessment info when needed; `failed_attempts - 1` is the fallback
  while that read is in flight (exact under the current invariant that
  `start_for_enrollment` refuses at zero sittings).

**Known gaps**
- Acceptance 1–5 (browser) and 7 (production) were run by the operator
  on YYYY-MM-DD: <outcome>.  (fill in)
- `/policies#retakes` scrolls only if the section exists when the hash
  is applied; the page loads its payload asynchronously, so a direct
  navigation may land at the top — the same caveat as 016's
  `/policies#refund` links.
- The reader contents column and the 320px layout were verified by
  tests and a production build, not by a screenshot; 025 used headless
  Chrome for that and 027 did not.
- `MyLesson.jsx` still resets its state synchronously inside its load
  effect (a pre-existing pattern oxlint warns about); `Player.jsx`'s
  unused `furthest` state variable predates this feature.
- The reader clip's `ended` affordance scrolls to the open question or
  opens the next section; it does not auto-advance, by design.
- On a video-only lesson, "Answer the review questions" re-asks every
  question in the lesson, not only the unanswered ones — the play
  payload carries no `answered` flag and adding one is a payload change
  this feature did not make.
- `deploy/`, `docs/OPERATIONS.md`: unchanged.
