# Implementation Plan: Customer Directory Service

## Overview

Implement a production-ready Fastify + TypeScript REST API with Prisma/SQLite persistence, API key auth, OpenTelemetry tracing, Prometheus metrics, Swagger docs, structured logging with correlation IDs, and graceful shutdown. The implementation follows a layered architecture: Fastify → Service → Repository → Prisma/SQLite.

## Tasks

- [ ] 1. Project scaffolding
  - Initialise `package.json` with all required dependencies (`fastify`, `@fastify/swagger`, `@fastify/swagger-ui`, `@fastify/cors`, `pino`, `prom-client`, `@opentelemetry/sdk-node`, `@opentelemetry/auto-instrumentations-node`, `@prisma/client`, `uuid`) and devDependencies (`typescript`, `vitest`, `fast-check`, `@vitest/coverage-v8`, `eslint`, `prettier`, `prisma`, `tsx`, `@types/node`, `@types/uuid`)
  - Create `tsconfig.json` targeting ES2022 with `strict: true`, `outDir: dist`, `rootDir: src`
  - Create `.eslintrc.json` and `.prettierrc` configuration files
  - Create the full folder structure: `src/db/`, `src/errors/`, `src/plugins/`, `src/routes/`, `src/schemas/`, `src/services/`, `src/repositories/`, `tests/unit/`, `tests/integration/`
  - Add `npm` scripts: `build`, `start`, `dev`, `lint`, `test`, `test:coverage`
  - _Requirements: 16.1, 16.2_

- [ ] 2. Prisma schema and migration
  - Create `prisma/schema.prisma` with the `Customer` model (`id` UUID PK, `name` String, `email` String unique, `phone` String nullable, `createdAt` DateTime, `updatedAt` DateTime)
  - Run `prisma migrate dev --name init` to generate the initial migration in `prisma/migrations/`
  - Generate the Prisma client (`prisma generate`)
  - _Requirements: 13.1, 13.2_

- [ ] 3. Core infrastructure modules
  - [ ] 3.1 Implement `src/config.ts`
    - Read and validate all environment variables (`PORT`, `API_KEY`, `DATABASE_URL`, `LOG_LEVEL`, `NODE_ENV`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `npm_package_version`)
    - Throw a descriptive error and exit with code `1` if `API_KEY` is not set
    - Export a typed `config` object
    - _Requirements: 2.4, 2.5_

  - [ ] 3.2 Implement `src/logger.ts`
    - Create a Pino instance configured for JSON output to stdout
    - Respect `LOG_LEVEL` from config; default to `info`
    - Export the logger singleton
    - _Requirements: 9.1_

  - [ ] 3.3 Implement `src/errors/AppError.ts`
    - Define `AppError` base class extending `Error` with `statusCode` and `errorTitle` properties
    - Implement subclasses: `NotFoundError` (404), `ConflictError` (409), `ValidationError` (422), `UnauthorizedError` (401)
    - _Requirements: 8.1, 8.2_

  - [ ] 3.4 Implement `src/db/prisma.ts`
    - Create and export a singleton `PrismaClient` instance
    - _Requirements: 13.3_

- [ ] 4. Telemetry plugin
  - Implement `src/plugins/telemetry.ts`
  - Initialise the OTel Node SDK with HTTP and Prisma auto-instrumentations
  - Export an `initTelemetry()` function that must be called before any other imports in `src/index.ts`
  - When `OTEL_EXPORTER_OTLP_ENDPOINT` is not set, configure a `NoopSpanExporter` so the app starts cleanly
  - Register as a Fastify plugin so span context is available in route handlers
  - _Requirements: 10.1, 10.2, 10.4, 10.5_

