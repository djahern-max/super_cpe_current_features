# Current Feature

## Feature 004, Courses assembled from lesson packages

## Goal
A course is the credit-bearing unit: an ordered set of ingested lesson
packages sharing one field of study, one knowledge level, and one set of
prerequisites. After this feature an admin can create a course from packages,
order its lessons, swap a lesson to a newer package version, and see the
course-level facts the Standards require a participant to read (3.01.1,
3.02.1, 8.01). Nothing is published yet; that gate is feature 008.

## In scope
- `course_code` and `position` formalized as required manifest fields
- `courses` and `course_lessons` tables
- Course creation from packages, lesson ordering, version updates
- Derived course objectives and a course-level `content_updated_at`
- Deletion of unattached packages
- Admin course pages; a public catalog route that only serves published
  courses (so it serves nothing yet, correctly)

## Out of scope
- Credit (005), questions (006/007), review sign-off and publish (008)
- Editing course fields by hand. Course facts come from the packages, and a
  course whose title or level disagrees with its lessons is a course that
  was assembled wrong. See "Course facts are derived".
- Deleting a course that has ever been published or enrolled in. Nothing can
  be published yet, so plain delete is fine for now; 010 revisits.

## Contract edit, do first
In `docs/course-package.md`, add to the manifest as required fields:

```json
"course_code": "ASC842-PCX",
"position": 1,
```

`course_code` groups lessons into a course; `position` is the lesson's order
within it, a positive integer, unique within the course. Add both to rule 3's
required list in `packages.py` and to the factory. Keep `package_version: 1`
and note in the changelog that the only pre-existing package (`HAZWASTE-01`,
a fixture) predates the rule. video-tool already writes both fields.

## Locators
Read 3.01, 3.01.1, 3.02.1, 8.01, 8.01.1, and 8.01.2 before starting. Also
re-read 7.01.1's second paragraph (segments in multiple fields of study):
this feature does not implement it, and the changelog must say the course
requires a single field of study *because* 004 chose not to.

## Course facts are derived
A course's `title` is typed once by the admin. Everything else the Standards
require a participant to read comes from its lessons and must agree across
them:

- `field_of_study`, `knowledge_level`, `prerequisites`,
  `advance_preparation`: every attached package must carry identical values.
  Attaching a package that disagrees is refused with a message naming the
  field and both values. The course stores the agreed value on its own row
  for convenience, re-copied on every attach and version update.
- `learning_objectives`: the union of every lesson's objectives, grouped by
  lesson, in position order. Not a table; a service function
  `course_objectives(course) -> list[{lesson_id, position, objectives}]`.
  Objective ids are unique only within a package, so consumers key on
  (package_id, objective_id). Feature 006 will need that when mapping
  questions.
- `description`: a text field typed by the admin, the 8.01.1 course
  announcement copy. Required before publish (008), optional now.

This is the one place superCPE deliberately departs from abacadaba, which let
the admin type the level and prerequisites on the course and then validated
the typed values. Here the packages are the source; the admin cannot
introduce a contradiction between what the video says and what the catalog
says.

## Data model
Table `courses`:
- id, course_code (string, unique, not null), title (not null),
  description (text, not null, default '')
- field_of_study, knowledge_level, prerequisites, advance_preparation:
  copied from packages, nullable until the first lesson is attached
- status: string, not null, default 'draft', CHECK in ('draft',
  'published'). Only 'draft' is writable in this feature.
- content_updated_at: timezone-aware, not null. Bumped on every change that
  a participant could observe: attach, detach, reorder, version update,
  title or description edit. Later features derive "credit is stale" and
  "review is stale" from this single column, so it must be bumped from one
  choke point. Name it `touch(course)` in the service and call it nowhere
  else.
- created_at, updated_at

Table `course_lessons`:
- id, course_id (FK cascade), package_id (FK restrict), position (int)
- unique (course_id, position); unique (course_id, package_id)
- A package's `lesson_id` may appear once per course. Enforce in the service
  (two versions of the same lesson cannot both be attached) and test it.

## Behaviors
- Create course: `course_code`, `title`, optional `description`. Refuse a
  code already in use.
- Attach package: by package id. Checks, in order: package not already
  attached; no other version of the same `lesson_id` attached; agreement on
  the four derived fields (or first lesson, which sets them); manifest
  `course_code` equals the course's (mismatch is refused: the lesson was
  exported for a different course). Position defaults to the manifest's
  `position`; if taken, refuse and say so rather than silently appending.
