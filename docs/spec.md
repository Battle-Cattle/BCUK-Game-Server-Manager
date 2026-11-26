# BCUK Game Server Manager
## Multi-Game Server Management Platform

**Status:** Proposed  
**Initial games:** Minecraft and Hytale  
**Minecraft edition:** Undecided, architecture must support Java and Bedrock  
**Hosting:** New installations and existing server imports  
**Access:** Standalone web interface and BCUK-Bot-4 integration  
**Process management:** Independent process supervisor  
**Inactive server detection:** Automatic shutdown after a configurable period with zero players

---

## 1. Project objective

Create a standalone, multi-game server management platform that can be operated through its own web interface and integrated into the existing BCUK-Bot-4 dashboard.

The initial supported games will be Minecraft and Hytale.

The platform must support multiple independent server instances per game and provide a foundation for adding further games without redesigning the core system.

Users must be able to:
- Install new game servers through the manager.
- Import existing server installations.
- Start, stop and restart servers.
- Monitor server status and resource usage.
- Access live consoles and logs.
- Manage game-specific functionality.
- Configure backups and updates.
- Automatically stop inactive servers based on player count.

The manager must operate independently of BCUK-Bot-4. Bot or dashboard restarts must not interrupt running game servers.

## 2. Project separation

The solution will consist of two independent projects.

### 2.1 BCUK-Bot-4

The existing community bot and dashboard.

Responsibilities:
- Provide a Game Servers navigation section.
- Display server status and monitoring information.
- Provide common lifecycle controls.
- Provide access to server consoles.
- Integrate with the manager API.
- Authenticate users and pass authorised requests.
- Link to the standalone management interface.

BCUK-Bot must not directly launch, own or supervise game server processes.

### 2.2 BCUK-Game-Server-Manager

A separate repository and independently deployed service.

Suggested repository:

`Battle-Cattle/BCUK-Game-Server-Manager`

Responsibilities:
- Standalone web interface.
- Management API.
- Independent persistence and configuration.
- Server registration and installation.
- Game adapters.
- Independent process supervisor.
- Console and log management.
- Monitoring.
- Backups and restoration.
- Updates and scheduling.
- Inactive-server detection and automatic shutdown.

The manager must remain fully functional when BCUK-Bot is unavailable.

### 2.3 Deployment arrangement

Suggested hostnames:

- `bcuk.co.uk`: Existing BCUK-Bot dashboard.
- `servers.bcuk.co.uk`: Standalone game server manager.

These are proposed hostnames, subject to deployment configuration.

Both interfaces must communicate with the same manager API and server records. They must not maintain separate or conflicting server state.

### 2.4 Logical architecture

```text
                  Users
                    |
          +---------+---------+
          |                   |
      BCUK-Bot-4       servers.bcuk.co.uk
      Existing UI       Manager UI
          |                   |
          +---------+---------+
                    |
             Authenticated API
                    |
          BCUK Game Server Manager
                    |
             Server Registry
                    |
            Process Supervisor
                    |
              Game Adapters
                /       \
           Minecraft   Hytale
                \       /
             Server Instances
```

The manager's web interface and API must be separate from the process supervisor's lifecycle. Restarting the web application must not stop game servers.

## 3. Architectural principles

1. Multi-game support from the beginning.
2. Multiple independent instances of the same game.
3. Game-specific adapters behind a shared contract.
4. Shared functionality implemented once.
5. Capability-driven interface.
6. New installation and existing installation import support.
7. Game servers independent of BCUK-Bot.
8. Process supervisor independent of the manager web interface.
9. Manager owns its own database and configuration.
10. Secure authentication integration with BCUK.
11. Restricted process and filesystem access.
12. Future games must be addable without rewriting the core.

## 4. Repository assessment

Before implementation, inspect the existing BCUK-Bot-4 repository and deployment environment.

Assess:
- Backend and frontend architecture.
- Existing API and WebSocket conventions.
- Authentication and authorisation.
- Existing user and role model.
- Database and persistence conventions.
- Logging and error handling.
- Reverse proxy and deployment setup.
- Background tasks and scheduling.
- Server operating environment.
- Available process supervision mechanisms.
- Filesystem permissions and storage layout.

