# WhatsApp Clone — Architecture, APIs, Frontend, Keycloak, Setup

This repository contains a full-stack WhatsApp-like chat application built with:

- Angular frontend (whatsapp-clone-ui)
- Spring Boot backend (whatsappclone)
- Keycloak for authentication and identity
- PostgreSQL for persistence
- Docker Compose for local infrastructure

Use this README as a one-stop guide for schema design, backend APIs, frontend components, Keycloak flow, and how to set up and run everything locally.

---

## Architecture

- UI: Angular app serves the web client and integrates with Keycloak for auth.
- API: Spring Boot app exposes REST endpoints for users, chats, and messages.
- Auth: Keycloak provides OpenID Connect/OAuth2, tokens, and user management.
- DB: PostgreSQL stores users, chats, messages, presence/last-seen.
- Infra: Docker Compose starts Keycloak and Postgres locally.

Reference files:
- Compose: [whatsapp-clone/docker-compose.yml](whatsapp-clone/docker-compose.yml)
- Frontend SSO helper: [whatsapp-clone/whatsapp-clone-ui/public/silent-check-sso.html](whatsapp-clone/whatsapp-clone-ui/public/silent-check-sso.html)
- Example component: [whatsapp-clone/whatsapp-clone-ui/src/app/components/chat-list/chat-list.component.ts](whatsapp-clone/whatsapp-clone-ui/src/app/components/chat-list/chat-list.component.ts)
- Project resources: [whatsapp-clone/resources/whatsapp_clone.drawio](whatsapp-clone/resources/whatsapp_clone.drawio)

---

## Schema Design

Conceptual entities and relationships (one user can have many chats; a chat has many messages):

- Users
	- id (UUID)
	- username/email
	- firstName, lastName
	- online (boolean)
	- lastSeen (timestamp)

- Chats
	- id (UUID)
	- senderId (UUID → Users.id)
	- receiverId (UUID → Users.id)
	- name (display name for UI)
	- lastMessageTime (timestamp)

- Messages
	- id (UUID)
	- chatId (UUID → Chats.id)
	- senderId (UUID → Users.id)
	- content (text)
	- createdAt (timestamp)
	- delivered (boolean)
	- read (boolean)

Example minimal DDL (adapt as needed):

```sql
CREATE TABLE users (
	id UUID PRIMARY KEY,
	username TEXT UNIQUE NOT NULL,
	first_name TEXT,
	last_name TEXT,
	online BOOLEAN DEFAULT FALSE,
	last_seen TIMESTAMP NULL
);

CREATE TABLE chats (
	id UUID PRIMARY KEY,
	sender_id UUID NOT NULL REFERENCES users(id),
	receiver_id UUID NOT NULL REFERENCES users(id),
	name TEXT,
	last_message_time TIMESTAMP NULL
);

CREATE TABLE messages (
	id UUID PRIMARY KEY,
	chat_id UUID NOT NULL REFERENCES chats(id),
	sender_id UUID NOT NULL REFERENCES users(id),
	content TEXT NOT NULL,
	created_at TIMESTAMP NOT NULL DEFAULT NOW(),
	delivered BOOLEAN DEFAULT FALSE,
	read BOOLEAN DEFAULT FALSE
);
```

---

## Backend API

The backend provides REST endpoints around users, chats, and messages. The Angular client code and models (e.g., `ChatResponse`, `UserResponse`) indicate these core operations:

- Users
	- GET list of users (used to search new contacts)
	- GET current user profile (via Keycloak subject)

- Chats
	- POST create a chat between `sender-id` and `receiver-id` → returns new chat ID
	- GET chats for a user (e.g., by receiver/userId)

- Messages
	- GET all messages for a chat
	- POST send a message to a chat
	- PATCH/PUT mark as delivered/read (optional based on implementation)

Notes:
- Default Spring Boot port is typically `8080` (verify in `application.yml`).
- The frontend’s generated service layer under `src/app/services` calls these APIs.
- Endpoint shapes mirror the TypeScript models in `src/app/services/models`.

---

## Frontend Components

Angular app highlights:

- Pages
	- Main page and route config in `src/app/pages/main/*` and `src/app/app.routes.ts`.

- Components
	- Chat list: [whatsapp-clone/whatsapp-clone-ui/src/app/components/chat-list/chat-list.component.ts](whatsapp-clone/whatsapp-clone-ui/src/app/components/chat-list/chat-list.component.ts)
		- Displays existing chats
		- Searches contacts via `UserService.getAllUsers()`
		- Creates chats via `ChatService.createChat()` using Keycloak `userId`
		- Emits selected chat to parent

