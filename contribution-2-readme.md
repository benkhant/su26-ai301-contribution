# Contribution 2: Add Client-Side Full-Text Search for Reviews

**Contribution Number:** 2  
**Student:** Ben Khant  
**Issue:** https://github.com/mcgill-courses/mcgill.courses/issues/377  
**Status:** Phase II Complete

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

The review section on a course page has no way to search through review 
content. Users can only sort reviews by date or rating, or filter by 
instructor name — but there is no way to find reviews mentioning a 
specific topic or keyword.

### Expected Behavior

A search input should appear above the review list. Typing a keyword 
should filter the displayed reviews to only those containing that text, 
while leaving the ratings and difficulty chart above completely unaffected.

### Current Behavior

No search input exists in the review section. Users must scroll through 
all reviews manually to find relevant content.

### Affected Components

- `client/src/components/review-filter.tsx` — where the sort/filter UI 
  lives, and where the search input will be added
- `client/src/pages/course.tsx` — parent component that manages review 
  state and passes props to ReviewFilter
  
---

## Reproduction Process

### Environment Setup

**Tools required:** Docker, Rust/Cargo, pnpm

**Challenges encountered:**
- The README's `cargo run` command includes a `serve` subcommand that no 
  longer exists in the current codebase. Running `cargo run -- --help` 
  revealed the correct syntax. The working command is:
  `cargo run -- --source=seed --initialize --db-name=mcgill-courses`
- Initial seeding takes a long time since it loads course 
  data year by year
- Environment variables (`MS_CLIENT_ID`, `MS_CLIENT_SECRET`) require 
  joining the project Discord and DMing a moderator to obtain

**How each was resolved:**
- Outdated `serve` subcommand — used `--help` flag to find current syntax
- Seeding time — let it run in background while setting up frontend in 
  a separate terminal tab
- Env vars — joined Discord, DMed moderator Jeff, received credentials 
  within ~10 minutes
  
### Steps to Reproduce

1. Join the mcgill.courses Discord and DM a moderator to get
   the environment variables and contributor role
3. Fork and clone the repo, create branch `fix-issue-377`
4. Create `.env` in the project root with `MS_CLIENT_ID`, 
   `MS_CLIENT_SECRET`, and `MS_REDIRECT_URI=http://localhost:8000/api/auth/authorized`
5. Create `client/.env` with `VITE_API_URL=http://localhost:8000`
6. Run `docker compose up --no-recreate -d` to start MongoDB
7. Run `cargo run -- --source=seed --initialize --db-name=mcgill-courses` 
   to seed and start the backend
8. In a new terminal tab, run `cd client && pnpm install && cd .. && pnpm -r run dev`
9. Navigate to `http://localhost:5173/course/acct-352`
10. Scroll down to the reviews section — observe there is no search input, 
   only Sort By and Instructor dropdowns
11. To confirm the issue: there is no way to filter reviews by keyword

**Branch link:** https://github.com/benkhant/mcgill.courses/tree/fix-issue-377

### Reproduction Evidence

- **My findings:** The review section renders in two places in 
  `course.tsx` — once for desktop layout and once for mobile. Both 
  instances will need the search input. The ratings/stats chart is 
  rendered separately from the review list, so filtering reviews will 
  not affect it as long as changes are scoped to `showingReviews` state 
  only. The existing `reset` function in `review-filter.tsx` already 
  runs when the course changes, which is the correct place to also clear 
  the search query.

---

## Solution Approach

### Analysis

This is a missing feature rather than a bug. The review list filtering 
logic lives in `review-filter.tsx`, which already handles sort and 
instructor filtering via a `useEffect`. The fix adds a search layer on 
top of this using `flexsearch`, the same library already used for course 
search in `client/src/lib/search-index.ts`.

### Proposed Solution

Add a `flexsearch` index built from review content, a search input to 
the UI, and wire them together so typing filters the review list before 
the existing sort/instructor logic runs.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Users cannot search through review text. A search input above the review 
list should filter displayed reviews by keyword without affecting the 
ratings chart.

**Match:** `client/src/lib/search-index.ts` already uses `flexsearch` with 
`new Index({ tokenize: 'forward' })`, `.add(id, text)`, and 
`index.search(query, { limit })`. The review search follows the exact 
same pattern. The existing `SearchBar` component can be reused for the 
input UI.

**Plan:** 
1. In `client/src/components/review-filter.tsx`:
   - Add `searchQuery` and `setSearchQuery` to props
   - Build a `flexsearch` index from `allReviews` content using `useMemo`
   - Filter `allReviews` by search query before sort/instructor filter
   - Add `SearchBar` input above the sort/instructor dropdowns, styled 
     to match the Explore page searchbar
   - Clear `searchQuery` in the existing `reset` function

2. In `client/src/pages/course.tsx`:
   - Add `const [searchQuery, setSearchQuery] = useState('')`
   - Pass `searchQuery` and `setSearchQuery` to both `ReviewFilter` 
     instances (desktop and mobile)

**Implement:** https://github.com/benkhant/mcgill.courses/tree/fix-issue-377

**Review:** 
CONTRIBUTING.md only contains the license clause. Based on existing PR 
history: use TypeScript with strict typing, Tailwind CSS for styling, 
reuse existing components, keep changes minimal and focused.

**Evaluate:** 
- Type a keyword and confirm only matching reviews appear
- Confirm ratings/stats chart is unaffected while searching
- Confirm search clears when navigating to a different course
- Confirm search works alongside instructor filter and sort simultaneously
- Test both desktop and mobile layouts
- Test dark mode
- Follow existing test patterns in `course-review.test.tsx`

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
