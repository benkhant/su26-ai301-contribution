# Contribution 2: Add Client-Side Full-Text Search for Reviews

**Contribution Number:** 2  
**Student:** Ben Khant  
**Issue:** https://github.com/mcgill-courses/mcgill.courses/issues/377  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I picked this one because it's a clear, well-scoped feature: users should 
be able to search course reviews by content, similar to how course search 
already works on the rest of the site.

A previous contributor (Sergio-Na) had already tried this back in April 2024, 
but the PR went stale and was eventually closed by the maintainer in June 2025. 
The maintainer said the approach was good and that they'd likely pick the work 
back up someday.

So, before claiming it, I read through that entire closed PR's discussion, and 
it turned out to be really useful. The maintainers had already worked 
out a few important things:

- They wanted `flexsearch` instead of the `fuse.js` mentioned in the 
  original issue, since the project already uses `flexsearch` elsewhere 
  and didn't want two search libraries in the codebase
- Searching reviews had accidentally been affecting the ratings chart 
  above it, which a couple of collaborators flagged as confusing
- They wanted the searchbar to match the Explore page's style (rounder, 
  underline highlighting) instead of the homepage one
- The search input didn't clear when switching to a different course's 
  page, which needed fixing

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

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

- [PR #526](https://github.com/mcgill-courses/mcgill.courses/pull/526) — 
  previous attempt at this issue, contains valuable maintainer feedback on 
  approach (flexsearch vs fuse, UI style, known bugs)
- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
