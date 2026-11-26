# Phase 0: BCUK-Bot-4 assessment

Output of [#9](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/9) (part 1 of 8 of #1). Read-only: nothing in BCUK-Bot-4 was changed.

- **Checked against:** `Battle-Cattle/BCUK-Bot-4` `main` at commit `1b5555b643e3e6e659a3b7434100135ccca9cefb` (2026-11-26, shallow clone).
- **Method:** reading the repository (`CLAUDE.md` files, `README.md`, `DATABASE-SCHEMA.md`, `deploy/`, `deploy.sh`, `scripts/`, `src/`, `views/`). The live host was **not** inspected; anything about the running machine belongs to #10.
- **Status:** input to the ADRs (#11 stack, #12 supervisor, #13 auth). Not a decision. Each finding names its source.

## 1. Summary

| Question | Finding | Consequence for the manager |
|---|---|---|
| Stack | Node 24+, TypeScript 6/7, Express 5, EJS server-rendered pages, MySQL (`mysql2`), Vitest, ESLint | The "Node/TypeScript throughout" decision matches the bot's tooling and conventions |
| Sign-in | Discord OAuth2 (`identify` scope) or passkeys (WebAuthn). Twitch is for the bot and EventSub, not for panel sign-in | SSO means "a Discord identity the bot already whitelists" |
| Roles | Per-guild `AccessLevel` 0 to 3 in `guild_member`, plus a global `user.is_owner` flag | Map manager permissions onto these; do not invent a parallel user table |
| Sessions | Server-side in MySQL `sessions`, signed with `SESSION_SECRET`, cookie bound to the bot's origin | A cookie is not shareable across a different host or database; SSO needs an explicit hand-off (see section 4) |
| Live updates | Server-Sent Events (SSE) with caps and backpressure; polled JSON for the dashboard. No WebSocket server | Use SSE for status and console streaming |
| Deployment | One systemd unit, `deploy.sh` restarts it, Caddy in front, panel on `127.0.0.1:3000` | A bot deploy restarts only `bcuk-bot`; the manager and supervisor need their own units that `deploy.sh` never touches |
| Process model | Single instance, exits 1 on fatal errors, systemd restarts it | Exactly the fragility the supervisor must be independent of |

## 2. Architecture and conventions

### Backend and tooling
- **Runtime/language.** `package.json` `engines`: Node `>=24.22.3`, npm 11+. Type-checking and builds use `typescript7`; `typescript` 6.x stays only for tooling that needs the JS API (root `CLAUDE.md`, "TypeScript 6 + 7"). `strict` plus `noUncheckedIndexedAccess`.
- **Web.** Express 5, `helmet` with a custom CSP, `express-session` with `express-mysql-session`, `express-rate-limit` (`package.json`, `src/web/server.ts`).
- **Entry and layout.** `src/index.ts` boots DB ping, migrations, Discord, Twitch, the web panel and schedulers in a fixed order (root `CLAUDE.md`, "Startup Sequence"). Feature routes live in `src/web/routes/` and are mounted in order by `src/web/routes/index.ts`.
- **Quality gates.** Vitest (DB tests against real MySQL in CI), ESLint with complexity limits, JSDoc required on every function, `madge` circular check, `knip` unused-export check (`package.json` scripts, root `CLAUDE.md`).
- **Config.** One `.env` read by dotenv at import time in `src/shared/config.ts`. Required variables throw at startup; ports and numbers are range-validated (`parseIntEnvInRange`). dotenv does not override already-exported variables.

### Frontend
Server-rendered EJS (`views/`, `views/partials/`) with small plain browser scripts in `public/`. No SPA framework. The CSP is strict and the rules for it are in `views/CLAUDE.md`. A PWA service worker is registered (`src/web/routes/serviceWorker.ts`, `views/partials/pwa-*.ejs`).

### API and live-update conventions
- **HTML routes.** `POST` handlers redirect to `?error=code` on failure and never render an error (`src/web/CLAUDE.md`). CSRF tokens on unsafe methods (`src/web/csrf.ts`).
- **JSON routes.** `/api/*` returns `{ ok: boolean, error?: string }` shapes, with `requireAuthJson` answering `401 { ok:false, error:'unauthenticated' }` instead of redirecting (`src/web/middleware.ts`, `src/web/routes/api.ts`).
- **SSE, not WebSocket.** `src/web/routes/sseChannel.ts` implements per-channel connection sets with a 25 s keepalive, a process-wide cap (`SSE_MAX_TOTAL_CONNECTIONS`), and eviction of clients whose unflushed buffer exceeds 256 KiB. Dashboard status also has an SSE variant (`dashboardStatusEvents.ts`, `healthStatusEvents.ts`) plus a polled `/status` JSON endpoint. There is no WebSocket server.
- **Health.** `GET /healthz` is unauthenticated, sits ahead of session and rate-limit middleware, and answers only "ok" once the panel is listening (`src/web/CLAUDE.md`).
- **Rate limiting.** Two keys. The pre-session and general limiters are keyed on the client IP (`ipKey`, from `req.ip` with `trust proxy: 1`). The session limiters are keyed on the signed-in user's Discord id (`sessionLimiterKey`), so two signed-in users behind one IP get separate session buckets, though both still count against the pre-session IP limit. Some auth limiters combine IP with a path parameter (`src/web/server.ts`, `src/web/rateLimits.ts`).

### Persistence, logging, errors
- **Database.** MySQL 8.4 via `mysql2`. Schema changes are numbered files in `migrations/` (0001 to 0012 at this commit) with a SHA-256 checksum recorded in `schema_migrations`; startup fails if an applied file changed. `DATABASE-SCHEMA.md` is the schema reference. DB access goes through the `src/db.ts` facade only (root `CLAUDE.md`).
- **Logging.** `winston` with daily-rotated files in `logs/`; every `error`-level entry is also fed into an in-memory health store shown on an owner Health page (`src/shared/logger.ts`, `src/shared/healthStore.ts`). `console` is lint-banned.
- **Errors.** `uncaughtException` and `unhandledRejection` log and exit 1 on purpose, so systemd restarts from a clean state; all exits go through `exitProcess(code)` so logs flush (root `CLAUDE.md`).

## 3. Deployment and scheduling

- **Service.** `deploy/bcuk-bot.service`, installed by `scripts/install-systemd-service.sh` to `/etc/systemd/system/bcuk-bot.service`. `Restart=on-failure`, `StartLimitIntervalSec=0`, backoff to 5 minutes, `TimeoutStopSec=30`, `KillSignal=SIGTERM`. Heavily sandboxed: `ProtectSystem=strict`, `ProtectHome=tmpfs`, `NoNewPrivileges`, `CapabilityBoundingSet=` empty, `SystemCallFilter=@system-service`, `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`, `PrivateDevices`, `UMask=0077`. Writable paths are an explicit `BindPaths` list.
  - *Reusable idea:* the hardening set is a good template for the manager and supervisor units.
  - *Does not carry over as is:* the supervisor must spawn game-server processes, so it needs `PrivateDevices`, `RestrictAddressFamilies`, `SystemCallFilter`, `RestrictNamespaces` and write paths chosen for that job. Its sandbox is its own decision in #12, not a copy.
- **Deploy.** `deploy.sh` (`npm run deploy [debug [hours]]`): pre-checks `sudo -n systemctl restart bcuk-bot`, builds, restarts the one named unit, polls `/healthz` for up to 90 s, then requires 30 s of stability, and rolls back code, build and dependencies on failure. **Exception:** when a deployment brings new migrations, a failed restart is not rolled back automatically (`stop_without_rollback`). The new build stays in place because the previous build refuses to start against a database that records migrations it has no file for, and a half-applied migration leaves the schema neither old nor new. Recovery is a fix forward, or going back by hand with the steps `deploy.sh` prints. First check that no migration is still running (the report says so when the `bcuk_bot_migrations` lock is held); if one is, let it finish or check the bot's log before stopping, because stopping the service cuts it off part way. Then stop the service and restore the pre-deploy database dump (`scripts/restore-db.sh`). That replaces the live database with the dump, so any writes made after the dump was taken are lost from it: `restore-db.sh` first saves the current data as a `pre-restore` dump, but recovering those writes needs separate reconciliation. Then put the previous code back (`git reset --hard <previous sha>`), restore `dist` and `node_modules` from their `.old` copies, then start the service. Restoring the dump alone does not restore the previous build. The sudoers rule it relies on allows exactly `systemctl restart bcuk-bot` and nothing else (`scripts/install-systemd-service.sh`, `/etc/sudoers.d/bcuk-bot`).
  - *Consequence:* a bot deploy cannot restart the manager or supervisor, provided they are separate units with other names. Nothing in `deploy.sh` is generic over service names, so the manager needs its own deploy script and its own narrow sudoers rule.
- **Reverse proxy.** Caddy, one `X-Forwarded-For` hop (`trust proxy: 1`); the panel binds `127.0.0.1:3000` by default (`WEB_HOST`, `WEB_PORT` in `src/shared/config.ts`; README "Deployment"). The README requires the panel be reachable only through the proxy and `WEB_PORT` firewalled. **The Caddy configuration itself is not in the repository**, so the existing site blocks, TLS setup and body/timeout limits must be read from the host (#10).
- **Schedulers.** In-process `setInterval`-style tickers started in `main()` after the web panel (`startCounterScheduler`, `startRewardPricingScheduler`, `startTimerCommandScheduler`, DB health timer in `src/index.ts`). They assume a single process. No external job runner.
- **Backups.** `scripts/backup-db.sh` and `restore-db.sh` for the bot database, run from cron (README "Backups"). The script dumps the database only; it reads `.env` for the connection settings but does not back it up. The README asks operators to back up `.env` (above all `EVENTSUB_TOKEN_SECRET`, without which restored tokens cannot be decrypted) and the upload folders separately, off the server. Nothing here covers the manager.

## 4. Authentication, roles and what SSO can build on

### What exists
- **Sign-in methods.** Discord OAuth2 with `scope=identify`, state stored in the session with a 10-minute expiry (`src/web/routes/auth.ts`); passkeys via `@simplewebauthn` (`src/web/routes/passkeys*.ts`). Both end in the same `establishDashboardSession` (`src/web/loginSession.ts`), which regenerates the session id (fixation defence) and builds a `SessionUser`.
- **Whitelist.** Sign-in requires an existing `user` row (`findUser`); an unknown Discord id is refused (`LOGIN_KNOWN_ERRORS` includes `not_whitelisted`).
- **Roles.** `AccessLevel`: 0 User, 1 Mod, 2 Manager, 3 Admin, stored per guild in `guild_member(guild_id, discord_id, access_level)`. `user.is_owner` is a separate global super-admin, set only by hand in the DB, and resolves to Admin everywhere (`getEffectiveAccessLevelForUser` in `src/db/guildMembers.ts`; `DATABASE-SCHEMA.md`).
- **Guild context.** A session holds `currentGuildId`. `requireGuildContext` re-reads membership and owner status from the DB **on every request**, so revocation is immediate. It must run before any `requireAccessLevel` check (`src/web/CLAUDE.md`).
- **Session lifetime.** 24 h since the last page load (renewed only by `Sec-Fetch-Mode: navigate`), at most 7 days (`src/web/sessionLifetime.ts`). `user.session_generation` lets "Sign out other sessions" end all older sessions.
- **Machine auth.** Bearer-token middleware for Stream Deck and companion keys, stored only as SHA-256 hashes (`src/web/middleware.ts`, `SECURITY.md`).

### What this means for the manager (inputs to #13, not decisions)
1. The bot's session cookie is a signed id into the **bot's** MySQL `sessions` table. The manager must not read the bot database (spec rule), so it cannot validate that cookie directly.
2. Realistic options for #13: (a) the manager runs its own Discord OAuth app and keeps its own user and role tables, which breaks "one identity and role model"; (b) the bot acts as the identity provider and issues a short-lived, signed, single-use hand-off token the manager verifies, so the manager stores only an opaque Discord id plus the roles the token carries; (c) a shared-secret API where the manager asks the bot "is this Discord id Manager+ in guild G". In (c) the shared secret only authenticates the manager's request to the bot; it does not prove who the user is, so (c) still needs the manager to authenticate the user itself, for example through Discord OAuth, before asking. Option (a) needs nothing from the bot, but it gives up the shared role model unless a separate synchronisation mechanism is added. Options (b) and (c) need the bot to expose something, which falls under "do not change BCUK-Bot-4 without the owner's say-so" and so is Phase 6 work.
3. **Role granularity.** Levels are per guild, and the bot can serve several guilds. The manager needs a stated rule for which guild's level governs game servers (one configured guild id, or any guild the user is Manager+ in). This is an owner decision for #13.
4. **Revocation.** The bot re-checks on every request. A token-based hand-off does not keep that property by itself: a short lifetime only bounds how long a stale role claim can survive, and a manager session derived from the token may outlive it. Immediate revocation needs an authoritative check against the bot (option c) or an equivalent invalidation, including for sessions the manager has already created.
5. **Administrator recovery** must not depend on the bot being up. `is_owner` is DB-only today, so the manager needs its own local break-glass path (#13).

## 5. Adding a "Game Servers" navigation entry

- Navigation is a hand-written partial, `views/partials/nav.ejs`, with a `navbar` of `nav-item` links and `nav-group` dropdowns (Bot Commands, Overlays, Viewer Rewards, Devices). Groups are role-gated in the template; tests cover it (`src/web/navView.test.ts`, `navbarScript.test.ts`).
- Adding the section is a small template edit: a new link (or group) pointing at the manager's own origin, `servers.bcuk.co.uk` per the issue text, shown at Manager+ (or whatever #13 decides). Because the manager is a separate origin, this is a plain outbound link and does not need to share the bot's CSP or cookies.
- That edit belongs in BCUK-Bot-4, so it is Phase 6 and waits for the owner's say-so (#65 to #68 stay here until then).

## 6. Reuse, build, do not copy

**Reuse (as patterns, re-implemented in the manager; do not import across repos):**
- Strict TypeScript config, ESLint rules, Vitest layout, JSDoc and complexity standards.
- Config loading with eager validation and range-checked ports.
- Express middleware ordering ideas: `helmet` first, unauthenticated `/healthz` ahead of session middleware, separate JSON and HTML auth failures, per-IP rate limiting.
- SSE channel design: keepalive, global connection cap, backpressure eviction (`sseChannel.ts`).
- systemd hardening directives and the "install script fills placeholders" approach.
- Numbered, checksummed SQL migrations (`src/db/migrations.ts`).
- `winston` logging with rotated files.
- Hashed storage for machine tokens.

**Build in the manager:**
- Everything process-related: supervisor, identity verification, console streaming, adapters, backups, scheduling of game-server tasks.
- Its own database and migrations (the manager "owns its own database").
- Its own deploy script and units that never share a lifecycle with the bot.
- Its own session handling and whatever #13 chooses for sign-in.

**Do not copy:**
- **Single-instance assumptions.** The bot's in-memory cursors, dedup caches and tickers assume one process (root `CLAUDE.md`, "Single instance only"). The supervisor and web/API are separate processes by design.
- **`process.exit` on any uncaught error.** Correct for the bot, wrong for the supervisor: a crash of the manager API must never take game servers down, and the supervisor must not exit on a single bad request.
- **Shared database.** Never connect to the bot's MySQL, `user`, `guild_member` or `sessions` tables.
- **`deploy.sh` rollback model.** It swaps the checkout and restarts one named unit. The manager needs a deploy that cannot touch running game servers.
- **`PrivateDevices`, `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX` and `SystemCallFilter=@system-service`** unchanged on the supervisor unit.
- **Panel binding.** `trust proxy: 1` and a loopback bind are right for the manager's web/API too, but the supervisor's local interface should be a Unix socket or loopback with its own authentication, not an HTTP panel behind the proxy.

## 7. Risks and open items for the ADRs

| # | Item | Goes to |
|---|---|---|
| 1 | The bot is a single process that exits on error and is restarted by systemd. Nothing in the bot may own or supervise game servers (spec). | #12 |
| 2 | Caddy configuration, TLS and any existing site for `servers.bcuk.co.uk` are not in the repository; read them on the host. | #10 |
| 3 | `deploy.sh` and the sudoers rule are name-specific. Confirm the manager's units cannot be restarted by a bot deploy and the reverse. | #10, #12 |
| 4 | SSO needs a bot-side surface (token issuer or role-check API). This is a bot change, deferred until the owner agrees. | #13, Phase 6 |
| 5 | Multi-guild role mapping for the manager is undecided. | #13 |
| 6 | No WebSocket server exists in the bot; if the manager's console streaming wants one, that is a new choice, with SSE as the conforming default. | #11 |
| 7 | The bot's backup script dumps its database only; `.env` and upload folders need separate backups (the README says so; whether they are done on the host is for #10 to check). The manager's DB, config and supervisor state need their own backup story. | #10, #12, Phase 5 |
| 8 | The bot's sandbox blocks child-process-heavy work by design. Resource contention (CPU, memory, disk) between game servers and the bot on one machine is a host question. | #10 |

## 8. Recheck log

| Date | Checked against | Note |
|---|---|---|
| 2026-11-26 | `1b5555b643e3e6e659a3b7434100135ccca9cefb` | First assessment. Repository read only; live host not inspected. |
