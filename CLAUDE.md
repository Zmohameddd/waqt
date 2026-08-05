# Waqt

A daily routine app — prayer times, workout tracking, nutrition goals, and Quran reading — generated per day, with per-category streaks and prayer-time-aware push notifications.

## Stack
- Backend: FastAPI (Python 3.12), Postgres, deployed on AWS App Runner
- Frontend: React Native via Expo, deployed to the App Store via EAS
- Auth: JWT
- External API: Aladhan (prayer times, no key required)

## Repo structure
```
waqt/
├── backend/     — FastAPI app, Postgres models, API routes
├── app/         — Expo/React Native mobile app
├── CLAUDE.md
└── README.md
```
## Commands

### Backend (run from /backend)
- `uvicorn app.main:app --reload` — start dev server
- `pytest` — run tests
- `ruff check .` — lint
- `ruff format .` — auto-format

### Frontend (run from /app)
- `npx expo start` — start dev server (scan QR with Expo Go app, or run on simulator)
- `npm test` — run tests
- `npx eslint .` — lint

## Conventions
- Commits follow Conventional Commits: `feat:`, `fix:`, `chore:`, `refactor:`
- One feature per branch (`feat/branch-name`), reviewed before merge into `main`
- Python: snake_case, type hints on all function signatures, Pydantic models for all API request/response schemas
- TypeScript/React Native: camelCase, functional components only, no class components
- Never commit `.env` files or hardcoded secrets — always reference environment variables
- All new backend endpoints require a corresponding Pydantic schema and at least one test

## Security
- Validate and sanitize all user input — never trust client-supplied data
- Never render user-supplied text unescaped in any UI (XSS risk)
- Never construct outbound requests using a URL built from user input without allow-listing the target (SSRF risk)
- Use Python's `secrets` module for tokens/keys — never `random`
- Sanitize filenames on any file upload; never join user input directly into a filesystem path (path traversal risk)
- Treat any CodeQL finding in these categories as a real bug to fix, not noise to dismiss

## Architecture notes
- Per-category streaks (prayer, food, workout, water, Quran) are tracked independently — no single "day completed" boolean. Each category has its own streak counter that increments/resets on its own.

- Per-category streaks (prayer, food, workout, water, quran) are tracked independently in the Streak table — no single "day completed" boolean.
- Per-category completion criteria (strict — must hit full target):
  - Prayer: all 5 prayers logged
  - Food: sum(protein logged) >= protein_target_g
  - Workout: boolean session-complete flag
  - Water: sum(water logged) >= water_target_ml
  - Quran: sum(minutes logged) >= quran_target_minutes
- The "overall" streak is a SEPARATE row in Streak (category="overall"), NOT derived from the other five. It only requires at least one ProgressLog entry that day, regardless of category or whether any target was hit.

---

## API

REST convention: URLs are resources (nouns), HTTP methods are actions (verbs).

### Auth
- `POST /auth/register` — create account
- `POST /auth/login` — returns JWT

### Daily Goal
- `GET /daily-goal?date=YYYY-MM-DD` — get today's targets + prayer times + workout type. Auto-generates the DailyGoal row on first call for a given date if it doesn't exist yet (pulls Aladhan prayer times, derives workout day from split, applies standing protein/water/quran targets). No separate "generate" endpoint — read always works.

### Progress Logs
- `POST /logs` — log progress for a category (body: category, value, timestamp)
- `GET /logs?date=YYYY-MM-DD` — all logs for a given day

### Streaks
- `GET /streaks` — all 6 streaks (5 categories + overall), current + longest

### Weekly View
- `GET /weekly-summary?week_start=YYYY-MM-DD` — aggregated week view

### Quotes
- `GET /quotes/random` — one active quote for the reminder card

### Push Notifications
- `POST /devices/register` — register Expo push token for this user's device


## Working with Claude Code — behavioral guidelines
*Biases toward caution over speed. I'm learning this stack deliberately — don't skip the reasoning to save time.*

### 1. Think before coding
Don't assume, don't hide confusion, surface tradeoffs.
- State assumptions explicitly before implementing. If uncertain, ask rather than guess.
- If multiple valid interpretations exist, present them — don't silently pick one.
- If a simpler approach exists, say so, even if it means pushing back on how I phrased the request.
- If something is unclear, stop and name what's confusing before proceeding.

### 2. Simplicity first
Minimum code that solves the problem. Nothing speculative.
- No features beyond what was asked.
- No abstractions for single-use code, no "flexibility" that wasn't requested.
- No error handling for scenarios that can't actually occur.
- If a solution could be 1/4 the length, rewrite it shorter.

### 3. Surgical changes
Touch only what's necessary. Every changed line should trace directly to the request.
- Don't "improve" adjacent code, comments, or formatting while making an unrelated change.
- Match existing style even if you'd personally do it differently.
- If you notice unrelated dead code or issues, mention them — don't fix them unprompted.
- Only remove imports/variables/functions that your own change made unused.

### 4. Goal-driven execution
Define success criteria, then verify against them — don't declare done without checking.
- "Add validation" → write tests for invalid inputs, then make them pass.
- "Fix the bug" → write a test that reproduces it first, then make it pass.
- "Refactor X" → confirm tests pass before and after.
- For multi-step tasks, state a brief plan with a verification step per item before starting.