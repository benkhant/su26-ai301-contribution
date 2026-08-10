# Contribution 2: Add Client-Side Full-Text Search for Reviews

**Contribution Number:** 2  
**Student:** Ben Khant  
**Issue:** https://github.com/mcgill-courses/mcgill.courses/issues/377  
**Status:** Phase IV Complete

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
2. Fork and clone the repo, create branch `fix-issue-377`
3. Create `.env` in the project root with `MS_CLIENT_ID`, 
   `MS_CLIENT_SECRET`, and `MS_REDIRECT_URI=http://localhost:8000/api/auth/authorized`
4. Create `client/.env` with `VITE_API_URL=http://localhost:8000`
5. Run `docker compose up --no-recreate -d` to start MongoDB
6. Run `cargo run -- --source=seed --initialize --db-name=mcgill-courses` 
   to seed and start the backend
7. In a new terminal tab, run `cd client && pnpm install && cd .. && pnpm -r run dev`
8. Navigate to `http://localhost:5173/course/acct-352`
9. Scroll down to the reviews section — observe there is no search input, 
   only Sort By and Instructor dropdowns
10. To confirm the issue: there is no way to filter reviews by keyword

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
- [x] Search input renders above the Sort By and Instructor dropdowns
- [x] Typing a keyword calls setSearchQuery to update search state
- [x] Clicking Reset clears the search query back to empty string

### Integration Tests
- [x] Search filters review list correctly by keyword in the browser
- [x] Ratings/stats chart is unaffected while searching
- [x] Search clears when navigating to a different course page
- [x] Works correctly in both desktop and mobile layouts
- [x] Works correctly in dark mode

### Manual Testing
All manual testing was completed before the PR was opened. The database 
was accidentally wiped mid-setup (`docker compose down -v`), which 
required a full re-seed, but everything was verified once the environment 
was back up.

Tested in the browser at localhost:5173:
- Search box appears above the Sort By and Instructor dropdowns ✅
- Typing a keyword filters matching reviews in real time ✅
- Ratings/stats chart stays unchanged while searching ✅
- Search clears when navigating to a different course ✅
- Works in both desktop and mobile layouts ✅
- Works in dark mode ✅

---

## Implementation Notes

### Week 1 Progress
Set up the full local development environment — installed Docker, 
Rust/Cargo, and pnpm, got MongoDB running via Docker, seeded the 
database, and got the React frontend running at localhost:5173. 
Explored the codebase to understand how `flexsearch` is already used 
for course search and how the review filter component works.

### Week 2 Progress
Implemented the full-text search feature. Added `searchQuery` state 
to `course-page.tsx` and passed it to both `ReviewFilter` instances. 
Modified `review-filter.tsx` to build a `flexsearch` index from review 
content, filter reviews by search query before the existing 
sort/instructor filter runs, and added the search input UI using the 
existing `SearchBar` component. Added a test file with 3 tests covering 
search rendering, filtering, and reset behavior. All 218 tests pass and 
ESLint is clean.

### Code Changes
- **Files modified:**
  - `client/src/components/review-filter.tsx`
  - `client/src/pages/course-page.tsx`
  - `client/src/components/review-filter.test.tsx` (new file)
  - `client/src/components/course-review.tsx`
  - `client/src/components/course-review.test.tsx`

- **Key commits:**
  - `f0a5540` — feat: add client-side full-text search for reviews
  - `37a5950` — test: add tests for review search filter
  - `dfc70eb` — fix: prevent search from affecting course ratings chart
  - `5a39c8a` — fix: handle null return from flexsearch Index.search
  - `533eec4` — fix: clear search query immediately on course navigation
  - `4c359ea` — fix: simplify search result filtering and formatting
  - `fecd90e` — feat: highlight matched search terms in reviews
  - `6adf89d` — test: verify actual filtered results in search filter test
  - `c5e8f1f` — fix: keep per-instructor stats independent of search filtering