- Services (generated client layer)
	- Located under `src/app/services/services/*` and `src/app/services/models/*`
	- Provide typed calls for chat, message, and user endpoints

- Utilities
	- Keycloak integration service (used where `KeycloakService` is imported)
	- HTTP interceptor that attaches Bearer tokens to API calls
	- Silent SSO helper: [whatsapp-clone/whatsapp-clone-ui/public/silent-check-sso.html](whatsapp-clone/whatsapp-clone-ui/public/silent-check-sso.html)

---

## Keycloak Workflow

Authentication is handled by Keycloak with an SPA login flow:

1. On app load, the Keycloak client initializes and tries SSO using `silent-check-sso.html`.
2. If the session exists, the app retrieves an access token and user info (e.g., `sub`/userId).
3. The HTTP interceptor adds `Authorization: Bearer <token>` to API requests.
4. Backend validates the token (via Keycloak issuer and public keys) and authorizes requests.
5. User actions (search contacts, create chat, send message) carry the authenticated `userId`.

Keycloak admin console is exposed via Docker on `http://localhost:9090` by default (admin/admin from compose). Create a realm, client, and users as described below.

---

## Setup & Run (Windows)

### Prerequisites

- Docker Desktop
- Node.js (LTS: 18 or 20) + npm
- Java 17+ and Maven (or use the included Maven wrapper)

### Start Infrastructure (Postgres + Keycloak)

From the `whatsapp-clone` folder containing compose:

```powershell
cd d:\git_clone_whatsapp\whatsapp-clone
docker compose up -d
```

Services in compose: [whatsapp-clone/docker-compose.yml](whatsapp-clone/docker-compose.yml)

- PostgreSQL at `localhost:5432`, DB `whatsapp_clone`
- Keycloak admin at `http://localhost:9090` (admin/admin)

> If Postgres credentials need adjusting, edit the compose to use `POSTGRES_USER` and `POSTGRES_PASSWORD` environment variables as per the official image.

### Configure Keycloak

In `http://localhost:9090` (admin/admin):

1. Create a realm (e.g., `whatsapp-clone`).
2. Create a client (e.g., `whatsapp-web`):
	 - Type: Public (for local dev) or Confidential
	 - Valid Redirect URIs: `http://localhost:4200/*`
	 - Web Origins: `*` (or `http://localhost:4200`)
3. Create test users and set credentials.
4. (Optional) Add roles and mappers to include profile fields in tokens.

### Configure Backend

Update Spring Boot datasource and Keycloak config in `application.yml` (path: `whatsappclone/src/main/resources/application.yml`). Example:

```yaml
spring:
	datasource:
		url: jdbc:postgresql://localhost:5432/whatsapp_clone
		username: postgres
		password: <your_password>
	jpa:
		hibernate:
			ddl-auto: update
		properties:
			hibernate:
				format_sql: true

keycloak:
	issuer-uri: http://localhost:9090/realms/whatsapp-clone
	client-id: whatsapp-web
```

Adjust to match your realm/client settings.

### Run Backend

```powershell
cd d:\git_clone_whatsapp\whatsappclone
./mvnw.cmd spring-boot:run
```

The API will start (default `http://localhost:8080` unless configured otherwise).

### Run Frontend

```powershell
cd d:\git_clone_whatsapp\whatsapp-clone\whatsapp-clone-ui
npm install
npm start
```

The app serves at `http://localhost:4200`. Ensure Keycloak client settings and the frontend Keycloak config match (`silent-check-sso.html` is already present).

---

## Development Tips

- Angular generated clients under `src/app/services` reflect backend OpenAPI; regenerate if backend changes.
- Use environment-specific configs for Keycloak endpoints in dev vs prod.
- For CORS issues, verify backend CORS and Keycloak Web Origins.
- For token issues, check clock skew and token refresh handling.

---

## Troubleshooting

- Cannot log in:
	- Verify realm/client config and redirect URIs.
	- Check Keycloak is running at `http://localhost:9090`.

- 401/403 from API:
	- Confirm `Authorization: Bearer` header is sent.
	- Validate token audience and issuer match backend config.

- DB errors:
	- Ensure Postgres credentials match your `application.yml`.
	- Confirm database `whatsapp_clone` exists.

---

## Project Structure

- Infra and docs: [whatsapp-clone](whatsapp-clone)
- Frontend: [whatsapp-clone/whatsapp-clone-ui](whatsapp-clone/whatsapp-clone-ui)
- Backend: [whatsappclone](whatsappclone)

Contributions and improvements welcome.

# whatsapp-clone
WhatsApp clone application using Spring boot 3, Angular 19, Keycloak, PrimeNg
