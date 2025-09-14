SOW: LXC Application Updater for Proxmox VE 9
=============================================

_Yes, updating packages is adorable. Sadly your apps still rot. Let’s fix that properly._

1) Project Goal
---------------

Design and deliver a **scheduled, idempotent updater** that runs on a Proxmox VE 9 host and updates **applications inside LXC containers**, not just the container OS. It will support multiple app ecosystems, enforce maintenance windows, snapshot-before-update, run health checks after, and notify you when something inevitably breaks.

2) In-Scope
-----------

*   **Container targeting**
    
    *   Select LXC containers by tag, name pattern, or allow/deny list.
        
    *   Handle both **privileged** and **unprivileged** LXCs.
        
*   **Update engines**
    
    *   Linux package managers: `apt`, `apt-get`, `dnf`, `apk`, `pacman` (best-effort).
        
    *   Language/package ecosystems: `pip/pipx`, `npm/yarn/pnpm`, `composer`, `gem`, `go install` (optional per container).
        
    *   App-specific CLIs: e.g. `helm` for charts in an LXC, `snap`, `flatpak` where applicable.
        
    *   **Docker-in-LXC**: optional support to `pull` and `recreate` containers if you insist on nesting.
        
*   **Pre-flight**
    
    *   Verify node load, free disk, container state, network reachability, lock status.
        
    *   Create **Proxmox snapshot** or `vzdump` backup per container (configurable).
        
    *   Respect **maintenance windows** and concurrency limits.
        
*   **Execution**
    
    *   Enter container (e.g. `pct exec`) and run update steps per defined plan.
        
    *   **Dry-run mode** that lists planned operations without changing anything.
        
*   **Post-update**
    
    *   App/service **health checks** (HTTP probe/TCP/command exit status).
        
    *   **Rollback** via snapshot restore on hard failure (configurable thresholds).
        
    *   **Restart policies**: systemd services, app processes, docker compose stacks.
        
*   **Scheduling & Orchestration**
    
    *   `systemd` timer or cron on the PVE host.
        
    *   Staggered updates with jitter to avoid stampedes.
        
*   **Observability**
    
    *   Structured logs per container + host summary.
        
    *   Report: email, webhook (Slack/Discord/Teams/Telegram), and exit codes.
        
    *   Minimal dashboard-ready JSON artifact (dump to disk) for later parsing.
        
*   **Policy & Safety**
    
    *   Per-container **policy file**: which engines to run, exclusions, health check URL, timeouts.
        
    *   Global **kill-switch** flag file.
        
    *   Rate limiting and **max-runtime** cutoffs.
        

3) Out-of-Scope
---------------

*   Writing or fixing your app’s migrations. That’s your circus.
    
*   Proxmox cluster upgrades, kernel updates, or ZFS wizardry.
    
*   Kubernetes. Different beast, different leash.
    
*   Magic autocompliance with every distro under the sun. We’ll be sane, not mythical.
    

4) Assumptions
--------------

*   You have Proxmox VE 9 with `pct` access and permission to snapshot/backup LXCs.
    
*   LXCs have their respective package managers installed and can reach repositories.
    
*   You can tolerate a small maintenance window and temporary service restarts.
    
*   Snapshots/`vzdump` storage is available and not wheezing at 99% usage.
    

5) Deliverables
---------------

1.  **Design spec**: architecture diagram, data flows, update policy schema, failure matrix.
    
2.  **Configuration templates**:
    
    *   Global config: schedule, concurrency, notification sinks, backup method.
        
    *   Per-container policy files: runners, commands, health checks, exclusions.
        
3.  **Execution plan**: step-by-step runbook for:
    
    *   Dry run
        
    *   Standard run
        
    *   Partial fleet selection
        
    *   Rollback
        
4.  **Logging & reporting format**: JSON lines per container, consolidated report, examples.
    
5.  **Test plan**: unit-ish tests for runners, sandbox validation steps, failure drills.
    
6.  **Operational docs**: install, upgrade, rollback, adding new ecosystems, common pitfalls.
    

_(You said “don’t provide code.” So you get everything except the fun parts.)_

6) High-Level Architecture
--------------------------

*   **Host Scheduler**: `systemd` timer or cron triggers the updater.
    
*   **Target Resolver**: builds the container list from policy (allow/deny/tag).
    
*   **Pre-flight Guard**: checks host/container readiness; triggers snapshot/backup.
    
*   **Runner Pipeline** (per container):
    
    1.  OS package runner (optional)
        
    2.  App ecosystem runners (pip/npm/etc.) per policy
        
    3.  App-specific CLIs (as configured)
        
*   **Health Verifier**: runs HTTP/TCP/cmd checks with retries and backoff.
    
*   **Rollback Manager**: decides restore vs warn, based on thresholds.
    
*   **Reporter**: aggregates logs; sends notifications; writes JSON artifacts.
    

