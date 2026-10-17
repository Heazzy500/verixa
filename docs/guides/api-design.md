# API Design: contract before routes

`apps/api/openapi/openapi.yaml` is the source of truth for the REST API
contract. Define the operation, request/response schemas, error shape, and
security requirements there before implementing its route. This lets client
work and server work agree on observable behavior before either depends on
implementation details.

The current document reserves tags for Identity, Credentials, Sessions, MFA,
Authorization, Verification, Audit, Governance, and Notifications. Its empty
`paths` object is intentional: the API contract is introduced before this
phase's route catalogue. Add an operation under its bounded-context tag in
the same change that exposes it; do not invent an undocumented route as a
shortcut.

## Local documentation

In development, `@fastify/swagger` loads the static OpenAPI document and
`@fastify/swagger-ui` serves an interactive viewer at `/docs/`. The canonical
JSON document is available at `/docs/openapi.json`. These development routes
are disabled when `NODE_ENV=production`; publish or protect API documentation
as an explicit deployment decision rather than exposing it accidentally.

CI validates the YAML against OpenAPI using Redocly CLI. A change that parses
as YAML but violates the OpenAPI schema must not merge.

## Adding an operation

1. Add the operation and shared schema references to `openapi.yaml`.
2. Give it a bounded-context tag and document success and failure responses.
3. Implement request shape validation in the Fastify route, then delegate
   business rules to the application use case.
4. Keep authorization checks server-side; CORS and the OpenAPI document do
   not grant or deny access.
5. Add the route-registration wiring in `apps/api` and verify the contract
   against the behavior before merge.

The contract does not try to describe internal persistence details. That
boundary keeps clients stable when adapters change and makes compatibility
review a question about requests and responses, not the database schema.
