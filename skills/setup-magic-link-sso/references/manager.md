# Magic Link SSO Manager

## What Managed Mode Adds

Managed mode is optional. It adds the manager as a control plane for access
administration while keeping the Magic Link SSO server as the runtime auth
service.

Choose it when selected sites should have manager-owned grants and scope
catalogs instead of hand-edited access entries in one operator-maintained TOML
file.

## Ownership Boundaries

Keep file ownership explicit:

- operator-owned:
  - `magic-sso.base.toml`
- manager-owned mutable state:
  - `manager-state.json`
  - `magic-sso.runtime.toml`
  - `magic-sso.runtime.last-good.toml`
  - `manager-audit.ndjson`
  - `manager.lock`

The manager should read the base config, generate runtime config, and never
rewrite the base file in place.

## Minimum Setup Shape

Managed mode needs:

- a base Magic Link SSO server config
- a manager settings TOML referenced by `MAGICSSO_MANAGER_CONFIG_FILE`
- a generated runtime TOML that the server reads through `MAGICSSO_CONFIG_FILE`
- an initial manager state file
- managed site IDs chosen ahead of time

Useful starting points in the main repository:

- [docs/managed-mode.md](https://github.com/magic-link-sso/magic-sso/blob/main/docs/managed-mode.md)
- [docs/manager-architecture.md](https://github.com/magic-link-sso/magic-sso/blob/main/docs/manager-architecture.md)
- [docs/manager-operations.md](https://github.com/magic-link-sso/magic-sso/blob/main/docs/manager-operations.md)
- [manager/README.md](https://github.com/magic-link-sso/magic-sso/blob/main/manager/README.md)
- [manager/manager.example.toml](https://github.com/magic-link-sso/magic-sso/blob/main/manager/manager.example.toml)
- [manager/docker-compose.prod.yml](https://github.com/magic-link-sso/magic-sso/blob/main/manager/docker-compose.prod.yml)
- [manager/manager.prod.example.toml](https://github.com/magic-link-sso/magic-sso/blob/main/manager/manager.prod.example.toml)
- [manager/magic-gate.prod.example.toml](https://github.com/magic-link-sso/magic-sso/blob/main/manager/magic-gate.prod.example.toml)

## Required `manager-admin` Site

Keep a dedicated static `manager-admin` site in the base server config for the
manager itself.

That bootstrap access path should stay operator-managed rather than
manager-managed. When the manager UI is exposed to browsers, prefer Gate in
front of it and require the `manager-admin` site identity there.

For production-like deployments, prefer the published-image manager stack under
`manager/docker-compose.prod.yml`. It keeps the manager service private, exposes
only Gate publicly, and assumes the main Magic Link SSO server already runs
elsewhere.

## Apply Workflow

The safe operational loop is:

1. Validate manager settings and file paths.
2. Review managed site IDs and bootstrap admin access.
3. Run validation and diff before the first write.
4. Apply with confirmation.
5. Confirm the server is using the generated runtime config.

Typical commands in the main repository workflow are:

- `manager validate`
- `manager diff`
- `manager apply --yes`

## Reload Hook

The server reload hook is optional. It is an optimization for restart-free
applies, not a requirement for managed mode.

If used:

- keep it on private networking only
- protect it with a dedicated secret
- treat failures as non-destructive, with the current live config staying in
  place

## First Verification Path

Always validate this sequence:

1. The manager can read base config and state files.
2. Validation succeeds.
3. Diff output matches the intended access change.
4. Apply writes a new runtime config cleanly.
5. The server accepts the generated runtime config after reload or restart.
6. The bootstrap manager admin path still works.
