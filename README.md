# Nexora

A Discord-like communication platform with secure remote desktop access, built
as a pnpm + Turborepo monorepo of independently deployable NestJS services and
an Electron desktop client.

Messages, attachments and call media are end-to-end encrypted: the server
stores ciphertext and routes it, and never holds a key that opens it.

`CLAUDE.md` is the target architecture. `development/` tracks what is built,
why each decision was taken, and what is deliberately left open.

---

## Status

| Area | State |
| --- | --- |
| Accounts | Register, login, refresh-token rotation with reuse detection, Google and GitHub sign-in, admin panel |
| Servers | Servers, roles, per-member permission overrides, invites by slug, server settings |
| Channels | Public and private text channels, private channels as an allowlist, direct messages between friends |
| Messages | End-to-end encrypted, realtime over WebSocket, history paging, attachments of any type |
| Voice and video | LiveKit voice channels, camera, screen share with a source picker, end-to-end encrypted media |
| Presence | Online / idle / do not disturb / invisible, typing indicators, voice rosters |
| Notifications | Desktop notifications, system tray, start with the system, per-channel mute, quiet hours, persisted unread |
| Remote desktop | Not built - `remote-gateway` and `remote-agent` are scaffolds |

Known limits are written down rather than implied: see `development/E2EE.md`
for what the encryption does and does not protect, and `development/TODO.md`
for everything each phase left open on purpose.

---

## Architecture

Five concerns are kept apart, and the separation is the design: public ingress,
internal routing, application logic, realtime media, and remote access.

```
                              Internet
                                 │
                          Cloudflare Tunnel          (no inbound ports opened)
                                 │
                        ┌────────▼────────┐
                        │  Nginx :8080    │          routing, rate limits,
                        │  API gateway    │          body caps, WS upgrade
                        └────────┬────────┘
                                 │
   ┌──────────────┬──────────────┼───────────────┬──────────────┐
   │              │              │               │              │
┌──▼───┐   ┌──────▼─────┐  ┌─────▼─────┐  ┌──────▼──────┐ ┌─────▼──────┐
│ auth │   │  server    │  │   chat    │  │  presence   │ │notification│
│ 3001 │   │   3003     │  │   3004    │  │    3005     │ │    3006    │
└──┬───┘   └──────┬─────┘  └─────┬─────┘  └──────┬──────┘ └─────┬──────┘
   │              │              │               │              │
   └──────────────┴──────┬───────┴───────────────┴──────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
      ┌─────▼──────┐            ┌─────▼─────┐
      │ PostgreSQL │            │   Redis   │  pub/sub, presence,
      │   :5432    │            │   :6379   │  rate limits, sessions
      └────────────┘            └───────────┘

  Desktop ──── WebRTC (E2EE media) ───▶ LiveKit SFU :7880   ◀── tokens from
                                                                call-service :3007
```

Media never passes through a NestJS process. `call-service` decides who may
join a room and mints a LiveKit token; the client dials the SFU directly and
encrypts its own tracks with the channel key.

### How a request actually flows

Sending a message, from the keystroke to the other person's screen. Every layer
does one job, and the boundaries are where the security properties live.

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1  Electron renderer                                                 │
│    Zustand store → encrypt with the channel key (AES-256-GCM)        │
│    Plaintext stops here. Everything below sees ciphertext.           │
└───────────────────────────────┬──────────────────────────────────────┘
                                │  POST /api/v1/messages   (JWT bearer)
┌───────────────────────────────▼──────────────────────────────────────┐
│ 2  Nginx :8080          (in development, the Vite proxy stands in)   │
│    Route by path · rate limit · body cap · forward x-request-id      │
│    No business logic, no database, no idea what a message is.        │
└───────────────────────────────┬──────────────────────────────────────┘
                                │  http://chat-service:3004
┌───────────────────────────────▼──────────────────────────────────────┐
│ 3  chat-service (NestJS)                                             │
│    JwtAuthGuard         → who is this                                │
│    ValidationPipe / DTO → is the body the right shape                │
│    Controller (thin)    → hands off, decides nothing                 │
│    MessagesService      → resolveChannelAccess(user, channel):       │
│                           membership, role, overrides, allowlist     │
└───────────────────────────────┬──────────────────────────────────────┘
                                │  typed Prisma calls