- [ ] 5. Fastify app factory and remaining plugins
  - [ ] 5.1 Implement `src/plugins/correlation.ts`
    - `onRequest` hook: read `x-correlation-id` header or generate a UUID v4; attach to `request.correlationId`
    - Create a Pino child logger scoped to the request: `request.log = logger.child({ correlationId })`
    - `onSend` hook: write `x-correlation-id` back to the response header
    - _Requirements: 9.2, 9.3, 9.4, 9.5_

  - [ ] 5.2 Implement `src/plugins/auth.ts`
    - Register a `preHandler` hook scoped to a sub-router that covers all routes except `/health`, `/docs`, and `/docs/json`
    - Compare `x-api-key` header against `config.apiKey`; throw `UnauthorizedError` for missing or invalid keys
    - _Requirements: 2.1, 2.2, 2.3_

  - [ ] 5.3 Implement `src/plugins/errorHandler.ts`
    - `setErrorHandler`: map `AppError` subclasses to their HTTP status codes and format `{ error, message }`
    - Handle Fastify schema validation errors → 422
    - Handle all other errors → 500 with generic message; log full stack at `error` level
    - Never include stack traces or internal paths in the response body
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

  - [ ] 5.4 Implement `src/plugins/swagger.ts`
    - Register `@fastify/swagger` with OpenAPI 3.0 config (title, version, `x-api-key` security scheme)
    - Register `@fastify/swagger-ui` at `/docs`
    - Serve raw spec at `/docs/json`
    - _Requirements: 12.1, 12.2, 12.3, 12.4_

  - [ ] 5.5 Implement `src/plugins/metrics.ts`
    - Initialise `prom-client` default metrics (includes `process_uptime_seconds`)
    - Define `http_requests_total` counter (labels: `method`, `route`, `status`) and `http_request_duration_seconds` histogram
    - Update counters via `onResponse` hook
    - Register `GET /metrics` route (auth-protected, returns Prometheus text format)
    - _Requirements: 11.1, 11.2, 11.3_

  - [ ] 5.6 Implement `src/app.ts`
    - Export a `buildApp(opts?)` factory function that creates a Fastify instance
    - Register plugins in order: correlation → auth → errorHandler → swagger → metrics → telemetry plugin
    - Register routes: health, customers
    - _Requirements: 1.1, 2.1_

- [ ] 6. Route handlers
  - [ ] 6.1 Implement `src/schemas/customer.schema.ts`
    - Define JSON Schema objects for: create/update request body, customer response, list response, error response, pagination query params
    - `name`: non-empty string, maxLength 255; `email`: format `email`; `phone`: optional string, maxLength 50
    - `page`: integer ≥ 1, default 1; `pageSize`: integer 1–100, default 20
    - _Requirements: 3.4, 3.5, 3.6, 5.2, 5.3, 5.4, 5.5_

  - [ ] 6.2 Implement `src/routes/health.ts`
    - `GET /health` — no auth, responds with `{ status: "ok", version: "<semver>" }`
    - _Requirements: 1.1, 1.2, 1.3_

  - [ ] 6.3 Implement `src/routes/customers.ts`
    - `POST /customers` → 201 + created customer
    - `GET /customers` → 200 + paginated list with optional `search` query param
    - `GET /customers/:id` → 200 + customer or 404
    - `PUT /customers/:id` → 200 + updated customer or 404/409
    - `DELETE /customers/:id` → 204 or 404
    - Attach JSON Schema from `customer.schema.ts` to each route for validation and OpenAPI generation
    - Delegate all business logic to `CustomerService`
    - _Requirements: 3.1, 4.1, 4.2, 4.3, 5.1, 6.1, 7.1, 7.2, 7.3_

- [ ] 7. Service layer
  - Implement `src/services/CustomerService.ts`
  - `createCustomer`: generate UUID v4 for `id`, set `createdAt`/`updatedAt` to current UTC; delegate to repository
  - `getCustomerById`: delegate to repository; throw `NotFoundError` if null
  - `listCustomers`: pass `page`, `pageSize`, `search` to repository; return paginated envelope
  - `updateCustomer`: verify customer exists (throw `NotFoundError`); set `updatedAt`; delegate to repository
  - `deleteCustomer`: verify customer exists (throw `NotFoundError`); delegate to repository
  - Record a custom OTel span for the search operation including sanitised search term length as a span attribute
  - _Requirements: 3.2, 3.3, 4.2, 5.6, 5.7, 5.8, 6.2, 6.4, 7.2, 10.3_

