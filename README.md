# JustCrea8 Backend — Phase 1

Spring Boot REST/WebSocket backend for the JustCrea8 platform.

**Stack:** Spring Boot 3.3 (Java 17) · Spring Security + JWT · MongoDB Atlas · Redis · Docker · WebSocket (STOMP, wired for phase 2)

> This is **Phase 1**: Auth, User Profiles, Projects, Teams, and Task/Kanban boards — the modules everything else depends on.
> Phase 2 will add: Notes, Todos, Blog, Polls, Chat, and Group Discussion Rooms (real-time via the WebSocket
> infra already configured in `config/WebSocketConfig.java`).

## What's included

| Module | Endpoints base | Notes |
|---|---|---|
| Auth | `/api/auth/*` | register, verify-email, login, refresh, logout, forgot/reset password |
| Users | `/api/users/*` | profile get/update, search |
| Projects | `/api/projects/*` | CRUD, members, ownership rules, Redis-cached reads |
| Teams | `/api/teams/*` | CRUD, invite-code join, members, Redis-cached reads |
| Tasks (Kanban) | `/api/tasks/*` | CRUD + drag-and-drop `move`, broadcasts live updates over `/topic/tasks/{projectId}` |

Cross-cutting: JWT access/refresh tokens (refresh tokens tracked + revocable in Redis), BCrypt password
hashing, global exception handling with consistent JSON error shape, Swagger UI at `/swagger-ui.html`,
Actuator health check at `/actuator/health`, CORS configured for your frontend origin(s).

## Local setup

### 1. Prerequisites
- Java 17+, Maven 3.9+
- A MongoDB Atlas cluster (free tier is fine) — or run the commented-out local `mongo` service in
  `docker-compose.yml`
- Docker + Docker Compose (optional but recommended)

### 2. Configure environment
```bash
cp .env.example .env
```
Fill in:
- `MONGODB_URI` — from Atlas: **Connect → Drivers**, paste the connection string and append `/justcrea8` before the `?`.
- `JWT_SECRET` — generate with `openssl rand -base64 32` (must be Base64, ≥32 bytes).
- `MAIL_USERNAME` / `MAIL_PASSWORD` — optional; if left blank, verification/reset emails are skipped
  silently (logged as a warning) so auth still works without SMTP configured.

### 3a. Run with Docker (recommended)
```bash
docker compose up --build
```
This starts Redis + the backend (port `8080`), pointed at your Atlas cluster via `MONGODB_URI`.

### 3b. Run locally without Docker
Start Redis (`redis-server`, or `docker run -p 6379:6379 redis:7-alpine`), export the vars from `.env`,
then:
```bash
mvn spring-boot:run
```

### 4. Verify
- Swagger UI: `http://localhost:8080/swagger-ui.html`
- Health: `http://localhost:8080/actuator/health`

> **Note on this sandbox:** the code was written and manually reviewed here, but this container's network
> allowlist doesn't include Maven Central, so `mvn compile` couldn't be run to completion in this session.
> Run `mvn clean package` locally or in your CI (GitHub Actions, etc.) as the first step to confirm a clean
> build before deploying.

## Auth flow (JWT)

1. `POST /api/auth/register` → creates user, sends verification email with a Redis-backed one-time token (24h TTL).
2. `POST /api/auth/verify-email` → marks the account verified.
3. `POST /api/auth/login` → returns `{ accessToken, refreshToken, expiresIn, user }`. Access token: 15 min.
   Refresh token: 7 days, tracked in Redis so it can be revoked.
4. Send `Authorization: Bearer <accessToken>` on every subsequent request.
5. When the access token expires, `POST /api/auth/refresh` with the refresh token — this **rotates** it
   (old one is revoked, a new pair is issued).
6. `POST /api/auth/logout` revokes the refresh token and blacklists the current access token in Redis until
   it would have naturally expired.

## Redis usage in this build
- Refresh-token allow-list (`refresh:{userId}:{jti}`) + access-token blacklist on logout
- Email-verification and password-reset one-time tokens
- `@Cacheable` read-through cache for project/team/profile GETs (10 min TTL, evicted on writes)

## Project structure
```
src/main/java/com/justcrea8/backend/
  config/       Security, Redis, WebSocket, OpenAPI config
  security/     JWT service, filter, UserDetails, refresh-token store
  common/       ApiResponse/ErrorResponse wrappers, custom exceptions, global handler
  auth/         Register/login/refresh/logout/password-reset
  user/         Profile
  project/      Projects + membership
  team/         Teams + invite-code join
  task/         Kanban tasks (with realtime WS broadcast)
```

## Connecting the React frontend

The current frontend (`justCrea8-Official-main`) talks directly to Firebase (Auth + Firestore) — there's no
existing API client layer. To integrate this backend you'll want to:

1. Add an `axios` instance (`src/api/client.js`) that attaches `Authorization: Bearer <token>` and handles
   401 → refresh-token retry. See `frontend-integration/apiClient.js` for a ready-to-drop-in version.
2. Swap `AuthProvider.jsx`'s Firebase calls for calls to `/api/auth/*`, storing the returned tokens
   (e.g. in memory + `httpOnly`-style handling, or `localStorage` for a simpler first pass).
3. Replace direct Firestore reads/writes in your project/team/task pages with calls to the matching REST
   endpoints above.

This is a page-by-page migration — happy to wire up specific pages once you're ready; just point me at them.

## Next (Phase 2)
Notes, Todos, Blog, Polls, project Chat, and Group Discussion Rooms — the last two using the STOMP
WebSocket endpoint (`/ws`) already configured with JWT handshake auth in `config/WebSocketConfig.java`.