┌───────────────────────────────▼──────────────────────────────────────┐
│ 4  Prisma (@nexora/database)                                         │
│    One schema, one generated client, one connection pool.            │
│    Typed queries in, rows out - the only thing that speaks SQL.      │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│ 5  PostgreSQL :5432                                                  │
│    messages(id, channelId, authorId, content = ciphertext, …)        │
└───────────────────────────────┬──────────────────────────────────────┘
                                │  row written
┌───────────────────────────────▼──────────────────────────────────────┐
│ 6  Redis :6379 - publish `message.created`                           │
│    Fanout, not storage: every chat-service instance is subscribed,   │
│    so a client on another instance is reached just the same.         │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│ 7  /ws/chat gateways → sockets subscribed to that channel            │
│    → each renderer decrypts with its own copy of the channel key,    │
│      updates the store, and decides whether to notify                │
└──────────────────────────────────────────────────────────────────────┘
```

The sender's own client is on that same socket path: there is no optimistic
insert, so a message appears when it is real, and the copy that appears is the
one that came back through the fanout.

Reads take the first five steps and stop: `GET /api/v1/messages` is the same
guard, the same access check, and one indexed Prisma query on
`(channelId, createdAt)`. The client caches the decrypted result per channel in
memory, so reopening a conversation paints immediately and refreshes behind it.

Two paths deliberately skip this stack:

- **Media** goes `desktop → LiveKit SFU` over WebRTC. `call-service` only mints
  the token that says which room the user may join.
- **Attachments** are sealed in the renderer and uploaded as bytes; the server
  stores an object it cannot type and hands back a key.

### Services

Each service is its own package with its own `package.json`, `Dockerfile` and
`GET /health`, and can be deployed on its own.

| Service | Port | Owns |
| --- | --- | --- |
| `api-gateway` | 8080 | Nginx config: routing, rate limiting, request caps, WebSocket upgrade. No business logic |
| `auth-service` | 3001 | Registration, login, JWT access and refresh tokens with rotation, OAuth, the admin API |
| `server-service` | 3003 | Servers, members, roles, permission overrides, channels, invites |
| `chat-service` | 3004 | Messages, `/ws/chat` fanout, uploads, the E2EE key directory, friends and DMs |
| `presence-service` | 3005 | `/ws/presence`, online status, typing, voice rosters, all Redis-backed |
| `notification-service` | 3006 | Notification preferences, per-channel mutes, quiet hours, read markers |
| `call-service` | 3007 | LiveKit room permissions and access tokens |
| `remote-gateway` | 3008 | Remote machines, per-machine permissions, session relay on `/ws/remote`, audit log |
| `remote-agent` | — | Scaffold, for a headless machine. On a desktop the agent is the app itself |

`user-service` is a scaffold too; its routes (profiles, friends, user search)
are served by chat-service until it exists.

### Shared packages

Cross-cutting code only - no service's business logic lives here.

| Package | Holds |
| --- | --- |
| `@nexora/shared-types` | DTOs, API contracts, WebSocket event unions. One source for client and server |
| `@nexora/database` | Prisma schema and client, plus `resolveChannelAccess` and `resolveRemoteAccess` - the single answers to "may this user do this here" and "…to this machine" |
| `@nexora/auth` | JWT sign and verify, `JwtAuthGuard`, `@CurrentUser()`, secret sealing |
| `@nexora/permissions` | Role constants and the override arithmetic (deny beats grant beats role) |
| `@nexora/events` | Event names, payload contracts, and the Redis pub/sub bus |
| `@nexora/nest-common` | Bootstrap, request ids, the error contract, `/health`, Redis-backed rate limiting |
| `@nexora/storage` | Local-disk and S3 drivers behind one interface, including multipart upload |
| `@nexora/websocket` | Shared socket plumbing |
| `@nexora/logger` | Structured JSON logging with redaction |
| `@nexora/config` | Typed environment loading |

### Realtime

- **`/ws/chat`** - JWT handshake, per-channel subscriptions, fanout driven by
  Redis pub/sub so any number of chat-service instances stay in step.
- **`/ws/presence`** - status, typing and voice rosters, with Redis holding the
  live state rather than any single process.
- **`/ws/remote`** - the relay between a remote session's controller and the
  agent on the machine, with every input event checked against the permissions
  frozen on the session.
- **WebRTC to LiveKit** - voice, video, screen share and a remote machine's
  screen. Never over WebSocket.

### Data

PostgreSQL holds persistent state through Prisma: users, identities, servers,
members, roles, channels, channel allowlists, friendships, messages, device
keys, wrapped channel keys, notification settings, read markers, remote
machines, remote grants, remote sessions and the remote audit trail.

Redis holds what is live and cheap to lose: presence, typing, voice rosters,
pub/sub, rate-limit windows.

Object storage (local disk or any S3-compatible bucket) holds files. Postgres
keeps the metadata; blobs never go in a column.

---

## Security

- **End-to-end encryption.** ECDH P-256 identity key per device, HKDF key
  wrapping, AES-256-GCM for messages, attachments and call media. Private keys
  are sealed with the OS keychain through Electron `safeStorage` and never
  leave the machine. A call that cannot encrypt aborts rather than downgrading.
- **Attachments are encrypted before upload**, and their manifest - name, real
  type, size - travels inside the encrypted message body, not in columns. The
  server cannot type what it stores, so it serves everything as
  `application/octet-stream` with a download disposition.
- **Authorization is server-side and central.** `resolveChannelAccess` answers
  every channel question for chat-, call- and presence-service. A private
  channel is an allowlist that ownership does not override, and a channel the
  caller cannot see answers 404 rather than 403, so ids cannot be probed for.
- **Refresh-token rotation with reuse detection** - replaying a consumed token
  revokes the whole family.
- **Rate limiting twice**: per address at the gateway, and again in the service
  through Redis, so every instance shares one budget.
- **Hardened renderer**: context isolation on, node integration off, sandbox
  on, a permission handler that allows only microphone, camera and display
  capture, and navigation locked to the app origin.
- **Structured logs never carry** passwords, tokens, keys or message content.
  Every request logs a request id, and the id survives a hop between services.

`development/E2EE.md` states the threat model and the known gaps plainly.

---

## Requirements

- Node.js 20+
- pnpm 9
- Docker - Postgres, Redis and LiveKit in development; the whole stack in a
  deployment (see [Deploy with Docker](#deploy-with-docker))

## Quick start

```bash
cp .env.example .env            # then set the JWT secrets
pnpm dev:infra                  # Postgres, Redis, LiveKit in Docker
pnpm install
pnpm db:generate
pnpm db:migrate                 # creates the schema
pnpm db:seed                    # optional: demo@nexora.local / nexora123
pnpm dev:backend                # every service, no renderer (leave running)
pnpm dev:duo                    # second terminal: two signed-in test windows
```

Use `pnpm dev:backend`, not `pnpm dev`: the latter also starts the desktop
renderer on 5173, which `dev:duo` needs for its own Vite. `pnpm dev:desktop`
runs a single client against a running backend.

Default ports in development: gateway `8080`, auth `3001`, server `3003`, chat
`3004`, presence `3005`, notification `3006`, call `3007`, LiveKit `7880`,
renderer `5173`, admin panel `5174`.

---

## Deploy with Docker

`infrastructure/docker/docker-compose.yml` runs the whole stack - every
service, Nginx, the admin panel, Postgres, Redis and LiveKit - in containers.
Only the gateway and LiveKit publish ports; the data stores publish none.

### 1. Configure

Every command below is run **from the repository root**, which is where Compose
reads `.env`.

```bash
cp .env.example .env
```

The compose file supplies `DATABASE_URL`, `REDIS_URL` and the service ports
itself, so those entries in `.env` are ignored in this mode. What it does read
from `.env`:

| Variable | Set it to | Why |
| --- | --- | --- |
| `JWT_SECRET` | 48 random bytes, hex | Required; the stack refuses to start without it |
| `JWT_REFRESH_SECRET` | a *different* 48 random bytes | Required |
| `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` | your own pair, not `devkey` | Required; the shipped pair is development-only |
| `LIVEKIT_URL` | `/livekit` | **Change this.** `.env.example` ships the host-development value `ws://127.0.0.1:7880`, which points every client at its own machine. `/livekit` keeps the deployment on one address |
| `PUBLIC_API_URL` | the public base URL, e.g. `https://nexora.example.com` | The OAuth callback is built from it |
| `OAUTH_ALLOWED_REDIRECTS` | `<PUBLIC_API_URL>/admin` | Where a finished OAuth sign-in may return |
| `CORS_ORIGIN` | your public origin, or leave `*` | Browser clients only; the desktop app is not affected |
| `GATEWAY_PORT` | `8080`, or `127.0.0.1:8080` | The second form keeps the gateway off the LAN while a host tunnel can still reach it |
| `SETTINGS_SECRET` | optional, another random value | Seals OAuth client secrets at rest; falls back to `JWT_SECRET` |
| `POSTGRES_PASSWORD` | a real password | Defaults to `postgres` |

