# Requirements Document

## Introduction

The Customer Directory Service is a production-ready REST API that manages a directory of customer records. It is designed to run locally and in Kubernetes, targeting public-sector-style clients who require strong security, structured observability, comprehensive testing, and clear documentation. The service exposes CRUD operations over customer data, enforces API key authentication, emits OpenTelemetry traces and Prometheus metrics, and ships with a Swagger UI and a complete Docker-based local development setup.

## Glossary

- **Service**: The Customer Directory REST API application built with Node.js and Fastify.
- **Customer**: A record containing identity and contact information for a single customer entity.
- **API_Key**: A secret string passed in the `x-api-key` HTTP request header used to authenticate callers.
- **Validator**: The request validation layer responsible for checking input structure and field constraints.
- **Repository**: The data-access layer that mediates between the Service and the database via Prisma ORM.
- **Database**: The SQLite file-based database managed by Prisma.
- **Telemetry**: The OpenTelemetry SDK components responsible for emitting traces, metrics, and structured logs.
- **Correlation_ID**: A unique identifier attached to every request and propagated through logs and trace spans.
- **Error_Handler**: The centralized Fastify error hook that formats all error responses uniformly.
- **Pagination**: The mechanism that splits large result sets into discrete pages using `page` and `pageSize` parameters.
- **Search**: Case-insensitive substring matching applied to the `name` and `email` fields of Customer records.
- **Health_Check**: The `/health` endpoint that reports service liveness and version without requiring authentication.
- **Metrics_Endpoint**: The `/metrics` endpoint that exposes Prometheus-format metrics and requires API key authentication.
- **Swagger_UI**: The interactive API documentation served at `/docs`, generated from the OpenAPI specification.
- **Graceful_Shutdown**: The process by which the Service stops accepting new requests, drains in-flight requests, and closes database connections before exiting.

---

## Requirements

### Requirement 1: Health Check Endpoint

**User Story:** As an operator, I want a public health check endpoint, so that load balancers and Kubernetes liveness probes can verify the service is running without needing credentials.

#### Acceptance Criteria

1. WHEN a `GET /health` request is received, THE Service SHALL respond with HTTP 200 and a JSON body containing `{ "status": "ok", "version": "<semver>" }`.
2. THE Service SHALL serve `GET /health` without requiring an `x-api-key` header.
3. WHEN the `GET /health` request is received, THE Service SHALL respond within 500ms under normal operating conditions.

---

### Requirement 2: API Key Authentication

**User Story:** As a security officer, I want all non-health endpoints protected by an API key, so that only authorised callers can read or modify customer data.

#### Acceptance Criteria

1. WHEN a request is received for any endpoint other than `GET /health`, THE Service SHALL inspect the `x-api-key` request header.
2. IF the `x-api-key` header is absent, THEN THE Service SHALL respond with HTTP 401 and a JSON body `{ "error": "Unauthorized", "message": "Missing API key" }`.
3. IF the `x-api-key` header value does not match the configured API key, THEN THE Service SHALL respond with HTTP 401 and a JSON body `{ "error": "Unauthorized", "message": "Invalid API key" }`.
4. THE Service SHALL read the valid API key value exclusively from an environment variable named `API_KEY`.
5. IF the `API_KEY` environment variable is not set at startup, THEN THE Service SHALL terminate with a non-zero exit code and log a descriptive error message.

---

### Requirement 3: Create Customer

**User Story:** As an API consumer, I want to create a new customer record, so that I can add customers to the directory.

#### Acceptance Criteria

