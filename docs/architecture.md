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

- **Presentation tier decision**: Keep pages focused on rendering and user interaction; use the shared API client for relative `/api` requests. Represent loading, empty, and request-error states accessibly, and do not put persistence or business rules in the UI.
- **Logic tier decision**: Implement the API contract in Express routes/services. Validate request shape and referenced IDs before persistence, return deliberate 4xx responses for invalid or unknown resources, and keep business calculations out of routes where a service can own them.
- **Data tier decision**: Use Mongoose models for persisted fields, required values, and references. Keep seed data representative of all four capabilities and safe to run repeatedly.
- **Activity tracking**: Create activities only for existing users and teams; persist their references, type, and minutes together with centrally computed points and an occurrence date. Return useful activity data from list and create endpoints.
- **Team management approach**: Require a team name, preserve its mascot, and store member references. List and create teams through the API rather than accessing the database from the presentation tier.
- **Leaderboard approach**: Aggregate activity points by team in the logic tier, sort totals highest-first, and assign ranks starting at 1. Return the team name and total with each rank.
- **Workout suggestions approach**: Match the requested email to a seeded user, use that user's goal and recent activity to choose relevant suggestions, and include a reason for the first suggestion. Return a deliberate 4xx when no user matches.

## Test plan

Run contract tests in dependency order, then build both applications:

1. `npm run test:data --prefix octofit-tracker/backend` — verifies model validation, references, and repeatable representative seeding. Failures indicate a data-tier contract or seed issue.
2. `npm run test:api --prefix octofit-tracker/backend` — verifies API validation and responses, persistence, leaderboard ordering, and workout recommendations. Failures after data tests pass indicate logic-tier behavior or API/data integration issues.
3. `npm test --prefix octofit-tracker/frontend` — verifies API-backed page content and accessible loading, empty, and error states. Failures after API tests pass indicate presentation behavior or API-client integration issues.
4. `npm run build --prefix octofit-tracker/backend` — type-checks the backend. Failures indicate TypeScript errors or incompatible backend interfaces/types.
5. `npm run build --prefix octofit-tracker/frontend` — type-checks and bundles the frontend. Failures indicate frontend type errors or bundling/configuration issues.
