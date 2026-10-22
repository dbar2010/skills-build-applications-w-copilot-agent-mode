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

- Presentation tier decision: Keep page rendering and interaction in the React components, with API requests going through the shared client. Each capability view handles loading, empty, success, and accessible error states without embedding backend logic.
- Logic tier decision: Keep Express route handlers thin and put validation, persistence orchestration, and calculations in the service layer. Reject malformed or missing references with deliberate 4xx responses; do not add authentication in this exercise.
- Data tier decision: Represent relationships with Mongoose references and require the fields needed by each capability. Keep seed data representative and idempotent, preserving unrelated records and existing user/team relationships.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and positive `minutes`; validate ID shape and confirm both records exist before creating an activity. Store references as `user` and `team`, derive points deterministically from minutes, and set `occurredAt` on the server.
- Team management approach: Require a team name, preserve the submitted name and mascot, and default omitted members to an empty array. Validate submitted member references and expose member IDs in list responses.
- Leaderboard approach: Sum activity points per team, sort from highest to lowest, and assign 1-based ranks. Include teams with no activity at zero points so the leaderboard represents all teams.
- Workout suggestions approach: Resolve the user by email, use their fitness goal and recent activity to prioritize matching seeded workouts, and include a short reason for the recommendation. Return a deliberate 4xx when the email does not match a user.

## Test plan

Run the focused contract tests first, then both builds. The starter tests are expected to fail until their corresponding capability is implemented.

1. `npm run test:data --prefix octofit-tracker/backend` — model validation failures indicate missing required fields, references, or defaults; seed failures indicate incomplete fixtures, destructive/non-idempotent seeding, or broken relationships.
2. `npm run test:api --prefix octofit-tracker/backend` — failures identify incorrect status codes or response shapes, invalid-input handling, persistence/calculation errors, leaderboard ordering/ranks, or user-specific suggestion behavior.
3. `npm test --prefix octofit-tracker/frontend` — failures identify missing API-backed rendering, incorrect request paths, or missing loading, empty, error, or accessible content states.
4. `npm run build --prefix octofit-tracker/backend` — failures indicate backend TypeScript type or compile errors not necessarily exercised by the contract tests.
5. `npm run build --prefix octofit-tracker/frontend` — failures indicate frontend TypeScript or Vite production-build errors.