Generate a secret with:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

### 2. Start

```bash
docker compose -f infrastructure/docker/docker-compose.yml up -d --build
```

The one-shot `migrate` container applies the Prisma migrations before any
service accepts traffic, so there is no separate schema step. Check it came up:

```bash
docker compose -f infrastructure/docker/docker-compose.yml ps
curl http://localhost:8080/health
```

### 3. Create the first administrator

The admin panel has no sign-up. The first administrator is created against the
database, and the generated password is printed once:

```bash
docker compose -f infrastructure/docker/docker-compose.yml run --rm \
  -w /repo/packages/database migrate \
  ./node_modules/.bin/tsx prisma/create-admin.ts
```

This reuses the `migrate` container, which already has the schema, the Prisma
client and database access. It creates:

```
username  nexoraadmin
email     admin@nexora.local
password  printed once by the command above
```

The account must change that password at first login. Re-running the command
when the account exists does nothing; add `--reset` at the end of the command
to issue a new password and revoke every session the old one left behind.

Sign in at `http://localhost:8080/admin` (or `https://nexora.example.com/admin`
once the tunnel is up) - the panel is served behind the same gateway.

Ordinary user accounts need none of this: they register from the desktop app's
sign-up screen.

### 4. Open the ports that matter

| Port | Carries | Needed publicly? |
| --- | --- | --- |
| `8080/tcp` | Everything: REST, `/ws/chat`, `/ws/presence`, `/ws/remote`, uploads, `/admin`, LiveKit signalling | Yes - or point a Cloudflare Tunnel at it instead |
| `7881/tcp`, `50000-50019/udp` | WebRTC media to the SFU | Yes, for voice, video, screen share and remote desktop |
| `5432`, `6379` | Postgres, Redis | No - they publish no ports at all |