1. WHEN a `POST /customers` request is received with a valid JSON body, THE Service SHALL persist a new Customer record and respond with HTTP 201 and the created Customer object.
2. THE Service SHALL assign a UUID v4 value to the `id` field of every newly created Customer.
3. THE Service SHALL set `createdAt` and `updatedAt` to the current UTC timestamp at the moment of creation.
4. THE Validator SHALL require the `name` field to be a non-empty string with a maximum length of 255 characters.
5. THE Validator SHALL require the `email` field to be a non-empty string in valid RFC 5322 email format.
6. THE Validator SHALL accept the `phone` field as an optional string with a maximum length of 50 characters.
7. IF the `name` field is absent or empty, THEN THE Validator SHALL respond with HTTP 422 and a JSON error body listing the specific field violation.
8. IF the `email` field is absent, malformed, or not a valid email address, THEN THE Validator SHALL respond with HTTP 422 and a JSON error body listing the specific field violation.
9. IF a Customer with the same `email` already exists, THEN THE Repository SHALL respond with HTTP 409 and a JSON body `{ "error": "Conflict", "message": "A customer with this email already exists" }`.

---

### Requirement 4: Retrieve Customer by ID

**User Story:** As an API consumer, I want to retrieve a single customer by their ID, so that I can view their details.

#### Acceptance Criteria

1. WHEN a `GET /customers/:id` request is received with a valid UUID, THE Service SHALL respond with HTTP 200 and the matching Customer object.
2. IF no Customer record exists for the given `:id`, THEN THE Service SHALL respond with HTTP 404 and a JSON body `{ "error": "Not Found", "message": "Customer not found" }`.
3. IF the `:id` path parameter is not a valid UUID v4 format, THEN THE Validator SHALL respond with HTTP 422 and a JSON error body describing the format violation.

---

### Requirement 5: List and Search Customers

**User Story:** As an API consumer, I want to list customers with optional search and pagination, so that I can browse and find customers efficiently.

#### Acceptance Criteria

1. WHEN a `GET /customers` request is received, THE Service SHALL respond with HTTP 200 and a JSON body containing `{ "data": [...], "total": <integer>, "page": <integer>, "pageSize": <integer> }`.
2. THE Service SHALL default the `page` query parameter to `1` when it is absent.
3. THE Service SHALL default the `pageSize` query parameter to `20` when it is absent.
4. THE Validator SHALL reject a `pageSize` value greater than `100` with HTTP 422 and a descriptive error message.
5. THE Validator SHALL reject a `page` value less than `1` with HTTP 422 and a descriptive error message.
6. WHEN the `search` query parameter is provided, THE Repository SHALL filter results to Customer records where `name` or `email` contains the search string using case-insensitive matching.
7. THE Service SHALL return an empty `data` array and `total` of `0` when no customers match the search criteria.
8. THE Service SHALL return customers ordered by `createdAt` descending by default.

---

### Requirement 6: Update Customer

**User Story:** As an API consumer, I want to update an existing customer's details, so that I can correct or enrich their information.

#### Acceptance Criteria

1. WHEN a `PUT /customers/:id` request is received with a valid JSON body, THE Service SHALL update the matching Customer record and respond with HTTP 200 and the updated Customer object.
2. THE Service SHALL set `updatedAt` to the current UTC timestamp at the moment of each update.
3. THE Validator SHALL apply the same field constraints for `name`, `email`, and `phone` as defined in Requirement 3, Acceptance Criteria 4–6.
4. IF no Customer record exists for the given `:id`, THEN THE Service SHALL respond with HTTP 404 and a JSON body `{ "error": "Not Found", "message": "Customer not found" }`.
5. IF the updated `email` value conflicts with an existing Customer record with a different `id`, THEN THE Repository SHALL respond with HTTP 409 and a JSON body `{ "error": "Conflict", "message": "A customer with this email already exists" }`.

---

### Requirement 7: Delete Customer

**User Story:** As an API consumer, I want to delete a customer record, so that I can remove outdated or incorrect entries from the directory.

#### Acceptance Criteria

