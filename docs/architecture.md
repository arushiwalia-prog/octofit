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

- Data tier: Add the required Mongoose fields and validation for users, teams, activities, and workouts. Keep the existing seed data representative and idempotent.
- Logic tier: Implement the Express `/api/*` contracts in the backend, validate request data and referenced records, and return deliberate 4xx responses for invalid or unknown inputs. Keep API behavior in the logic tier and persistence in Mongoose models.
- Presentation tier: Implement the React routes using the shared API client and relative `/api` paths. Provide accessible loading, empty, error, and populated states without moving API or persistence logic into the UI.
- Activity tracking approach: Validate `userId` and `teamId` before creating an activity; persist the corresponding user/team references, submitted type and minutes, and computed points and occurrence time.
- Team management approach: Require a name, preserve the submitted name and mascot, and support listing teams with their members.
- Leaderboard approach: Aggregate activity points by team and return teams in descending point order with 1-based ranks.
- Workout suggestions approach: Match the requested email to a user, use that user's goal or recent activities to choose a relevant first suggestion, and include a reason; return a deliberate 4xx when no user matches.
- Keep MongoDB on port `27017`, the Express API on port `8000`, and Vite on port `5173`. Authentication remains out of scope.

## Test plan

Run checks in implementation order: data tier, logic/API tier, presentation tier, then backend and frontend builds.

- Data tier: `npm run test:data --prefix octofit-tracker/backend`. Missing required-field failures indicate incomplete model validation; the seed idempotence test must continue to pass.
- Logic/API tier: `npm run test:api --prefix octofit-tracker/backend`. Failures indicate incorrect status codes or response data for activity/team requests, leaderboard ordering or totals, or workout suggestion matching and reasons.
- Presentation tier: `npm test --prefix octofit-tracker/frontend`. Failures indicate missing or incorrect API-backed rendering, route behavior, or accessible loading, empty, and error states.
- Backend build: `npm run build --prefix octofit-tracker/backend`. A failure indicates backend TypeScript compilation or type errors after the data and API changes.
- Frontend build: `npm run build --prefix octofit-tracker/frontend`. A failure indicates frontend TypeScript compilation or Vite production-build errors after the presentation changes.
