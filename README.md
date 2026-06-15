# Contribution 1: Tech Debt: Refactor Configuration-Dependent Models

**Contribution Number:** 1  
**Student:** Ben Khant  
**Issue:** https://github.com/bcgov/cas-registration/issues/4702  
**Status:** Phase II Complete

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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