A tunnel carries `8080` fine; it cannot carry the media path. Voice and screen
share need those LiveKit ports reachable, or a TURN relay in front of them.

### 5. Day to day

```bash
# logs, all services or one
docker compose -f infrastructure/docker/docker-compose.yml logs -f
docker compose -f infrastructure/docker/docker-compose.yml logs -f chat-service

# update to a new build (migrations reapply on the way up)
git pull
docker compose -f infrastructure/docker/docker-compose.yml up -d --build

# stop, keeping the database and uploaded files
docker compose -f infrastructure/docker/docker-compose.yml down

# stop and delete them too
docker compose -f infrastructure/docker/docker-compose.yml down -v
```

Postgres, Redis and the uploads live in named volumes (`postgres-data`,
`redis-data`, `upload-data`), so `down` without `-v` keeps everything.

## Desktop client

Electron with a hardened preload, React, Tailwind and Zustand. It keeps running
in the system tray when the window is closed - which is what makes a
notification possible while it is "shut" - and starts with the system by
default, with both switches in Settings → Notifications.

```
pnpm dev:desktop     one client against a running backend
pnpm dev:duo         two windows, two profiles, two encryption identities
pnpm dev:admin       the admin panel on 5174
pnpm admin:create    bootstrap the first administrator (password printed once)
```

### One address, and how to change it

A deployment is **one URL**. REST, `/ws/chat`, `/ws/presence`, uploaded files
and LiveKit's signalling handshake are all behind the same gateway, so there is
a single variable to set:

```
VITE_API_URL="https://nexora.example.com"
```

Which URL that is:

| Where the backend runs | Point the desktop app at |
| --- | --- |
| `pnpm dev:backend` on this machine | nothing - `pnpm dev` proxies to the services itself |
| Docker compose on this machine | `http://localhost:8080` (the built-in default) |
| Docker compose on another machine on the LAN | `http://<its-ip>:8080` |
| Behind a Cloudflare Tunnel | `https://nexora.example.com` |

The port is `GATEWAY_PORT`, never a service port: `3001`, `3004` and the rest
are internal to the Docker network and are not what a client talks to.

