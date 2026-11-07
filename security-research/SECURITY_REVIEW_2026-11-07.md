# Full Chain Sentinel — Security Review

**Review date:** 2026-11-07  
**Repository:** `Gold3310/full-chain-sentinel-system`  
**Baseline commit:** `961f34e822abebb9d9c58ae940322c89a9f8d9a4`  
**Research branch:** `security-hardening/fail-closed-bridge-default-2026-11-07`  
**Draft PR:** [#2 — Fail closed and default to bridge networking](https://github.com/Gold3310/full-chain-sentinel-system/pull/2)

## Executive summary

The reviewed application is a local penetration-testing agent with host-side file access, containerized command execution, MCP servers, Android/iOS tooling, and optional AI approval. The README explicitly documented fail-open behavior when the AI judge timed out or failed, and project containers defaulted to host networking.

The research branch hardens those defaults. The application is **not yet safe to treat as a general-purpose autonomous agent on a workstation that holds wallet keys or other secrets**. Several privileged execution paths remain outside the approval gate.

## Confirmed design risks and branch changes

### FS-01 — Gatekeeper failure previously allowed guarded operations

**Severity:** High execution-control weakness  
**Affected code:** `packages/web/src/tool-gate.ts`, `packages/web/src/sentinel-session.ts`, `packages/web/src/gatekeeper.ts`.

Before this branch, bash, device-shell and MCP gate call sites treated a `failed` judge result like `allow`. A timeout, endpoint error or unparsable answer therefore degraded AI Approval into execution.

**Branch remediation:** Only an explicit `allow` proceeds. `block`, `failed`, and `aborted` stop the guarded operation. The MCP tool gate uses the same fail-closed rule.

### FS-02 — Plugin-host MCP tools were excluded from AI judging by default

**Severity:** High policy-coverage gap  
**Affected code:** `packages/web/src/gatekeeper-config.ts`, `packages/web/src/tool-gate.ts`.

The default configuration enabled custom MCP judging but left `includePlugins=false`, allowing plugin-host tools to bypass AI Approval.

**Branch remediation:** New/malformed configurations now default `includePlugins=true`. An existing explicit `false` remains an operator opt-out and must be reviewed manually.

### FS-03 — Project containers defaulted to host networking

**Severity:** High isolation risk  
**Affected code:** `packages/web/src/container-network.ts`.

Host networking shares the host network namespace. A compromised or prompt-injected container process could reach services bound to host loopback and other host-reachable networks.

**Branch remediation:** Missing or malformed configuration now selects Docker bridge networking. Host mode remains an explicit opt-in for authorized reverse-callback workflows.

## Remaining high-priority risks — not fixed by this branch

### FS-04 — Host-side file tools can access paths outside the project

**Severity:** Critical local confidentiality/integrity risk  
**Affected behavior:** `read`, `write`, and `edit` execute in the console process on the host. The repository documentation states that absolute paths are unrestricted.

**Impact:** If an agent is prompt-injected by an untrusted repository or document, it may read or overwrite files outside the selected project, including local credentials, SSH material, and environment files.

**Required remediation:** Enforce canonical-path containment against the project root for every host-side file tool, including symlink resolution; separate operator-only host file access from agent tools. Do not run this application on a workstation containing production wallet keys until this is addressed.

### FS-05 — Several built-in tools bypass the approval gate

**Severity:** High execution-control gap  
**Affected behavior:** Frida, packet capture, APK analysis, file-panel operations, and some host-side tools have independent execution paths. The file panel runs with container root and follows symlinks under `/work`.

**Required remediation:** Add explicit per-tool authorization and audit hooks, default dangerous operations to operator approval, and resolve symlinks before enforcing file-panel path boundaries.

### FS-06 — AI Approval is not a security boundary

**Severity:** High residual risk  
The Gatekeeper prompt is designed for authorized penetration testing and intentionally allows sensitive-file reads, webshell uploads, password attacks, and scanning. It judges individual tool calls and cannot reliably determine whether a target is authorized or whether a script hides destructive behavior.

**Required remediation:** Use an OS-level sandbox and explicit target allowlists. Do not rely on model judgment as the sole control for data access, network scope, or destructive operations.

### FS-07 — Host-side MCP stdio servers

**Severity:** High host execution risk  
Custom stdio MCP servers run as child processes of the console on the host, outside the Kali container. Their command and arguments can execute host programs.

**Required remediation:** Allow only pinned, reviewed server binaries and fixed argument lists; isolate them in separate containers or OS users; do not pass wallet credentials to these processes.

### FS-08 — Container hardening is incomplete

**Severity:** Medium/high  
The container runs as root, lacks resource limits, cap-drop, seccomp and read-only hardening. Bridge mode reduces network exposure but does not provide a full sandbox.

**Required remediation:** Drop unnecessary capabilities, use a non-root UID, add seccomp/AppArmor, read-only root filesystem, CPU/memory/PID limits, and per-project isolation.

### FS-09 — Console network exposure

**Severity:** High if internet-exposed  
The README documents a default listen address of `0.0.0.0`. A random entry token is not a substitute for network isolation or strong authentication.

**Required remediation:** Bind to `127.0.0.1` by default. For remote use, require TLS reverse proxy, strong authentication, firewall allowlisting, and secure session cookies.

## Validation

A GitHub Actions workflow was added to run:
- `npm ci`
- `npm --prefix packages/web test`
- `npm --prefix packages/web run check:webui`
- `npm --prefix packages/web run build`

CI status is pending at the time of this report. No test pass is claimed until the workflow completes successfully.

## Operating recommendation

Do not connect production wallet keys, seed phrases, signing services, or unrestricted RPC credentials to this agent. Keep Auto mode disabled. Use Review mode, bridge networking, only trusted project directories, and a disposable VM until FS-04 through FS-09 are resolved.