7) Configuration Model
----------------------

*   `global.yml`
    
    *   schedules, maintenance window, max\_concurrency, jitter\_ms
        
    *   snapshot: enabled, method (`snapshot|vzdump`), retention, storage
        
    *   notifications: email/webhooks, throttling
        
    *   defaults: timeouts, retries, health-check method
        
*   `containers/CTID.yml`
    
    *   enabled: true
        
    *   selectors: tags/names
        
    *   runners:
        
        *   os: apt|dnf|apk|none
            
        *   pip: list of venv paths or system
            
        *   npm: working dirs or `package.json` paths
            
        *   custom: ordered shell commands
            
    *   health\_checks: list of probes (http/tcp/cmd) with thresholds
        
    *   exclusions: packages, paths, services
        
    *   restart: services/compose stacks to restart
        

8) Scheduling & Windows
-----------------------

*   Default: **weekly** production-style window, **night hours** local time.
    
*   Optional **daily** quick pass for low-risk updates.
    
*   Stagger with random delay to avoid disk and network thrash.
    
*   Hard stop if outside window unless `--force` is set.
    

9) Backup & Rollback Strategy
-----------------------------

*   Pre-update **snapshot** or **vzdump** per container.
    
*   Keep N latest per policy; prune old ones automatically.
    
*   Rollback triggers:
    
    *   Health check failures beyond retry budget
        
    *   Non-zero critical runner exit codes
        
    *   Operator-issued `--rollback` using last good snapshot
        

10) Health Checks
-----------------

*   HTTP: 2xx within latency SLO
    
*   TCP: port open within timeout
    
*   Command: zero exit code; optional string match
    
*   Multi-step: warm-up delay then probe with exponential backoff
    
*   Record baseline success rate to cut noise
    

11) Notifications
-----------------

*   On success: compressed summary
    
*   On degraded: include failing CTIDs and last 20 lines of logs
    
*   On failure/rollback: loud message with restore details and next steps
    
*   Optional: create a **change report** artifact for audit
    

12) Security
------------

*   Run with least privilege on the host.
    
*   No secrets in logs. Secrets consumed via env files or Proxmox host store.
    
*   Validate policy files strictly. Refuse to run unknown commands by default.
    
*   Optional `allowlist` of binaries runnable inside containers.
    

13) Performance & Concurrency
-----------------------------

*   Tune `max_concurrency` by CPU, IO, repo bandwidth.
    
*   Per-runner **timeout** and **retry** caps.
    
*   Backpressure if host load/disk usage crosses thresholds.
    

14) Extensibility
-----------------

*   Pluggable runners for new ecosystems.
    
*   Per-container hooks: `pre_update` and `post_update`.
    
*   Simple schema for app-specific update sequences.
    

15) Risks & Mitigations
-----------------------

*   **Broken upstream repos** → dry-run + staged rollouts + snapshot restore.
    
*   **Schema drift per distro** → per-container policy with explicit runners.
    
*   **Long-running migrations** → timeouts and independent service restarts.
    
*   **Disk exhaustion** → pre-flight checks + snapshot retention policy.
    
*   **Human creativity** → kill-switch file halts everything.
    

16) Test Plan
-------------

*   **Sandbox**: one privileged and one unprivileged LXC, each with different ecosystems.
    
*   **Chaos cases**: simulate repo failures, network drops, disk-full, bad health check.
    
*   **Rollback drill**: prove snapshot restore works and services return healthy.
    
*   **Load test**: N containers with max concurrency and jitter.
    

17) Acceptance Criteria
-----------------------

*   Selective updates by policy work on at least 3 distinct ecosystems.
    
*   Snapshots created and pruned per retention rules.
    
*   Health checks gate success; failures trigger rollback per policy.
    
*   Logs are structured; summary and per-CTID artifacts generated.
    
*   Notifications fire with correct severity and payloads.
    
*   Dry-run mode prints the exact plan without changes.
    
*   Execution is idempotent and safe to re-run.
    

18) Timeline & Phases
---------------------

1.  **Design & Policy Schema**: finalize config model, failure matrix.
    
2.  **MVP Pipeline**: target resolution, pre-flight, one ecosystem runner, health check, logging.
    
3.  **Rollback & Backups**: snapshot/vzdump integration, restore path, retention.
    
4.  **Multi-Ecosystem Support**: pip/npm/compose/custom runners.
    
5.  **Scheduling & Notifications**: timers/cron, webhooks/email, concurrency tuning.
    
6.  **Hardening & Tests**: chaos cases, sandbox sign-off, docs.
    

_(You’ll do this in VS Code/Codex. I’ll pretend not to judge your extensions folder.)_

19) Handover Artifacts
----------------------

*   Final architecture doc and diagrams
    
*   Config templates with commented examples
    
*   Runbooks: install, operate, rollback, extend
    
*   Test evidence: logs, artifacts, and a smug checklist