Produce a technical assessment and integration plan before major implementation work.

Avoid introducing unnecessary dependencies or duplicating existing functionality.

## 5. Independent process supervisor

This is a mandatory architectural component.

The supervisor must run independently of BCUK-Bot-4 and the manager web interface.

### Responsibilities

- Start and stop game server processes.
- Perform graceful shutdown.
- Support forced termination as a separate recovery operation.
- Verify process identity and ownership.
- Monitor process health and exit codes.
- Detect crashes and unexpected exits.
- Handle startup and shutdown timeouts.
- Prevent duplicate launches.
- Recover and reconcile process state after restarts.
- Handle configured machine-reboot startup policies.
- Report process state to the manager API.
- Continue scheduled inactivity checks when the web interface is unavailable.

### Expected behaviour

| Event | Expected result |
|---|---|
| BCUK-Bot restarts | Servers continue running |
| BCUK-Bot crashes | Servers continue running |
| Manager web/API restarts | Servers continue running |
| Supervisor restarts | Existing processes are rediscovered |
| Game server crashes | Supervisor detects and records the failure |
| Machine reboots | Servers follow configured startup policies |
| Manager reconnects | Current server state is reconciled |

### Persistent process state

Maintain sufficient state to recover management after a restart:

- Server instance ID.
- Process ID.
- Process start time.
- Executable identity.
- Working directory.
- Launch configuration identity.
- Supervisor ownership metadata.
- Desired state.
- Last known lifecycle state.

Never trust a saved PID alone. Verify executable identity, process ownership and other identifying information before signalling or terminating a process. Protect against PID reuse.

Use appropriate operating-system process or service facilities where practical.

### Communication

The supervisor should communicate through a restricted local interface, such as a Unix domain socket or authenticated loopback API.

- Never expose process-management endpoints publicly.
- Authenticate and authorise supervisor requests.
- Validate server IDs and requested operations.
- Only execute registered and validated launch definitions.
- Do not provide arbitrary shell execution.
- Audit sensitive lifecycle actions.

## 6. Server instance model

Each server instance should have:

- Unique ID.
- Display name.
- Game identifier.
- Edition or variant.
- Server implementation or distribution.
- Installation directory.
- Data directory where separate.
- Validated launch configuration.
- Working directory.
- Runtime requirements.
- Memory and resource limits where supported.
- Network and port information.
- Current lifecycle state.
- Desired lifecycle state.
- Installation source, either manager-installed or imported.
- Game-specific configuration metadata.
- Creation and modification timestamps.

Support optional game-specific metadata without forcing all games into the same configuration model.

Use versioned configuration schemas and migrations.

## 7. Installation and importing

### New installations

Users should be able to:

1. Select a game.
2. Select an available edition or server implementation.
3. Select a supported version.
4. Choose an installation directory.
5. Configure initial settings.
6. Review the installation.
7. Start installation.
8. Monitor progress and logs.
9. Complete setup and launch the server.

Validate runtime requirements, permissions, disk space, destination paths and detectable port conflicts.

Installation definitions must belong to the relevant game adapter.

### Existing installations

Users must be able to import installations created outside the manager.

The workflow should:

1. Select a game or request detection.
2. Select the installation directory.
3. Validate the directory.
4. Detect game, edition, version and implementation where possible.
5. Detect configuration and runtime information.
6. Request missing required information.
7. Preview the detected installation.
8. Register it without overwriting existing data.

Importing must not automatically update, move, delete or rewrite server files.

If detection is incomplete, explain what could not be identified rather than making assumptions.

### Filesystem safety

- Restrict installations to configured permitted directories.
- Validate and canonicalise paths.
- Prevent path traversal and symlink escape.
- Avoid unrestricted filesystem browsing.
- Never overwrite an existing installation without confirmation.
- Keep installation logs.
- Avoid unrestricted installer privileges.

## 8. Game adapter architecture

Each game integration must implement a shared contract and declare its capabilities.

Illustrative interface:

