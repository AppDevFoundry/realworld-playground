You are an expert level backend software engineer specializing in APIs built on Nx + Node/Express + Prisma stack.

You are currently working in the civiclens git repo, with is a main repo that contains four submodules (civiclens-android, civiclens-api, ciclens-ios, ciclens-web). For this task, you will primarily be working with civiclens-api submodule, but feel free to check if there should be any additional relevant updates in other repos. For example, we recently just finished updating civiclens-web in the most recent pull request. 

Here is the specification for this task:

# Congress.gov API Integration

## Objective

Build a first-class integration layer inside `civiclens-api/` (Nx + Express + Prisma) that can talk to the [Congress.gov API](https://gpo.congress.gov/). The result should be a strongly typed, fully tested internal client that CivicLens services can call to fetch bill, member, committee, nomination, and hearing data. Exposing this internally is enough for now; a follow-up task will decide how to surface the data through our own REST API.

## Repository & Environment Notes

- Project root: `civiclens-api/`. It is an Nx workspace running a Node/Express server (`src/main.ts`) with routes mounted under `/api`.
- TypeScript throughout; linting and testing are provided via Nx/Jest (`npm run test` executes `nx test`).
- `axios` is already installed and should be the HTTP client.
- The environment variable `CONGRESS_API_KEY` is available (and must stay out of source control). Optionally support `CONGRESS_API_BASE_URL` with default `https://api.congress.gov/v3`.
- Tests live under `src/tests/**`; Prisma-specific mocks already exist in `src/tests/prisma-mock.ts` but are unrelated to this feature.

## Implementation Requirements

### 1. Configuration & Secrets

- Create `src/app/config/congress.config.ts` (or similar) that safely reads `CONGRESS_API_KEY`, optional `CONGRESS_API_BASE_URL`, and default request parameters (`format=json`, `limit`, etc.). Throw a descriptive error if the key is missing.
- Update `.env.example` / documentation so other developers know to set `CONGRESS_API_KEY` (and optionally `CONGRESS_API_BASE_URL`). Do **not** commit real keys.

### 2. HTTP Client Foundation

- Add `src/app/services/congress/congress.types.ts` defining shared enums/types (e.g., `BillType`, `PaginationParams`, `ApiError`, domain models shaped after the Congress.gov schema).
- Implement `src/app/services/congress/congress.client.ts` that:
  - Creates a dedicated `axios` instance with base URL + API key injected as a query param (Congress API expects `api_key` in the query string) and `format=json`.
  - Normalizes successful responses into `{ data, pagination, meta }`.
  - Converts API failures into typed `CongressApiError` objects (capture `status`, `detail`, `requestId` if present).
  - Provides helpers for pagination (`next`, `previous` URLs), date filtering, and optional throttling/backoff (lightweight retry-on-429 with exponential backoff is sufficient).
  - Exposes generic `request<T>` plus resource-specific helpers such as `getCollection<T>(path, params)` to reduce duplication.

### 3. Resource Modules

Create dedicated modules under `src/app/services/congress/resources/` that wrap the highest-priority endpoints listed in the [API docs](https://github.com/LibraryOfCongress/api.congress.gov/):

1. `bills.service.ts`
   - `listBills(params: BillsQueryParams)` with filters for congress session, bill type, introduced date range, sponsor, and pagination.
   - `getBillById({ congress, billType, billNumber })`.
   - Provide TypeScript interfaces for the summary data, actions, subjects, and latest action metadata commonly needed downstream.
2. `members.service.ts`
   - `listMembers(params)` filtering by chamber/state/party.
   - `getMemberDetail(memberId)`.
3. `committees.service.ts`
   - List/search committees plus detail (include subcommittees array parsing).
4. `nominations.service.ts`
   - Fetch nomination lists and detail (status, organization, latest action).
5. `hearings.service.ts`
   - Fetch committee hearings with date filters and location metadata.

Each module should export a plain object (or class) with well-named async methods that internally call `CongressApiClient`. Structure everything so new resources can be added by dropping new files into `resources/` and wiring them through `index.ts`.

### 4. Internal Access Point

- Add `src/app/services/congress/index.ts` that instantiates the client + resource modules and exports a single `CongressApi` object (e.g., `CongressApi.bills.listBills(...)`) for other services to import.
- Optionally provide a lightweight diagnostics route (`GET /api/congress/ping`) that uses the client to call the API status endpoint and returns sanitized metadata. Keep it behind the existing Express router structure (`src/app/routes/routes.ts`).

### 5. Testing & Tooling

- Use Jest with `nock` or `axios-mock-adapter` (add the library to `devDependencies`) to stub Congress.gov responses—tests must never hit the real API.
- Unit tests to cover:
  - Configuration errors when the API key is missing.
  - Client behavior (query param injection, pagination helpers, retry-on-429 logic, error translation).
  - Each resource module’s happy-path and error-path behavior with representative fixtures (store fixtures under `src/tests/fixtures/congress/`).
- Add a top-level integration-style test that wires `CongressApi` end-to-end with mocked HTTP to ensure modules share the same client instance.
- Ensure `npm run test` (aka `nx test`) passes locally.

### 6. Documentation & Developer Experience

- Update `civiclens-api/README.md` (or create `docs/congress-api.md`) summarizing:
  - How to configure the API key.
  - The new service entry points (code examples).
  - How to run the new tests.
- Include inline code comments only where logic is non-obvious (e.g., retry helpers).

## Acceptance Criteria

- Strongly typed Congress API client and resource modules live under `src/app/services/congress/**`.
- `CONGRESS_API_KEY` is validated at runtime; missing config fails fast with a clear message.
- All new functions are covered by Jest tests that isolate network calls via mocks.
- Repository docs mention the new integration and updated env requirements.
- `npm run test` succeeds.

## References

- API docs: https://gpo.congress.gov/ (and https://github.com/LibraryOfCongress/api.congress.gov/ for schema examples & changelog).
- Existing Express/Nx setup: `src/main.ts`, `src/app/routes/routes.ts`.