- Detach, reorder (move up/down with the two-pass renumber that dodges the
  unique constraint).
- Update version: replace an attached package with a newer version of the
  same `lesson_id`. Refuse if the newer version disagrees on derived fields.
- Delete package: only if unattached; removes the storage object too.
- Delete course: only while draft; detaches lessons, does not delete
  packages.
- Every mutation calls `touch`.

## Backend tasks
1. Models and migration `create courses`. Hand-add CHECKs. Verify round-trip.
2. `app/services/courses.py` with the behaviors above; `course_objectives`;
   `touch`. Rule violations raise `CourseRuleViolation`, translated to the
   same 422 `{"errors": [...]}` shape as 002 and 003.
3. `app/services/packages.py`: `delete_package(db, storage, id)` refusing
   when attached.
4. Routers under `/api/v1/admin/courses` (CRUD, attach, detach, reorder,
   update-version) and `DELETE /api/v1/admin/packages/{id}`. Public
   `GET /api/v1/courses` and `GET /api/v1/courses/{course_code}` returning
   published courses only, with the full 8.01 disclosure payload: title,
   description, field of study, level, prerequisites, advance preparation,
   objectives grouped by lesson, lesson titles and durations. Write the
   public schema now even though it serves nothing; 008 flips the status.
5. `app/services/packages.py::list_packages` gains an `attached_to` field so
   the admin can see which packages are free.
6. Tests, `tests/test_courses.py`:
   - create, duplicate code refused
   - attach sets derived fields from the first package
   - attach with a differing knowledge level refused, message names both
   - attach with a different manifest `course_code` refused
   - two versions of one lesson_id cannot both be attached
   - position collision refused
   - reorder keeps positions dense and unique
   - update-version swaps the package and bumps `content_updated_at`
   - detach and reorder bump `content_updated_at`; a no-op read does not
   - delete attached package refused; delete unattached removes the storage
     object
   - public list is empty while every course is draft; public detail 404s
     for a draft course
   - a `course_code`-less manifest is now refused at ingest

## Frontend tasks
1. `/admin/courses`: list with code, title, lesson count, status, last
   content change. Create form (code, title).
2. `/admin/courses/:code`: title and description editable inline; derived
   fields shown read-only with a note that they come from the lessons; the
   lesson table in position order with move up/down, detach, and an "update
   to vN" button when a newer version of that lesson exists; an attach panel
   listing unattached packages whose `course_code` matches. 422 errors shown
   per line, as before.
3. `/admin/packages`: show `attached_to`; a delete button on unattached
   packages with a confirm.
4. `/courses` public catalog page rendering the published list (empty state:
   a plain sentence, not a placeholder graphic). `/courses/:code` public
   course page laying out every 8.01 disclosure field in reading order:
   title, description, what you will learn (objectives by lesson), level,
   prerequisites, advance preparation, field of study, lessons with
   durations. Build it now against a draft course by temporarily flipping
   status in the database to check the layout, then flip it back; do not add
   a publish button.
5. Design brief for the public page: this is the first participant-facing
   surface. Generous line length, one accent, no cards-in-cards. The
   disclosure fields are not a sidebar; they are the page.

## COMPLIANCE.md
Rows for 3.01 (objectives derived from lesson packages, shown per lesson),
3.02.1 (course-level agreement enforced on attach), 8.01.1 and 8.01.2
(public course payload and page carry every element; not yet reachable
because nothing is published), and 7.01.1 with the Gap that multi-field
courses are refused rather than allocated.

## Acceptance
- Contract edit made; ingest refuses a manifest without `course_code`
- `pytest` passes with the new suite
- Migration round-trips
- Admin: create `ASC842-PCX`, attach the fixture package → refused because
  its `course_code` is `HAZWASTE-01` (correct); delete the fixture package;
  ingest a factory package with `course_code: ASC842-PCX` and attach it
- Public `/courses` shows nothing; flipping the row to published in psql
  shows the course page with every disclosure field; flipping back hides it

## When done
Append the 004 entry. Under Decisions: course facts derived from packages,
single field of study per course, `touch` as the one choke point. Under
Known gaps: no publish; no multi-field allocation (7.01.1); objectives are
not editable in superCPE, only in the package. Add to ROADMAP.md's
Improvement notes anything you noticed. Then stop.
