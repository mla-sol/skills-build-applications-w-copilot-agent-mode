# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Keep API access in the shared frontend client and render each capability in its own route component. Show accessible loading, empty, success, and error states; use relative `/api` paths so Vite's proxy remains the default.
- Logic tier decision: Keep request validation, HTTP status codes, and response shapes in the Express API, with business rules in backend services where appropriate. Reject invalid or missing resources with deliberate 4xx responses rather than leaking cast errors or returning placeholder success responses.
- Data tier decision: Use Mongoose models to enforce required fields, valid relationships, and sensible defaults. Seed stable representative records for all capabilities using upserts that preserve unrelated records and existing relationships.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and `minutes`; validate that both IDs are well-formed and refer to existing records before creating an activity. Persist references, compute points, and set `occurredAt` on creation; return stored activities from the list endpoint.
- Team management approach: Require a name, retain the submitted name and mascot, and default members to an empty array. Create and list teams with their member references intact.
- Leaderboard approach: Sum activity points by team, sort teams from highest to lowest total, and return each team's name, total points, and 1-based rank.
- Workout suggestions approach: Find the user by email and use their fitness goal and recent activity to prioritize a matching workout. Include an explanatory reason with each suggestion, and return a deliberate 4xx response when the email has no matching user.

## Test plan

Run focused tests after each capability, then run both builds before considering the tiers complete. The initial baseline was captured with the full backend and frontend test commands below; build commands have not been run.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — checks model validation, seeded records, relationships, idempotency, and preservation of unrelated records. The backend baseline had 4 data-contract failures: required user identity/goals, team fields, activity fields, and workout fields were not enforced.
- API command: `npm run test:api --prefix octofit-tracker/backend` — checks activity and team validation/creation, leaderboard totals and ranking, and user-specific workout suggestions. The backend baseline had 4 API-contract failures: activity/team validation returned `501`, the leaderboard was empty, and Mona's first workout suggestion did not match her endurance activity.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — checks capability data rendering, the workout-suggestions request, and loading/empty/error states for every route. Baseline: all 15 tests failed; expected data and accessible states were missing.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — verifies TypeScript compilation; not run in the initial baseline.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — verifies TypeScript compilation and the Vite production build; not run in the initial baseline.
- Baseline commands run: `npm test --prefix octofit-tracker/backend` (8 failed, 1 passed) and `npm test --prefix octofit-tracker/frontend` (15 failed). A failure in a focused suite identifies a contract gap in that tier; build failures indicate compilation or production-bundling issues.
