# Current Feature

## Feature 005, Credit measurement

## Goal
A course knows how many CPE credits it recommends, how that number was
arrived at, and whether it is still true. The calculation is stored with its
inputs per lesson, shown term by term in the admin, disclosed on the public
course page, and exportable as the 9.02.2(2)(ii) record. It refuses to be
used when stale.

## In scope
- Method 2, the word count formula (7.02.6), including its all-video form
  (7.02.7), computed at the course level
- Per-lesson inputs read from the stored packages, never typed
- Rounding down to one-fifth (7.01), with a minimum awardable credit
- Stored result plus a stored breakdown, staleness derived not stored
- Admin credit panel, public disclosure of recommended credit
- The calculation as a plain-text record for the audit bundle

## Out of scope
- Method 1 pilot testing (7.02.1–7.02.4). superCPE does not use it. Do not
  build a half version.
- Adaptive learning path averaging (7.02.6, second paragraph).
- Question minimums (5.01.2.1, 6.01.2). Those are 006 and 007; they read the
  credit this feature produces.
- Per-jurisdiction rounding policy. Roadmap 019. Leave the comment.
- Publish. 008 calls `is_stale` and refuses; this feature only exposes it.

## Locators
Read 7.01, 7.01.1, 7.02, 7.02.5, 7.02.6, 7.02.7, and 9.02.2(2)(ii) before
writing code. The formula in 7.02.6:

```
[(words / 180) + actual A/V minutes + (questions × 1.85)] / 50 = credit
```

7.02.7: when audio/video is additional learning rather than narration of the
text, its actual duration enters the A/V term; when the whole program is
video, the word term is absent. Four things to get right:

1. **A/V duration is the measured one.** It comes from
   `lesson_packages.duration_seconds`, which 002 verified with ffprobe and
   which carries the video-tool attestation. Nothing in this feature accepts
   a typed duration.
2. **A segment contributes either its A/V time or its words, per the
   manifest's `av_is_additional_learning`.** True: `duration_seconds` enters
   the A/V term. False: the audio merely reads the text, so `word_count`
   enters the word term instead and the duration does not count.
3. **Every question counts.** Review questions, including those above the
   minimum, and assessment questions. Count them from each package's stored
   `questions` JSON by `kind` in {review, assessment}. Leave a comment at the
   count saying 006 and 007 must not narrow this to one kind.
4. **Round down, never up.** 7.01 permits self study credit in one-fifth or
   one-half increments; 7.02.6 says round down. Use one-fifth uniformly.

## Data model
Add to `courses`:
- `credit_award`: numeric(4,1), nullable — the rounded recommendation
- `credit_raw_minutes`: numeric(8,2), nullable — the numerator before ÷ 50
- `credit_word_count`, `credit_av_seconds`, `credit_question_count`: int,
  nullable
- `credit_breakdown`: JSONB, nullable — per-lesson rows, see below
- `credit_formula_version`: string, nullable
- `credit_computed_at`: timezone-aware, nullable

Do not add `credit_is_stale`. Staleness is derived:

```
stale = credit_computed_at is None
     or credit_computed_at < content_updated_at
     or credit_formula_version != CREDIT_FORMULA_VERSION
```

`content_updated_at` is bumped only by 004's `touch`, and every attach,
detach, reorder, and version update already goes through it. Verify rather
than assume; if any path that changes a credit input skips `touch`, that is
a bug in this feature.

`credit_breakdown` is a list, one entry per lesson in position order:
`{lesson_id, package_id, version, position, title, duration_seconds,
av_is_additional_learning, av_seconds_counted, word_count,
words_counted, review_questions, assessment_questions}`. This is the
9.02.2(2)(ii) "supporting documentation for the data used": when the bundle
is exported, the calculation can be reproduced line by line from this alone.

## Constants
`app/constants/credit.py`, every one a number NASBA chose:
- `CREDIT_FORMULA_VERSION = "2026-7.02.6"`
- `MINUTES_PER_CREDIT = 50`
- `WORDS_PER_MINUTE = 180`
- `MINUTES_PER_QUESTION = Decimal("1.85")`
- `CREDIT_INCREMENT = Decimal("0.2")`
- `MIN_AWARDABLE = Decimal("0.2")`

When `CREDIT_FORMULA_VERSION` changes, every stored credit is stale by the
comparison above — no migration needed.

## Backend tasks
1. The eight columns, migration `add credit measurement`. Check numeric
   precision survives autogenerate. Round-trip.
