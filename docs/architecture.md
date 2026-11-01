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

- Presentation tier decision: Keep route components focused on rendering and user interaction. Use the shared API client for relative `/api` requests, and give each capability accessible loading, empty, and error states. Do not duplicate backend validation or business rules in the browser.
- Logic tier decision: Keep Express route handlers thin: translate HTTP requests and responses, and delegate validation and domain behavior to `octofitService.ts`. Return deliberate 4xx responses for invalid or missing resources; do not leak Mongoose cast errors as 500 responses. Do not add authentication in this exercise.
- Data tier decision: Use Mongoose schemas to enforce required fields and defaults, including an empty default team-members array. Keep user/team/activity references consistent. Seed representative users, teams, activities, and workouts idempotently without deleting unrelated records.
- Activity tracking approach: Accept only `userId`, `teamId`, `type`, and `minutes`; validate ID shape and referenced records before writing. Store the user and team references, and compute `points` and `occurredAt` on the server using one consistent points rule.
- Team management approach: List and create teams through the service. Require a name, preserve submitted names and mascots, and maintain valid member references without replacing unrelated teams or users.
- Leaderboard approach: Aggregate activity points by team, sort highest total first, and return `teamName`, `totalPoints`, and sequential 1-based ranks.
- Workout suggestions approach: Find the user by email, use their fitness goal and recent activity to prioritize matching workouts, and include a short reason for the recommendation. Return a deliberate 4xx response when the user does not exist.

## Test plan

Run the focused contract suites in data, API, then presentation order while implementing. Run both builds after the contracts pass. A failure in a focused suite identifies a contract in that tier; a build failure identifies a compile-time or bundling issue rather than a runtime contract failure.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`. Checks schema validation/defaults and idempotent seed behavior. Baseline: 4 of 5 tests fail because model fields are not yet required; the seed test passes.
- API command: `npm run test:api --prefix octofit-tracker/backend`. Checks input validation, persistence, leaderboard totals/ranks, and user-specific recommendations. Baseline: all 4 tests fail against the incomplete service implementations.
- Frontend command: `npm test --prefix octofit-tracker/frontend`. Checks API data rendering and accessible loading, empty, and error states across capability routes. Baseline: all 15 tests fail because the capability components remain implementation placeholders.
- Backend build command: `npm run build --prefix octofit-tracker/backend`. Checks TypeScript compilation; a failure indicates backend type errors.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`. Checks frontend TypeScript compilation and Vite bundling; a failure indicates a frontend type or build integration issue.
