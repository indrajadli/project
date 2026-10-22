 # BloodBond Backend Guide

This guide documents the Spring Boot backend, its REST APIs, authorization rules, key workflows, map data flow, real-time chat, and Groq-powered assistant.

> BloodBond is a community coordination and discovery tool. It does not verify blood type, establish clinical compatibility, assess donor eligibility, or replace a blood bank or healthcare professional.

## Contents

- [Backend overview](#backend-overview)
- [Run and configure](#run-and-configure)
- [Authentication and common responses](#authentication-and-common-responses)
- [REST API reference](#rest-api-reference)
- [Maps and donor discovery](#maps-and-donor-discovery)
- [Chat: REST and real-time WebSocket](#chat-rest-and-real-time-websocket)
- [Groq assistant](#groq-assistant)
- [End-to-end workflows](#end-to-end-workflows)
- [Persistence and migrations](#persistence-and-migrations)
- [Errors and troubleshooting](#errors-and-troubleshooting)
- [Privacy and safety](#privacy-and-safety)

## Backend overview

The backend is a Java 21 / Spring Boot 3 application. Its main components are:

- Spring Web REST controllers under `backend/src/main/java/com/bloodbond/`.
- Spring Security with stateless JWT authentication and role-based authorization.
- Spring Data JPA for relational persistence.
- Flyway migrations for schema changes.
- MySQL for the default configuration; an in-memory H2 database for local development and tests.
- STOMP over SockJS for authenticated live chat and room notifications.
- Groq's hosted, OpenAI-compatible chat-completions API for the educational assistant. The configured default model is `qwen/qwen3.8-27b`.

The API base URL in local development is `http://localhost:8080`; REST paths below include the `/api` prefix unless otherwise stated.

## Run and configure

From the repository root on Windows, start the backend using its local profile:

```powershell
.\backend\mvnw.cmd -f backend\pom.xml spring-boot:run "-Dspring-boot.run.profiles=local"
```

If the installed JDK is Java 17 and the project build requires compatibility overrides:

```powershell
.\backend\mvnw.cmd -f backend\pom.xml spring-boot:run "-Dspring-boot.run.profiles=local" "-Dmaven.compiler.release=17" "-Dmaven.compiler.source=17" "-Dmaven.compiler.target=17"
```

The local profile uses in-memory H2; its data is lost when the backend stops. The default application configuration uses MySQL. Flyway applies database migrations at startup, and Hibernate validates the schema.

Configuration can be supplied through environment variables and the supported `.env` locations:

| Variable | Purpose | Default |
|---|---|---|
| `SPRING_DATASOURCE_URL` | JDBC connection URL | Local MySQL URL |
| `SPRING_DATASOURCE_USERNAME` | Database user | `bloodbond` |
| `SPRING_DATASOURCE_PASSWORD` | Database password | `bloodbond` |
| `JWT_SECRET` | Signing secret for JWTs; use a strong secret outside local development | Development placeholder |
| `GROQ_API_KEY` | Server-side Groq credential | Not configured |
| `BLOODBOND_GROQ_URL` | Groq chat-completions endpoint | `https://api.groq.com/openai/v1/chat/completions` |
| `BLOODBOND_GROQ_MODEL` | Hosted model identifier | `qwen/qwen3.8-27b` |
| `BLOODBOND_ADMIN_EMAIL` | Seed admin email in the development profile | Not configured |
| `BLOODBOND_ADMIN_PASSWORD` | Seed admin password in the development profile | Not configured |

Never put the Groq key in frontend configuration or commit a real `.env` file. `.env.example` is a placeholder template.

Useful local links:

- Health: `http://localhost:8080/api/health`
- Swagger UI: `http://localhost:8080/swagger-ui.html`
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`
- H2 console (local profile only): `http://localhost:8080/h2-console`

## Authentication and common responses

Registration and login are public. All other application REST routes require a valid JWT in the HTTP header:

```http
Authorization: Bearer <jwt>
```

JWTs are stateless and expire after the configured duration (24 hours by default). Passwords are hashed using BCrypt. Registration supports `DONOR` and `RECIPIENT`; an administrator is provisioned through the development seed configuration, not public self-registration.

Successful API responses use the `ApiResponse<T>` envelope:

```json
{
  "timestamp": "2026-10-22T00:00:00Z",
  "status": 200,
  "message": "Operation completed",
  "data": {}
}
```

Validation and application errors use an error body with `timestamp`, `status`, `error`, `message`, and `path`. Common HTTP statuses include `201` for created resources, `400` for invalid input or invalid workflow actions, `401` for missing/invalid authentication, `403` for a role restriction, `404` for missing or inaccessible resources, `503` for an unavailable Groq assistant, and `500` for unexpected server errors.

### Role summary

| Role | Main capabilities |
|---|---|
| `DONOR` | Manage donor profile, view/respond to matching open requests, participate in chats, use the assistant |
| `RECIPIENT` | Manage recipient profile, search donors, create/cancel requests, initiate chats, use the assistant |
| `ADMIN` | Read aggregate statistics at `/api/admin/overview` |

## REST API reference

### Health and identity

| Method and path | Access | Purpose |
|---|---|---|
| `GET /api/health` | Public | Returns service status (`UP`). |
| `POST /api/auth/register` | Public | Creates a donor or recipient account and returns a JWT and user summary. |
| `POST /api/auth/login` | Public | Verifies credentials and returns a JWT and user summary. |
| `GET /api/users/me` | Any authenticated user | Returns the signed-in account's `id`, `name`, `email`, and `role`. |

Registration request:

```json
{
  "name": "Sam Lee",
  "email": "sam@example.com",
  "password": "ExamplePass123!",
  "role": "RECIPIENT"
}
```

The password must be 8–72 characters and contain an uppercase letter, lowercase letter, number, and special character. Registration responds with HTTP `201`; login responds with HTTP `200`. Both return an envelope whose `data` contains `token` and `user`.

Login request:

```json
{
  "email": "sam@example.com",
  "password": "ExamplePass123!"
}
```

### Profiles

| Method and path | Access | Purpose |
|---|---|---|
| `GET /api/donors/me` | `DONOR` | Loads the current user's donor profile. |
| `PUT /api/donors/me` | `DONOR` | Creates or updates the donor profile. |
| `GET /api/recipients/me` | `RECIPIENT` | Loads the current user's recipient profile. |
| `PUT /api/recipients/me` | `RECIPIENT` | Creates or updates the recipient profile. |

Donor profile update example:

```json
{
  "bloodGroup": "O+",
  "dateOfBirth": "1990-03-12",
  "city": "Springfield",
  "approxLat": 37.42,
  "approxLng": -122.08,
  "isAvailable": true
}
```

Supported groups are `A+`, `A-`, `B+`, `B-`, `AB+`, `AB-`, `O+`, and `O-`. A donor must be at least 18. Latitude and longitude must be within `-90..90` and `-180..180`. Coordinates represent approximate locations, not exact addresses.

Recipient profile update example:

```json
{
  "city": "Springfield",
  "approxLat": 37.42,
  "approxLng": -122.08
}
```

### Donor search

| Method and path | Access | Purpose |
|---|---|---|
| `GET /api/match/search` | `RECIPIENT` | Finds available donors with saved coordinates within the selected radius, ordered by distance. |

Query parameters:

| Parameter | Required | Default | Meaning |
|---|---:|---:|---|
| `lat` | Yes | — | Search-center latitude |
| `lng` | Yes | — | Search-center longitude |
| `bloodGroup` | No | No blood-group filter | Supported blood group |
| `radiusKm` | No | `50` | Search radius, from `1` to `250` km |

Example:

```http
GET /api/match/search?lat=37.42&lng=-122.08&bloodGroup=O%2B&radiusKm=50
```

The response `data` is a list of matches with donor profile ID, display name, blood group, city, approximate latitude/longitude, and distance in kilometers. It intentionally excludes the donor's email, phone number, date of birth, and exact address.

### Blood requests

| Method and path | Access | Purpose |
|---|---|---|
| `POST /api/requests` | `RECIPIENT` | Creates an open request for a blood group and approximate area. |
| `GET /api/requests/mine` | `RECIPIENT` | Lists the current recipient's requests and their response statuses. |
| `DELETE /api/requests/{requestId}` | `RECIPIENT` | Cancels the current user's open request. |
| `GET /api/requests/open` | `DONOR` | Lists open, unanswered requests in the donor's search area. |
| `POST /api/requests/{requestId}/respond` | `DONOR` | Records one donor's `ACCEPTED` or `DECLINED` response. |

Create request example:

```json
{
  "bloodGroup": "O+",
  "city": "Springfield",
  "approxLat": 37.42,
  "approxLng": -122.08,
  "radiusKm": 50
}
```

The radius must be between `1` and `250` km. Both coordinates must be supplied together or omitted together. When both donor and request have coordinates, the backend uses approximate distance and the request radius. If coordinates are not available, it falls back to a case-insensitive city match. Donor feeds omit recipient identity/contact details.

Respond example:

```json
{ "decision": "ACCEPTED" }
```

A donor needs a completed, available profile and can respond once per request. `DECLINED` records the response while leaving the request open. `ACCEPTED` marks it fulfilled, creates a private donor-recipient chat room, and notifies the recipient through the WebSocket room queue. A recipient may cancel only their own open request; a fulfilled or already-cancelled request cannot be cancelled.

### Private chat REST routes

| Method and path | Access | Purpose |
|---|---|---|
| `GET /api/chat/rooms` | Authenticated room participant | Lists only the signed-in user's rooms. |
| `POST /api/chat/rooms` | `RECIPIENT` | Starts or reuses a room with an available donor. |
| `GET /api/chat/rooms/{roomId}/messages` | Room participant | Loads persisted messages in chronological order. |
| `POST /api/chat/rooms/{roomId}/messages` | Room participant | Persists and returns a message; the live WebSocket path is preferred for chat UI. |

Create-room request:

```json
{
  "donorProfileId": 5,
  "bloodGroup": "O+",
  "city": "Springfield",
  "approxLat": 37.42,
  "approxLng": -122.08
}
```

The backend returns `201` for a newly created room and `200` if the pair already has a room. The server creates the associated blood request and notifies the donor on `/user/queue/chat/rooms` when the room is new. Message request:

```json
{ "content": "Hello, I saw your donor profile." }
```

Message content must be non-empty and no longer than 2,000 characters. A user who is not one of the room participants cannot read or send messages; inaccessible room IDs are treated as not found.

### Assistant routes

| Method and path | Access | Purpose |
|---|---|---|
| `GET /api/assistant/messages` | `DONOR` or `RECIPIENT` | Loads up to the latest 100 persisted messages for the current account. |
| `POST /api/assistant/messages` | `DONOR` or `RECIPIENT` | Saves a user message, requests a Groq completion, and saves/returns the reply. |
| `DELETE /api/assistant/messages` | `DONOR` or `RECIPIENT` | Deletes that account's assistant conversation and messages. |

Send request:

```json
{ "content": "What should I expect when donating blood?" }
```

Message content is required and limited to 2,000 characters. A successful send returns an exchange containing both the persisted user message and assistant reply. Groq/provider failures are reported as HTTP `503`; no placeholder response is returned.

### Admin route

| Method and path | Access | Purpose |
|---|---|---|
| `GET /api/admin/overview` | `ADMIN` | Returns aggregate counts for all users, donors, recipients, open/fulfilled requests, chat rooms, and donor profiles pending review. |

This endpoint returns totals only, not a list of users or their personal data. A pending review count does not represent actual verification or clinical clearance.

## Maps and donor discovery

There is no separate map-tile or geocoding API in the Spring Boot backend. The map is rendered in the browser by the frontend with Leaflet and OpenStreetMap tiles. The backend supplies approximate coordinates and distance-ordered match data through:

1. `PUT /api/donors/me` to save a donor's approximate coordinates.
2. `PUT /api/recipients/me` to save the recipient's approximate coordinates.
3. `GET /api/match/search?lat=...&lng=...&bloodGroup=...&radiusKm=...` to get nearby donor pins and distances.
4. The frontend plots those results over OpenStreetMap tiles; selecting a donor can start a private chat using `POST /api/chat/rooms`.

Browser geolocation is optional and initiated by the user in the frontend. The client rounds captured coordinates to two decimal places before saving. Matching is a discovery aid only: it does not determine clinical compatibility, donor eligibility, availability for a procedure, or blood-bank inventory. OpenStreetMap tile delivery requires internet access and is separate from the backend.

## Chat: REST and real-time WebSocket

REST loads room lists and saved history. Live messages are sent and received using STOMP over SockJS.

### Connect and authenticate

- SockJS endpoint: `http://localhost:8080/ws`
- The endpoint is a transport connection; a valid JWT is required in the STOMP `CONNECT` native header:

```text
Authorization: Bearer <jwt>
```

- STOMP application prefix: `/app`
- Broker destinations: `/topic` and `/queue`
- User destination prefix: `/user`

### Destinations

| Action | Destination | Details |
|---|---|---|
| Subscribe to room messages | `/topic/chat/rooms/{roomId}` | Only a donor or recipient participant of that room may subscribe. |
| Subscribe to room notifications | `/user/queue/chat/rooms` | Authenticated user-specific notifications for newly created rooms. |
| Publish a message | `/app/chat/rooms/{roomId}/messages` | Payload is `{"content":"..."}`; server verifies room participation and persists before broadcast. |

The backend authenticates the JWT at connection time and checks membership both for room subscriptions and message sends. Clients cannot publish to broker `/topic` or `/user` destinations. Messages are broadcast after persistence to authorized room subscribers, including the sender. New-room events are delivered privately to the relevant user.

## Groq assistant

The assistant uses Groq's hosted chat-completions API with an OpenAI-compatible request shape. The configured model default is `qwen/qwen3.8-27b`; Qwen is hosted by Groq here and is not downloaded or run locally.

### Request flow

1. The authenticated donor or recipient submits text to `POST /api/assistant/messages`.
2. The backend stores the user's message in an account-scoped conversation.
3. It loads at most the latest 12 messages for context and supplies those messages plus the user's role to Groq.
4. It stores the assistant reply and returns both sides of the exchange.
5. `GET /api/assistant/messages` reads persisted history; `DELETE` clears that user's history.

The assistant is configured by `GROQ_API_KEY`, `BLOODBOND_GROQ_URL`, and `BLOODBOND_GROQ_MODEL`. The API key stays on the server. Missing key, invalid model/endpoint, quota/rate limit, and provider outage conditions produce a clear service-unavailable response rather than a fabricated answer.

The assistant is for general education about BloodBond, blood donation, and basic blood-group concepts. It must not determine individual compatibility, diagnose, prescribe, or clear a person to donate. Users should rely on qualified blood-bank staff and treating clinicians for decisions and should not share identifying or sensitive medical information.

Private donor-recipient chat messages and profile details are not added to Groq context. The assistant conversation itself is persisted until the user deletes it. As with any external AI service, the conversation text and user role used for the completion are sent to Groq to generate a reply.

## End-to-end workflows

### Register, sign in, and set up a profile

1. Register using `POST /api/auth/register` with role `DONOR` or `RECIPIENT`.
2. Save the returned JWT and send it as a Bearer token on protected calls.
3. Read the account summary with `GET /api/users/me`.
4. Set up the role-specific profile with `PUT /api/donors/me` or `PUT /api/recipients/me`.
5. Use `GET` on the matching profile route to reload the saved profile.

### Recipient searches and starts a conversation

1. Save approximate recipient location with `PUT /api/recipients/me`, or explicitly choose a browser location in the UI.
2. Call `GET /api/match/search` with location, optional blood group, and radius.
3. The browser displays returned approximate donor locations on the map.
4. Start a room with `POST /api/chat/rooms`; the backend reuses an existing room for the pair or creates one and alerts the donor.
5. Load saved messages over REST, then connect to SockJS/STOMP and subscribe to the room topic to exchange new messages live.

### Recipient publishes a request; donor responds

1. Recipient calls `POST /api/requests` with a blood group, search radius, and optional approximate location/city.
2. Donor calls `GET /api/requests/open` to see unanswered requests within their matching area.
3. Donor calls `POST /api/requests/{requestId}/respond` with `{"decision":"DECLINED"}` or `{"decision":"ACCEPTED"}`.
4. Declining records that donor's response but leaves the request open. Accepting fulfills the request, creates a private chat, and sends the recipient a private room notification.
5. Recipient can inspect statuses using `GET /api/requests/mine`; they can cancel their own request only while it remains open.

### Donor or recipient uses the assistant

1. Call `GET /api/assistant/messages` to restore saved assistant history.
2. Send a question to `POST /api/assistant/messages`.
3. The server supplies a bounded conversation context and account role to Groq and returns the stored exchange.
4. Call `DELETE /api/assistant/messages` to permanently remove that account's assistant conversation.

## Persistence and migrations

Flyway migration files are in `backend/src/main/resources/db/migration/`:

| Migration | Main schema |
|---|---|
| `V1__init.sql` | Users, donor profiles, recipient profiles |
| `V2__matching_and_chat.sql` | Blood requests, chat rooms, chat messages |
| `V3__assistant_and_request_workflow.sql` | Assistant conversations/messages and donor request responses |

The database stores password hashes rather than plaintext passwords. Assistant history and private chat messages are persisted separately; user-to-user chat is not part of the assistant's prompt history.

## Errors and troubleshooting

| Symptom | Likely cause / check |
|---|---|
| `401 Unauthorized` | Missing, malformed, or expired Bearer JWT; sign in again and use the returned token. |
| `403 Forbidden` | Authenticated account has the wrong role for the endpoint. |
| `400 Bad Request` | Request validation failed, such as unsupported blood group, invalid coordinates, invalid radius, or an invalid lifecycle action. Read the response `message`. |
| `404 Not Found` | Requested resource does not exist or is intentionally hidden because the account is not a participant/owner. |
| Assistant returns `503` | Check that `GROQ_API_KEY` is configured server-side, model and endpoint are correct, and Groq account quota is available; restart the backend after environment changes. |
| WebSocket chat does not connect | Verify backend is running at port `8080`, connect to `/ws`, send the JWT as a STOMP native `Authorization` header, and subscribe only to a room in which the user participates. |
| Map has no tiles | OpenStreetMap tiles load from the browser and require internet access; donor results also need approximate saved coordinates and a recipient query location. |
| H2 console is unavailable | Start with the `local` Spring profile and use the H2 in-memory JDBC URL documented in the project README. |

## Privacy and safety

- Discovery and request matching use approximate locations; exact addresses and donor contact details are not returned by donor search.
- Chat rooms and messages are participant-only. Avoid sending precise addresses, identity documents, or sensitive medical data in chat.
- Assistant history is private to the signed-in account and can be cleared through the API.
- Only a bounded assistant history and role are sent to Groq; profile details and donor-recipient private chat are not included.
- Matching and assistant responses are informational. BloodBond does not provide emergency response, clinical compatibility decisions, donor eligibility verification, or medical clearance.