1. WHEN a `DELETE /customers/:id` request is received, THE Service SHALL delete the matching Customer record and respond with HTTP 204 and no response body.
2. IF no Customer record exists for the given `:id`, THEN THE Service SHALL respond with HTTP 404 and a JSON body `{ "error": "Not Found", "message": "Customer not found" }`.
3. IF the `:id` path parameter is not a valid UUID v4 format, THEN THE Validator SHALL respond with HTTP 422 and a JSON error body describing the format violation.

---

### Requirement 8: Centralised Error Handling

**User Story:** As a security officer, I want all error responses to follow a consistent format without exposing internal details, so that the API surface is predictable and does not leak implementation information.

#### Acceptance Criteria

1. THE Error_Handler SHALL intercept all unhandled errors before they reach the HTTP response.
2. THE Error_Handler SHALL respond with a JSON body conforming to `{ "error": "<short title>", "message": "<human-readable description>" }` for all error responses.
3. IF an unhandled internal error occurs, THEN THE Error_Handler SHALL respond with HTTP 500 and the body `{ "error": "Internal Server Error", "message": "An unexpected error occurred" }`.
4. THE Error_Handler SHALL log the full error details including stack trace to the structured log at `error` level.
5. THE Error_Handler SHALL never include a stack trace or internal file path in the HTTP response body.

---

### Requirement 9: Structured Logging and Correlation IDs

**User Story:** As an operator, I want every log line to be structured JSON with a correlation ID, so that I can trace requests across distributed systems and query logs programmatically.

#### Acceptance Criteria

1. THE Service SHALL emit all log output as newline-delimited JSON to stdout.
2. THE Service SHALL generate a unique Correlation_ID for each incoming request.
3. WHEN a request includes a `x-correlation-id` header, THE Service SHALL use that value as the Correlation_ID instead of generating a new one.
4. THE Service SHALL include the Correlation_ID in every log line emitted during the processing of that request.
5. THE Service SHALL include the Correlation_ID in the HTTP response as the `x-correlation-id` header.
6. THE Service SHALL log each completed request at `info` level with fields: `method`, `url`, `statusCode`, `responseTimeMs`, and `correlationId`.

---

### Requirement 10: OpenTelemetry Observability

**User Story:** As an operator, I want distributed traces and metrics exported via OpenTelemetry, so that I can monitor service performance and diagnose latency issues in a Kubernetes environment.

#### Acceptance Criteria

1. THE Telemetry SHALL instrument all incoming HTTP requests as OpenTelemetry spans with attributes: `http.method`, `http.route`, `http.status_code`.
2. THE Telemetry SHALL instrument all Prisma database queries as child spans with the query operation name as the span name.
3. THE Telemetry SHALL record a custom span for the customer search operation including the sanitised search term length as a span attribute.
4. THE Telemetry SHALL export traces to an OTLP endpoint configurable via the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable.
5. WHERE the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable is not set, THE Telemetry SHALL default to no-op trace export without crashing.

---

### Requirement 11: Prometheus Metrics Endpoint

**User Story:** As an operator, I want a `/metrics` endpoint in Prometheus format, so that I can scrape service metrics from a Prometheus instance.

#### Acceptance Criteria

1. WHEN a `GET /metrics` request is received with a valid API key, THE Service SHALL respond with HTTP 200 and a Prometheus text exposition format body.
2. THE Metrics_Endpoint SHALL expose at minimum: `http_requests_total` (labelled by method, route, status), `http_request_duration_seconds` (histogram), and `process_uptime_seconds`.
3. THE Service SHALL protect `GET /metrics` with the same API key authentication defined in Requirement 2.

---

### Requirement 12: OpenAPI Documentation

**User Story:** As a developer integrating with the API, I want interactive API documentation, so that I can explore endpoints and test requests without reading source code.

#### Acceptance Criteria