- [ ] 8. Repository layer
  - Implement `src/repositories/CustomerRepository.ts`
  - `create`: Prisma `create`; catch `PrismaClientKnownRequestError` with code `P2002` and re-throw `ConflictError`
  - `findById`: Prisma `findUnique`; return `null` if not found
  - `findMany`: Prisma `findMany` with `where` clause for case-insensitive `name`/`email` search (SQLite `mode: 'insensitive'` or `contains`), `orderBy: { createdAt: 'desc' }`, `skip`/`take` for pagination; also run `count` for `total`
  - `update`: Prisma `update`; catch `P2002` and re-throw `ConflictError`
  - `delete`: Prisma `delete`
  - Map Prisma model to domain `Customer` type (ISO 8601 string timestamps)
  - _Requirements: 3.9, 5.6, 5.8, 6.5, 13.1_

- [ ] 9. Entry point and graceful shutdown
  - Implement `src/index.ts`
  - Call `initTelemetry()` as the very first statement (before any other imports)
  - Call `validateConfig()`, then `buildApp()`, then `prisma.$connect()` + run pending migrations (`prisma migrate deploy`)
  - Call `fastify.listen()` on the configured port
  - Register `SIGTERM` and `SIGINT` handlers: call `fastify.close()` (max 10s timeout), then `prisma.$disconnect()`, then `sdk.shutdown()`, then `process.exit(0)`
  - If `fastify.close()` does not resolve within 10 seconds, force `process.exit(1)`
  - _Requirements: 2.5, 13.3, 13.4, 14.1, 14.2, 14.3, 14.4_

- [ ] 10. Checkpoint — verify the service starts and responds
  - Ensure all TypeScript compiles without errors (`npm run build`)
  - Ensure `GET /health` returns 200 with the correct shape
  - Ensure `GET /customers` with a valid API key returns the paginated envelope
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Unit tests
  - [ ] 11.1 Implement `tests/unit/validator.test.ts`
    - Test valid create/update payloads are accepted
    - Test missing `name` → 422
    - Test empty `name` → 422
    - Test `name` > 255 chars → 422
    - Test malformed `email` → 422
    - Test missing `email` → 422
    - Test `phone` > 50 chars → 422
    - Test `pageSize` > 100 → 422
    - Test `page` < 1 → 422
    - _Requirements: 3.4, 3.5, 3.6, 3.7, 3.8, 5.4, 5.5, 15.1_

  - [ ]* 11.2 Write property test for name field validation boundary (P3)
    - **Property 3: Name field validation boundary**
    - Use `fc.string()` — assert empty or >255 char strings → 422; 1–255 char strings → accepted
    - **Validates: Requirements 3.4, 3.7**

  - [ ]* 11.3 Write property test for email field validation (P4)
    - **Property 4: Email field validation**
    - Use `fc.emailAddress()` for valid emails and `fc.string()` filtered to non-email strings
    - Assert valid emails are accepted; invalid strings → 422
    - **Validates: Requirements 3.5, 3.8**

  - [ ] 11.4 Implement `tests/unit/customerService.test.ts`
    - Mock `CustomerRepository` with vi.fn() stubs
    - Test `createCustomer`: UUID v4 assigned to `id`; `createdAt`/`updatedAt` set to current UTC; repository `create` called with correct args
    - Test `getCustomerById`: returns customer when found; throws `NotFoundError` when repository returns null
    - Test `listCustomers`: passes `page`, `pageSize`, `search` to repository; returns paginated envelope
    - Test `updateCustomer`: throws `NotFoundError` when customer not found; sets `updatedAt`; calls repository `update`
    - Test `deleteCustomer`: throws `NotFoundError` when customer not found; calls repository `delete`
    - _Requirements: 3.2, 3.3, 4.2, 6.2, 6.4, 7.2, 15.2_