```ts
interface GameAdapter {
  id: string;
  name: string;

  getCapabilities(): GameCapabilities;

  validateConfig(config: unknown): ValidationResult;

  detectInstallation(path: string): Promise<DetectionResult>;

  install(
    server: ServerInstance,
    options: InstallOptions
  ): Promise<InstallResult>;

  getStatus(server: ServerInstance): Promise<ServerStatus>;

  getPlayerCount(server: ServerInstance): Promise<PlayerCountResult>;

  getConsole(server: ServerInstance): Promise<ConsoleOutput>;

  sendCommand(
    server: ServerInstance,
    command: string
  ): Promise<void>;

  prepareBackup(
    server: ServerInstance
  ): Promise<BackupPreparationResult>;

  update(server: ServerInstance): Promise<UpdateResult>;
}
```

This is illustrative. The final contract must be based on actual game requirements.

The process supervisor owns process lifecycle. Game adapters provide game-specific launch definitions and communication methods.

### Capabilities

Possible capabilities include:

- Console.
- Remote console protocol.
- Player count and player list.
- Player management.
- World management.
- Configuration management.
- Plugin management.
- Mod management.
- Version detection.
- Automated updates.
- Safe backup preparation.

Only declare functionality that the adapter genuinely supports.

The UI must hide or clearly mark unsupported functionality. Missing player-count support must never be interpreted as zero players.

## 9. Minecraft integration

Minecraft must be supported from the initial release.

The architecture must remain open to Java and Bedrock Edition. The initial edition and implementation scope will be decided during technical assessment.

### Java Edition

Potential implementations, subject to prioritisation:

- Vanilla.
- Paper.
- Fabric.
- Forge.
- NeoForge.

Potential functionality:

- Installation and version management.
- Existing installation detection.
- Server properties.
- World management.
- Player list and player count.
- Whitelist.
- Bans.
- Operator permissions.
- RCON where supported.
- Plugin management for compatible implementations.
- Mod management for compatible implementations.
- Version detection and updates.

Do not commit to supporting every distribution in the first release.

### Bedrock Edition

Keep the architecture open to:

- Bedrock-specific installation.
- Bedrock configuration.
- World management.
- Player information and management where supported.
- Console and administration mechanisms.
- Version detection and updates.

Do not assume Java configuration, RCON or modding behaviour applies to Bedrock.

Treat Java and Bedrock as distinct variants with their own configuration and adapter behaviour.

## 10. Hytale integration

Hytale is the second initial game.

Implement based on verified server capabilities and supported installation mechanisms.

Potential functionality:

- New installation.
- Existing installation import.
- Version management.
- Lifecycle management.
- Console and logs.
- Configuration.
- World management where supported.
- Player information where supported.
- Mod or plugin management where supported.
- Safe backup preparation.

Do not assume Hytale uses Minecraft's protocols or configuration formats.

## 11. Shared server functionality

### Dashboard

Display:
- Server name.
- Game and edition.
- Current status.
- Uptime.
- CPU and memory usage where available.
- Player count where supported.
- Process health.
- Warnings and recent errors.

Provide filters and sorting by game and status.

### Lifecycle management

Support:
- Start.
- Graceful stop.
- Restart.
- Forced termination as a separately confirmed recovery action.

Track actual process state. A successful start request is not proof that the server started successfully.

Prevent duplicate launches and detect startup timeouts.

### Console

Provide:
- Live server output.
- Command input.
- Timestamps.
- Reconnection handling.
- Bounded output and history.
- Server isolation.
- Persistent logs independent of browser sessions.
- Permission checks for command input.

Use the communication method appropriate to each game.

### Logs

Provide access to:
- Server output.
- Historical logs.
- Installation logs.
- Update logs.
- Backup logs.
- Process and supervisor events.

Implement retention and size limits.

### Resource monitoring

Monitor where available:
- CPU.
- Memory.
- Process uptime.
- Disk usage.
- Network activity.

Clearly distinguish unavailable information from zero values.

### Backups

Provide:
- Manual backups.
- Scheduled backups.
- Retention policies.
- Backup history and metadata.
- Restore operations.
- Validation and progress reporting.

Use game-specific preparation to avoid inconsistent world data. Do not assume copying live files is safe.

Store backups separately from live installations. Restore must require confirmation and protect existing data.

