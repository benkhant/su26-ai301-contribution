# Contribution 1: Tech Debt: Refactor Configuration-Dependent Models

**Contribution Number:** 1  
**Student:** Ben Khant  
**Issue:** https://github.com/bcgov/cas-registration/issues/4702  
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I chose this issue because it felt like a realistic and meaningful first contribution. 
The scope is clear, a maintainer already approved the approach and pointed to an 
existing example to follow, and no one has claimed it yet.

It also aligns with what I want to learn. I have a Python background but haven't worked 
much with Django ORM patterns in a real production codebase, so building a custom 
Manager from scratch felt like a good way to learn while actually 
contributing something useful to the project.

---

## Understanding the Issue

### Problem Description

The codebase has several models that are tied to a configuration period — 
meaning a record is only valid within a specific date range. Whenever the 
code needs to fetch one of these models for a given date, it manually 
writes out the same long filter conditions every time instead of having 
a shared helper that does it once.

### Expected Behavior

Each affected model should have a custom Django Manager with a method that 
wraps the date lookup logic internally. The rest of the codebase should 
just pass in a date and get back the right record — without needing to 
know the filter details.

### Current Behavior

The same two filter conditions appear copy-pasted across multiple service 
and test files:
- `valid_from__valid_from__lte=self.valid_date`
- `valid_to__valid_to__gte=self.valid_date`

This means if the lookup logic ever needs to change, every copy across 
the codebase has to be updated manually — which is error-prone and 
harder to maintain.

### Affected Components

- `reporting/models/activity_json_schema.py` — needs a custom Manager
- `reporting/models/activity_source_type_json_schema.py` — needs a custom Manager
- `reporting/service/report_activity_save_service.py` — two query call sites to simplify
- `reporting/tests/service/test_report_activity_save_service/infrastructure.py` — two call sites to update
- `reporting/tests/service/test_report_activity_save_service/test_report_activity_save_service_with_real_data.py` — two call sites to update

---

## Reproduction Process

### Environment Setup
Setting up the local environment required resolving several issues:

- **Postgres version conflict:** The machine had Homebrew PostgreSQL 14 
  installed, but the project uses PostgreSQL 16 via asdf. Fixed by setting 
  `asdf set -u postgres 16.2` and updating the shell config to prioritize 
  asdf shims over Homebrew binaries.

- **Missing ICU library:** Postgres 16 build failed with 
  `configure: error: ICU library not found`. Fixed by running 
  `brew install icu4c` and `brew link icu4c@78 --force`.

- **Missing pango library:** Running `make migrate` failed because 
  `weasyprint` could not load `libpango-1.0-0`. Fixed by running 
  `brew install pango`.

- **Missing Postgres role:** `make create_db` failed with 
  `role "aungminkhant" does not exist`. Fixed by connecting as the 
  `postgres` superuser and creating the role manually.

- **1Password env values:** The setup docs reference a private 1Password 
  vault for env values. Resolved by filling in safe local defaults for 
  non-sensitive variables (database name, host, port, debug flags) and 
  leaving external service credentials empty since they are not needed 
  for backend model work.

### Steps to Reproduce

1. Clone the repository and navigate to `bc_obps/`
2. Run `grep -rn "valid_from__valid_from__lte" .` in the terminal
3. Observe the same filter pattern repeated across multiple service
4. Open `reporting/service/report_activity_save_service.py` and note 
   the pattern at lines 79-80 and 112-113
5. Confirm no custom Manager exists on `ActivityJsonSchema` or 
   `ActivitySourceTypeJsonSchema` by checking their model files

### Reproduction Evidence