- [ ] 12. Integration tests
  - [ ] 12.1 Implement `tests/integration/health.test.ts`
    - `GET /health` → 200 with `{ status: "ok", version: string }`
    - No `x-api-key` required
    - Response time < 500ms
    - _Requirements: 1.1, 1.2, 1.3, 15.3_

  - [ ] 12.2 Implement `tests/integration/customers.test.ts` — auth and basic CRUD
    - `GET /customers` without API key → 401 `{ error: "Unauthorized", message: "Missing API key" }`
    - `GET /customers` with wrong API key → 401 `{ error: "Unauthorized", message: "Invalid API key" }`
    - `POST /customers` with valid body → 201 + customer object with UUID `id` and ISO timestamps
    - `GET /customers/:id` → 200 + matching customer
    - `PUT /customers/:id` → 200 + updated customer with new `updatedAt`
    - `DELETE /customers/:id` → 204; subsequent `GET` → 404
    - `GET /customers/:id` for non-existent id → 404 `{ error: "Not Found", ... }`
    - `POST /customers` with duplicate email → 409 `{ error: "Conflict", ... }`
    - `POST /customers` with invalid body → 422
    - _Requirements: 2.2, 2.3, 3.1, 4.1, 4.2, 6.1, 7.1, 7.2, 15.3_

  - [ ] 12.3 Implement round-trip integration test in `tests/integration/customers.test.ts`
    - Create customer via `POST /customers`
    - Retrieve via `GET /customers/:id` — verify `name`, `email`, `phone` match
    - Update via `PUT /customers/:id` — verify updated fields and `updatedAt` ≥ `createdAt`
    - Delete via `DELETE /customers/:id` — verify 204
    - Verify `GET /customers/:id` returns 404 after deletion
    - _Requirements: 15.4_

  - [ ] 12.4 Implement pagination and search integration tests in `tests/integration/customers.test.ts`
    - Seed multiple customers; verify default `page=1`, `pageSize=20` in response envelope
    - `pageSize` > 100 → 422
    - `page` < 1 → 422
    - `GET /customers?search=<term>` returns only matching records (name or email contains term, case-insensitive)
    - `GET /customers?search=<no-match>` returns `{ data: [], total: 0 }`
    - Verify `data` array is ordered by `createdAt` descending
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 15.3_

  - [ ] 12.5 Implement `tests/integration/metrics.test.ts`
    - `GET /metrics` without API key → 401
    - `GET /metrics` with valid API key → 200 with Prometheus text format body containing `http_requests_total`, `http_request_duration_seconds`, `process_uptime_seconds`
    - _Requirements: 11.1, 11.2, 11.3, 15.3_

  - [ ]* 12.6 Write property test for customer creation round-trip (P1)
    - **Property 1: Customer creation round-trip**
    - Use `fc.record({ name: fc.string({minLength:1, maxLength:255}), email: fc.emailAddress(), phone: fc.option(fc.string({maxLength:50})) })`
    - For each generated input: POST → GET by id → assert `name`, `email`, `phone` match exactly
    - **Validates: Requirements 3.1, 4.1**

  - [ ]* 12.7 Write property test for server-assigned fields on creation (P2)
    - **Property 2: Server-assigned fields are valid on creation**
    - Same arbitraries as P1
    - Assert `id` matches UUID v4 regex; `createdAt` and `updatedAt` are valid ISO 8601 UTC strings within a reasonable window of `Date.now()`
    - **Validates: Requirements 3.2, 3.3**

  - [ ]* 12.8 Write property test for search results always matching the search term (P5)
    - **Property 5: Search results always match the search term**
    - Use `fc.array(customerArb)` + `fc.string({minLength:1})`
    - Seed customers, search, assert every returned record has `name` or `email` containing the term (case-insensitive); assert no non-matching record appears
    - **Validates: Requirements 5.6, 5.7**

  - [ ]* 12.9 Write property test for list ordering (P6)
    - **Property 6: List response is ordered by createdAt descending**
    - Use `fc.array(customerArb, {minLength:2})`
    - Seed customers, fetch list, assert `data[i].createdAt >= data[i+1].createdAt` for all `i`
    - **Validates: Requirements 5.8**

  - [ ]* 12.10 Write property test for update round-trip (P7)
    - **Property 7: Update round-trip**
    - Use `customerArb` for create + `customerArb` for update input
    - Create, update, GET — assert `name`, `email`, `phone` match update input; `updatedAt` ≥ `createdAt`
    - **Validates: Requirements 6.1, 6.2**

  - [ ]* 12.11 Write property test for delete removes the record (P8)
    - **Property 8: Delete removes the record**
    - Use `customerArb`
    - Create, DELETE → assert 204; GET → assert 404
    - **Validates: Requirements 7.1**

  - [ ]* 12.12 Write property test for error response schema conformance (P9)
    - **Property 9: Error responses always conform to the error schema**
    - Generate various error-triggering inputs (missing fields, bad UUID, wrong API key, duplicate email)
    - Assert every 4xx/5xx response body has exactly `{ error: string, message: string }` and no stack trace
    - **Validates: Requirements 8.2, 8.3, 8.5**

  - [ ]* 12.13 Write property test for correlation ID echo (P10)
    - **Property 10: Correlation ID is echoed in response**
    - Use `fc.uuid()` for requests with `x-correlation-id` header — assert response header matches
    - Use requests without the header — assert response header contains a valid UUID v4
    - **Validates: Requirements 9.3, 9.5**