### Updates

Provide:
- Current version detection.
- Available update detection where supported.
- Manual update initiation.
- Update history.
- Optional pre-update backup.
- Progress and error reporting.

Do not automatically update production servers unless explicitly configured.

### Scheduling

Support:
- Scheduled backups.
- Scheduled restarts.
- Scheduled updates where enabled.
- Maintenance windows.
- Inactivity checks.

Scheduling must run within the manager environment and must not depend on BCUK-Bot being online.

### Activity history

Record:
- Server creation and import.
- Start, stop and restart.
- Crashes and unexpected exits.
- Configuration changes.
- Backup and restore.
- Updates.
- Inactivity-triggered shutdowns.
- Installation failures.

Include timestamps and the responsible user or system process.

## 12. Inactive-server detection and automatic shutdown

The manager must automatically stop inactive servers to reduce unnecessary resource usage.

### Inactivity definition

A server is considered inactive when its reported player count is exactly zero.

**Zero players is the sole inactivity condition.**

Do not use CPU usage, memory usage, network activity or lack of console activity as alternative inactivity signals.

### Behaviour

1. Monitor player count for servers whose adapters support reliable player-count reporting.
2. When the player count reaches zero, start the configured inactivity countdown.
3. If a player joins before the countdown expires, cancel or reset the pending shutdown.
4. If the player count remains zero for the full configured period, request a graceful shutdown.
5. Record the shutdown as inactivity-triggered.
6. If graceful shutdown fails, report the failure and follow the separately configured recovery policy.

Do not silently force-kill a server because it failed to shut down gracefully.

### Configuration

Provide per-server settings:

- Enable or disable automatic inactive shutdown.
- Configure the zero-player waiting period.
- View whether a shutdown countdown is pending.
- View the last inactivity-triggered shutdown.

The waiting period must be configurable. Do not hard-code an assumed timeout as a universal requirement.

### Safety and reliability

- Inactivity checks run within the manager/supervisor environment, not BCUK-Bot.
- The feature continues working if BCUK-Bot is offline.
- Persist enough state to handle manager restarts safely.
- Require a valid, fresh player-count result.
- If player-count monitoring becomes unavailable or stale, suspend the countdown. Do not stop the server on that basis.
- If the adapter cannot reliably report player count, disable the feature for that server and explain why.
- Recheck player count immediately before shutdown.
- A player joining during the shutdown sequence should cancel it if safely possible.
- Record automatic shutdowns and failures.

Do not enable this feature by inference when importing a server. Its setting and timeout must be visible to the user.

## 13. Standalone manager interface

Provide a complete web interface at a dedicated subdomain, for example:

`servers.bcuk.co.uk`

The interface should support:

- Server overview.
- Adding and importing servers.
- Individual server management.
- Console and logs.
- Players and worlds where supported.
- Mods and plugins where supported.
- Configuration.
- Backups and restoration.
- Updates.
- Scheduling.
- Inactivity settings.
- Activity history.
- Manager and supervisor health.

The standalone manager must remain usable when BCUK-Bot-4 is offline, subject to its configured authentication provider being available.

## 14. BCUK-Bot integration

Add a Game Servers section to the existing BCUK-Bot navigation.

Provide:

- Server overview and status.
- Start, stop and restart controls.
- Player counts and resource summaries.
- Console access where appropriate.
- Links to full management pages.
- Clear manager connection status.

BCUK-Bot must communicate through the manager API. It must not duplicate server records or directly manipulate processes.

## 15. Authentication and permissions

Use a shared identity experience where practical.

Preferred approach:

- BCUK remains the primary identity provider.
- The manager trusts a secure, documented authentication integration.
- Direct visits to the manager subdomain use a secure login or SSO flow.
- The manager validates permissions independently on every API request.
- Server-specific permissions determine who can view, start, stop, configure or administer each instance.
- The manager retains a secure administrator recovery method if BCUK-Bot is unavailable.

Do not trust browser-supplied user IDs or roles.

Do not expose long-lived API credentials in frontend code.

Use HTTPS, secure session handling, CSRF protection where applicable, authenticated WebSockets and audit logging.