- **Branch:** [https://github.com/benkhant/cas-registration/tree/fix-issue-4702](https://github.com/benkhant/cas-registration/tree/fix-issue-4702)
- **My findings:** The repeated query pattern appears in 2 service files 
  and 2 test files across 6 total call sites. Both affected models 
  (`ActivityJsonSchema` and `ActivitySourceTypeJsonSchema`) currently 
  use the default Django Manager with no custom query abstraction.
  
---

## Solution Approach

### Analysis

The root cause is that the configuration-based date lookup logic is not 
abstracted. Two models — `ActivityJsonSchema` and 
`ActivitySourceTypeJsonSchema` — both use the default Django Manager, 
which means any code that needs to query them by date has to manually 
write out the full filter conditions every time. There is no shared 
method that encapsulates this logic, leading to duplication across 
service and test files.

### Proposed Solution

Add a custom Django Manager to each affected model with a method that 
wraps the `valid_from__valid_from__lte` and `valid_to__valid_to__gte` 
filter logic internally. Then update all call sites in the service and 
test files to use the new cleaner interface.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Multiple files repeat the same complex date filter when querying 
configuration-dependent models. The fix is to encapsulate that logic 
once in a custom Manager so no caller has to know the internal details.

**Match:** The codebase already has `TimeStampedModelManager` in 
`registration/models/time_stamped_model.py` as a reference. It extends 
`models.Manager` and overrides queryset behavior. The same pattern will 
be used to add a `get_by_date()` method.

**Plan:** 
1. Add a custom Manager to `reporting/models/activity_json_schema.py` 
   with a `get_by_date(date)` method that filters by configuration date range
2. Attach the Manager to the `ActivityJsonSchema` model
3. Add the same custom Manager to 
   `reporting/models/activity_source_type_json_schema.py`
4. Attach it to the `ActivitySourceTypeJsonSchema` model
5. Update `reporting/service/report_activity_save_service.py` lines 
   79-80 and 112-113 to use the new Manager methods
6. Update the two test files to use the new Manager methods
7. Run `make test` to confirm all existing tests still pass
8. Run `grep -rn "valid_from__valid_from__lte" .` to confirm no 
   instances remain in service files

**Implement:** Branch: https://github.com/benkhant/cas-registration/tree/fix-issue-4702

**Review:** 
- Use `refactor:` prefix for commit messages per project conventions
- Run `prek run --all-files` before submitting PR
- Ensure no new tests are broken
- Follow the existing code style in model files

**Evaluate:** 
- All existing tests pass after the refactor
- The repeated query pattern no longer appears in service or test files
- The Manager methods are covered by at least one unit test each
- Running `grep -rn "valid_from__valid_from__lte" .` returns no results 
  in service files

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: `test_get_by_date` — confirms `ActivityJsonSchema.objects.get_by_date()` 
  returns the correct schema when given a date within the configuration's valid range
- [x] Test case 2: `test_get_by_date_no_match` — confirms `ActivityJsonSchema.objects.get_by_date()` 
  raises `DoesNotExist` when given a date outside the configuration's valid range
- [x] Test case 3: `test_get_by_date` and `test_get_by_date_no_match` — same two scenarios 
  repeated for `ActivitySourceTypeJsonSchema`, with the additional `source_type` parameter

### Integration Tests

- [x] Integration scenario 1: `test_creates_report_activity_data` in 
  `test_report_activity_save_service_with_real_data.py` — confirms the full 
  `ReportActivitySaveService.save()` workflow still produces correct results 
  after switching to the new Manager methods
- [x] Integration scenario 2: Full test suite for `test_report_activity_save_service/` 
  (39 tests) confirms no regressions across the entire save service

### Manual Testing

Ran `grep -rn "valid_from__valid_from__lte" .` and 
`grep -rn "valid_to__valid_to__gte" .` after all changes to confirm the 
repeated filter pattern only exists inside the two new Manager methods 
themselves, and nowhere else in the codebase.

---

## Implementation Notes

### Week 1 Progress

- Completed Phase I: selected the issue, ran the selection checklist, 
  claimed the issue with a comment, and posted in the course portal
  
### Week 2 Progress

- Completed Phase II: reviewed the codebase, traced the repeated query 
  pattern across files, studied the existing `TimeStampedModelManager` 
  pattern, and wrote the UMPIRE implementation plan
- Set up local development environment, resolving Postgres version 
  conflicts, missing ICU/pango system libraries, and Postgres role 
  creation issues
- Created `fix-issue-4702` branch

### Week 3 Progress

- Added `ActivityJsonSchemaManager` with `get_by_date()` to `activity_json_schema.py`
- Added `ActivitySourceTypeJsonSchemaManager` with `get_by_date()` to 
  `activity_source_type_json_schema.py`
- Resolved mypy type errors on the Manager methods using `typing.cast`
- Updated `report_activity_save_service.py` to use both new Manager methods
- Updated test infrastructure and assertions to use the new Manager methods
- Added 4 new dedicated unit tests covering both match and no-match cases 
  for both models
- Ran full test suite (39 + 10 + 10 tests) — all passed
- Cleaned up commit history with interactive rebase to fix a typo in one 
  commit message
- Confirmed no unnecessary changes via `git diff develop --stat`
- Updated Contribution README with implementation summary

### Code Changes

- **Files modified:**
  - `reporting/models/activity_json_schema.py`
  - `reporting/models/activity_source_type_json_schema.py`
  - `reporting/service/report_activity_save_service.py`
  - `reporting/tests/service/test_report_activity_save_service/infrastructure.py`
  - `reporting/tests/service/test_report_activity_save_service/test_report_activity_save_service_with_real_data.py`
  - `reporting/tests/models/test_activity_json_schema.py`
  - `reporting/tests/models/test_activity_source_type_json_schema.py`

- **Key commits:**
  - [refactor: add custom manager to ActivityJsonSchema](https://github.com/benkhant/cas-registration/commit/ba37dea5d26802930abb73b884c6f0ec039032eb)
  - [refactor: add custom manager to ActivitySourceTypeJsonSchema](https://github.com/benkhant/cas-registration/commit/dd12684b07f755e1e9e891511dde0afea3283c1d)
  - [refactor: update service to use manager methods](https://github.com/benkhant/cas-registration/commit/e65a18e431b8dc8b526c61fbd8bfd99dc8529d94)
  - [test: update tests to use manager methods](https://github.com/benkhant/cas-registration/commit/ef22825ab13e1635915484583fdf56f9abff2bed)
  - [test: add unit tests for ActivityJsonSchema custom manager](https://github.com/benkhant/cas-registration/commit/0f17bfdfc11724980ebe7c127dbbb3c026941964)
  - [test: add unit tests for ActivitySourceTypeJsonSchema custom manager](https://github.com/benkhant/cas-registration/commit/6cecad1f126bef31fe3418a4b228e002e3f5f937)

- **Approach decisions:**
  - Used `typing.cast()` instead of an intermediate variable assignment to 
    satisfy mypy's strict return type checking on the Manager methods, since 
    Django's generic `.get()` return type doesn't automatically narrow to 
    the specific model subclass
  - Kept the test files' existing naming conventions consistent (camelCase 
    in `test_activity_json_schema.py`, snake_case in 
    `test_activity_source_type_json_schema.py`) rather than imposing a 
    single style across both
  - Chose test dates one year before/after the test fixture's own 
    configuration range (e.g., `5024` vs `5025`) rather than arbitrary 
    real-world dates, to keep the no-match test independent of any 
    unrelated fixture data that may exist in the database
    
---

## Pull Request

**PR Link:** https://github.com/bcgov/cas-registration/pull/4835

**PR Description:** Added a custom Django Manager with a `get_by_date()` 
method to `ActivityJsonSchema` and `ActivitySourceTypeJsonSchema`, 
replacing six repeated manual query filters across the service and test 
files with a single shared, reusable method.

**Maintainer Feedback:**
- 2026-06-21: Copilot's automated review completed with no comments 
  flagged across all 7 changed files
- 2026-06-21: Posted a comment tagging the maintainer (@Sepehr-Sobhani) 
  who originally approved this approach on issue #4702, requesting a 
  review
- 2026-06-23: Sepehr-Sobhani replied that the repo is not currently 
  accepting contributions from external contributors, and closed the PR

**Status:** Closed — not accepted (external contribution policy)

---

## Learnings & Reflections

### Technical Skills Gained

This was my first time working with Django's custom Manager pattern, 
and it clicked once I saw how it just slots into the `.objects` interface 
to hide repeated query logic behind a clean method. I also got more 
comfortable with mypy on Django querysets — specifically why Django's 
generic `.get()` doesn't automatically narrow to the right model type, 
and how `typing.cast()` fixes that. On top of the code, I picked up a lot 
of practical debugging experience just getting the environment running, 
dealing with asdf, Postgres versions, and a couple of missing system 
libraries on macOS.

### Challenges Overcome

The environment setup ended up being the hardest part, honestly. There 
were several issues stacked on top of each other — a Postgres version 
mismatch between Homebrew and asdf, a missing ICU library blocking the 
Postgres build, a missing pango library blocking PDF generation, and a 
missing Postgres role. I had to work through each one individually, 
reading the actual error message carefully instead of guessing. On the 
code side, getting mypy to accept the Manager's return type took a couple 
of tries — first with type hints alone, then switching to `typing.cast()` 
once that wasn't enough.

The biggest challenge though wasn't technical at all. After fully 
implementing the fix, writing tests, and opening a clean PR, the 
maintainer replied that the repo isn't currently accepting contributions 
from external contributors, and closed it. That stung a bit, since the 
code itself was solid, Copilot's automated review flagged zero issues 
across all 7 files, but it just came down to a contribution policy I did
not know about beforehand. The maintainer was kind about it 
though, which helped.

### What I'd Do Differently Next Time

I'd check the project's required tool versions before running any setup 
commands, so I can catch version conflicts early instead of debugging 
them after hitting build errors. But more importantly, I'd try to check 
whether a project is actually open to outside contributions before 
investing real time in it, maybe by seeing if non-team contributors have 
gotten PRs merged recently. A "good first issue" label doesn't always 
mean the team is currently accepting outside PRs, especially for 
something like a government codebase.

Even though this one didn't get merged, I still walked away with a real 
working implementation, a clean review, and a much better sense of what 
contributing to a production Django codebase actually looks like.

---

## Resources Used

- [bcgov/cas-registration docs folder](https://github.com/bcgov/cas-registration/tree/develop/docs)
- [Django custom Managers documentation](https://docs.djangoproject.com/en/stable/topics/db/managers/)
- `registration/models/time_stamped_model.py` — the existing `TimeStampedModelManager` pattern I used as a template
- [mypy documentation on cast()](https://mypy.readthedocs.io/en/stable/type_narrowing.html)
