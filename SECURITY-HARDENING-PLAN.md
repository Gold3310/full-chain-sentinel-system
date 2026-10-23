# Security Hardening Plan — Full Chain Sentinel System

## Scope and handling

This is a repository-level hardening plan based on the public README, root package scripts, Dockerfile, release workflow, and security policy reviewed on 2026-10-23. Treat these as evidence-led review observations, not proof that every runtime path has been independently tested. Work only in systems and targets covered by written authorization. Do not run intrusive scans against public targets from CI.

## Immediate priorities

### P0 — Fail-closed approval gates
The README states that AI approval mode allows an operation when the gate model fails, times out, or returns an unparsable result. It also says some tools (including file operations, MCP/code mode, Frida, packet capture and APK analysis) bypass the command approval gate.

- Make approval-gate errors, timeouts, and invalid outputs deny by default.
- Apply a central policy check to every tool class, not just shell commands.
- Add unit tests for allow, deny, timeout, malformed output, unavailable model, and tool-bypass attempts.
- Preserve a clearly named operator-approved break-glass path with audit logging, rather than silently failing open.

### P0 — Container and host boundary
The README documents root execution inside the project container, default host networking, and a host-side console/agent process that is not itself sandboxed. It also notes project bind mounts.

- Default new projects to bridge networking; require explicit per-project confirmation for host networking.
- Run as a non-root UID where tool compatibility permits; drop Linux capabilities by default and add only documented capabilities per tool.
- Never mount the Docker socket into an agent-controlled container.
- Restrict bind mounts to the selected project workspace; use read-only mounts where possible.
- Treat Docker as an isolation layer, not a complete sandbox; use a disposable VM for high-risk testing.

### P1 — Tool and plugin authorization
The README documents MCP stdio servers executing on the host and plugin tools that may not pass through the same per-command approval path.

- Default custom MCP and plugins to disabled until individually reviewed and enabled.
- Display requested capabilities, network access, host execution, and privilege requirements before installation.
- Enforce authorization centrally at invocation time and record actor, tool, target, decision, and result.
- Add tests for disabled plugins, revoked permissions, unauthorized targets, and policy changes during a session.

### P1 — CI and supply chain
A minimal PR/push CI workflow has been added on this branch. It uses read-only token permissions, SHA-pinned checkout/setup-node actions, npm ci --ignore-scripts, automated tests, and the offline build. It does not publish artifacts or deploy.

- Keep release publishing separate from untrusted pull-request validation.
- Give release jobs only the permissions they need; review whether the release workflow's top-level actions: write permission is necessary for every job.
- Keep third-party actions pinned to verified full commit SHAs.
- Add Dependabot updates, dependency review, secret scanning/push protection, and CodeQL/Actions workflow analysis in repository settings where available.
- Never expose deployment or API secrets to fork PR jobs; do not use pull_request_target to execute untrusted PR code.

### P1 — Network and credential safety
- Use bridge networking and explicit target allowlists for routine testing; avoid host networking and broad egress by default.
- Keep credentials out of container environment dumps, shell history, logs, prompts, and evidence bundles.
- Redact tokens and personal data from reports; rotate any credential that may have been exposed.
- Separate read-only reconnaissance from active validation and exploitation; require an explicit scope and time window before active tests.

## Validation workflow

1. Review the diff and the new CI workflow.
2. Run npm ci --ignore-scripts, npm test, and npm run build:offline on a clean runner.
3. Add tests before changing approval policy, tool dispatch, container privileges, or network defaults.
4. Validate security changes against a local disposable lab and synthetic targets first.
5. Require code review before merge. Do not auto-merge or publish from this CI workflow.

## Out of scope for this change

- No live target scans or exploit attempts.
- No wallet signing, transaction broadcasting, asset transfers, Safe owner/threshold/module changes, or key recovery attempts.
- No container image publishing, GHCR push, release, deployment, or merge to main.
- No Foundry configuration was added: the repository root currently exposes a TypeScript/JavaScript monorepo and no root-level foundry.toml was found in the inspected file listing. Add a dedicated Solidity/Foundry package only if Solidity contracts are part of the intended project scope.