## 16. API requirements

Provide a documented, versioned API for the manager interface and BCUK-Bot.

Illustrative routes:

```text
GET    /api/v1/servers
POST   /api/v1/servers
GET    /api/v1/servers/:id
DELETE /api/v1/servers/:id

POST   /api/v1/servers/:id/start
POST   /api/v1/servers/:id/stop
POST   /api/v1/servers/:id/restart

GET    /api/v1/servers/:id/status
GET    /api/v1/servers/:id/logs
GET    /api/v1/servers/:id/players
GET    /api/v1/servers/:id/capabilities

GET    /api/v1/servers/:id/backups
POST   /api/v1/servers/:id/backups

GET    /api/v1/supervisor/status
```

These routes are illustrative and should follow the chosen API conventions.

Provide authenticated real-time updates through WebSockets or another suitable streaming mechanism.

Unsupported capabilities must be reported explicitly.

## 17. Persistence and storage

The manager owns its own database and configuration. It must not depend on the BCUK-Bot database.

Persist:
- Server instances.
- Game and adapter identifiers.
- Edition and implementation metadata.
- Installation and data paths.
- Runtime configuration.
- Desired and observed state.
- Supervisor and process metadata.
- Monitoring preferences.
- Backup policies and history.
- Scheduling settings.
- Inactivity settings and countdown state.
- Update preferences and history.
- Activity and audit history.

Do not store world files, large logs or backup archives in the application database.

Use migrations and versioned schemas.

## 18. Security

- Restrict manager and supervisor privileges.
- Enforce server-side authorisation.
- Restrict supervisor communication to a local authenticated interface.
- Validate filesystem paths.
- Prevent path traversal and symlink escape.
- Construct process arguments safely.
- Do not provide arbitrary shell execution.
- Never expose environment secrets in logs or API responses.
- Protect destructive operations with confirmation.
- Verify process identity before signalling or terminating.
- Isolate server files and console streams.
- Apply resource, log and request limits.
- Audit sensitive actions.
- Protect public access with HTTPS and suitable network controls.

## 19. Suggested project structure

The exact structure should follow the chosen stack and repository conventions.

```text
BCUK-Game-Server-Manager/
  apps/
    web/
    api/
    supervisor/
  packages/
    core/
    adapters/
      minecraft/
      hytale/
    shared-types/
  data/
  docs/
  tests/
```

This is a logical suggestion, not a requirement to introduce unnecessary packages or a monorepo.

The supervisor must be independently deployable and restartable.

## 20. Implementation phases

### Phase 0: Technical assessment

- Inspect BCUK-Bot-4.
- Assess manager deployment environment.
- Choose API and authentication integration.
- Define manager-owned persistence.
- Assess supervisor and operating-system process facilities.
- Assess Minecraft Java and Bedrock requirements.
- Verify Hytale requirements.
- Define adapter contract and capabilities.
- Define player-count reliability requirements.
- Produce architecture decisions and scoped issues.

**Exit criteria:** Architecture and deployment approach agreed.

### Phase 1: Manager foundation and supervisor

- Create standalone manager project.
- Establish configuration and persistence.
- Implement server registry.
- Implement adapter registration.
- Implement independent supervisor.
- Implement process identity and recovery.
- Implement lifecycle operations.
- Implement local authenticated communication.
- Implement service startup and restart behaviour.
- Implement health reporting.

**Exit criteria:** Multiple independent test processes can be managed, survive web/API restarts and be safely rediscovered after supervisor restart.

### Phase 2: Standalone interface and API

- Implement manager web interface.
- Implement versioned API.
- Implement authentication and permissions.
- Implement server overview and details.
- Implement lifecycle controls.
- Implement console and logs.
- Implement real-time status.
- Implement registration and import workflows.

**Exit criteria:** Manager can be operated directly without BCUK-Bot.

### Phase 3: Minecraft integration

- Decide initial edition scope.
- Implement selected server type or types.
- Implement installation.
- Implement import and detection.
- Implement launch and shutdown.
- Implement console.
- Implement supported configuration and capabilities.
- Implement reliable player-count reporting where supported.
- Test multiple instances.

