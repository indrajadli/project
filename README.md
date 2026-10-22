# BloodBond

BloodBond is a community platform for connecting blood donors with recipients. It combines a Spring Boot API, JWT authentication, Flyway-managed relational storage, and a React application with approximate donor discovery and private participant chat.

> BloodBond supports community connections. It does not verify blood type, determine medical eligibility, or replace advice from a qualified healthcare professional.

## Project status

### Implemented

- Public landing page, donor/recipient registration, and sign-in.
- Stateless JWT authentication and role-aware frontend routes.
- Donor and recipient profiles, including approximate location capture.
- Donor availability and blood group profile fields.
- Recipient donor search by blood group and distance.
- Interactive map with approximate donor pins and distance labels.
- Private donor-recipient chat rooms and stored messages, delivered live over authenticated WebSocket connections.
- Recipient-created blood requests with donor accept/decline responses, cancellation, radius/city filtering, and an accepted-request chat.
- A Groq-powered educational assistant for donors and recipients with account-scoped, persistent conversation history.
- Role-protected admin overview with aggregate user, request, chat, and pending donor-review counts.
- Local H2 and MySQL database configurations with Flyway migrations.
- Responsive interface using the supplied sunset image as a full-app background.

### Current limitations

- Chat messages and new conversation notifications use authenticated STOMP over SockJS WebSockets. REST remains available for initial room and message history loading.
- Matching filters on the selected blood group and distance. It is a discovery aid, not clinical compatibility advice.
- Donor and recipient locations are approximate coordinates, rounded to two decimal places for map results; the UI never displays an exact address or donor email/phone.
- The local H2 database is in-memory. Its data is lost when the backend stops.
- OpenStreetMap tiles are fetched by the browser and require an internet connection.
- Donor review counts are informational; BloodBond does not verify blood group, donor eligibility, or clinical compatibility.
- The assistant sends its conversation and the signed-in user's role to Groq. It does not send donor-recipient chat history or profile details. Assistant history is retained until the user deletes it.
- The assistant provides general educational information only. Clinical decisions and urgent needs must be handled by qualified blood-bank staff, the treating hospital, or local emergency services.

## Repository layout

```text
backend/     Spring Boot 3 API (Java 21, Maven)
frontend/    React 19 + TypeScript + Vite application
docs/        API contract and project documentation
```

## Requirements

- Java 21 for the project’s configured backend build. If using Java 17 locally, use the compatibility options shown below.
- Node.js and npm.
- No Docker or MySQL is required for local development with the `local` Spring profile.

## Run locally on Windows

Open two PowerShell terminals in the repository.

### 1. Start the backend

From the `backend` folder:

```powershell
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.profiles=local"
```

If the installed Java is 17 rather than 21, use:

```powershell
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.profiles=local" "-Dmaven.compiler.release=17" "-Dmaven.compiler.source=17" "-Dmaven.compiler.target=17"
```

