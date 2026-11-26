# BCUK-Game-Server-Manager: agent instructions

A standalone, multi-game server manager that BCUK-Bot-4 integrates with. **The repo is in planning: there is no code yet.** Read these before doing anything:

1. [`docs/spec.md`](docs/spec.md): the requirements (the source of truth).
2. [`docs/roadmap.md`](docs/roadmap.md): the phases, the issue map, the order of work and the owner's decisions.
3. The GitHub issues: 8 phase parents (#1 to #8) and 68 numbered sub-issues (#9 to #76). Each sub-issue has its scope, tests and a "Depends on" list.

## Where to start

**Phase 0 (#1) first.** It is documentation and decisions only; write no implementation code until its ADRs are accepted by the repo owner. Suggested order: #9 (assess BCUK-Bot-4) and #10 (host), then #11, #12, #13, with #14 and #15 in parallel, then #16. Phase 0 outputs go in `docs/phase-0/` and `docs/adr/` (`0001` to `0005`, mapped in the roadmap).

Phases 1 to 7 are **provisional until the Phase 0 ADRs land**. When picking one up, recheck its issues against the ADRs and add a dated note with the new commit sha instead of silently rewriting.

## Owner decisions already made (2026-11-26)

- Deployed on **the same machine as BCUK-Bot-4**, as separate services.
- **Node/TypeScript throughout**, supervisor included, as its own process and unit.
- First Minecraft target: **Java Vanilla**; Paper, other distributions and Bedrock are deferred (#47).
- Hytale server software is said to be available now; **verify it** (#15), don't assume.
- Bot-side integration issues (#65 to #68) **stay in this repo**; mirror them into BCUK-Bot-4 only when the owner asks.

These set direction; the ADRs still verify and record them.

## Rules that must not be broken

From the spec; the issues repeat them where they apply.

- The process supervisor is **independent** of BCUK-Bot-4 and the manager web/API. Game servers survive restarts of both. Never trust a saved PID alone: verify executable identity, ownership and start time before any signal.
- **Zero players is the sole inactivity condition**, and only from a valid, fresh player count. Unavailable or stale counts suspend the countdown and are never zero. Never use CPU, memory, network or console activity. Never enable it by inference on import. Graceful shutdown only: no silent force-kill.
- No arbitrary shell execution; only registered, validated launch definitions. Validate and canonicalise paths; prevent traversal and symlink escape.
- Adapters declare only capabilities they genuinely support; the UI hides or marks the rest. Nothing Java-specific is assumed for Bedrock, and nothing from Minecraft for Hytale.
- The manager owns its own database. World files, large logs and backups stay on the filesystem. It must not depend on the BCUK-Bot-4 database.
- BCUK-Bot-4 never launches or supervises processes and never keeps its own server records. Do not change BCUK-Bot-4 from this repo's work without the owner's say-so.

## Issue conventions

Mirrors BCUK-Bot-4 (see its `issue-management` skill):

- **Severity** is a label (`severity:critical|high|medium|low`) plus a type label (`enhancement`, `documentation`, `bug`). **Priority** (Urgent/High/Medium/Low) and **Effort** (High/Medium/Low) are the org issue fields; every open issue has both.
- Large work is split into `<topic> n/N: <what this part does>` sub-issues linked as real GitHub sub-issues, with a "Depends on" list. A parent's Priority and severity are the highest of its open parts; its Effort covers the whole plan.
- Closing the last open sub-issue completes the parent. A part not needed is closed as "not planned" with a note on the parent.
- Record the commit an issue was checked against; add dated notes on recheck. Mark finished items done, don't delete them.
- Check for existing issues before creating new ones.

Practical notes: the GitHub tools cannot create a label that does not exist: create `severity:*` labels in the repo's label settings first (they currently have no colour or description). Setting Priority and Effort needs the `issue_fields` argument on an update, after creation.

## Process

- Docs-only changes may go straight to `main` when the owner has said so; otherwise use a branch and a draft PR.
- The `workflow-guard` and `precommit` hooks belong to BCUK-Bot-4's session setup and may block `git commit` when working here. They do not apply to this repo; if one blocks you, ask the owner instead of working around it.
- Add this repo's own build, lint, test and commit conventions here once #17 (project scaffold) and #11 (stack ADR) define them.

## PR review and merge rules

Mirrors BCUK-Bot-4's rules (decided 2026-11-26). Docs-only PRs follow them too.

- **A clean CodeRabbit review is required before merging.** Every finding is fixed (after checking it against the source, since some are wrong) or explicitly accepted by the owner, and CodeRabbit's **Merge Risk** is resolved or explicitly accepted. A review that only says "addressed" on its threads, or a review of an earlier commit, does not count: it must cover the current head.
- **Qlty's findings are this repo's standards.** Never dismiss or resolve one without fixing it.
- **CI is the gate, not a local run.** A PR is not done until its checks are green on the current head.
- **The reviewer closes its own threads** (CodeRabbit, Qlty or a person), after checking the fix. Reply on the thread saying what changed and the commit; do not resolve it yourself.
- **Nobody merges without the owner's say-so.** Agents never approve or merge.
- CodeRabbit's free OSS review allowance is limited (the next review can be about an hour away). When it is used up, wait for it rather than merging without the review.
- **The agent monitoring the PR requests the review.** CodeRabbit does not always review automatically (drafts, a used-up limit). When a push has no review on the current head, the monitoring agent posts `@coderabbitai review` on the PR once the allowance has reset, without waiting to be asked, and says so. Pushes that need a fresh review are batched, since each one needs its own review.
- Add further rules here as they come up, with the date they were decided.
