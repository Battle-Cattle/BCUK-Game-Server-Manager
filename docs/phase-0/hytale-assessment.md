# Phase 0: Hytale server assessment

Output of [#15](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/15) (part 7 of 8 of #1). Documentation only: nothing was run, installed or changed.

- **Checked against:** `main` @ `bc960149e62a87721a55462e03ab75df6d3c2c51` (2026-11-28).
- **Source:** the official *Hytale Server Manual* (Hypixel Studios support site, article 45326769420827), saved by the repo owner as an HTML file and read in full. The article showed "Updated 1 month ago" at capture; the capture date was not recorded, so re-read the live page before relying on version-specific details. Hytale is **Early Access** and the manual says several features are still to come.
- **Not read:** the *Server Provider Authentication Guide* (linked from the manual), the downloader's `QUICKSTART.md` (inside its zip), and the full `--help` output (the manual truncates it with `[...]`). Items that depend on them are marked unconfirmed.
- **Not tested:** no Hytale server was run. Everything below is what the documentation says, not observed behaviour.
- **Status:** input to ADR 0005 (#16) and to Phase 4 (#5). Not a decision.

Legend: **Confirmed** = stated in the official manual. **Unconfirmed** = not stated, or only from a third party, so it must be tested before an adapter relies on it. **Unsupported** = the manual says it does not exist yet.

## 1. Summary

| Question | Finding | Status |
|---|---|---|
| Is a dedicated server available? | Yes. Three ways to get the files; the Hytale Downloader CLI is recommended for production | Confirmed |
| Unattended install allowed? | The downloader and bootstrap both need an interactive OAuth2 **device login** by a person with a Hytale account. Automatic/large-scale authentication is covered by a separate *Server Provider Authentication Guide* that has not been read | Unconfirmed |
| Licence limit | 500 servers per Hytale game licence | Confirmed |
| Runtime | Java 25, 4 GB RAM minimum, x64 or arm64 | Confirmed |
| Network | QUIC over **UDP**, default port 5520, set with `--bind` | Confirmed |
| Graceful stop | The manual gives **no stop command or signal behaviour**. Only the update path (exit code 8) is described | Unconfirmed |
| Console | The server has a console that takes slash commands (`/auth`, `/update`). Whether stdin is scriptable is not stated | Unconfirmed |
| Player count | **No built-in count in the server's own interface is documented.** An official partner plugin, `Nitrado:Query`, "exposes server status (player counts, etc.) via HTTP". Its endpoint, freshness and requirements are not documented in the manual | Unconfirmed |
| Official query/telemetry API | Planned ("Server Telemetry", first-party endpoints). Discovery counts come from client telemetry, not the server | Unsupported (planned) |
| Backups | Built in: `--backup`, `--backup-dir`, `--backup-frequency` (default 30 min in the launcher). Consistency guarantees are not stated | Confirmed / unconfirmed |
| Updates | Built in: `/update` commands, staging directory, exit code 8, wrapper script | Confirmed |
| Mods/plugins | Drop `.zip` or `.jar` into `mods/`. Java plugin API; the server jar is published to a Maven repository | Confirmed |

## 2. Obtaining, updating and versioning

Three options (manual, "Server Files"):

| Option | Manual's advice | Notes for the manager |
|---|---|---|
| Copy from a launcher install | "Quick testing. Annoying to keep updated" | Needs the game installed. Not suitable for managed installs, but it is a valid **import source** |
| Hytale Downloader CLI | "Production servers. Easy to keep updated" | OAuth2 authentication. Flags: none (download latest), `-print-version`, `-version`, `-check-update`, `-download-path <file>`, `-patchline pre-release`, `-skip-update-check`. Sets up the layout the updater needs |
| Bootstrap from `HytaleServer.jar` | "Installing from just HytaleServer.jar with no separate downloader" | `java -jar HytaleServer.jar --bootstrap`, then `/auth login device`, then `/update setup` (or `/update download`). Extracts the payload and shuts down. Carries `auth.enc`, `auth.key` and `config.json` into `Server/` |

- **Versioning.** `hytale-downloader -print-version` shows the game version without downloading; `/update status` shows the running version; `-patchline pre-release` and `/update patchline [name]` select a channel. Maven publishes the last five releases, with versions such as `2026.01.22-6f8bdbdc4` (`maven.hytale.com/release` and `/pre-release`). *Confirmed.*
- **Updating.** The server checks hourly by default, downloads to `updater/staging/`, then, when applying, exits with **code 8**. The wrapper script (`start.sh`/`start.bat`) sees code 8, replaces only server binaries and assets, keeps config, saves and mods, and starts the new version. The server "does not restart itself". Updates can be disabled with the `HYTALE_DISABLE_UPDATES` environment variable. *Confirmed.*
- **Required layout.** The updater is disabled unless the server runs from `Server/` with `Assets.zip` and the launcher script in the parent:

  ```text
  game/
  ├── Assets.zip
  ├── start.sh | start.bat
  ├── jvm.options        (optional, one JVM argument per line)
  └── Server/
      ├── HytaleServer.jar
      ├── HytaleServer.aot
      ├── config.json
      └── ...
  ```

- **Authentication.** `/auth login device` prints a code and URL (`accounts.hytale.com/device`), valid for 900 seconds, and the server then reports `Mode: OAUTH_DEVICE`. Credentials are stored in `auth.enc` and `auth.key`. `--auth-mode <authenticated|offline>` exists, default `AUTHENTICATED`. The manual says authenticated servers are needed to talk to Hytale's service APIs and "to counter abuse". *Confirmed.* What `offline` allows, and whether it is permitted for hosting, is **unconfirmed**.
- **Terms.** Whether unattended installation by a third-party tool is permitted is not stated in the manual. The *Server Provider Authentication Guide* is the place to check. **Unconfirmed; owner to read it** (see section 11).

## 3. Runtime, launch, ports and shutdown

- **Requirements.** Java 25 (Adoptium Temurin recommended), at least 4 GB memory, x64 or arm64. CPU scales with players and entities; RAM with view distance and explored area. View distance is the main RAM driver; the manual recommends at most 12 chunks (384 blocks). *Confirmed.*
- **Launch.** Direct: `java -XX:AOTCache=HytaleServer.aot -jar HytaleServer.jar --assets Assets.zip`. Through the launcher script the defaults are `--assets ../Assets.zip --backup --backup-dir backups --backup-frequency 30`, run from inside `Server/`, with extra JVM options read from `../jvm.options`. *Confirmed.*
- **Flags seen** (the list is truncated in the manual): `--accept-early-plugins`, `--allow-op`, `--assets <Path>`, `--auth-mode`, `-b/--bind <InetSocketAddress>` (default `0.0.0.0:5520`), `--backup`, `--backup-dir <Path>`, `--backup-frequency <Integer>` (minutes, default 30), `--disable-sentry`, `--bootstrap`. Full list needs `java -jar HytaleServer.jar --help`. *Partly confirmed.*
- **Network.** QUIC over UDP only; TCP is not needed. Default port 5520. Symmetric NAT may cause trouble. *Confirmed.* A manager running several instances needs one UDP port per instance, set with `--bind`.
- **Exit codes.** `8` means "restart for update"; the launcher loop restarts only on 8 and otherwise exits. Other codes are not described, and a normal-shutdown code is not stated. *Confirmed for 8; unconfirmed for the rest.*
- **Graceful shutdown.** **Not documented.** There is no `/stop` command, signal handling or timeout described. Needed before the adapter may declare a graceful stop (spec rule: no silent force-kill). *Unconfirmed.*
- **Telemetry.** The server reports crashes to Sentry unless `--disable-sentry` is passed. Whether the manager should disable that is an owner decision (privacy and noise), not a technical one.

## 4. Console and administration

- The console accepts slash commands, seen in the manual: `/auth login device`, `/update check|download|apply --confirm|cancel|status|patchline|setup`. A `--allow-op` flag exists and `permissions.json` holds the permission configuration. *Confirmed.*
- **Not stated:** how input reaches the console when the server is run headless (stdin, a socket, or a plugin), whether output is machine-readable, and any command to list players or stop the server. The manual does not mention RCON.
- **Third-party option (not verified):** a community plugin, HyRCON (`curseforge.com/hytale/mods/hyrcon`, MIT, source `dustinrouillard/hyrcon`), says it runs a TCP RCON server inside Hytale and forwards commands to the console. Its listing says it defaults to `0.0.0.0:25575` with the password `changeme` and states no encryption, rate limiting or command allow-list. Source code not reviewed, behaviour not tested, and only the "Early Access" game version is listed. **Unconfirmed; do not depend on it in the adapter until tested** (section 11).

## 5. Configuration, worlds and backup

- **File structure** (`Server/` working directory): `.cache/`, `logs/`, `mods/`, `universe/` (world and player data), `bans.json`, `config.json`, `permissions.json`, `whitelist.json`. Worlds live in `universe/worlds/<name>/` each with a `config.json` (seed, world generation, PvP and ticking flags, `IsSavingPlayers`, `IsSavingChunks`, `RequiredPlugins`, and so on). *Confirmed.*
- **Editing while running.** Configuration is read at startup and written when in-game actions occur; "manual changes while the server is running are likely to be overwritten". So the manager must not edit these files while the server runs. Edits go through console commands, or happen while stopped. *Confirmed.*
- **Backups.** `--backup` enables periodic backups; `--backup-dir` sets the location; `--backup-frequency` is in minutes. `Update.RunBackupBeforeUpdate` (default true) runs a full backup before an update and needs a configured backup directory; `BackupConfigBeforeUpdate` backs up `config.json`, `permissions.json`, `bans.json` and `whitelist.json`. *Confirmed.*
- **Not stated:** whether a backup is a consistent snapshot while the world is saving, any "save-off"/"save-all" equivalent, the backup file format, and how to restore. A safe manager backup of a running server is therefore **unconfirmed**; the conservative path is to back up only while the server is stopped, or to rely on the server's own backup feature once its guarantees are tested.
- **Secrets on disk.** `Server/auth.enc` and `Server/auth.key` are server credentials. They must never be shown in the UI or API, must be excluded from any downloadable file listing, and need an explicit decision on whether they belong in backups.

## 6. Player count and player list (decides inactivity shutdown)

| Source | What the manual says | Usable by the manager? |
|---|---|---|
| Server console command | None documented | Unconfirmed |
| Query port or protocol | None documented. A third-party service (`hytlapi.com`) says it is on hold "until Hytale add a public query port" | Not available as far as known |
| `Nitrado:Query` plugin | Official partner plugin: "Exposes server status (player counts, etc.) via HTTP". Needs the `Nitrado:WebServer` base plugin. Endpoint, format, authentication and freshness not documented | **Most promising.** Test before relying on it |
| `ApexHosting:PrometheusExporter` | "Exposes detailed server and JVM metrics". Whether a player gauge is included is not stated | Possible; test |
| First-party API ("Server Telemetry") | Planned: report status and player count | Unsupported today |
| Server discovery | Counts come from **client** telemetry; servers may still report an unverified count to users who add the server directly | Not for our use |

**Consequence for the adapter contract (#16) and inactivity (#61).** The spec makes zero players the only inactivity condition, and only from a valid, fresh count. Nothing in the manual provides that without a plugin. Until a source is tested, the Hytale adapter declares **no player count** and therefore no inactivity shutdown. If `Nitrado:Query` (or another source) is verified, the adapter may declare it, with these rules: the count is valid only if it was fetched successfully within the freshness window; plugin missing, HTTP error, timeout or an unparseable response is "unavailable", never zero; and the manager never installs the plugin by inference on import.

The server's own `AutoApplyMode: WhenEmpty` shows it can tell when no players are online, but that setting applies updates, not shutdowns, and the manager must not use it as an inactivity signal.

## 7. Mods and plugins

- **Install.** Put `.zip` or `.jar` files in `mods/`. `--accept-early-plugins` acknowledges that "early plugins" are unsupported and may cause instability. Worlds can list `RequiredPlugins`. *Confirmed.*
- **Plugin API.** Java, targeting the `HytaleServer` jar, which is published to Maven (`com.hypixel.hytale:Server`). Plugin classes seen include `PlayerRef` and `PlayerSetupConnectEvent`. The manual does not document a list-online-players call. *Confirmed (existence); detail unconfirmed.*
- **Recommended plugins** (maintained by Nitrado and Apex Hosting): `Nitrado:WebServer`, `Nitrado:Query`, `Nitrado:PerformanceSaver`, `ApexHosting:PrometheusExporter`.
- **Own plugin.** A small first-party plugin that tracks join/leave and publishes a timestamped count is possible but is new scope in a second language. Only worth it if the official/partner options cannot meet the freshness and validity rules. Plugin/mod management in the manager is later work (spec: "where supported").

## 8. Detecting an existing installation

Candidates for the import detector (#50), all from the documented layout. None is tested.

| Marker | Meaning |
|---|---|
| `Server/HytaleServer.jar` | Server binary (version via `/update status` or the downloader; reading it from the jar is unconfirmed) |
| `Assets.zip` in the parent directory | Required assets |
| `start.sh` / `start.bat` | Launcher scripts; their absence disables the updater |
| `Server/config.json`, `Server/universe/` | Server configuration and world data |
| `Server/auth.enc`, `Server/auth.key` | The server is authenticated. Do **not** read these |
| `Server/mods/` | Installed mods; the import lists them, never changes them |
| `jvm.options` | Custom JVM arguments to carry into the launch definition |

Import must not enable inactivity shutdown, install a plugin, or modify files. It records what it found and what is missing.

## 9. Findings that affect the plan

1. **Account link blocks fully unattended install.** A person must complete a device login for each server (or use the provider guide). The install workflow (#49) needs a "waiting for authorisation" state with the code and URL shown to the operator and a 900-second expiry, not a silent background task. Reading the *Server Provider Authentication Guide* is a prerequisite for deciding if anything more automatic is allowed.
2. **The built-in updater and the supervisor overlap.** `start.sh` loops on exit code 8 and spawns Java as a child. The supervisor must therefore either run the launcher script and track the Java child (process identity must verify the actual Java process, not the shell), or run Java directly and implement the update step itself. This is a design decision for ADR 0002 (#12) and the adapter contract. The manager should also own the update policy: set `Update.AutoApplyMode` to `Disabled` (the manual's default) so the server never restarts itself under the supervisor's feet, or accept exit code 8 as an expected restart.
3. **No documented graceful stop.** The adapter cannot declare graceful stop until a stop command or signal is verified. Without it, the only options are a verified console command or a confirmed forced stop, and the spec forbids silent force-kill.
4. **No console input path is documented for headless runs.** The supervisor may need to own stdin of the server process, or a plugin (RCON/HTTP) must supply the channel. This feeds directly into ADR 0002 and #33.
5. **Player count needs a plugin.** Declare it only after `Nitrado:Query` (or equivalent) is tested. Inactivity shutdown for Hytale depends on it.
6. **Config and world files are rewritten by the server.** No manager edits while running; backups of a running server are unproven.
7. **Protocol is exact-match.** Clients and servers must match, so a server update is time-critical when the game updates. This matters for the update scheduler (#59/#60) and for the "update available" display. A "protocol tolerance" change is announced but not available.
8. **Early Access.** Several features the manager would want (first-party query API, fallback servers, tolerance) are "coming soon". Capabilities must be re-verified when the game moves out of Early Access.

## 10. Fit with the planned host

From the host assessment ([host-assessment.md](host-assessment.md), #10; read there, not re-probed here):

| Hytale need | Host (as assessed) | Consequence |
|---|---|---|
| Java 25 | One JDK installed, OpenJDK 25.0.5.1 | Meets the stated runtime. Whether the installed build is acceptable for the server (the manual recommends Adoptium Temurin) is unconfirmed |
| x64 or arm64, 4 GB minimum RAM | x86_64, 11 GiB RAM, **no swap**, shared with the bot, MySQL and Caddy | One Hytale server fits; two or three alongside Minecraft may not. Per-server `MemoryMax` and OOM protection for MySQL and the bot are needed (ADR 0002). Hytale's view distance is its main RAM driver, so the launch definition should expose it and `-Xmx` |
| UDP port per instance (default 5520) | No port conflicts; `ufw` inactive and iptables ACCEPT, provider firewall unknown | Opening a UDP game port may be a provider-side action. The owner needs to say whether a provider firewall exists |
| Process survives manager restarts | Children of a systemd service stop with it unless the unit is set up otherwise, and pipes to the console are lost when the supervisor exits | Reinforces finding 4: console input must come from a persistent endpoint (FIFO, socket, broker or multiplexer) or from a plugin channel (ADR 0002) |
| Disk for `Assets.zip`, worlds and backups | One 100 GB disk, 63 GB free | The launcher's `Assets.zip` was about 3.3 GB in the manual's example listing, per install. Per-instance quotas matter |

## 11. Open items and what would unblock them

| Open item | What would unblock it | Needed by |
|---|---|---|
| Unattended install and auth terms | Owner reads the *Server Provider Authentication Guide* and the downloader's `QUICKSTART.md` and records the answer here | #49 (install), ADR 0005 |
| Graceful stop command or signal | Run a real server, try `/stop` or SIGTERM, record behaviour and exit code | #51, ADR 0005 |
| Headless console input | Test stdin piping of a running server, or HyRCON/plugin alternatives | #51, #33, ADR 0002 |
| Player-count source | Install `Nitrado:WebServer` and `Nitrado:Query`, record endpoint, response, freshness and failure modes. Optionally check `PrometheusExporter` | #52, #61, ADR 0005 |
| Full `--help` | Run `java -jar HytaleServer.jar --help` and paste the output | #48 (launch definition) |
| Backup safety | Check whether `--backup` is consistent while running; record the format and a restore | #53, #56 |
| Exit codes other than 8 | Observe a normal shutdown and a crash | #51, #25 |
| HyRCON suitability | Review the source and test behaviour and security | Optional, only if the console path needs it |
| Sentry | Owner decides whether managed servers pass `--disable-sentry` | #48 |

## 12. Effect on the Phase 4 issues (#5)

Provisional until the ADRs land; recheck each against them and add a dated note.

| Issue | Effect |
|---|---|
| #48 adapter skeleton and launch definition | Launch definition is `start.sh`/`java ... --assets ... --bind`; per-instance UDP port and directory; `jvm.options`. Depends on the launcher-vs-direct decision (finding 2) |
| #49 installation and versioning | Downloader or bootstrap, with an operator-visible device-login step; version from `-print-version` / `/update status`; patchline selection |
| #50 import and detection | Use the markers in section 8; never read the credential files |
| #51 lifecycle, console, startup detection | Blocked on graceful stop and console input; startup detection source not documented (log line unconfirmed) |
| #52 config, worlds, players | Config only while stopped; world list from `universe/worlds/`; player count only if a source is verified |
| #53 backup preparation | Layout is documented; consistency unproven |
| #54 multi-instance tests | Per-instance UDP ports and directories; real-server tests need a licence |