2. `app/services/credit.py`, pure and read-only except `store`:
   - `compute(db, course_id) -> CreditBreakdown`: a dataclass carrying the
     per-lesson rows, the three totals, raw minutes, raw credit, rounded
     award, and formula version. Reads packages through `course_lessons`;
     writes nothing.
   - `round_down(raw: Decimal) -> Decimal`: `floor(raw / 0.2) * 0.2`,
     `Decimal` throughout. Below `MIN_AWARDABLE` returns `Decimal("0.0")`;
     do not raise. Comment: state boards differ on increments; superCPE will
     need a per-jurisdiction policy (roadmap 019); this is the finest legal
     granularity and it never rounds up.
   - `store(db, course_id) -> CreditBreakdown`: compute, then write the
     eight columns. Does not call `touch`; computing credit is not a content
     change.
   - `is_stale(course) -> bool` as above.
   - `as_text(breakdown) -> str`: the calculation written out the way a
     reviewer would want to read it — each lesson's line, the three sums,
     the formula with numbers substituted, the raw result, the rounding.
     This string goes into the audit bundle in 011 and can be shown in the
     admin now.
3. Recompute automatically: call `store` at the end of every `courses`
   service mutation that goes through `touch` (attach, detach, move,
   update-version). An admin should never see a stale credit on a course
   they just edited; staleness exists for formula-version changes and for
   defense in depth, not as a normal state. Keep the explicit
   `POST /admin/courses/{code}/credit/recompute` anyway.
4. Public payload: `GET /courses/{code}` and the list gain
   `recommended_credit` (string like "0.8", from `credit_award`) and
   `credit_basis` ("Word count formula, 2026 Standards 7.02.6"). 8.01
   requires the recommended credit in descriptive materials. Serve `null`
   while stale rather than a stale number.
5. Admin payload: the full breakdown, `is_stale`, `as_text`.
6. Tests, `tests/test_credit.py`, using `Decimal` assertions:
   - one all-video lesson, 486 s, 8 questions → raw 0.458, award 0.4
     (7.02.7's own shape; this is the number abacadaba's session notes
     recorded, use it as the golden case)
   - a lesson with `av_is_additional_learning: false` and 900 words
     contributes 5.00 word-minutes and 0 A/V seconds
   - review and assessment questions both count; a package with 5+3 counts 8
   - two lessons sum; the breakdown has two rows in position order
   - raw 0.19 → award 0.0; raw 0.20 → 0.2; raw 0.99 → 0.8; raw 1.0 → 1.0
     (never up)
   - attach recomputes and clears staleness; detach recomputes
   - changing `CREDIT_FORMULA_VERSION` (monkeypatch) makes a stored credit
     stale
   - public payload serves `recommended_credit` when fresh and `null` when
     stale
   - `as_text` reproduces the award when its numbers are re-added by hand in
     the test

## Frontend tasks
1. `/admin/courses/:code`: a Credit panel between the derived facts and the
   lesson table. Shows the award large, then the three terms as a small
   table (words ÷ 180, A/V minutes, questions × 1.85), the sum, ÷ 50, raw,
   rounded. Below, the per-lesson rows. A "Show calculation" toggle reveals
   `as_text` in a monospace block. If stale, a plain amber line stating why
   (content changed since / formula version changed) and a Recompute button.
2. `/courses/:code` public page: recommended credit in the disclosure list,
   with the basis line beneath it in smaller type. When `null`, omit the row
   entirely; never show "0.0" or "pending" to a participant.
3. `/admin/courses` list: an award column, with a stale marker.

## COMPLIANCE.md
Rows for 7.01 (rounding), 7.02 (method 2 chosen; method 1 not implemented,
as a Gap), 7.02.5 (word count excludes transcript; only `word_count` from
the manifest counts, and superCPE cannot verify that number against the
package text — Gap), 7.02.6, 7.02.7, 9.02.2(2)(ii) (breakdown and `as_text`
retained; export arrives in 011), and 8.01 (recommended credit now
disclosed). The 7.01.1 row's Gap is unchanged.

## Acceptance
- `pytest` passes with the new suite
- Migration round-trips
- Admin: the ASC842-PCX course with its factory package shows a credit
  panel whose numbers you can re-add by hand from the terms shown
- Detaching the lesson clears the award; reattaching restores it; the
  public page shows the credit row only while a lesson is attached
- "Show calculation" text matches the panel

## When done
Append the 005 entry. Under Decisions: one-fifth rounding uniformly and why;
auto-recompute on mutation; breakdown stored for 9.02.2(2)(ii). Under Known
gaps: method 1 absent; `word_count` trusted from the manifest; no
per-jurisdiction policy. Then stop.