- **Approach decisions:**
  - Used `flexsearch` with `tokenize: 'forward'` to match the existing 
    course search pattern — no new libraries needed
  - Indexed only `review.content` since instructor filtering already 
    has a dedicated dropdown
  - Reused the existing `SearchBar` component for consistent UI styling
  - Managed `searchQuery` state in `course-page.tsx` (parent) and 
    passed it down to `ReviewFilter` (child) following React's 
    standard pattern for shared state
  - Search clears via the existing `reset` function which already runs 
    on course navigation — no extra logic needed
    
---

## Pull Request

**PR Link:** https://github.com/mcgill-courses/mcgill.courses/pull/1178

**PR Description:** Added client-side full-text search for course reviews 
using `flexsearch`, allowing users to filter reviews by keyword in real 
time without affecting the ratings/stats chart above.

**Maintainer Feedback:**

`39bytes` left four points on review: highlight matched search terms, 
simplify some unnecessary filtering logic, fix a CI formatting issue, 
and explain why the ratings chart's data source changed. Copilot 
separately flagged a test that only checked a setter fired, not that 
filtering actually worked.

A day after those were addressed, `39bytes` followed up: the chart fix 
had accidentally removed a working feature, per-instructor stats 
filtering, that existed before search was added.

After that fix got approved, the PR hit a new blocker: `CI / server` and 
`CI / coverage` started failing. Dug into the logs and traced it to a 
panic in shared backend test setup code (`src/state.rs:69`), nothing to 
do with any file this PR actually touched. Synced the branch with 
`master` a couple of times over the last week just to check whether it 
had already been fixed upstream, it hadn't, same failure both times, 
even after `master` picked up several unrelated merges from `terror` 
during that stretch. After a week of silence, I sent a polite follow-up 
tagging both `39bytes` and `terror`, since `terror` had been 
the more active maintainer lately.

**Week of Aug 3–9 update:** No response yet from either maintainer after 
the follow-up. Not sending a second follow-up yet, since it's only 
been about a week since the last follow up, planning to give it a bit more 
time before checking in again.

**Status:** Approved by `39bytes`, blocked on an unrelated CI 
infrastructure issue. Followed up after one week of no response; as of 
this check-in, a second week has passed with still no reply. Not yet 
merged.

---

## Learnings & Reflections

### Technical Skills Gained
- Learned how `flexsearch` works — building an index, adding items,
  and querying it for fast client-side search
- Got comfortable reading an unfamiliar TypeScript/React codebase and
  understanding how state flows between parent and child components
- Learned how to set up a full-stack local dev environment with Docker,
  Rust/Cargo, and pnpm running together

### Challenges Overcome
- The README's `cargo run` command was outdated — used `--help` to
  discover the correct syntax rather than getting stuck
- The database got wiped accidentally with `docker compose down -v` —
  learned the difference between `down` and `down -v` the hard way
- `SearchBar` required extra props (`searchSelected`, `setSearchSelected`)
  that weren't obvious from the component name — reading the source
  code directly resolved it quickly
- Getting the replica set to initialize correctly required waiting
  longer after Docker started before running cargo

### What I'd Do Differently Next Time
- Read component source files before using them to understand required
  props upfront, not after hitting TypeScript errors
- Never use `docker compose down -v` unless intentionally wiping data —
  use `docker compose down` only
- Test in the browser earlier in the process rather than waiting until
  all the code is written
- Check the existing test files before writing implementation code, so
  the test patterns are familiar from the start

---

## Resources Used
- [PR #526](https://github.com/mcgill-courses/mcgill.courses/pull/526) —
  previous attempt at this issue, contains valuable maintainer feedback on
  approach (flexsearch vs fuse, UI style, known bugs)
- [flexsearch docs](https://github.com/nextapps-de/flexsearch) —
  used to understand Index configuration and tokenize options
- [search-index.ts](https://github.com/mcgill-courses/mcgill.courses/blob/master/client/src/lib/search-index.ts) —
  existing flexsearch usage in the codebase, used as reference for
  the same pattern in review search
- [React Testing Library docs](https://testing-library.com/docs/react-testing-library/intro/) —
  referenced for render, screen, waitFor, and userEvent patterns