1. THE Service SHALL expose a Swagger UI at `GET /docs`.
2. THE Service SHALL serve the raw OpenAPI 3.0 JSON specification at `GET /docs/json`.
3. THE Swagger_UI SHALL document all endpoints defined in Requirements 1–7 and 11, including request schemas, response schemas, and authentication requirements.
4. THE Service SHALL serve `GET /docs` and `GET /docs/json` without requiring an API key.

---

### Requirement 13: Persistence and Data Model

**User Story:** As a developer, I want the customer data persisted in a Prisma-managed SQLite database, so that data survives service restarts and the schema is version-controlled.

#### Acceptance Criteria

1. THE Database SHALL store Customer records with fields: `id` (UUID, primary key), `name` (string, required), `email` (string, unique, required), `phone` (string, nullable), `createdAt` (datetime), `updatedAt` (datetime).
2. THE Service SHALL manage all database schema changes through Prisma migrations stored in the `prisma/migrations/` directory.
3. WHEN the Service starts, THE Service SHALL verify the database connection and apply any pending migrations before accepting requests.
4. IF the database connection fails at startup, THEN THE Service SHALL terminate with a non-zero exit code and log a descriptive error message.

---

### Requirement 14: Graceful Shutdown

**User Story:** As an operator, I want the service to shut down gracefully on SIGTERM, so that in-flight requests complete and database connections are closed cleanly during Kubernetes pod termination.

#### Acceptance Criteria

1. WHEN the Service receives a `SIGTERM` or `SIGINT` signal, THE Service SHALL stop accepting new incoming connections.
2. WHEN the Service receives a `SIGTERM` or `SIGINT` signal, THE Service SHALL wait for all in-flight requests to complete before exiting, up to a maximum of 10 seconds.
3. WHEN the Service receives a `SIGTERM` or `SIGINT` signal, THE Service SHALL disconnect from the Database before the process exits.
4. IF in-flight requests do not complete within 10 seconds of receiving the shutdown signal, THEN THE Service SHALL forcibly close remaining connections and exit with code `1`.

---

### Requirement 15: Unit and Integration Testing

**User Story:** As a developer, I want a comprehensive test suite, so that I can confidently refactor and deploy the service.

#### Acceptance Criteria

1. THE Service SHALL include unit tests for the Validator covering valid inputs, missing required fields, malformed email addresses, and out-of-range pagination values.
2. THE Service SHALL include unit tests for the service layer covering create, read, update, delete, and list operations using a mocked Repository.
3. THE Service SHALL include integration tests for each API endpoint defined in Requirements 1–7 and 11, using a temporary SQLite database isolated per test run.
4. THE Service SHALL include a round-trip integration test: create a Customer via `POST /customers`, retrieve it via `GET /customers/:id`, update it via `PUT /customers/:id`, and verify the updated fields match.
5. WHEN `npm test` is executed, THE Service SHALL run all unit and integration tests and exit with code `0` when all tests pass.
6. THE Service SHALL achieve a minimum of 80% line coverage across the `src/` directory as reported by the test runner.

---

### Requirement 16: Build, Lint, and Containerisation

**User Story:** As a developer, I want a reproducible build, lint, and Docker packaging pipeline, so that the service can be built and deployed consistently across environments.

#### Acceptance Criteria

1. WHEN `npm run build` is executed, THE Service SHALL compile TypeScript sources to the `dist/` directory without errors.
2. WHEN `npm run lint` is executed, THE Service SHALL report zero ESLint violations across all `src/` and `tests/` files.
3. THE Service SHALL include a `Dockerfile` that produces a minimal production image using a Node.js LTS base, copies only compiled output and production dependencies, and exposes port `3000`.
4. THE Service SHALL include a `docker-compose.yml` that starts the Service container, mounts a named volume for the SQLite database file, and passes required environment variables from a `.env` file.
5. WHEN `docker compose up` is executed with a valid `.env` file, THE Service SHALL start and respond to `GET /health` within 30 seconds.
6. THE Service SHALL include a `.env.example` file documenting all required and optional environment variables with descriptions and example values.