`VITE_API_URL` is read from the repo-root `.env` and baked in at build time, so
it only affects a packaged build:

```bash
VITE_API_URL="https://nexora.example.com" pnpm --filter @nexora/desktop package
```

`pnpm dev` ignores it deliberately - the Vite dev server proxies to the
services itself, and its own origin is then the gateway.

It is only the default. **Connect to a self-hosted instance** on the login
screen (and *Change server* in Settings → My Account) points the window at any
other deployment: the address is checked before it is stored, and connecting
elsewhere signs the window out and reloads. So a build can ship pointed at one
deployment without being locked to it.

WebRTC media is the one exception, and it is inherent rather than a shortcut:
the SFU negotiates its own path on `7881/tcp` and `50000-50019/udp`. One
hostname carries signalling, not media.

## Remote desktop

A machine offers itself by turning on **Settings → Remote Access**. It enrols
under the account signed in on it, keeps its credential in the OS keychain and
dials *out* to the gateway - nothing listens, no port is opened, and 3389 is
published nowhere in this stack.

Access is granted per person per machine, never by a server role: owning a
machine grants everything on it, and anybody else holds exactly what they were
given, optionally until a date. `REMOTE_VIEW`, `REMOTE_CONTROL` and
`REMOTE_CLIPBOARD` are implemented; `REMOTE_FILE_TRANSFER` and `REMOTE_AUDIO`
exist in the vocabulary and do nothing yet.

The screen travels over the same SFU voice channels use. The gateway relays
input and refuses anything the session was not granted, so a view-only session
cannot type however the client is built - and refusals are audited alongside
the sessions themselves, which a machine's owner can read.

The owner connecting to their own machine starts immediately; anyone else
raises a prompt on the machine that refuses itself if nobody answers, and a
banner stays up for as long as the session does. Mouse and keyboard injection
is Windows-only for now - elsewhere a session can watch but not touch.

## Public ingress

Nexora is one public hostname. If a `cloudflared` already runs on the server,
add one ingress entry:

```yaml
- hostname: nexora.example.com
  service: http://localhost:8080     # GATEWAY_PORT
```

and reload it - no extra container. To let Nexora bring its own tunnel instead:

```bash
CLOUDFLARE_TUNNEL_TOKEN=... docker compose   -f infrastructure/docker/docker-compose.yml --profile public up -d
```

`infrastructure/cloudflare/tunnel.yml` documents both, including the one thing
that does not go through a tunnel: WebRTC media negotiates its own UDP path to
the SFU (`7881/tcp`, `50000-50019/udp`). Chat, files and everything else are
complete over the tunnel on their own.

## File storage

`@nexora/storage` picks its driver from the environment:

- **S3 variables empty (default):** files land in `LOCAL_STORAGE_PATH`
  (`./storage-data`) and chat-service serves them. Nothing to configure.
- **`S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` all set:** the
  S3 driver takes over. Partial configuration stays on local disk rather than
  half-working.
- `STORAGE_DRIVER=local|s3` forces one. Forcing `s3` without credentials fails
  at boot instead of silently falling back.

Keys are UUID-based, so a client filename never decides where a file lands.
Anything over 8 MB uploads in parts, with the session carried by the client as
a sealed ticket rather than held as state in a service.

---

## Repository layout

```
apps/
  desktop/                Electron + React + Tailwind + Zustand client
  admin/                  Admin panel (React, served under /admin)
  services/
    api-gateway/          Nginx configuration
    auth-service/         Accounts, tokens, OAuth, admin API
    server-service/       Servers, roles, channels
    chat-service/         Messages, /ws/chat, uploads, E2EE directory
    presence-service/     /ws/presence, status, typing, voice rosters
    notification-service/ Preferences, mutes, quiet hours, read markers
    call-service/         LiveKit tokens and room permissions
    remote-gateway/       Remote machines, permissions, session relay, audit
    remote-agent/         Scaffold - for a headless machine with no app on it
    user-service/         Scaffold
packages/                 shared-types, database, auth, permissions, events,
                          nest-common, storage, websocket, logger, config
infrastructure/           docker compose, nginx, cloudflare, livekit
development/              planning, MVP, E2EE design, testing guide, TODO
```

## Common commands