- [ ] 13. Checkpoint — full test suite passes
  - Run `npm test` and verify all unit and integration tests pass with exit code `0`
  - Verify line coverage across `src/` meets the 80% threshold
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 14. Docker packaging and environment configuration
  - [ ] 14.1 Create `Dockerfile`
    - Multi-stage build: `builder` stage compiles TypeScript; `production` stage copies `dist/`, `prisma/`, and production `node_modules` onto a Node.js LTS slim base
    - Expose port `3000`; set `NODE_ENV=production`; `CMD ["node", "dist/index.js"]`
    - _Requirements: 16.3_

  - [ ] 14.2 Create `docker-compose.yml`
    - Define `customer-api` service using the built image
    - Mount a named volume `sqlite-data` to `/app/data/` for the SQLite database file
    - Pass environment variables from a `.env` file
    - Map host port `3000` to container port `3000`
    - _Requirements: 16.4, 16.5_

  - [ ] 14.3 Create `.env.example`
    - Document all variables: `PORT`, `API_KEY`, `DATABASE_URL`, `LOG_LEVEL`, `NODE_ENV`, `OTEL_EXPORTER_OTLP_ENDPOINT`
    - Include descriptions and example values for each variable
    - _Requirements: 16.6_

- [ ] 15. README
  - Create `README.md` documenting: prerequisites, local setup (`npm install`, `prisma migrate dev`, `npm run dev`), Docker Compose usage, environment variables reference, available npm scripts, API overview with example `curl` commands for each endpoint, and test instructions
  - _Requirements: 15.5, 16.1_

- [ ] 16. Final checkpoint — end-to-end verification
  - Run `npm run build` — verify zero TypeScript errors
  - Run `npm run lint` — verify zero ESLint violations
  - Run `npm test` — verify all tests pass and coverage ≥ 80%
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- `initTelemetry()` in `src/index.ts` **must** be the very first statement — before any other imports — so OTel auto-instrumentation patches Node.js built-ins correctly
- Property tests use **fast-check** and run a minimum of 100 iterations each; tag each test with a comment: `// Feature: customer-directory-service, Property N: <title>`
- Integration tests use a temporary isolated SQLite database per test run (set `DATABASE_URL` to a temp file path in test setup/teardown)
- Checkpoints ensure incremental validation at logical milestones
