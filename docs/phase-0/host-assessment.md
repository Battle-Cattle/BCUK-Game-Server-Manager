# Phase 0: deployment host assessment

Output of [#10](https://github.com/Battle-Cattle/BCUK-Game-Server-Manager/issues/10) (part 2 of 8 of #1). Read-only: nothing on the host was changed.

- **Method:** a read-only probe script run by the owner on the live host, as the bot's service user with passwordless sudo, on **2026-11-28 11:19:59 UTC**. Output lines with secret-looking words were redacted and addresses masked before sharing. The script is not in this repo.
- **Bot code checked:** the live bot checkout is at commit `55ff49d` (dated 2026-11-28). Facts about the bot's `deploy.sh`, unit template and installer come from BCUK-Bot-4 `1b5555b`, as in [`bot-assessment.md`](bot-assessment.md). The live checkout's commit is dated two days later than that, so recheck those files against `55ff49d` before relying on them.
- **Status:** input to ADRs #11 (stack) and #12 (supervisor). Not a decision. Where this document suggests something it says so.
- **Public repo:** the panel's domain name and the host's addresses are left out on purpose.

## 1. Summary

| Question | Finding | Consequence for the manager |
|---|---|---|
| Machine | One KVM guest: 6 vCPU, 11 GiB RAM, **no swap**, one 100 GB disk (63 GB free). Ubuntu 26.04.1 LTS, kernel 7.0, x86_64 | Game servers share all of it with the bot, its database and Caddy. Memory limits per game server are needed, and with no swap an out-of-memory kill is the failure mode |
| Init and isolation | systemd 259, cgroup v2 with `cpu`, `memory`, `io`, `pids` controllers, `systemd-run` and `systemd-analyze` present | Per-server limits and a dedicated slice are possible with plain systemd properties |
| Who the bot runs as | The interactive admin user (`ubuntu`: groups `sudo`, `adm`, `lxd`, passwordless sudo), sandboxed by the unit (exposure 1.5 OK) | The manager and supervisor should run as **their own service users without sudo**, not as this user |
| Reverse proxy | Caddy 2.11 as a system service, one site block (the bot's panel), proxying to `127.0.0.1:3000` | `servers.bcuk.co.uk` needs a second site block and a DNS record; nothing else changes |
| Firewall | `ufw` inactive, iptables default ACCEPT, only fail2ban sets in nftables | Host rules restrict nothing inbound. Whether the provider's network firewall does is **unknown** (section 6) |
| Storage for backups | A single volume. Bot backups, the database, uploads and any future game data all sit on it | No separate backup volume. Backups need an off-host copy or provider snapshots (owner to confirm) |
| Reboots | Rebooted at intervals of 2 to 24 days over the last six boots, each after a clean-looking shutdown; units are enabled at boot; journald is persistent | The supervisor must rediscover servers after a reboot and decide what restarts (sections 7 and 9) |

## 2. Operating system, init and isolation (probe sections 1 and 2)

- Ubuntu 26.04.1 LTS, kernel `7.0.0-43-generic`, x86_64, `systemd-detect-virt` = `kvm` (a `qemu-guest-agent` service runs). Up 14 days 17 hours at probe time (boot on 2026-11-13).
- Clock is UTC, NTP synchronised.
- systemd **259**. cgroup v2 (`cgroup2fs`), controllers `cpuset cpu io memory hugetlb pids rdma misc dmem`. So `MemoryMax`, `CPUQuota`, `IOWeight`, `TasksMax` and slices all work; the bot's `RestartSteps` and `RestartMaxDelaySec` (need 254+) are supported.
- `systemd-run`, `loginctl` and `systemd-analyze` are installed. `max_user_namespaces` is 40085.
- **Linger is off** for the `ubuntu` user. A `systemctl --user` manager would stop when its user's last session ends unless `loginctl enable-linger` is set, so user units are not a safe default for something that must survive logouts. System units (or a service user with linger) are.
- Tools present: `tmux`, `screen`, `python3`, `curl`, `unzip`, `git`, Node 24.22.3 (`/usr/local/bin/node`, exactly the bot's `engines` minimum) and **one JDK, OpenJDK 25.0.5.1**. Whether Java 25 suits the Minecraft version chosen, or older versions need other JDKs, is #14's question.

## 3. The bot's service on this host (probe sections 3 and 4)

- Unit `/etc/systemd/system/bcuk-bot.service`, enabled and active. `User=ubuntu`, `WorkingDirectory=/home/ubuntu/bcuk-bot`, `ExecStart=/usr/local/bin/node dist/index.js`, `Restart=on-failure`, `RestartSec=10`, `StartLimitIntervalSec=0`, `TimeoutStopSec=30`. It follows the repo's template with the installer's placeholders filled in; some comment wording differs from `1b5555b`, and the probe cut the unit text at 90 lines, so the last lines are not shown.
- Sandbox: `ProtectSystem=strict`, `ProtectHome=tmpfs`, `NoNewPrivileges`, empty capability set, `@system-service` syscall filter, `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`, `PrivateDevices`. `systemd-analyze security` reports **1.5 OK**. Read-only: the checkout. Read-write: `logs`, `overlay-videos`, `alert-assets`, `sfx`, `upload-incoming` under the checkout.
- `ProtectHome=tmpfs` hides `/home` from the bot. Manager data under `/home` would be invisible to it, which is what we want.
- **No resource limits today**: `MemoryMax`, `MemoryHigh` and `CPUQuota` are infinity, `TasksMax` 12025, `LimitNOFILE` 524288, slice `system.slice`.
- Non-secret `.env` values: panel on port 3000, database on `localhost:3306`, `SFX_FOLDER=./sfx`. The panel listens on loopback only, as the bot's README requires.
- `ubuntu` has **passwordless sudo** (the probe's `sudo -n` worked). So on this host the installer's narrow sudoers rule (restart `bcuk-bot` only) is not the real boundary: the account itself can do anything as root. The sudoers file's content was redacted by the probe (it contains the word `NOPASSWD`), so what that file says is unconfirmed.
- No crontab for `ubuntu`. The backup directory still holds 14 dumps, newest from the morning of the probe, which is consistent with `deploy.sh`'s pre-deploy dump and the default `BACKUP_KEEP` of 14. Whether a nightly backup exists (root crontab or a systemd timer) was **not probed**.

## 4. Filesystem layout, space and permissions (probe section 5)

| Path | Owner and mode | Size now |
|---|---|---|
| `/home/ubuntu/bcuk-bot` (checkout) | `ubuntu:ubuntu` 775 | 350 MB |
| `…/overlay-videos` | `ubuntu:ubuntu` 755 | 77 MB |
| `…/sfx` | `ubuntu:ubuntu` 755 | 27 MB |
| `…/logs` | `ubuntu:ubuntu` 775 | 184 KB |
| `…/alert-assets`, `…/upload-incoming` | `ubuntu:ubuntu` 755 | 4 KB each |
| `/home/ubuntu/bcuk-bot-backups` | `ubuntu:ubuntu` 700 | 1.1 MB, 14 files |

- Single root filesystem `/dev/sda1`, ext4 (`rw,relatime,discard,errors=remount-ro,commit=30`), 96 GB, **34 GB used, 63 GB free (35%)**. `/boot` 1 GB, `/boot/efi` 105 MB. Inodes 2% used. There is **no second volume** for backups or game data.
- The bot's own growth is capped by its settings (1 GB per streamer and 10 GB for the shared library by default, per the bot's README), which together could use a large share of the 63 GB. Game worlds, server installs and backups need their own quotas or a disk-use check in the manager (#55 covers disk monitoring, #56 backups on separate storage and #58 retention).
- The journal is persistent and uses 3.8 GB, which is also on this disk.
- **Suggested layout, not a decision** (for #11 and #12 to confirm): keep the manager and supervisor out of `/home`, under their own service users, for example code under `/opt/`, state and database under `/var/lib/`, and server installs and worlds under `/srv/`, each owned by the supervisor's user and mode 750 or tighter. That keeps them out of the bot's `ProtectHome=tmpfs` view and out of reach of the `ubuntu` account when it is not using sudo. Because `ubuntu` has passwordless sudo, this layout does not stop root access from that account; removing or restricting that sudo is a separate decision (open item 2).

## 5. Resource contention (probe section 8)

- Snapshot at probe time: load 0.16, 1.4 GiB used of 11 GiB (7 GiB is page cache), **no swap**. Largest users: `mysqld` (about 0.7 GiB), the bot (about 140 MB), journald, Caddy.
- **Memory:** a Java Minecraft server commonly wants one to several GiB, so two or three servers plus MySQL and the bot can exhaust 11 GiB. With no swap the kernel OOM killer picks a victim, and it could be the database or the bot. Per-server `MemoryMax`, and `MemoryLow` or `OOMScoreAdjust` protection for MySQL and the bot, are options for #12. Adding swap is a host change for the owner.
- **CPU:** 6 vCPUs, no limits. `CPUWeight` on slices can favour the bot and database over game servers or the reverse. A shared KVM guest may also suffer CPU steal from other tenants; the probe took a single snapshot and cannot show it.
- **Deploys:** `deploy.sh` builds and runs the bot's tests on this host, a burst of CPU and memory while game servers may be running. A bot deploy must also never restart game servers: it restarts only `bcuk-bot` (section 8).

## 6. Networking, firewall, reverse proxy and TLS (probe section 6)

- One network interface with a single public IPv4 and an IPv6 address.
- **Ports already in use, so game servers must avoid them:** 22 (ssh), 80 and 443 (Caddy, TCP, plus UDP 443 for HTTP/3), 3000 (bot panel, loopback), 2019 (Caddy admin API, loopback), 53 (resolver, loopback), 3306 and 33060 (MySQL). Everything else is free on the host, including the usual Minecraft ports.
- **Firewall:** `ufw` is inactive, iptables policies are ACCEPT, and nftables holds only fail2ban's sets (an `sshd` jail and a Caddy scan jail that block 22 and 80/443). So nothing on the host restricts inbound ports. A game port opened by the supervisor is reachable from the internet unless the **provider's network firewall** blocks it, which the probe cannot see. The owner should say whether one exists, and the ADRs should decide where port policy lives.
- **Caddy** 2.11 runs as its own user and is enabled at boot. The Caddyfile has **one site block**, the bot's panel domain, with JSON access logs under `/var/log/caddy/` (50 MiB rolls, 5 kept, 30 days), gzip, and `reverse_proxy 127.0.0.1:3000`. TLS is Caddy's automatic HTTPS. The Caddyfile has **no site for `servers.bcuk.co.uk`**.
- Adding it means a second site block proxying to the manager's loopback port (or Unix socket), plus a DNS record pointing at this host. Whether that DNS record exists was not checked. The Caddy scan jail's name suggests it reads the bot site's access log, so a new site would need its own log path and jail to get the same protection (the jail's config was not read).
- The bot's panel binds loopback only and sits behind Caddy, which matches its `trust proxy: 1`. The manager's web/API should do the same. The supervisor's local interface should be a Unix socket or loopback port with its own authentication, never reachable through Caddy.

## 7. Reboots and power loss (probe section 7)

- `bcuk-bot`, `caddy` and `mysql` are enabled at boot, so they come back by themselves. A game server is not a unit today, so nothing restarts it: the manager needs an explicit policy for what starts again after a reboot.
- The last six boots (2026-09-27 to now) lasted between 2 and 24 days. Each boot began about 8 to 10 seconds after the previous one's last journal entry, which looks like clean reboots rather than power loss. `unattended-upgrades` is running and kernel livepatch is installed. Whether automatic reboots are enabled was **not checked**.
- Root is ext4 with a journal. After an unclean stop it replays the journal to keep the filesystem structure consistent, but that does not guarantee consistent file contents, and `commit=30` is only the journal's commit interval, not a bound on lost data: what survives depends on what applications had flushed to disk and on the storage's own behaviour. MySQL and any game world therefore need their own flush and backup safety, and #12 should not take 30 seconds as a recovery point. How the provider treats host power loss, and whether it offers snapshots, is unknown.
- `last` is not installed, so a reboot/shutdown history beyond the journal's boot list was not available.

## 8. How a manager deployment avoids interfering with the bot

- The bot's `deploy.sh` restarts exactly one unit (`systemctl restart bcuk-bot`), checks `/healthz`, and rolls back inside the bot's checkout. It never touches other units. The manager and supervisor therefore need **different unit names, different directories and their own deploy script**, and the bot's deploy must not be able to reach them.
- On this host the bot's sudoers rule gives no isolation, because the same account has full sudo. Real separation comes from separate users: a bot deploy runs as `ubuntu`, so the supervisor's state, unit files and game data should be owned by other users that `ubuntu` has no business with (although `ubuntu` can still sudo; the owner may want to decide whether that is acceptable).
- A game server must survive a restart of the manager's web/API and of the bot. If the supervisor is a systemd service, children left in its cgroup are stopped when it stops, unless the unit sets `KillMode=process` or `none`, or each server runs in its own transient unit or scope. Surviving is not enough, though: if the supervisor owns the only parent ends of the pipes (or the pty master) wired to a server's console, its exit closes them. The server then sees end-of-file on input or `SIGPIPE`/`EPIPE` on output (a closed pty master hangs up the session), and a replacement supervisor cannot recover those descriptors. So `KillMode` alone does not keep console control. A persistent console endpoint is an additional requirement: for example a FIFO or socket held by systemd or a small separate broker process, or a multiplexer such as `tmux` that keeps the pty. This is a core question for #12.
- Give game servers their own slice (a name for #12 to choose) so `MemoryMax`, `CPUWeight` and `TasksMax` apply to them as a group without touching `bcuk-bot.service` or `mysql`.

## 9. Candidate supervision mechanisms on this host

Facts only; the choice is #12's.

| Mechanism | Process identity and rediscovery | Console access | Limits and isolation | Needs on this host |
|---|---|---|---|---|
| Own Node supervisor daemon spawning children | Verify executable, owner and start time from `/proc` ourselves; keep our own state file | Pipes or a pty we own, fully controllable while we run; the descriptors close when the daemon exits, so a restart loses console control unless a persistent endpoint is added (section 8) | Only what the unit and `setrlimit` give, unless children are placed in cgroups | A unit with `KillMode` that lets children outlive it; its own service user |
| systemd transient units (`systemd-run --unit …`) | The unit name and its cgroup are the identity, immune to PID reuse; `systemctl` and D-Bus list them after a supervisor restart, but a host reboot removes them, so server configuration must be persistent | Needs stdin wiring (socket, FIFO or pty), which is the awkward part | Native `MemoryMax`, `CPUQuota`, sandbox properties per server | System manager: privilege (root, or a polkit rule for the service user). User manager: linger, which is off |
| `tmux` or `screen` sessions | Session name plus PID, weakest identity; rediscovery by `tmux ls` for that user | Easy: attach, `send-keys`; the tmux server holds the pty, so the console survives a supervisor restart (not a host reboot) | None unless combined with a scope; the processes sit in whichever cgroup started tmux | The tools exist; same `KillMode` issue as the first row |

## 10. Risks and open items

| # | Item | For |
|---|---|---|
| 1 | No swap and no resource limits: one memory-hungry game server can trigger the OOM killer against MySQL or the bot | #12 |
| 2 | The bot's account has full passwordless sudo, so a process running as it is not isolated from root. The manager and supervisor need their own service users; removing or restricting that account's sudo is a separate decision for the owner | #11, #12 |
| 3 | No host firewall rules; whether the provider's firewall filters inbound traffic, and any DDoS protection for game ports, is unknown | owner, #12 |
| 4 | Single disk: backups and game data share it with the database. Off-host copy or provider snapshots needed; disk quotas for worlds and installs | owner, #12, Phase 5 |
| 5 | Nightly backups: no `ubuntu` crontab; root crontab and systemd timers were not probed | owner |
| 6 | Automatic reboot setting for `unattended-upgrades`, and the provider's behaviour on power loss, unknown | owner |
| 7 | `servers.bcuk.co.uk`: no Caddy site yet, DNS not checked; Caddy's scan jail and logging need a decision for the new site | #11, Phase 6 |
| 8 | A supervisor under systemd must not take its servers down on restart (`KillMode`, or one unit per server), and console descriptors must outlive it (persistent FIFO, socket, broker or multiplexer) | #12 |
| 9 | Only OpenJDK 25 is installed; whether it suits the chosen Minecraft version is for #14 | #14 |
| 10 | The live bot is at `55ff49d`, newer than the `1b5555b` assessed in `bot-assessment.md`; recheck `deploy.sh`, the unit template and the installer | #9 recheck |
| 11 | The content of the bot's sudoers file was redacted by the probe; confirm it | owner |

## 11. Recheck log

| Date | Checked against | Note |
|---|---|---|
| 2026-11-28 | Live host probe 2026-11-28 11:19:59 UTC; BCUK-Bot-4 `1b5555b` (repo facts); live bot checkout `55ff49d` | First assessment. Provider-level settings and root-only schedules were not covered. |
