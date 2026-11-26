# Roadmap and issue map

Planned 2026-11-26 against `main` @ `f571055` (README only). This is the plan the GitHub issues implement; the requirements live in [spec.md](spec.md). Nothing here is built yet.

Every phase is a parent issue, split into numbered sub-issues (`<topic> n/N: ...`) that are linked as real GitHub sub-issues. The parent's Priority and severity are the highest of its parts. Phases 1 to 7 are **provisional until Phase 0's decisions land**: recheck each against the ADRs and add a dated note rather than silently rewriting.

## Phases

| Phase | Parent | Parts | Priority | Severity | Goal |
|---|---|---|---|---|---|
| 0 | [#1](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/1) Technical assessment and architecture decisions | #9 to #16 | Urgent | high | Decisions and verified facts; no code |
| 1 | [#2](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/2) Manager foundation and independent process supervisor | #17 to #26 | High | critical | Supervised processes survive web/API restarts and are rediscovered after a supervisor restart |
| 2 | [#3](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/3) Standalone web interface and API | #27 to #37 | High | high | Operable without BCUK-Bot-4 |
| 3 | [#4](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/4) Minecraft integration | #38 to #47 | High | high | First real game, edition scope decided in #14 |
| 4 | [#5](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/5) Hytale integration | #48 to #54 | Medium | medium | Second game, verified capabilities only |
| 5 | [#6](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/6) Shared advanced functionality | #55 to #64 | Medium | high | Monitoring, backups, updates, scheduling, inactivity shutdown |
| 6 | [#7](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/7) BCUK-Bot-4 integration and deployment | #65 to #70 | Medium | medium | Both interfaces on the same manager and data |
| 7 | [#8](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/8) Hardening and documentation | #71 to #76 | Low | high | Production-ready, with tested recovery procedures |

## Order of work

```text
Phase 0  decisions (ADRs)            #9 #10 -> #11 #12 #13   #14 #15 -> #16
   |
Phase 1  foundation + supervisor     #17 -> #18 -> #19, #20, #21 -> #22 -> #23 -> #24 -> #25 -> #26
   |                                 (the survival demo #26 gates everything below)
Phase 2  API + standalone UI         #27 -> #28 -> #29 -> #30 -> #31, #32, #33, #34 -> #35 -> #36 -> #37
   |
Phase 3  Minecraft   Phase 4  Hytale (parallel after Phase 2; Hytale also needs #15 to confirm the server is usable)
   |
Phase 5  shared services (real player counts need #42; backups need #44)
   |
Phase 6  bot integration + deployment
   |
Phase 7  hardening + documentation
```

## Decisions made in Phase 0

Each becomes an ADR in `docs/adr/` once the owner accepts it.

| ADR | Issue | Decides |
|---|---|---|
| 0001 | #11 | Stack, repository layout, API style, database, tooling |
| 0002 | #12 | Process supervisor mechanism, process identity, local interface |
| 0003 | #13 | Authentication, SSO, permissions model, administrator recovery |
| 0004 | #14 | Minecraft initial edition and implementation scope |
| 0005 | #16 | Adapter contract, capability model, player-count reliability |

Supporting assessments (`docs/phase-0/`): BCUK-Bot-4 (#9), deployment host (#10), Minecraft (#14), Hytale (#15).

## Rules the plan depends on

These come straight from the spec and are repeated in the issues that enforce them:

- **The supervisor is independent.** Game servers survive restarts of BCUK-Bot-4 and of the manager web/API. A saved PID is never trusted alone.
- **Zero players is the sole inactivity condition.** It needs a valid, fresh count; an unavailable or stale count suspends the countdown and is never zero. Never CPU, memory, network or console activity. Never enabled by inference on import. Graceful shutdown only; no silent force-kill.
- **Unsupported is explicit.** Adapters declare only what they genuinely support; the UI hides or marks the rest.
- **The manager owns its data.** Own database; world files, logs and backups stay on the filesystem.
- **Bot-side work stays in the bot.** Phase 6 records it here so the plan has one home; it is mirrored to BCUK-Bot-4 issues, following that repo's rules, when it is ready to build.

## Issue conventions

Mirrors BCUK-Bot-4's issue management:

- **Severity** is a label (`severity:critical`, `high`, `medium`, `low`) saying how much it matters if never done, plus a type label (`enhancement`, `documentation`, `bug`).
- **Priority** (Urgent, High, Medium, Low) and **Effort** (High, Medium, Low) are the organisation issue fields. Every open issue has both.
- Large issues are split into `<topic> n/N: <what this part does>` sub-issues, each with its scope, tests and a "Depends on" list. Optional parts are marked `(optional)`.
- Close the last open sub-issue to complete the parent. A part found unnecessary is closed as "not planned", with a note on the parent.
- Issues record the commit they were checked against; add a dated note with the new sha when rechecking.

## Open questions for the owner

1. **Host:** is the manager deployed on the same machine as BCUK-Bot-4 or elsewhere? (#10)
2. **Stack:** the bot's Node/TypeScript stack, or something else, particularly for the supervisor? (#11, #12)
3. **Minecraft:** Java, Bedrock or both first, and which implementation? (#14)
4. **Hytale:** is dedicated server software available and permitted to run unattended? (#15)
5. **Bot-side issues:** should the Phase 6 bot work be created in BCUK-Bot-4 now or only when Phase 2 is done?
