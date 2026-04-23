---
id: sysdesign-api-contract-prompt
title: API Contract Generator
type: prompt
version: "1.0.0"
status: active
workstream:
  - system-designers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - api
  - openapi
  - contract
  - system-design
  - documentation
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
description: |
  Generates a complete OpenAPI 3.0 YAML contract stub from a plain-language API description,
  including paths, request/response schemas, authentication, and standard error responses —
  accelerating API-first design and ensuring consistent contracts across services.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: API_DESCRIPTION
    type: string
    description: Plain-language description of the API, its purpose, resources, and operations
    required: true
  - name: RESOURCE_NAME
    type: string
    description: Primary resource name (e.g., "User", "Order", "Product")
    required: true
  - name: AUTH_METHOD
    type: string
    description: "Authentication method: none | api-key | bearer | oauth2 (default: bearer)"
    required: false
  - name: ERROR_CODES
    type: string
    description: "Comma-separated list of additional HTTP error codes to include (e.g., 409, 422)"
    required: false
outputs:
  - name: OPENAPI_CONTRACT
    type: string
    description: Complete OpenAPI 3.0 YAML contract stub ready for import into design tools
---

## Overview

Use this prompt to produce a well-structured OpenAPI 3.0 YAML contract from a plain-language description. The output is ready to import into API design tools (Swagger Editor, Postman, Stoplight) and serves as the source of truth for both frontend and backend teams during API-first development.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{API_DESCRIPTION}}` | string | Yes | Plain-language description of the API |
| `{{RESOURCE_NAME}}` | string | Yes | Primary resource the API manages |
| `{{AUTH_METHOD}}` | string | No | `none` / `api-key` / `bearer` / `oauth2` (default: `bearer`) |
| `{{ERROR_CODES}}` | string | No | Additional error codes to include, comma-separated |

## Outputs

| Name | Type | Description |
|---|---|---|
| `OPENAPI_CONTRACT` | string | OpenAPI 3.0 YAML contract stub |

## Prompt

```
[System]
You are an API design expert who produces accurate, standards-compliant OpenAPI 3.0 contracts
from plain-language descriptions.

[Task]
Generate a complete OpenAPI 3.0 YAML contract for the following API.

API Description:
{{API_DESCRIPTION}}

Primary resource: {{RESOURCE_NAME}}
Authentication method: {{AUTH_METHOD}}
Additional error codes to include: {{ERROR_CODES}}

[Rules]
- Use OpenAPI 3.0.3 specification.
- Include a complete `info` block (title, version: "1.0.0", description).
- Define all CRUD operations that are implied by the description (GET list, GET by ID, POST, PUT, DELETE).
- Use `$ref` components for all request/response schemas — no inline schemas in paths.
- Define all schemas under `components/schemas` with `required` arrays and property descriptions.
- Include standard error responses: 400 Bad Request, 401 Unauthorized (if auth is not none),
  404 Not Found, 500 Internal Server Error — plus any codes listed in ERROR_CODES.
- For bearer auth, add a `BearerAuth` security scheme; for api-key, add `ApiKeyAuth`; for oauth2
  use the `authorizationCode` flow with placeholder URLs.
- Add meaningful `operationId` values using camelCase (e.g., listUsers, getUserById, createUser).
- Add `summary` and `description` for every operation.
- Use plural lowercase kebab-case for path segments (e.g., /user-accounts/{id}).

[Output Format]
Output only the YAML content inside a fenced ```yaml code block. No prose before or after.
```

## Usage

### LLM Chat (current)

```
Generate an OpenAPI 3.0 contract for this API:
Description: {{API_DESCRIPTION}}
Resource: {{RESOURCE_NAME}}
Auth: {{AUTH_METHOD}}
```

### Standalone

1. Copy the prompt above.
2. Fill in the four placeholders.
3. Paste into your LLM interface.
4. Import the YAML output into Swagger Editor or Postman to validate.

## Examples

### Example 1 — User management API with bearer auth

**Input:**
- `API_DESCRIPTION`: `A RESTful API for managing users in a multi-tenant SaaS platform. Supports creating, reading, updating, and deactivating user accounts. Users belong to an organisation and have a role (admin, member, viewer).`
- `RESOURCE_NAME`: `User`
- `AUTH_METHOD`: `bearer`
- `ERROR_CODES`: `409, 422`

**Output (abbreviated):**

```yaml
openapi: 3.0.3
info:
  title: User Management API
  version: "1.0.0"
  description: >
    RESTful API for managing users in a multi-tenant SaaS platform.
    Users belong to an organisation and have a role (admin, member, viewer).

servers:
  - url: https://api.example.com/v1

security:
  - BearerAuth: []

paths:
  /users:
    get:
      operationId: listUsers
      summary: List all users
      description: Returns a paginated list of users for the authenticated organisation.
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        "200":
          description: Paginated user list
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserList"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "500":
          $ref: "#/components/responses/InternalServerError"
    post:
      operationId: createUser
      summary: Create a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateUserRequest"
      responses:
        "201":
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "400":
          $ref: "#/components/responses/BadRequest"
        "409":
          $ref: "#/components/responses/Conflict"
        "422":
          $ref: "#/components/responses/UnprocessableEntity"

  /users/{id}:
    get:
      operationId: getUserById
      summary: Get a user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          description: User found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "404":
          $ref: "#/components/responses/NotFound"

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    User:
      type: object
      required: [id, email, role, organisationId, active]
      properties:
        id:
          type: string
          format: uuid
          description: Unique user identifier
        email:
          type: string
          format: email
          description: User email address
        role:
          type: string
          enum: [admin, member, viewer]
          description: User role within the organisation
        organisationId:
          type: string
          format: uuid
          description: ID of the organisation the user belongs to
        active:
          type: boolean
          description: Whether the user account is active

    CreateUserRequest:
      type: object
      required: [email, role, organisationId]
      properties:
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, member, viewer]
        organisationId:
          type: string
          format: uuid

  responses:
    BadRequest:
      description: Invalid request body or parameters
    Unauthorized:
      description: Missing or invalid authentication token
    NotFound:
      description: Resource not found
    Conflict:
      description: Resource already exists
    UnprocessableEntity:
      description: Request is well-formed but contains semantic errors
    InternalServerError:
      description: Unexpected server error
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Produces valid OpenAPI 3.0.3 YAML. Occasionally needs a follow-up to add pagination headers. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More complete `$ref` usage and richer property descriptions than gpt-4o. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