The backend listens on `http://localhost:8080`. Confirm it is running at [http://localhost:8080/api/health](http://localhost:8080/api/health).

### 2. Start the frontend

From the `frontend` folder:

```powershell
npm install
npm run dev
```

Open the local URL printed by Vite, normally [http://localhost:5173](http://localhost:5173).

### 3. Try donor discovery and chat

1. Register a donor account. Open **Manage donor profile**, select a blood group, enter a date of birth (donors must be 18 or older), capture or set an approximate location, and mark the profile available.
2. Register a separate recipient account. Save an approximate location in **Recipient profile**.
3. Open **Find donors**, select a blood group and radius, and select a donor marker to start a private chat.
4. Sign into the donor account in another browser or private window and open **Messages** to read and reply.

The map requires an internet connection for OpenStreetMap tiles. Browser location access requires the site to be served in a secure context; `localhost` is supported by modern browsers.

## View the local database

With the backend running under the `local` profile, open [http://localhost:8080/h2-console](http://localhost:8080/h2-console):

| Field | Value |
|---|---|
| JDBC URL | `jdbc:h2:mem:bloodbond-local;MODE=MySQL;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE` |
| User name | `sa` |
| Password | Leave blank |

The H2 console is enabled only in the `local` profile. Tables are in the `PUBLIC` schema. Local H2 data is temporary and resets when the backend process stops.

## MySQL / Docker

The default Spring configuration connects to MySQL using environment variables. For a persistent local database:

1. Configure `.env` from `.env.example`, including `GROQ_API_KEY`. The backend loads the key from a `.env` in the repository root or `backend` folder (case-insensitive key name; quoted values are accepted). The Groq model defaults to `qwen/qwen3.8-27b`; the key remains server-side and must never use a `VITE_` prefix.
2. Start the database with `docker compose up -d mysql`.
3. Run the backend with the `dev` profile and the MySQL environment variables configured.
4. Start the frontend as described above.

For development, the `dev` profile can seed an administrator. Set `BLOODBOND_ADMIN_EMAIL` and `BLOODBOND_ADMIN_PASSWORD` in the backend environment to configure the seed credentials. Do not use sample credentials outside local development.

## Backend and data model

The backend uses Spring Boot 3, Spring Web, Spring Data JPA, Spring Security, Bean Validation, JWT, and Flyway. MySQL is the default deployment database; H2 is used for local development and tests. Hibernate validates the schema while Flyway applies versioned changes.

| Table | Purpose |
|---|---|
| `users` | Account name, email, password hash, role, enabled flag, creation time |
| `donor_profiles` | Blood group, date of birth, city, approximate coordinates, availability, verification status, donation dates |
| `recipient_profiles` | City and approximate coordinates for recipient search |
| `blood_requests` | Recipient requests and their open/fulfilled lifecycle |
| `blood_request_responses` | A donor's one-time accept or decline response to a request |
| `chat_rooms` | Donor-recipient conversation participants and request reference |
| `chat_messages` | Message body, sender, room, and creation time |
| `assistant_conversations` | A private assistant conversation scoped to one account |
| `assistant_messages` | Persisted user and assistant messages used for limited context |

Database migrations are under `backend/src/main/resources/db/migration/`:

- `V1__init.sql` creates accounts and profile tables.
- `V2__matching_and_chat.sql` creates blood requests and chat storage.
- `V3__assistant_and_request_workflow.sql` adds assistant history and donor request responses.

## Frontend

The frontend uses React, TypeScript, Vite, React Router, Axios, React Hook Form, Zod, Leaflet, and OpenStreetMap tiles. The map and chat pages are lazy-loaded. The shared supplied sunset image is served from `frontend/public/bloodbond-sunset.png` and used as the full-app backdrop with a readability overlay.

Routes:

| Route | Access | Purpose |
|---|---|---|
| `/` | Public | Welcome page |
| `/login` | Public | Sign in |
| `/register` | Public | Create donor or recipient account |
| `/dashboard` | Signed in | Role-based next steps |
| `/donor/profile` | Donor | Maintain donor profile |
| `/recipient/profile` | Recipient | Maintain recipient location |
| `/discover` | Recipient | Search and view nearby donors on the map |
| `/chat` | Donor or recipient | Read and send participant-only messages |
| `/requests` | Donor or recipient | Donors respond to open recipient requests; recipients publish and track requests |
| `/assistant` | Donor or recipient | Ask educational BloodBond and donation questions; manage assistant history |
| `/admin` | Admin | View aggregate platform statistics |

The frontend API base URL defaults to `http://localhost:8080/api`; set `VITE_API_URL` to override it.

## API documentation

See [docs/api.md](docs/api.md) for route authorization, request parameters, response examples, matching behavior, privacy notes, and errors.

See [backend.md](backend.md) for the comprehensive Spring Boot backend API reference, authentication, workflows, maps, WebSocket chat, Groq assistant configuration, and troubleshooting.

## Quality checks

Run frontend checks from `frontend`:

```powershell
npm run build
npm run lint
npm test
```

Run backend tests from the repository root:

```powershell
.\backend\mvnw.cmd -f backend\pom.xml test
```

If your machine only has Java 17, add these Maven options to the backend test command:
`"-Dmaven.compiler.release=17" "-Dmaven.compiler.source=17" "-Dmaven.compiler.target=17"`.

## Privacy and safety

- Match responses include donor display name, blood group, city, approximate map coordinates, and rounded distance; they do not include donor email or phone.
- Frontend-captured coordinates are rounded to two decimal places before saving; match responses are rounded again before they are returned.
- Chat endpoints only return rooms and messages to their donor or recipient participants. Non-participant room access is reported as not found.
- Assistant history is scoped to the signed-in account. Only the assistant conversation and account role are sent to Groq; private donor-recipient chats, profile details, and contact information are not added to model context. A user can delete the stored assistant history from the assistant page.
- Donors can respond only once to an open request. Accepting creates a private chat and marks the request fulfilled; declining leaves it open. Acceptance is a choice to start a conversation, not a medical or compatibility decision.
- Donor availability and matching information are self-reported. BloodBond does not provide medical clearance or an emergency response service.