**Exit criteria:** Selected Minecraft implementation can be installed or imported and managed reliably.

### Phase 4: Hytale integration

- Verify current server requirements.
- Implement installation and import.
- Implement lifecycle and console.
- Implement supported configuration and game features.
- Implement reliable player-count reporting if available.
- Test multiple instances.

**Exit criteria:** Hytale can be managed through the same platform.

### Phase 5: Shared advanced functionality

- Backups and restoration.
- Updates.
- Resource monitoring.
- Scheduling.
- Inactive-server detection and automatic shutdown.
- Activity history.
- Notifications.

**Exit criteria:** Shared services work reliably and respect adapter capabilities.

### Phase 6: BCUK-Bot integration and deployment

- Add Game Servers navigation.
- Integrate authentication and SSO.
- Add dashboard summaries and controls.
- Add links or embedded management views.
- Configure manager subdomain and reverse proxy.
- Configure service supervision and machine startup.
- Test independent bot and manager restarts.

**Exit criteria:** Both interfaces operate against the same manager and server data.

### Phase 7: Hardening and documentation

- Security testing.
- Process recovery testing.
- Crash and power-loss scenarios.
- Backup and restore testing.
- Inactivity timer edge cases.
- Permission testing.
- Performance and resource testing.
- Deployment and recovery documentation.
- Adapter development documentation.

**Exit criteria:** Production-ready operational and security documentation with tested recovery procedures.

## 21. Testing requirements

Test:
- Multiple instances of the same game.
- Multiple games.
- Adapter registration and unsupported capabilities.
- New installation and import.
- Invalid paths and symlink escape.
- Process start, graceful stop and restart.
- Forced termination safeguards.
- PID reuse and process identity checks.
- Duplicate launch prevention.
- Bot restart while servers are running.
- Manager web/API restart while servers are running.
- Supervisor restart and process reconciliation.
- Machine reboot policies.
- Console isolation.
- API and WebSocket authorisation.
- Backup and restore.
- Update failures.
- Zero-player inactivity countdown.
- Player joining before timeout.
- Player-count unavailable or stale.
- Player joining during shutdown where cancellation is possible.
- Inactivity policy persistence across restarts.
- Permissions and audit history.

Use mock adapters for core tests, but perform integration tests against real supported server implementations.

## 22. Definition of done

The platform is complete when:

- It exists as a separate project and service.
- It has its own standalone web interface and subdomain.
- BCUK-Bot-4 integrates through an authenticated API.
- It owns its own persistence and configuration.
- The supervisor operates independently of BCUK-Bot and the manager web interface.
- Running servers survive bot and web/API restarts.
- Process state is safely reconciled after supervisor restart.
- Multiple server instances per game are supported.
- New installations and existing imports are supported.
- Minecraft and Hytale have independent adapters.
- Minecraft edition scope has been explicitly decided.
- Shared functionality is reused.
- Unsupported game features are handled honestly.
- Inactive shutdown is based only on a valid zero-player count and configured waiting period.
- Backups, restoration, lifecycle operations and permissions are tested.
- Security requirements are met.
- Deployment, recovery and adapter documentation are available.

## 23. Development agent instructions

Start with Phase 0. Do not immediately begin implementing game-specific integrations.

Inspect the actual BCUK-Bot-4 repository and deployment environment. Confirm what can be reused and what must be built in the new project.

Create the manager as an independent project. Do not put process management inside BCUK-Bot-4.

Treat the independent supervisor as mandatory. Demonstrate that game processes survive BCUK-Bot and manager web/API restarts before proceeding to full game integrations.

Verify real Minecraft and Hytale server capabilities before committing to feature claims. Keep Java and Bedrock open in the architecture until the initial edition scope is agreed.

Implement inactivity shutdown only when the adapter provides a reliable, fresh player count. Zero players is the sole inactivity condition. Do not substitute CPU, network or console inactivity.

Create appropriately scoped issues with dependencies. Check for existing issues and avoid duplicates. Follow the project's existing issue and pull-request standards.

The objective is a secure, maintainable, standalone multi-game server-management platform that integrates naturally with BCUK-Bot-4 while remaining independently operable.
