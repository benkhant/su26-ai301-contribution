# Contribution 1: Tech Debt: Refactor Configuration-Dependent Models

**Contribution Number:** 1  
**Student:** Ben Khant  
**Issue:** https://github.com/bcgov/cas-registration/issues/4702  
**Status:** Phase I Complete

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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