| Command | Effect |
| --- | --- |
| `pnpm dev:backend` | Every service, no renderer |
| `pnpm dev` | Everything, including the desktop renderer |
| `pnpm dev:duo` | Two signed-in desktop windows for chat, voice and presence |
| `pnpm build` | Build every package, service and the desktop bundle |
| `pnpm typecheck` | Type-check the whole monorepo |
| `pnpm check` | Package self-checks - crypto, storage, logger, auth, permissions, websocket |
| `pnpm db:migrate` / `db:seed` / `db:studio` | Schema, demo data, Prisma Studio |
| `pnpm admin:create` | Create the first administrator |
| `pnpm --filter @nexora/chat-service smoke` | End-to-end check against running services |
| `pnpm --filter @nexora/presence-service smoke` | Presence, typing and voice rosters |
| `pnpm --filter @nexora/notification-service smoke` | Preferences, unread counting, read markers |
| `pnpm --filter @nexora/remote-gateway smoke` | Enrolment, grants, and what the remote relay refuses |

## Testing and CI

Three layers, all runnable locally:

1. **Self-checks** (`pnpm check`) need no infrastructure: the crypto
   primitives, the storage drivers including a multipart round trip, the
   permission arithmetic, the logger's redaction.
2. **Smoke scripts** drive the real HTTP and WebSocket surface against running
   services and exit non-zero on a failed assertion.
3. **Manual walkthroughs** for what only a human can judge - two people in a
   voice call, a shared screen, a notification arriving from the tray.
   `development/TESTING.md` lists them.

GitHub Actions runs install → lint → typecheck → build → self-checks, then an
integration job with Postgres and Redis containers that applies the migrations,
starts the services and runs every smoke script.

## API surface

```
POST /api/v1/auth/register|login|refresh|logout    GET /api/v1/auth/me
GET|POST /api/v1/servers          POST /api/v1/servers/join
GET|PATCH|DELETE /api/v1/servers/:id       POST /api/v1/servers/:id/leave
GET /api/v1/servers/:id/members   PATCH|DELETE /api/v1/servers/:id/members/:userId
GET|POST /api/v1/channels         PATCH|DELETE /api/v1/channels/:id
GET|PUT /api/v1/channels/:id/members       GET|POST /api/v1/messages
GET /api/v1/users/search          GET|POST /api/v1/friends
POST /api/v1/friends/:id/accept   DELETE /api/v1/friends/:id
GET|POST /api/v1/dm
POST /api/v1/uploads              GET /api/v1/uploads/:key
GET|POST /api/v1/e2ee/devices     GET /api/v1/e2ee/keys/:channelId
POST /api/v1/e2ee/keys            POST /api/v1/calls/token
GET|PATCH /api/v1/notifications/preferences
GET /api/v1/notifications/unread  POST /api/v1/notifications/read
GET|POST /api/v1/remote/machines  PATCH|DELETE /api/v1/remote/machines/:id
GET|PUT /api/v1/remote/machines/:id/grants
GET /api/v1/remote/machines/:id/audit
POST /api/v1/remote/sessions      DELETE /api/v1/remote/sessions/:id
GET /api/v1/admin/...             (administrators only)
WS   /ws/chat                     WS  /ws/presence      WS /ws/remote
GET  /health                      (every service)
```

Errors share one shape everywhere:

```json
{ "error": { "code": "CHANNEL_NOT_FOUND", "message": "Channel not found", "requestId": "..." } }
```

## Conventions

- TypeScript strict everywhere; no `any` in committed code.
- Controllers stay thin, services hold the logic, persistence stays in services.
- Every service exposes `GET /health` and answers the shared error contract.
- Conventional commits (`feat:`, `fix:`, `chore:`, `docs:`).

## Licence

**Source-available, view only. Not open source, and not MIT.**

You may read this code and refer to it. You may not copy, modify, redistribute,
or use it - in whole or in part, in source or compiled form, commercially or
not - without written permission from the copyright holder. Full terms in
[`LICENSE`](LICENSE).

Third-party dependencies keep their own licences.

## Documentation

| Document | Covers |
| --- | --- |
| `CLAUDE.md` | Target architecture, in full |
| `development/PLANNING.md` | Phase map, every architectural decision and why |
| `development/MVP.md` | What the first runnable version covered |
| `development/E2EE.md` | Encryption design, threat model, known limits |
| `development/TESTING.md` | Running two clients locally, and what to try |
| `development/TODO.md` | Ordered backlog, including what each phase left open |
